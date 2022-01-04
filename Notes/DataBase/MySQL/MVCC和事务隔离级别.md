## MVCC和事务隔离级别

### 事务隔离级别

#### 四种事务隔离级别

- **读未提交（READ UNCOMMITTED）**：一个事务还没有提交时，它的变更就能被别的事务看到。
- **读已提交（READ COMMITTED）**：一个事务提交之后，它的变更才能被其他事务看到。
- 可重复读（REPEATABLE READ）：一个事务执行中看到的数据总是在这个事务启动时看到的数据是一致的。当然在可重复读的个里脊别瞎，未提交变更对其他事务也是不可见的。
- **串行化（SERIALIZABLE）**：对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等待前一个事务执行完成，才能继续执行。

#### 隔离级别解决的问题

- **脏读（dirty read）**：如果一个事务读到了另一个未提交事务修改过的数据。

![脏读](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/脏读.25wdatpzin9c.jpg)

- **不可重复读（non-repeatable read）**：如果一个事务只能读到另一个已提交事务修改过的数据，并且其他事务每次对该数据进行一次修改并提交后，该事物都能查询到最新值。

  ![不可重复读](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/不可重复读.1z0twx5r2zj4.jpg)

- **幻读（phantom read）**：如果一个事务先根据某些条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来。

  ![幻读](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/幻读.6nx6hguh5ww0.jpg)

#### 如何设置事务的隔离级别

```
SET [GLOBAL|SESSION] TRANSACTION ISOLATION LEVEL level;//等级就是上面的几种
```

#### 如何启动事务

启动事务一般就是：

1. 显式启动事务语句， **begin** 或 **start transaction**，配套的提交语句是**commit**，回滚语句是**rollback**。
2. **set autocommit=0**，这个命令会将这个线程的**自动提交**关掉，意味着如果你只执行一个select语句，这个事务就启动了，而且并不会自动提交。这个事务持续存在直到你主动执行**commit** 或 rollback 语句，或者断开连接。
3. **长事务**是大家需要注意的，因为一旦**set autocommit=0**自动开启事务，所有的查询也都会在事务里面了，有慢SQL那数据库也容易被拖垮的。



### 视图

**视图**，这是事务隔离实现的根本，数据库里面会创建一个视图，访问的时候以视图的逻辑结果为准。

在MySQL里，有两个“视图”的概念：

- 一个**view**，它是一个用于查询语句定义的虚拟表，在调用的时候执行查询语句并且生成结果。创建视图的语法时create view...，而它的查询方法与表一样。
- 另一个时InnoDB在实现**MVCC**时用到的一致性读视图。即**consistent read view**，用于支持**RC（Read Committed，读提交）**和**RR（Repeatable Read，可重复读）**隔离级别的实现。

在**可重复读**的隔离级别下，这个视图是在事务启动时创建的，整个事务存在期间都使用这个视图。



### 版本链

在**可重复读**的隔离级别下，事务在启动时就“拍了快照”，注意，这个快照时基于整库的。

InnoDB中每个事务都有一个**唯一的事务ID**，叫做**transaction id**，它是在事务开始的时候向InnoDB的事务系统申请的，是按照申请顺序严格递增的。

每行数据也都是有多个版本的，每次事务更新数据的时候，都会生成一个新的数据版本，并且把transaction id赋值给这个数据版本的事务ID，记为**row_trx_id**。同时，旧的数据版本要保留，并且在新的数据版本中，能够有信息直接拿到它。

也就是说，表中的一行记录，其实可能**有多个版本（row）**，每个版本有自己的**row_trx_id**。

这是一个隐藏列，还有另外一个**roll_pointer**：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到undo日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。

两者都在InnoDB的聚簇索引中：

![幻读](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/幻读.6nx6hguh5ww0.jpg)

**undo log**的回滚机制也是依靠这个版本链，每次对记录进行改动，都会记录一条undo日志，每条undo日志也都有一个**roll_pointer**属性（INSERT操作对应的undo日志没有该属性，因为该记录并没有更早的版本），可以将这些undo日志都连起来，串成一个链表，所以现在的情况就像下图一样：

![幻读](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/幻读.6nx6hguh5ww0.jpg)

### 事务隔离级别与MVCC的关系

在**读提交**的隔离级别下，这个视图是在每个SQL语句开始执行的时候创建的，在这个隔离级别下，事务每次查询开始时都会生成一个独立ReadView。

![幻读](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/幻读.6nx6hguh5ww0.jpg)

**可重复读**，在第一次读取数据时生成一个**ReadView**，对于使用**REPEATABLE READ**隔离级别的事务来说，只会在第一次执行查询语句时生成一个**ReadView**，之后的查询就不会重复生成了，所以一个事务的查询结果每次都是一样的。

![可重复读](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/可重复读.7ateapwf8rc.jpg)

这里需要注意的是，**读未提交**隔离级别下直接返回记录上的最新值，没有视图概念，也就是图中丙丙那一栏，脏读，幻读，不可重复读都有可能发生。

而**串行化**隔离级别下直接用**加锁**的方式来避免并行访问。

MVCC里面跟事务隔离级别相关的，只有**可重复读**和**读已提交**这两种。



### 查看事务隔离级别

```
show variables 
```



### 总结

所谓的**MVCC（Multi-Version Concurrency Control，多版本并发控制）**指的就是在使用**读已提交（READ COMMITTD）、可重复读（REPEATABLE READ）**这两种隔离级别的事务在执行普通**SELECT**操作时访问记录的**版本链**的过程。这样子可以使不同事务的读-写、写-读操作并发执行，从而提升系统性能。

这两个隔离级别的一个很大不同就是：**生成ReadView的时机不同**，READ COMMITTD在每一次进行普通**SELECT**操作前都会生成一个**ReadView**，而**REPEATABLE READ**只在第一次进行普通**SELECT**操作前生成一个ReadView，数据的可重复读其实就是**ReadView**的重复使用。

**MVCC** 的实现依赖有**隐藏字段+Read View+Undo log**。隐藏字段中比较重要的是修改**（insert | update）**的事务ID与指向当前记录行的undo log信息；Read View主要保存了**对本事务不可见的其他活跃事务**；**Undo log中存储的是老版本数据**，当一个事务需要读取记录行时，如果当前记录行不可见，可以顺着undo log链找到满足其可见性条件的记录行版本。