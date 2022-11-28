## 一、什么影响了数据库查询速度

### 1.1 影响数据库查询速度的四个因素

  ![clipboard.png](https://segmentfault.com/img/bV5m06)

### 1.2 风险分析

> **QPS：**`Queries Per Second`意思是“每秒查询率”，是一台服务器每秒能够相应的查询次数，是对一个特定的查询服务器在规定时间内所处理流量多少的衡量标准。

> **TPS：**是`TransactionsPerSecond`的缩写，也就是事务数/秒。它是软件测试结果的测量单位。客户机在发送请求时开始计时，收到服务器响应后结束计时，以此来计算使用的时间和完成的事务个数。

  **Tips：**最好不要在主库上数据库备份，大型活动前取消这样的计划。

1. 效率低下的`sql`：超高的`QPS`与`TPS`。
2. 大量的并发：数据连接数被占满（`max_connection`默认`100`，一般把连接数设置得大一些）。
   并发量:同一时刻数据库服务器处理的请求数量
3. 超高的`CPU`使用率：`CPU`资源耗尽出现宕机。
4. 磁盘`IO`：磁盘`IO`性能突然下降、大量消耗磁盘性能的计划任务。解决：更快磁盘设备、调整计划任务、做好磁盘维护。

### 1.3 网卡流量：如何避免无法连接数据库的情况

1. 减少从服务器的数量（从服务器会从主服务器复制日志）
2. 进行分级缓存（避免前端大量缓存失效）
3. 避免使用`select *` 进行查询
4. 分离业务网络和服务器网络

### 1.4 大表带来的问题（**`重要`**）

#### 1.4.1 大表的特点

1. 记录行数巨大，单表超千万
2. 表数据文件巨大，超过`10`个`G`

#### 1.4.2 大表的危害

1.慢查询：**很难在短时间内过滤出需要的数据**
  查询字区分度低 -> 要在大数据量的表中筛选出来其中一部分数据会产生大量的磁盘`io` -> 降低磁盘效率

2.对`DDL`影响：

  **建立索引需要很长时间：**

- `MySQL -v<5.5` 建立索引会锁表
- `MySQL -v>=5.5` 建立索引会造成主从延迟（`mysql`建立索引，先在组上执行，再在库上执行）

  **修改表结构需要长时间的锁表：**会造成长时间的主从延迟('480秒延迟')

#### 1.4.3 如何处理数据库上的大表

> 分库分表把一张大表分成多个小表

  **难点：**

1. 分表主键的选择
2. 分表后跨分区数据的查询和统计

### 1.5 大事务带来的问题（**`重要`**）

#### 1.5.1 什么是事务

  ![clipboard.png](https://segmentfault.com/img/bV5pO8)

#### 1.5.2事务的`ACID`属性

> 1、原子性（`atomicity`)：全部成功，全部回滚失败。银行存取款。

> 2、一致性（consistent)：银行转账的总金额不变。

> 3、隔离性（isolation)：

  **隔离性等级：**

- 未提交读(`READ UNCOMMITED`) **脏读**,两个事务之间互相可见；-- `存在脏读、不可重复读、幻读的问题`
- 已提交读(`READ COMMITED`)符合隔离性的基本概念,一个事务进行时，其它已提交的`事物`对于该事务是可见的，即可以获取其它事务提交的数据（**不可重复读**）。-- `解决脏读的问题`，存在不可重复读、幻读的问题。
- 可重复读(`REPEATABLE READ`) **`InnoDB的默认隔离等级`**。事务进行时，其它所有事务对其不可见，即多次执行读，得到的结果是一样的！ -- mysql 默认级别，`解决脏读、不可重复读的问题`，存在幻读的问题。使用 MVCC（多版本并发控制，提高并发，不加锁）机制 实现可重复读。**`But，MySQL在可重复读级别已经解决幻读，插不进数据，有间隙锁。`**
- 可串行化（`SERIALIZABLE`） 在读取的每一行数据上都加锁，会造成大量的锁超时和锁征用，严格数据一致性且没有并发是可使用。 -- `解决脏读、不可重复读、幻读`，可保证事务安全，但完全串行执行，性能最低。

