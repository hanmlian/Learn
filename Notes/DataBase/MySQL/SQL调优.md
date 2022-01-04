## SQL调优

### MySQL基础结构图

![MySQL结构图](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/MySQL结构图.40qxexo8bm00.jpg)

所谓的调优也就是在，执行器执行之前的分析器，优化器阶段完成的。



### 调优操作

#### 排除缓存干扰

因为存在缓存，所以有时候sql在第一次执行之后都是比较快的速度，如果没有注意到就会比较容易被干扰。

加上SQL NoCache去跑SQL，这样跑出来的时间就是真实的查询时间了。

缓存失效比较频繁的原因就是，只要一对表进行更新，那这个表所有的缓存都会被清空。

#### Explain字段

expain出来的信息有10列，分别是id、select_type、table、type、possible_keys、key、key_len、ref、rows、Extra

概要描述：

- id：选择标识符
- select_type：查询的类型
- table：输出结果集的表
- type：表的连接类型
- possible_keys：查询时可能使用的索引
- key：实际使用的索引
- key_len：索引字段的长度
- ref：列于索引的比较
- rows：估算需要扫描的行数
- Extra：执行情况的描述和说明

详细解释：

- **id**

  select标识符，select的查询序列号。

  1. id相同时，执行顺序由上至下。
  2. 如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行。
  3. id如果相同，可以认为是一组，从上往下顺序执行；在所有组中，id值越大，优先级越高，越先执行

- **select_type**

  查询中每个select字句的类型

  1. SIMPLE(简单SELECT，不使用UNION或子查询等)
  2. PRIMARY(子查询中最外层查询，查询中若包含任何复杂的子部分，最外层的select被标记为PRIMARY)
  3. UNION(UNION中的第二个或后面的SELECT语句)
  4.  DEPENDENT UNION(UNION中的第二个或后面的SELECT语句，取决于外面的查询)
  5. UNION RESULT(UNION的结果，union语句中第二个select开始后面所有select)
  6. SUBQUERY(子查询中的第一个SELECT，结果不依赖于外部查询)
  7. DEPENDENT SUBQUERY(子查询中的第一个SELECT，依赖于外部查询)
  8. DERIVED(派生表的SELECT, FROM子句的子查询)
  9. UNCACHEABLE SUBQUERY(一个子查询的结果不能被缓存，必须重新评估外链接的第一行)

- **table**

  所访问数据库中表名称（显示这一行的数据是关于哪张表的），有时不是真实的表名字，可能是简称

- **type**

  访问方式，表示MySQL在表中找到所需行的方式，又称“访问类型”。

  常用的类型有： **ALL、index、range、 ref、eq_ref、const、system、NULL（从左到右，性能从差到好）**

  1. ALL：Full Table Scan， MySQL将遍历全表以找到匹配的行
  2. index: Full Index Scan，index与ALL区别为index类型只遍历索引树
  3. range:只检索给定范围的行，使用一个索引来选择行
  4. ref: 表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值
  5. eq_ref: 类似ref，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配，简单来说，就是多表连接中使用primary key或者 unique key作为关联条件
  6. const、system: 当MySQL对查询某部分进行优化，并转换为一个常量时，使用这些类型访问。如将主键置于where列表中，MySQL就能将该查询转换为一个常量，system是const类型的特例，当查询的表只有一行的情况下，使用system
  7. NULL: MySQL在优化过程中分解语句，执行时甚至不用访问表或索引，例如从一个索引列里选取最小值可以通过单独索引查找完成。

- **possible_keys**

  **MySQL能使用哪个索引在表中找到记录，查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询使用（该查询可以利用的索引，如果没有任何索引显示 null）**

  该列完全独立于EXPLAIN输出所示的表的次序。这意味着在possible_keys中的某些键实际上不能按生成的表次序使用。

- **key**

  **key列显示MySQL实际决定使用的键（索引），必然包含在possible_keys中（不一定）**

  如果没有选择索引，键是NULL。要想强制MySQL使用或忽视possible_keys列中的索引，在查询中使用**FORCE INDEX、USE INDEX**或者**IGNORE INDEX**。

- **key_len**

  **表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度（key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的）**

  不损失精确性的情况下，长度越短越好

