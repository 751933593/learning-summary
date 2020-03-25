## MySQL>5.0 引擎innodb

### 1.innodb 的行级锁：共享锁和排他锁

- 共享锁（S Lock） 允许事务读一行数据
- 排他锁（X Lock） 允许事务删除或更新一行数据 

### 2.innodb的表级锁：意向锁

- 意向共享锁（IS lock）事务想要获取表中某几行的共享锁
- 意向排他锁（IX lock）事务想要获取表中某几行的排他锁

例子：如果没有意向锁，当已经有人使用行锁对表中的某一行进行修改时，如果另外一个要求对全表进行修改，那么就需要对所有的行进行扫描，查看是否该行有锁，这样效率是十分低的。但是如果引入意向锁，意向排他锁（IX）会对全表上锁，然后再使用排他锁（X）对行上锁，当进行全表修改时，只有该表的意向排他锁（IX）消失后才能全表修改。

**因此，意向锁不会阻塞除全表操作之外的任何请求。**

| 锁兼容 | X    | IX       | S        | IS       |
| ------ | ---- | -------- | -------- | -------- |
| X      | 冲突 | 冲突     | 冲突     | 冲突     |
| IX     | 冲突 | **兼容** | 冲突     | **兼容** |
| S      | 冲突 | 冲突     | **兼容** | **兼容** |
| IS     | 冲突 | **兼容** | **兼容** | **兼容** |



### 3.线程上的锁：latch 闩锁

一种轻量级的锁，因为其要求锁定的时间非常短，若时间持续很长，则性能会变差。在innodb中latch分为mutex（互斥锁）和rwlock（读写锁），其目的用来保证并发线程操作临界资源的正确性，没有死锁检测机制。

lock的锁事务，latch是锁线程，其关系如下图：

|          | lock                                                   | latch                                                        |
| -------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| 对象     | 事务                                                   | 线程                                                         |
| 保护     | 数据库内容                                             | 内存数据结构                                                 |
| 持续时间 | 整个事务过程                                           | 临界资源                                                     |
| 模式     | 行锁、表锁、意向锁                                     | 读写锁、互斥锁                                               |
| 死锁     | 通过waits-for graph、time out 等机制进行死锁检测与处理 | 没有死锁检测机制，通过应用程序加锁的顺序保证无死锁的情况发生 |
| 存在于   | lock manager的哈希表中                                 | 每个数据结构的对象中                                         |



### 4.一致性非锁定读

​		innodb通过多版本控制MVCC（multi-version concurrency control）来读取当前时间数据库的行数据，而不需要该行的X锁释放（innodb默认读取方式）

- 分为快照读和按版本读（同一事务中，select是按照快照读，当insert，update，delete是读快照后面的版本）

- 该技术不会有额外的开销，因为读取的快照数据其实是行数据之前的版本数据，该实现是通过undo （undo log 事务回滚  自行了解）段来完成

- 不同的事务隔离级别下，读取的方式不同（并不是在每个事务隔离级别下都是采用非锁定的一致性读，自行思考，tip：脏读，不可重复读，幻读）

### 5.一致性锁定读

- select ... for update 对读取的行加一个X锁，其他事务不能对已上锁的行加任何锁
- select ... lock in share mode 对读取的行记录加一个S锁，其他事务可以向这些行加S锁，但如果加X锁，则会被阻塞

  ### 6.行锁的三种算法

- record lock ： 单个记录上的锁

- gap lock ： 间隙锁，锁定一个范围，但不包括记录本身

- next-key lock ： gap lock+recordlock，锁定一个范围，并且锁定记录本身（可解决phantom problem）

  当查询的索引含有唯一属性时，innodb会将next-key lock降级到record key，锁住索引本身，而不是范围

### 7.锁问题（事务的并发问题）

- 脏读

  即事务可以读到其他事务未提交的数据

- 重复读

  同一事务中多次读取同一数据集合，前后两次读取到的数据集合不一致（其他事务提交后造成了数据不一致）

- 幻读

  同一事务中多次读取同一数据集合，前后两次读取到的数据数量不一致（其他事务提交后造成了数据不一致）

### 8.事务的隔离级别

- read uncommitted 不能防脏读
- read committed 可防脏读，不能防重复读  （oracle默认）
- repeatable read 可防重复读，不能防幻读  （mysql默认）
- serializable 可防幻读

set session transaction isolation level 设置数据库事务的隔离级别

### 9.死锁（dead lock）的例子

[关于mysql死锁(Deadlock)的两个详细经典案例]:(https://blog.csdn.net/llf_1241352445/article/details/83472715)

- record lock - record lock

  先在A事务中执行：select * from t where id=1 for update ，不提交

  再在B事务中执行：select * from t where id=2 for update，不提交
  
  然后在A事务中执行：select * from t where id=2 for update，此时A事务会等待B事务提交后才能执行
  
  最后在B事务中执行：select * from t where id=1 for update ，此时B事务会等待A事务提交，死锁
  
- record lock - next-key lock 

  已知表：id主键 为 record lock ，age 为next-key lock，tel unique 为record lock

  | id   | name | age  | tel     |
  | ---- | ---- | ---- | ------- |
  | 1    | 张三 | 10   | 1234561 |
  | 2    | 李四 | 20   | 1234562 |
  | 3    | 王五 | 30   | 1234563 |
  | 6    | 赵柳 | 40   | 1234564 |

  首先事务A执行 delete FROM user_info WHERE age = 25，A事务获取了age范围为[20,30]的Next-key Lock

  再在事务B执行 inert into user_info values('admin',50,'110')，B事务获取了 tel=110的record lock

  然后事务A执行 inert into user_info values('admin1',50,'110')，由于tel=110已经被B锁，所以A等待B

  最后事务B执行 inert into user_info values('admin2',21,'111')，由于age=[20,30]已经被A锁，所以B等待A

  

### 10.事务的分类

- 扁平事务
- 带有保存点的事务
- 链事务
- 嵌套事务
- 分布式事务