## 02 - Mysql 更新语句执行过程

#### ddl 和更新语句

```sql
create table T(ID int primary key, c int);
update T set c=c+1 where ID=2;
```

![](../imgs/1.png)

执行前使用`连接器`连接数据库，表更新会导致与该表有关的`查询缓存`都失效，因此上面的 update sql 会把表 T 上的所有缓存结果都清空，因此查询缓存效率比较低，一般不建议使用查询缓存。

`分析器`通过词法和语法解析知道这是更新语句，`优化器`决定要使用 ID 这个索引，`执行器`负责具体执行，找到该行后更新。

更新流程涉及到 两个日志模块:  `redo log (重做日志)` 和 `binlog （归档日志）`

###  redo log

`《孔乙己》里面，酒店掌柜有一个粉板，专门用来记录客人的赊账记录。`

`如果赊账的人不多，那么他可以把顾客名和账目写在板上。但如果赊账的人多了，粉板总会有记不下的时候，这个时候掌柜一定还有一个专门记录赊账的账本。`

`如果有人要赊账或者还账的话，掌柜一般有两种做法：`

- `一种做法是直接把账本翻出来，把这次赊的账加上去或者扣除掉；`

- `另一种做法是先在粉板上记下这次的账，等打烊以后再把账本翻出来核算。`

`在生意红火柜台很忙时，掌柜一定会选择后者，因为前者操作实在是太麻烦了。首先，你得找到这个人的赊账总额那条记录。你想想，密密麻麻几十页，掌柜要找到那个名字，可能还得带上老花镜慢慢找，找到之后再拿出算盘计算，最后再将结果写回到账本上。`

在 MYSQL 中，如果每次 更新操作都需要写入磁盘，磁盘找到对应记录，再更新，整个过程 IO成本，查找成本很高。

MySQL WAL (Write-Ahead Logging) 技术,先写日志，再写磁盘，也就是先写粉板，等不忙时候再写账本，即当有记录需要更新，InnoDB 引擎先把记录写到 `redo log` , 并更新内存，此时更新就算完成。InnoDB 引擎在适当时候将该操作记录更新到磁盘，这个更新往往在系统空闲时候做。

`如果今天赊账的不多，掌柜可以等打烊后再整理。但如果某天赊账的特别多，粉板写满了，又怎么办呢？这个时候掌柜只好放下手中的活儿，把粉板中的一部分赊账记录更新到账本中，然后把这些记录从粉板上擦掉，为记新账腾出空间。`

InnoDB 的  `redo log` 为固定大小，可配置为一组 4个文件，每个文件大小 1GB，那么这个“粉板” 一共可记录 4GB 的操作，从头开始写，写到末尾又到开头循环写

![](../imgs/redolog.png)

`write pos` 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。`checkpoint` 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。`write pos` 和 `checkpoint` 之间的是“粉板”上还空着的部分，可以用来记录新的操作。如果 `write pos` 追上 `checkpoint`，表示“粉板”满了，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 `checkpoint` 推进一下。有了 `redo log`，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 `crash-safe`。

crash-safe : 只要赊账记录记在粉板上或者写在账本上，之后即使掌柜忘记，比如突然停业几天，恢复生意后依然可以通过账本和粉板上的数据明确赊账项目。

binlog

MySQL 整体分两块

- Server 层: 主要做 MySQL 功能层面事情
- 引擎层:  负责存储相关具体事宜
- redo log 是 InnoDB 引擎特有日志，Server 层自己的日志为 binlog (归档日志)

最开始 MySQL 里面并没有 InnoDB 引擎，MySQL 自带的引擎是 MyISAM，但是 MyISAM 没有 `crash-safe` 能力，binlog 日志只能用于归档，InnoDB 是另一个公司以插件形式 引入 MySQL ，既然只依靠 binlog 是没有 crash-safe 能力的，所以 InnoDB 使用另外一套日志系统——也就是 redo log 来实现 crash-safe 能力。

redo log 和 binlog 都保存在硬盘上

[redo log](https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html)

[bin log](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)

#### 物理日志 redo log 和逻辑日志  binlog 区别

1. redo log 是 InnoDB 引擎特有，binlog 是 MySQL 的 Server 层实现，所有引擎都可以用
2. redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

###  执行器和 InnoDB 在执行 update 语句时候的流程