- **Extra**

  **该列包含MySQL解决查询的详细信息,有以下几种情况：**

  1. **Using where**：使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。**注意**：Extra列出现Using where表示MySQL服务器将存储引擎返回服务层以后再应用WHERE条件过滤。（**回表**）
  2. **Using temporary**：表示MySQL需要使用**临时表来存储结果集**，常见于排序和分组查询，常见 **group by , order by**
  3. **Using filesort**：当Query中包含 order by 操作，而且**无法利用索引完成的排序**操作称为“文件排序”
  4. **Using join buffer**：使用了连接缓存：**Block Nested Loop**，连接算法是块嵌套循环连接;**Batched Key Access**，连接算法是批量索引连接
  5. **Impossible where**：这个值强调了where语句会导致没有符合条件的行（通过收集统计信息不可能存在结果）。
  6. **Select tables optimized away**：这个值意味着仅通过使用索引，优化器可能仅从聚合函数结果中返回一行
  7. **No tables used**：Query语句中使用from dual 或不含任何from子句
  8. **Using index**：说明查询是覆盖了索引的，不需要读取数据文件，从索引树（索引文件）中即可获得信息。如果同时出现**using where**，表明索引被用来执行索引键值的查找，没有using where，表明索引用来读取数据而非执行查找动作。这是MySQL服务层完成的，但无需再回表查询记录。
  9. **distinct**: 一旦mysql找到了与行相联合匹配的行，就不再搜索了

- **总结：**
  • **EXPLAIN不会告诉你关于触发器、存储过程的信息或用户自定义函数对查询的影响情况**
  • **EXPLAIN不考虑各种Cache**
  • **EXPLAIN不能显示MySQL在执行查询时所作的优化工作**
  • **部分统计信息是估算的，并非精确值**
  • **EXPALIN只能解释SELECT操作，其他操作要重写为SELECT后查看执行计划**

#### Explain分析

- Explain统计的行数知识一个接近的数字，并不是完全正确的，索引也不是一定就是走最优的。

  原因：

  1. MySQL中的**数据单位都是页**，MySQL又采用了采样统计的方法，采样统计的时候，InnoDB默认会选择N个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。
  2. 我们的数据时一直会变的，所以索引的统计信息也是会变的，会根据一个阈值，重新统计。
  3. MySQL索引一般走错都是因为优化器在选择的时候发现，走A索引没有额外的代价，比如走B索引并不能直接拿到我们的值，还需要回到主键索引才可以拿到，多了一次回表的过程，这个也是会被优化器考虑进去的。
  4. 如果是上面的统计信息错了，那简单，我们用**analyze table tablename** 就可以重新统计索引信息了，所以在实践中，如果你发现explain的结果预估的rows值跟实际情况差距比较大，可以采用这个方法来处理。
  5. 还有一个方法就是force index强制走正确的索引，或者优化SQL，最后实在不行，可以**新建索引，或者删掉错误的索引**。

#### 覆盖索引

​	建立的索引上就已经有我们需要的字段，就不需要**回表**

​	由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。

#### 联合索引

- 商品表举例，需要根据他的名称，去查他的库存，假设这是一个很高频的查询请求，怎么建立索引呢？
  1. 建立一个**名称和库存的联合索引**，这样名称查出来就可以看到库存了，不需要查出id之后去回表再查询库存了（回表）。
  2. 联合索引在开发过程中也是常见的，但是并不是可以一直建立的，大家要思考**索引占据的空间**。

#### 最左匹配原则

#### 索引下推

```
select * from itemcenter where name like '敖%' and size=22 and age = 20;
```

在MySQL 5.6之前，只能从ID开始一个个回表，到主键索引上找出数据行，再对比字段值。

![索引下推_origin](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/索引下推_origin.15rye5b5v8f4.jpg)

而MySQL 5.6 引入的**索引下推优化（index condition pushdown)**， 可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数

![索引下推_new](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/索引下推_new.1ei5jkxq2zcw.jpg)

#### 唯一索引普通索引选择难题

- **change buffer**

  当更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InnoDB会将这些更新操作缓存在change buffer中，这样就不需要从磁盘中读入这个数据页了。

  下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行change buffer中与这个页有关的操作，通过这种方式就能保证这个数据逻辑的正确性。

  需要说明的是，虽然叫做change buffer，实际上是**可以持久化**的数据。也就是说，change buffer在内存中有拷贝，也会被写入到磁盘上。

  将change buffer中的操作应用到原数据页，得到最新结果的过程称为**merge**。

  除了**访问这个数据页会触发merge**外，系统有**后台线程会定期merge**。在数据库**正常关闭（shutdown）的过程中，也会执行merge操作**。

  显然，如果能够将更新操作先记录在change buffer，减少读磁盘，语句的执行速度会得到明显的提升。而且，**数据读入内存是需要占用buffer pool**的，所以这种方式还能够**避免占用内存，提高内存利用率**

![change-buffer](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/change-buffer.5uhwk34ag980.jpg)

- **change buffer使用**

  对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束。

  要判断表中是否存在这个数据，而这必须要将数据页读入内存才能判断，如果都已经读入到内存了，那直接更新内存会更快，就没必要使用change buffer。

  因此，**唯一索引的更新就不能使用change buffer**，实际上也只有普通索引可以使用。

  change buffer用的是**buffer pool**里的内存，因此不能无限增大，change buffer的大小，可以通过参数**innodb_change_buffer_max_size**来动态设置，这个参数设置为50的时候，表示change buffer的大小最多只能占用buffer pool的50%。

  将数据从**磁盘读入内存涉及随机IO**的访问，是数据库里面成本最高的操作之一，**change buffer因为减少了随机磁盘访问**，所以对更新性能的提升是会很明显的。

