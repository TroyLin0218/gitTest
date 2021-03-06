### MySQL日志

#### redo log

假如MySQL的每一次更新都操作都要写入磁盘，然后磁盘需要找到那条记录，然后修改，整个IO过程、查找成本很高，效率很低，因此就有了WAL技术，write-ahead logging，关键点就是先写日志再写磁盘，并且写磁盘的时候是在cpu不忙的时候。

简单的说，当一条记录需要更新的时候，InnoDB就会先把记录写到redo log日志中，冰更新内存，这时候就返回给客户端更新好了，同时，在适当的时机，将操作更新到磁盘中。

其中，InnoDB的redo log大小是有限制的，可以配置一组4哥文件，每个文件1GB大小，每次从头写，写到末尾就回到开头循环写。

`cash-safe`：由于有redo log，MySQL可以在数据库发生异常重启，并且记录不会丢失

#### binlog

Server层的log日志叫做binlog（归档日志）

1. redo log是物理日志，比如某个数据页上做了什么修改；binlog是逻辑日志，记录的是原始逻辑，比如“给id是2的这一行c字段加1”

2. redo log是循环写的，空间固定，binlog是追加写，不会覆盖以前的日志。

InnoDB引擎执行update语句：

```sql
update T set c=c+1 where ID=2;
```

1. 执行器首先找到id=2这一行，id是主键，引擎用树搜索到这一行，如果id=2这一行本来就在内存中，直接返回给执行器，否则需要读入内存然后再返回。
2. 执行器拿到引擎给的含数据，把字段值修改，得到一行新数据，在调用接口写入系数据
3. 引擎将这行数据更新到内存中，同时记录到redo log中，此时redo log处于prepare状态。然后告诉执行器，执行完了，随时可以提交事务。
4. 执行器生成操作的binlog，并把binlog写入磁盘，
5. 执行器调用引擎的食物接口，引擎把redo log的状态改为提交状态，更新完成。

#### 两阶段提交：

先让redo log是prepare状态，然后写binlog，之后再让redo log是提交状态

#### 如果不是两阶段提交会发生什么？

由于redo log和binlog是两个独立的逻辑，如果不用两阶段提交，要么就是先写完redo log再写binlog，或者采用反过来的顺序。再假设执行update语句过程中在写完第一个日志后，第二个日志还没有写完期间发生了crash。

1. **先写redo log后写binlog**。假设在redo log写完，binlog还没有写完的时候，MySQL进程异常重启。由于我们前面说过的，redo log写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行c的值是1。
   但是由于binlog没写完就crash了，这时候binlog里面就没有记录这个语句。因此，之后备份日志的时候，存起来的binlog里面就没有这条语句。
   然后你会发现，如果需要用这个binlog来恢复临时库的话，由于这个语句的binlog丢失，这个临时库就会少了这一次更新，恢复出来的这一行c的值就是0，与原库的值不同。
2. **先写binlog后写redo log**。如果在binlog写完之后crash，由于redo log还没写，崩溃恢复以后这个事务无效，所以这一行c的值是0。但是binlog里面已经记录了“把c从0改成1”这个日志。所以，在之后用binlog来恢复的时候就多了一个事务出来，恢复出来的这一行c的值就是1，与原库的值不同。

### 总结：

1. redo log的概念是什么? 为什么会存在？
2. 什么是WAL(write-ahead log)机制, 好处是什么.
3. redo log 为什么可以保证crash safe机制.
4. binlog的概念是什么, 起到什么作用, 可以做crash safe吗?
5. binlog和redolog的不同点有哪些?
6. 物理一致性和逻辑一直性各应该怎么理解?
7. 执行器和innoDB在执行update语句时候的流程是什么样的?
8. 如果数据库误操作, 如何执行数据恢复
9. 什么是两阶段提交, 为什么需要两阶段提交, 两阶段提交怎么保证数据库中两份日志间的逻辑一致性(什么叫逻辑一致性)?
10. 如果不是两阶段提交, 先写redo log和先写bin log两种情况各会遇到什么问题?