1. 执行器先找引擎取 id =2 这一行，id 是主键，引擎直接用树搜索找到这一行，如果 id = 2 这一行所在的数据页本来就在内存中，就直接返回给执行器，否则，需要先从磁盘读入内存，然后再返回。

2. 执行器拿到引擎给的数据，把这个值 +1，比如原来是 N，现在就是 N+1,得到新的一行数据，再调用引擎接口写入这行新数据。

3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于  `prepare` 状态，然后告知执行器执行完成了，随时可以提交事务。

4. 执行器生成这个操作的 `binlog`，并把 binlog写入磁盘。

5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 `redo log` 改成 提交 ( `commit` ) 状态，更新完成。

   update 语句执行流程图，图中浅色框表示是在 InnoDB 内部执行的，深色框表示是在执行器中执行的。

   ![](../imgs/update-process.png)

   redo log 写入拆成两步: `prepare` 和 commit ,这就是 "两阶段提交"

   ### 两阶段提交

   保持两份日志之间逻辑一致。

   如何 让数据库恢复到半个月内任意一秒的状态？

   - binlog 采用 `追加写` 形式记录所有的逻辑操作，备份系统保存最近半个月的所有 binlog，同时系统会定期做整个数据库的备份，`定期` 取决于重要性，可以是一天一备，也可以是一周一备。

   - 当需要回复到指定的某一秒时候，比如某天下午两点发现中午12点有一次误删表，需要找回，可以这么操作:

     - 先找到最近一次全量备份，从这个备份恢复数据到临时库
     - 从备份的时间点开始，将备份的 binlog 依次取出，重放到中午误删表之前的那个时刻。
     - 此时临时库和误删之前的线上库一样了，可以把表数据从临时库取出，按需恢复到线上库

### 为何需要两阶段提交？

反证法

假设当前 id = 2的行，字段 c  的值是0，在假设执行 update 语句过程中在写完第一个日志后，第二个日志还没有写完期间发生了 crash ，会出现什么情况？

1. `先写 redo log 后写 binlog`。假设在 redo log 写完，binlog 还没有写完的时候，MySQL 进程异常重启。由于我们前面说过的，redo log 写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行 c 的值是 1。但是由于 binlog 没写完就 crash 了，这时候 binlog 里面就没有记录这个语句。因此，之后备份日志的时候，存起来的 binlog 里面就没有这条语句。然后你会发现，如果需要用这个 binlog 来恢复临时库的话，由于这个语句的 binlog 丢失，这个临时库就会少了这一次更新，恢复出来的这一行 c 的值就是 0，与原库的值不同。
2. `先写 binlog 后写 redo log`。如果在 binlog 写完之后 crash，由于 redo log 还没写，崩溃恢复以后这个事务无效，所以这一行 c 的值是 0。但是 binlog 里面已经记录了“把 c 从 0 改成 1”这个日志。所以，在之后用 binlog 来恢复的时候就多了一个事务出来，恢复出来的这一行 c 的值就是 1，与原库的值不同。

不使用两阶段提交，那么数据库的状态可能和用它的日志恢复出来的库的状态不一致。

除了误操作后需要这个过程来恢复数据，需要扩容时候（再多搭建一些备库来增加系统的读能力的时候），常见做法也是用全量备份加上应用 binlog 实现，这个`不一致` 会导致线上出现主从数据库不一致的情况。

redo log 用于保证 crash-safe 能力。innodb_flush_log_at_trx_commit 这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘。这个参数我建议你设置成 1，这样可以保证 MySQL 异常重启之后数据不丢失。

sync_binlog 这个参数设置成 1 的时候，表示每次事务的 binlog 都持久化到磁盘。这个参数设置成 1，这样可以保证 MySQL 异常重启之后 binlog 不丢失。两阶段提交是跨系统维持数据逻辑一致性时常用的一个方案。

### 一天一备和一周一备对比

一周一备`最长恢复时间更短`

- 一天一备 :  最坏情况下需要应用一天的 binlog，比如，你每天0点做一次全量备份，而要恢复出一个到昨天晚上23点的备份。
- 一周一备: 需要应用一周的 binlog

当然这个是有成本的，因为更频繁全量备份需要消耗更多存储空间，所以这个 RTO 是成本换来的，就需要你根据业务重要性来评估了