- **change buffer的使用场景**

  因为merge的时候是真正进行数据更新的时刻，而**change buffer的主要目的就是将记录的变更动作缓存**下来，所以在一个数据页做merge之前，change buffer记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大。

  因此，对于**写多读少的业务**来说，页面在写完以后马上被访问到的概率比较小，此时change buffer的使用效果最好，这种业务模型常见的就是账单类、日志类的系统。

  反过来，假设一个业务的**更新模式是写入之后马上会做查询**，那么即使满足了条件，将更新先记录在change buffer，但之后由于马上要访问这个数据页，会立即触发merge过程。这样随机访问IO的次数不会减少，反而增加了change buffer的维护代价，所以，对于这种业务模式来说，change buffer反而起到了副作用。

#### 前缀索引

​	定义**字符串的一部分作为索引**。默认地，如果你创建索引的语句不指定前缀长度，那么索引就会包含整个字符串。

​	使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。

​	覆盖索引是不需要回表的，但是前缀索引，即使你的联合索引已经包涵了相关信息，他还是会回表，因为他不确定你到底是不是一个完整的信息，就算你是一个	完整的邮箱去查询，他还是不知道你是否是完整的，所以他需要回表去判断一下。

- **很长的字段，想做索引我们怎么去优化他呢？**

  因为存在一个磁盘占用的问题，索引选取的越长，占用的磁盘空间就越大，相同的数据页能放下的索引值就越少，搜索的效率也就会越低。

  只要区分度过高，都可以，那我们可以采用倒序，或者删减字符串这样的情况去建立我们自己的区分度，**注意的是，调用函数也是一次开销**

#### 条件字段函数操作

如果对字段**做了函数计算，就用不上索引**了，这是MySQL的规定。

对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能。

- **隐式类型转换**

```
select * from t where id = 1
```

如果id是字符类型的，1是数字类型的，用explain会发现走了全表扫描，根本用不上索引。

因为MySQL底层会对你的比较进行转换，相当于加了 **CAST( id AS signed int)** 这样的一个函数，上面说过**函数会导致走不上索引**。

- **隐式字符编码转换**

  如果两个表的字符集不一样，一个是utf8mb4，一个是utf8，因为utf8mb4是utf8的超集，所以一旦两个字符比较，就会转换为utf8mb4再比较。

  转换的过程相当于加了CONVERT(id USING utf8mb4)函数，那又回到上面的问题了，用到函数就用不上索引。

#### flush

**redo log**就是对数据库操作的日志，他是在内存中的，每次操作一旦写了redo log就会立马返回结果，但是这个redo log总会找个时间去更新到磁盘，这个操作就是**flush**。

在更新之前，当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为**“脏页”**。

内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为**“干净页“**。

- **什么时候flush**
  1. InnoDB的redo log写满了，这时候系统会停止所有更新操作，把checkpoint往前推进，redo log留出空间可以继续写。
  2. 系统内存不足，当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘。
  3. 如果刷脏页一定会写盘，就保证了每个数据页有两种状态：
     - 一种是内存里存在，内存里就肯定是正确的结果，直接返回
     - 另一种是内存里没有数据，就可以肯定数据文件上是正确的结果，读入内存后返回。这样的效率最高。
  4. MySQL认为系统“空闲”的时候，只要有机会就刷一点“脏页”。
  5. MySQL正常关闭，这时候，MySQL会把内存的脏页都flush到磁盘上，这样下次MySQL启动的时候，就可以直接从磁盘上读数据，启动速度会很快。

![flush](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/flush.6ihfm2xbjww0.jpg)

- **flush的时机**

  Innodb刷脏页控制策略，每个电脑主机的io能力是不一样的，你要正确地告诉InnoDB所在主机的IO能力，这样InnoDB才能知道需要全力刷脏页的时候，可以刷多快。

  这就要用到**innodb_io_capacity**这个参数了，它会告诉InnoDB你的磁盘能力，这个值建议设置成磁盘的IOPS，磁盘的IOPS可以通过**fio**这个工具来测试。

  刷脏页的时候，旁边如果也是脏页，会一起刷掉的，并且如果周围还有脏页，这个连带责任制会一直蔓延，这种情况其实在机械硬盘时代比较好，一次IO就解决了所有问题。

  **innodb_flush_neighbors=0**这个参数可以不产生连带制

![flush_连带](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/flush_连带.5g14m8jps800.jpg)

### 总结

![SQL调优](https://cdn.jsdelivr.net/gh/hanmlian/image-hosting@master/DataBase/SQL调优.1p4ajgsxkrxc.jpg)