```css
不可重复读：事务A首先读取了一条数据，然后执行逻辑的时候，事务B将这条数据改变了，然后事务A再次读取的时候，发现数据不匹配了，就是所谓的不可重复读了。

也就是说，当前事务先进行了一次数据读取，然后再次读取到的数据是别的事务修改成功的数据，导致两次读取到的数据不匹配，也就照应了不可重复读的语义。

幻读：事务A首先根据条件索引得到N条数据，然后事务B改变了这N条数据之外的M条或者增添了M条符合事务A搜索条件的数据，导致事务A再次搜索发现有N+M条数据了，就产生了幻读。

也就是说，当前事务读第一次取到的数据比后来读取到数据条目少。

不可重复读和幻读比较：
两者有些相似，但是前者针对的是update或delete，后者针对的insert。
```

  **查看系统的事务隔离级别：**`show variables like '%iso%'`;
  **开启一个新事务：**`begin`;
  **提交一个事务：**`commit`;
  **修改事物的隔离级别：**`set session tx_isolation='read-committed'`;

> 4、持久性(`DURABILITY`)：从数据库的角度的持久性，磁盘损坏就不行了

  ![clipboard.png](https://segmentfault.com/img/bV5p77)

  `redo log`机制保证事务更新的**一致性**和**持久性**

### 1.5.3 大事务

> 运行时间长，操作数据比较多的事务；

  **风险：锁定数据太多，回滚时间长，执行时间长。**

1. 锁定太多数据，造成大量阻塞和锁超时；
2. 回滚时所需时间比较长，且数据仍然会处于锁定；
3. 如果执行时间长，将造成主从延迟，因为只有当主服务器全部执行完写入日志时，从服务器才会开始进行同步，造成延迟。

  **解决思路：**

1. 避免一次处理太多数据，可以分批次处理；
2. 移出不必要的`SELECT`操作，保证事务中只有必要的写操作。

## 二、什么影响了MySQL性能（**`非常重要`**）

### 2.1 影响性能的几个方面

1. 服务器硬件。
2. 服务器系统（系统参数优化）。
3. **存储引擎**。
   `MyISAM`： 不支持事务，表级锁。
   `InnoDB`: 支持事务，支持行级锁，事务`ACID`。
4. **数据库参数配置。**
5. **`数据库结构设计和SQL语句。（重点优化）`**

### 2.2 MySQL体系结构

  **分三层：客户端->服务层->存储引擎**

  ![clipboard.png](https://segmentfault.com/img/bV5uXi)

1. `MySQL`是**`插件式的存储引擎`**，其中存储引擎分很多种。只要实现符合mysql存储引擎的接口，可以开发自己的存储引擎!
2. 所有跨存储引擎的功能都是在服务层实现的。
3. MySQL的**存储引擎是针对表的，不是针对库的**。也就是说在一个数据库中可以使用不同的存储引擎。但是不建议这样做。

### 2.3 InnoDB存储引擎

  `MySQL5.5`及之后版本**默认的存储引擎**：`InnoDB`。

#### 2.3.1 InnoDB使用表空间进行数据存储。

```
show variables like 'innodb_file_per_table
```

 如果innodb_file_per_table 为 ON 将建立独立的表空间，文件为tablename.ibd；

 如果innodb_file_per_table 为 OFF 将数据存储到系统的共享表空间，文件为ibdataX（X为从1开始的整数）；

 `.frm` ：是服务器层面产生的文件，类似服务器层的数据字典，**记录表结构**。

#### 2.3.2 (MySQL5.5默认)系统表空间与(`MySQL5.6`及以后默认)独立表空间

  1.1 系统表空间无法简单的收缩文件大小，造成空间浪费，并会产生大量的磁盘碎片。

  1.2 独立表空间可以通过`optimeze table` 收缩系统文件，不需要重启服务器也不会影响对表的正常访问。

  2.1 如果对多个表进行刷新时，实际上是顺序进行的，会产生IO瓶颈。

  2.2 独立表空间可以同时向多个文件刷新数据。

**强烈建立对Innodb 使用独立表空间，优化什么的更方便，可控。**

#### 2.3.3 系统表空间的表转移到独立表空间中的方法

  1、使用mysqldump 导出所有数据库数据（存储过程、触发器、计划任务一起都要导出 ）可以在从服务器上操作。

  2、停止MYsql 服务器，修改参数（my.cnf加入innodb_file_per_table），并删除Inoodb相关文件（可以重建Data目录）。

  3、重启MYSQL，并重建Innodb系统表空间。

  4、 重新导入数据。

  **或者** `Alter table` 同样可以的转移，但是无法回收系统表空间中占用的空间。

### 2.4 InnoDB存储引擎的特性

#### 2.4.1 特性一：事务性存储引擎及两个特殊日志类型：Redo Log 和 Undo Log

1. `Innodb` 是一种**事务性存储引擎**。
2. 完全支持事务的`ACID`特性。
3. 支持事务所需要的两个特殊日志类型：`Redo Log` 和`Undo Log`

  **Redo Log：**实现事务的持久性(已提交的事务)。
  **Undo Log：**未提交的事务，独立于表空间，需要随机访问，可以存储在高性能io设备上。

> `Undo`日志记录某数据被修改前的值，可以用来在事务失败时进行`rollback`；`Redo`日志记录某数据块被修改后的值，可以用来恢复未写入`data file`的已成功事务更新的数据。

#### 2.4.2 特性二：支持行级锁

1. InnoDB支持行级锁。
2. 行级锁可以最大程度地支持并发。
3. 行级锁是由存储引擎层实现的。

### 2.5 什么是锁

#### 2.5.1 锁

  ![clipboard.png](https://segmentfault.com/img/bV5vh9)

#### 2.5.2 锁类型

  ![clipboard.png](https://segmentfault.com/img/bV5vkS)

```n1ql
S锁（读锁）： select ... lock in share mode
X锁（写锁）：select ... for update (update、delete、)
```

#### 2.5.3 锁的粒度

MySQL的事务支持**不是绑定在MySQL服务器本身**，**`而是与存储引擎相关`**

  ![clipboard.png](https://segmentfault.com/img/bV5vFB)

  将**table_name**加表级锁命令：`lock table table_name write`; **`写锁会阻塞其它用户对该表的‘读写’操作，直到`**写锁被释放：`unlock tables`；

1. **锁的开销越大，粒度越小，并发度越高。**
2. 表级锁通常是在服务器层实现的。
3. 行级锁是存储引擎层实现的。innodb的锁机制，服务器层是不知道的

#### 2.5.4 锁的分类

（1）悲观锁

总是假设最坏的情况，每次拿数据都认为别人会修改数据，所以要加锁，别人只能等待，直到我释放锁才能拿到锁；数据库的行锁、表锁、读锁、写锁都是这种方式，java中的synchronized和ReentrantLock也是悲观锁的思想。

（2）乐观锁

总是假设最好的情况，每次拿数据都认为别人不会修改数据，所以不会加锁，但是更新的时候，会判断在此期间有没有人修改过；一般基于版本号机制实现。

（3）使用场景

乐观锁适用于读多写少的情况，即冲突很少发生；如果是多写的情况，应用会不断重试，反而会降低系统性能，这种情况最好用悲观锁，因为等待到锁被释放后，可以立即获得锁进行操作。

拓展： [(1)图解悲观锁和乐观锁](https://link.segmentfault.com/?enc=t5yAi%2Bos5wxDKYrA%2FwUWtg%3D%3D.qG0wfg97tNGvqcia35SOAah2qjGoYgMEubhGfW80BCJmdYBTTOjb24IM6lhtpbXv)
[(2)什么是乐观锁与悲观锁？](https://link.segmentfault.com/?enc=ZCYdX9RiqqH1F4VKvD91Xg%3D%3D.4ZkE%2FOA9RbhFDI07lbo4hqUC4th8wkpEkv6Qq54N3XQ%2FgXdemLXbRE%2BH%2FFy2ggCG)

#### 2.5.4 阻塞和死锁

  （1）阻塞是由于资源不足引起的排队等待现象。
  （2）死锁是由于两个对象在拥有一份资源的情况下申请另一份资源，而另一份资源恰好又是这两对象正持有的，导致两对象无法完成操作，且所持资源无法释放。

### 2.6 如何选择正确的存储引擎

  **参考条件：**

1. 事务
2. 备份(`Innobd`免费在线备份)
3. 崩溃恢复
4. 存储引擎的特有特性

  **总结:****`Innodb`大法好。**
  **`注意:`**尽量别使用混合存储引擎，比如回滚会出问题在线热备问题。

### 2.7 配置参数

#### 2.7.1 内存配置相关参数

> 确定可以使用的内存上限。

```undefined
内存的使用上限不能超过物理内存，否则容易造成内存溢出；（对于32位操作系统，MySQL只能试用3G以下的内存。）
```

> 确定MySQL的**每个连接`单独`**使用的内存。

```mipsasm
sort_buffer_size #定义了每个线程排序缓存区的大小，MySQL在有查询、需要做排序操作时才会为每个缓冲区分配内存（直接分配该参数的全部内存）；
join_buffer_size #定义了每个线程所使用的连接缓冲区的大小，如果一个查询关联了多张表，MySQL会为每张表分配一个连接缓冲，导致一个查询产生了多个连接缓冲；
read_buffer_size #定义了当对一张MyISAM进行全表扫描时所分配读缓冲池大小，MySQL有查询需要时会为其分配内存，其必须是4k的倍数；
read_rnd_buffer_size #索引缓冲区大小，MySQL有查询需要时会为其分配内存，只会分配需要的大小。
```

  **`注意：`**以上四个参数是为一个线程分配的，如果有100个连接，那么需要×100。

> MySQL数据库实例：
>
> 　①MySQL是**`单进程多线程`**（而oracle是多进程），也就是说`MySQL`实例在系统上表现就是一个服务进程，即进程；
>
> 　②MySQL实例是线程和内存组成，实例才是真正用于操作数据库文件的；
>
> **一般情况下**一个实例操作一个或多个数据库；**集群情况下**多个实例操作一个或多个数据库。

  **如何为缓存池分配内存：**
  `Innodb_buffer_pool_size`，定义了Innodb所使用缓存池的大小，对其性能十分重要，必须足够大，但是过大时，使得Innodb 关闭时候需要更多时间把脏页从缓冲池中刷新到磁盘中；

```undefined
总内存-（每个线程所需要的内存*连接数）-系统保留内存
```

  `key_buffer_size`，定义了MyISAM所使用的缓存池的大小，由于数据是依赖存储操作系统缓存的，所以要为操作系统预留更大的内存空间；

```axapta
select sum(index_length) from information_schema.talbes where engine='myisam'
```

  **注意：**即使开发使用的表全部是Innodb表，也要为MyISAM预留内存，因为MySQL系统使用的表仍然是MyISAM表。

  `max_connections` 控制允许的最大连接数， 一般2000更大。
  **不要使用外键约束保证数据的完整性。**

### 2.8 性能优化顺序

  **从上到下：**

  ![clipboard.png](https://segmentfault.com/img/bV5wVH)

## 三、数据库结构优化（**`非常重要`**）

### 3.1 数据库结构优化目的

  **1、减少数据冗余：**（数据冗余是指在数据库中存在相同的数据，或者某些数据可以由其他数据计算得到），注意，尽量减少不代表完全避免数据冗余；

 **2、尽量避免数据维护中出现更新，插入和删除异常：**

​    ![clipboard.png](https://segmentfault.com/img/bV5AYv)

​    总结：要避免异常，需要对数据库结构进行**`范式化设计`**。

  **3、节约数据存储空间。**

  **4、提高查询效率。**

### 3.2 数据库结构设计步骤

  1、**需求分析：**全面了解产品设计的存储需求、数据处理需求、数据安全性与完整性；

  2、**逻辑设计（`重要`）：**设计数据的逻辑存储结构。**数据实体之间的逻辑关系，解决数据冗余和数据维护异常。数据范式可以帮助我们设计；**

  3、**物理设计：**表结构设计，**存储引擎与列的数据类型**；

  4、**维护优化：****索引优化**、存储结构优化。

### 3.3 数据库范式设计与反范式化

  [**传送门：**数据库逻辑设计之三大范式通俗理解，一看就懂，书上说的太晦涩](https://segmentfault.com/a/1190000013695030)

### 3.4 物理设计

  ![clipboard.png](https://segmentfault.com/img/bV5Eux)

  ![clipboard.png](https://segmentfault.com/img/bV5EuI)

  ![clipboard.png](https://segmentfault.com/img/bV5EYl)

##### [**相关传送门：**MySQL中字段类型与合理的选择字段类型；int(11)最大长度是多少？，varchar最大长度是多少](https://segmentfault.com/a/1190000010012140)

## 四、高可用架构设计

  ![clipboard.png](https://segmentfault.com/img/bV5E2S)

  ![clipboard.png](https://segmentfault.com/img/bV5FbX)

### 4.1 读写分离

  ![clipboard.png](https://segmentfault.com/img/bV5FCy)

  [MaxScale：实现MySQL读写分离与负载均衡的中间件利器](https://link.segmentfault.com/?enc=IKBAufkKkV6fW9zUku45pA%3D%3D.CtcPND9nlhv%2FxSywQBLU2BBNtr8y6Za%2FA%2FYouwZQpbtREZdnLltsJj%2FMnXViw7Ct)

## 五、数据库索引优化（**`非常重要`**）

### 5.1 两种主要数据结构：B-tree和Hash

#### 5.1.1 B-tree结构

  ![clipboard.png](https://segmentfault.com/img/bV5FXp)

##### **B-tree索引的限制：**

  ![clipboard.png](https://segmentfault.com/img/bV5F3S)

#### 5.1.2 Hash结构

  ![clipboard.png](https://segmentfault.com/img/bV5FYs)

##### **Hash索引的限制：**

- Hash索引必须进行二次查找
- Hash索引无法用于排序
- Hash索引**不支持部分索引查找也不支持范围查找**
- Hash索引中Hash码的计算**可能存在Hash冲突，不适合重复值很高的列，如性别**，身份证比较合适。

#### 5.1.3 MySQL常见索引和各种索引区别

```pgsql
PRIMARY KEY（主键索引）  ALTER TABLE `table_name` ADD PRIMARY KEY ( `column` ) 
UNIQUE(唯一索引)     ALTER TABLE `table_name` ADD UNIQUE (`column`)
INDEX(普通索引)     ALTER TABLE `table_name` ADD INDEX index_name ( `column` ) 
FULLTEXT(全文索引)      ALTER TABLE `table_name` ADD FULLTEXT ( `column` )
组合索引   ALTER TABLE `table_name` ADD INDEX index_name ( `column1`, `column2`, `column3` ) 
```

1. 普通索引：最基本的索引，没有任何限制
2. 唯一索引：与"普通索引"类似，不同的就是：索引列的值必须唯一，但允许有空值。
3. 主键索引：它 是一种特殊的唯一索引，不允许有空值。
4. 全文索引：仅可用于 MyISAM 表，针对较大的数据，生成全文索引很耗时好空间。
5. 组合索引：为了更多的提高mysql效率可建立组合索引，遵循”最左前缀“原则。

### 5.2 使用索引好处和索引缺陷

#### 5.2.1 为什么要使用索引

  **1、索引大大减少了存储引擎需要扫描的数据量；**

  **2、索引可以帮助我们进行排序以避免使用临时表；**

  **3、索引可以把随机I/O变为顺序I/O。**

#### 5.2.2 索引不是越多越好

  **1、索引会增加写操作的成本；**

  **2、太多的索引会增加查询优化器的选择时间。**

> 索引就好比一本书的目录，它会让你更快的找到内容，显然目录（索引）并不是越多越好，假如这本书1000页，而有500页是目录，它当然效率低，目录是要占纸张的,而索引是要占磁盘空间的。

### 5.3 索引优化策略

#### 5.3.1 索引列上不能使用表达式和函数

  ![clipboard.png](https://segmentfault.com/img/bV5LaJ)

#### 5.3.2 前缀索引和索引列的选择性

> `Innodb`索引列最大宽度为`667`个字节(`utf-8` 差不多`255`个字符),`MyIsam`索引类宽度最大为`1000`个字节，于是出现前缀索引，索引的选择性。

  对于列的值较长，比如`BLOB、TEXT、VARCHAR`，就必须建立前缀索引，即将值的前一部分作为索引。这样既可以节约空间，又可以提高查询效率。但无法使用前缀索引做 `ORDER BY` 和 `GROUP BY`，也无法使用前缀索引做覆盖扫描。

  **语法：** `ALTER TABLE table_name ADD KEY(column_name(prefix_length))`

  ![clipboard.png](https://segmentfault.com/img/bV5LkN)

##### **如何选择索引列的顺序：**

  1、经常会被使用到的列优先（**`选择性`**差的列不适合，如性别，**查询优化器**可能会认为全表扫描性能更好）；

  2、**`选择性高`**的列优先；

  3、宽度小的列优先（一页中存储的索引越多，降低`I/O`，查找越快）；

#### 5.3.3 组合/联合索引策略

  如果索引了多列，要遵守**`最左前缀`**法则。指的是查询从索引的最左前列开始并且**不跳过索引中的列**。
[**深入理解请移步：**最左前缀原理与相关优化](https://link.segmentfault.com/?enc=B%2B90U8mkrGs0pQjc3xDwuw%3D%3D.hzWUzVsErv8jvt8F%2BNlU4zwr9YuIDw%2FRDO0Q7bpmP6nIPutlClDP2XLCVMPVMPW%2Bn3RCH94ciuwRPTJvI%2BxvkA%3D%3D)

#### 5.3.4 覆盖索引策略

  跟组合索引有点类似，如果**`索引包含所有满足查询需要的数据的索引则成为覆盖索引`**(Covering Index)，也就是平时所说的不需要回表操作。即**索引的叶子节点上面包含了他们索引的数据(hash索引不可以)**。

  判断标准：使用`explain`，可以通过输出的`extra`列来判断，对于一个索引覆盖查询，显示为`using index`,`MySQL`查询优化器在执行查询前会决定是否有索引覆盖查询。

##### **优点:**

  1、可以优化缓存,减少磁盘IO操作；
  2、可以减少随机IO,变随机IO操作变为顺序IO操作；
  3、可以避免对InnoDB主键索引的二次查询；
  4、可以避免MyISAM表进行系统调用；

##### **无法使用覆盖索引的情况：**

  1、存储引擎不支持覆盖索引；
  2、**`查询中使用了太多的列`**（如`SELECT *` ）；
  3、使用了双`%`号的`like`查询（底层API所限制）；

  [mysql高效索引之覆盖索引](https://link.segmentfault.com/?enc=W2Jc7kugZ1tQ4khDR5SZuA%3D%3D.sTHwKZLT%2BUDcuVv%2FObfDZ8%2FiOOWI4qt0DZTQIeLpSZGHrsyxiBNGW%2Bvn5s%2FLh3fULQET2dSIKaDP5RP7zt3sIQ%3D%3D)

#### 5.3.5 SQL索引优化总结口诀（**`套路重点`**）

> **`全值匹配我最爱，最左前缀要遵守；`**
> **`带头大哥不能丢，中间兄弟不能断；`**
> **`索引列上不计算，范围之后全失效；`**
> **`LIKE百分写最右，覆盖索引不写 \*；`**
> **`不等空值还有or，索引失效要少用；`**
> **`字符单引不可丢，SQL高级也不难 ；`**

  [MySQL高级-索引优化](https://link.segmentfault.com/?enc=4OkMEqzoUu1Dfk%2Fojh8Jng%3D%3D.vTBwOSAsZeRJ9FYWvVAUpYOflPWr2aJzsvco0xzmgFzRIlsS3WHhfJNvbzD4w3nng%2Bm%2Bcyb1QeNDCIDKAB%2Bfmw%3D%3D)

### 5.4 使用索引来优化查询

#### 5.4.1 利用索引排序

  1、`group by` **`实质是先排序后分组`**，遵照索引的最佳左前缀。；

  2、索引中所有列的方向(**升序、降序**)和Order By子句完全一致；

  3、当无法使用索引列，增大`max_length_for_sort_data`参数的设置+增大`sort_buffer_size`参数的设置；

  4、如果最左列使用了范围，则排序会失效；

  5、`where` **高于**`having`，能写在`where`限定的条件就不要去`having`去限定了

### 5.5 索引的维护和优化

#### 5.5.1 删除重复索引

  ![clipboard.png](https://segmentfault.com/img/bV5PWy)

  **注：**主键约束相当于(唯一约束 + 非空约束)

  一张表中最多有一个主键约束,如果设置多个主键,就会出现如下提示：`Multiple primary key defined!!!`

#### 5.5.2 删除冗余索引

  ![clipboard.png](https://segmentfault.com/img/bV5PXH)

  **检查工具：****pt-duplicate-key-checker**

#### [**`扩展阅读： `**MySQL索引背后的数据结构及算法原理](https://link.segmentfault.com/?enc=Vs9P8l1A%2FD1PboOr%2BVAJtA%3D%3D.GV0uMrhEOoN0Cz2M8XQK%2B98KqYS4lQQ3iPteUaexN4M17cBbvcyhPtWXzz1W9KRqeukO7HpPIAopRl43lqJScQ%3D%3D)

> explain 查询计划
>
> Using where：表示优化器需要通过索引回表查询数据；
>
> Using index：表示直接访问索引就足够获取到所需要的数据，不需要通过索引回表，如覆盖索引；

## 六、SQL查询优化（**`重要`**）

### 6.1 获取有性能问题SQL的三种方式

1. 通过用户反馈获取存在性能问题的SQL；
2. 通过慢查日志获取存在性能问题的SQL；
3. 实时获取存在性能问题的SQL；

#### 6.1.2 慢查日志分析工具

**相关配置参数：**

```apache
slow_query_log # 启动停止记录慢查日志，慢查询日志默认是没有开启的可以在配置文件中开启(on)
slow_query_log_file # 指定慢查日志的存储路径及文件，日志存储和数据从存储应该分开存储

long_query_time # 指定记录慢查询日志SQL执行时间的阀值默认值为10秒通常,对于一个繁忙的系统来说,改为0.001秒(1毫秒)比较合适
log_queries_not_using_indexes #是否记录未使用索引的SQL
```

 **常用工具：**`mysqldumpslow`和`pt-query-digest`

```autoit
pt-query-digest --explain h=127.0.0.1,u=root,p=p@ssWord  slow-mysql.log
```

### 6.1.3 实时获取有性能问题的SQL（**推荐**）

![clipboard.png](https://segmentfault.com/img/bV5TPp?w=363&h=170)

```pgsql
SELECT id,user,host,DB,command,time,state,info
FROM information_schema.processlist
WHERE TIME>=60
```

 查询当前服务器执行超过`60s`的`SQL`，可以通过脚本周期性的来执行这条`SQL`，就能查出有问题的`SQL`。

### 6.2 SQL的解析预处理及生成执行计划（**`重要`**）

#### 6.2.1 查询过程描述（**`重点！！！`**）

![clipboard.png](https://segmentfault.com/img/bV5XzQ?w=643&h=654)

[**上图原文连接**](https://link.segmentfault.com/?enc=KFofsqpoj4cUv8lQ5xIj%2BA%3D%3D.GW%2BZ1EJ%2F4%2Bsj6nav2oEvzGHM3qQkk390pId9%2FZcyNDA5Ku40PMO3FsiCTMPaGsRyJOpSw3IhvJPNC9%2FgvmX2wQIjgl2mfGbBY2ZFWwy4uME%3D)

##### 通过上图可以清晰的了解到MySql查询执行的大致过程：

1. 发送`SQL`语句。
2. 查询缓存，如果命中缓存直接返回结果。
3. `SQL`解析，预处理，再由优化器生成对应的查询执行计划。
4. 执行查询，调用存储引擎API获取数据。
5. 返回结果。

#### 6.2.2 查询缓存对性能的影响（**建议关闭缓存**）

**第一阶段：**
**相关配置参数：**

```nginx
query_cache_type # 设置查询缓存是否可用
query_cache_size # 设置查询缓存的内存大小
query_cache_limit # 设置查询缓存可用的存储最大值（加上sql_no_cache可以提高效率）
query_cache_wlock_invalidate # 设置数据表被锁后是否返回缓存中的数据
query_cache_min_res_unit # 设置查询缓存分配的内存块的最小单
```

> **缓存查找是利用对大小写敏感的`哈希查找`来实现的**，Hash查找只能进行全值查找（sql完全一致），如果缓存命中，检查用户权限，如果权限允许，直接返回，查询不被解析，也不会生成查询计划。

##### **`在一个读写比较频繁的系统中，建议关闭缓存，因为缓存更新会加锁`**。将`query_cache_type`设置为`off`,`query_cache_size`设置为`0`。

#### 6.2.3 **第二阶段：**MySQL依照执行计划和存储引擎进行交互

 **这个阶段包括了多个子过程：**

 ![clipboard.png](https://segmentfault.com/img/bV5Zpr?w=352&h=103)

 ![clipboard.png](https://segmentfault.com/img/bV5Zpy?w=375&h=95)

 ![clipboard.png](https://segmentfault.com/img/bV5ZpI?w=368&h=84)

##### **`一条查询可以有多种查询方式`**，查询优化器会对每一种查询方式的（存储引擎）统计信息进行比较，找到成本最低的查询方式，**`这也就是索引不能太多的原因`**。

### 6.3 会造成MySQL生成错误的执行计划的原因

  1、统计信息不准确
  2、成本估算与实际的执行计划成本不同

  ![clipboard.png](https://segmentfault.com/img/bV5Zp4?w=354&h=109)

  3、给出的最优执行计划与估计的不同

  ![clipboard.png](https://segmentfault.com/img/bV5Zql?w=283&h=39)

  4、MySQL不考虑并发查询
  5、会基于固定规则生成执行计划
  6、MySQL不考虑不受其控制的成本，如存储过程，用户自定义函数

### 6.4 MySQL优化器可优化的SQL类型

> 查询优化器：对查询进行优化并查询mysql认为的成本最低的执行计划。 为了生成最优的执行计划，查询优化器会对一些查询进行改写

 可以优化的sql类型

 1、重新定义表的关联顺序；

 ![clipboard.png](https://segmentfault.com/img/bV5ZqK?w=327&h=35)

 2、将外连接转换为内连接；

 3、使用等价变换规则；

 ![clipboard.png](https://segmentfault.com/img/bV5Zrm?w=245&h=25)

 4、优化count(),min(),max()；

 ![clipboard.png](https://segmentfault.com/img/bV5ZrR?w=388&h=107)

 5、将一个表达式转换为常数；
 6、子查询优化；

 ![clipboard.png](https://segmentfault.com/img/bV5Zr3?w=336&h=45)

 7、提前终止查询，如发现一个不成立条件(如`where id = -1`)，立即返回一个空结果；

 8、对in()条件进行优化；

### 6.5 查询处理各个阶段所需要的时间

#### 6.5.1 使用profile(目前已经不推荐使用了)

```nginx
set profiling = 1; #启动profile,这是一个session级的配制执行查询

show profiles; # 查询每一个查询所消耗的总时间的信息

show profiles for query N; # 查询的每个阶段所消耗的时间
```

#### 6.5.2 performance_schema是5.5引入的一个性能分析引擎（5.5版本时期开销比较大）

启动监控和历史记录表：`use performance_schema`

```pgsql
update setup_instruments set enabled='YES',TIME = 'YES' WHERE NAME LIKE 'stage%';

update set_consumbers set enabled='YES',TIME = 'YES' WHERE NAME LIKE 'event%';
```

  ![clipboard.png](https://segmentfault.com/img/bV5ZsD?w=547&h=220)

  ![clipboard.png](https://segmentfault.com/img/bV5UmN?w=445&h=263)

### 6.6 特定SQL的查询优化

#### 6.6.1 大表的数据修改

  ![clipboard.png](https://segmentfault.com/img/bV5ZsJ?w=223&h=232)

  ![clipboard.png](https://segmentfault.com/img/bV5ZsY?w=516&h=242)

#### 6.6.2 大表的结构修改

  ![clipboard.png](https://segmentfault.com/img/bV5Zs8?w=295&h=117)

1. 利用主从复制，先对从服务器进入修改，然后主从切换
2. （推荐）

> 添加一个新表（修改后的结构），老表数据导入新表，老表建立触发器，修改数据同步到新表， 老表加一个排它锁（重命名）， 新表重命名， 删除老表。

  ![clipboard.png](https://segmentfault.com/img/bV5ZtG?w=330&h=169)

**修改语句这个样子：**

```sql
alter table sbtest4 modify c varchar(150) not null default ''
```

**利用工具修改：**

  ![clipboard.png](https://segmentfault.com/img/bV5ZtX?w=466&h=118)

#### 6.6.3 优化not in 和 <> 查询

 **子查询改写为关联查询：**

 ![clipboard.png](https://segmentfault.com/img/bV5Zuh?w=489&h=177)

## 七、分库分表

### 7.1 分库分表的几种方式

> 分担读负载 可通过 一主多从，升级硬件来解决。

#### 7.1.1 把一个实例中的多个数据库拆分到不同实例（集群）

![clipboard.png](https://segmentfault.com/img/bV5YaN?w=554&h=120)

  **拆分简单,不允许跨库。但并不能减少写负载。**

#### 7.1.2 把一个库中的表分离到不同的数据库中

![clipboard.png](https://segmentfault.com/img/bV5Yct?w=550&h=149)

  **该方式只能在一定时间内减少写压力。**

  以上两种方式只能暂时解决读写性能问题。

#### 7.1.3 数据库分片

> 对一个库中的相关表进行水平拆分到不同实例的数据库中

![clipboard.png](https://segmentfault.com/img/bV5YEy?w=537&h=158)

##### 7.1.3.1 如何选择分区键

1. 分区键要能尽可能避免跨分区查询的发生
2. 分区键要尽可能使各个分区中的数据平均

##### 7.1.3.2 分片中如何生成全局唯一ID

  ![clipboard.png](https://segmentfault.com/img/bV5ZuI?w=414&h=112)

##### [**扩展：表的垂直拆分和水平拆分**](

作者：唐成勇

来源：https://segmentfault.com/a/1190000013672421