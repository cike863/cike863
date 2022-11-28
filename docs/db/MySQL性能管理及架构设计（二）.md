## 一、数据库结构优化（**`非常重要`**）

### 1.1 数据库结构优化目的

  **1、减少数据冗余：**（数据冗余是指在数据库中存在相同的数据，或者某些数据可以由其他数据计算得到），注意，尽量减少不代表完全避免数据冗余；

 **2、尽量避免数据维护中出现更新，插入和删除异常：**

​    ![clipboard.png](https://segmentfault.com/img/bV5AYv)

​    总结：要避免异常，需要对数据库结构进行**`范式化设计`**。

  **3、节约数据存储空间。**

  **4、提高查询效率。**

### 1.2 数据库结构设计步骤

  1、**需求分析：**全面了解产品设计的存储需求、数据处理需求、数据安全性与完整性；

  2、**逻辑设计（`重要`）：**设计数据的逻辑存储结构。**数据实体之间的逻辑关系，解决数据冗余和数据维护异常。数据范式可以帮助我们设计；**

  3、**物理设计：**表结构设计，**存储引擎与列的数据类型**；

  4、**维护优化：****索引优化**、存储结构优化。

### 1.3 数据库范式设计与反范式化

  [**传送门：**数据库逻辑设计之三大范式通俗理解，一看就懂，书上说的太晦涩](https://segmentfault.com/a/1190000013695030)

### 1.4 物理设计

  ![clipboard.png](https://segmentfault.com/img/bV5Eux)

  ![clipboard.png](https://segmentfault.com/img/bV5EuI)

  ![clipboard.png](https://segmentfault.com/img/bV5EYl)

##### [**相关传送门：**MySQL中字段类型与合理的选择字段类型；int(11)最大长度是多少？，varchar最大长度是多少](https://segmentfault.com/a/1190000010012140)

## 二、高可用架构设计

  ![clipboard.png](https://segmentfault.com/img/bV5E2S)

  ![clipboard.png](https://segmentfault.com/img/bV5FbX)

### 2.1 读写分离

  ![clipboard.png](https://segmentfault.com/img/bV5FCy)

  [MaxScale：实现MySQL读写分离与负载均衡的中间件利器](https://link.segmentfault.com/?enc=IKBAufkKkV6fW9zUku45pA%3D%3D.CtcPND9nlhv%2FxSywQBLU2BBNtr8y6Za%2FA%2FYouwZQpbtREZdnLltsJj%2FMnXViw7Ct)

## 三、数据库索引优化（**`非常重要`**）

### 3.1 两种主要数据结构：B-tree和Hash

#### 3.1.1 B-tree结构

  ![clipboard.png](https://segmentfault.com/img/bV5FXp)

##### **B-tree索引的限制：**

  ![clipboard.png](https://segmentfault.com/img/bV5F3S)

#### 3.1.2 Hash结构

  ![clipboard.png](https://segmentfault.com/img/bV5FYs)

##### **Hash索引的限制：**

- Hash索引必须进行二次查找
- Hash索引无法用于排序
- Hash索引**不支持部分索引查找也不支持范围查找**
- Hash索引中Hash码的计算**可能存在Hash冲突，不适合重复值很高的列，如性别**，身份证比较合适。

#### 3.1.3 MySQL常见索引和各种索引区别

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

### 3.2 使用索引好处和索引缺陷

#### 3.2.1 为什么要使用索引

  **1、索引大大减少了存储引擎需要扫描的数据量；**

  **2、索引可以帮助我们进行排序以避免使用临时表；**

  **3、索引可以把随机I/O变为顺序I/O。**

#### 3.2.2 索引不是越多越好

  **1、索引会增加写操作的成本；**

  **2、太多的索引会增加查询优化器的选择时间。**

> 索引就好比一本书的目录，它会让你更快的找到内容，显然目录（索引）并不是越多越好，假如这本书1000页，而有500页是目录，它当然效率低，目录是要占纸张的,而索引是要占磁盘空间的。

### 3.3 索引优化策略

#### 3.3.1 索引列上不能使用表达式和函数

  ![clipboard.png](https://segmentfault.com/img/bV5LaJ)

#### 3.3.2 前缀索引和索引列的选择性

> `Innodb`索引列最大宽度为`667`个字节(`utf-8` 差不多`255`个字符),`MyIsam`索引类宽度最大为`1000`个字节，于是出现前缀索引，索引的选择性。

  对于列的值较长，比如`BLOB、TEXT、VARCHAR`，就必须建立前缀索引，即将值的前一部分作为索引。这样既可以节约空间，又可以提高查询效率。但无法使用前缀索引做 `ORDER BY` 和 `GROUP BY`，也无法使用前缀索引做覆盖扫描。

  **语法：** `ALTER TABLE table_name ADD KEY(column_name(prefix_length))`

  ![clipboard.png](https://segmentfault.com/img/bV5LkN)

##### **如何选择索引列的顺序：**

  1、经常会被使用到的列优先（**`选择性`**差的列不适合，如性别，**查询优化器**可能会认为全表扫描性能更好）；

  2、**`选择性高`**的列优先；

  3、宽度小的列优先（一页中存储的索引越多，降低`I/O`，查找越快）；

#### 3.3.3 组合/联合索引策略

  如果索引了多列，要遵守**`最左前缀`**法则。指的是查询从索引的最左前列开始并且**不跳过索引中的列**。
[**深入理解请移步：**最左前缀原理与相关优化](https://link.segmentfault.com/?enc=B%2B90U8mkrGs0pQjc3xDwuw%3D%3D.hzWUzVsErv8jvt8F%2BNlU4zwr9YuIDw%2FRDO0Q7bpmP6nIPutlClDP2XLCVMPVMPW%2Bn3RCH94ciuwRPTJvI%2BxvkA%3D%3D)

#### 3.3.4 覆盖索引策略

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

#### 3.3.5 SQL索引优化总结口诀（**`套路重点`**）

> **`全值匹配我最爱，最左前缀要遵守；`**
> **`带头大哥不能丢，中间兄弟不能断；`**
> **`索引列上不计算，范围之后全失效；`**
> **`LIKE百分写最右，覆盖索引不写 \*；`**
> **`不等空值还有or，索引失效要少用；`**
> **`字符单引不可丢，SQL高级也不难 ；`**

  [MySQL高级-索引优化](https://link.segmentfault.com/?enc=4OkMEqzoUu1Dfk%2Fojh8Jng%3D%3D.vTBwOSAsZeRJ9FYWvVAUpYOflPWr2aJzsvco0xzmgFzRIlsS3WHhfJNvbzD4w3nng%2Bm%2Bcyb1QeNDCIDKAB%2Bfmw%3D%3D)

### 3.4 使用索引来优化查询

#### 3.4.1 利用索引排序

  1、`group by` **`实质是先排序后分组`**，遵照索引的最佳左前缀。；

  2、索引中所有列的方向(**升序、降序**)和Order By子句完全一致；

  3、当无法使用索引列，增大`max_length_for_sort_data`参数的设置+增大`sort_buffer_size`参数的设置；

  4、如果最左列使用了范围，则排序会失效；

  5、`where` **高于**`having`，能写在`where`限定的条件就不要去`having`去限定了

### 3.5 索引的维护和优化

#### 3.5.1 删除重复索引

  ![clipboard.png](https://segmentfault.com/img/bV5PWy)

  **注：**主键约束相当于(唯一约束 + 非空约束)

  一张表中最多有一个主键约束,如果设置多个主键,就会出现如下提示：`Multiple primary key defined!!!`

#### 3.5.2 删除冗余索引

  ![clipboard.png](https://segmentfault.com/img/bV5PXH)

  **检查工具：****pt-duplicate-key-checker**

#### [**`扩展阅读： `**MySQL索引背后的数据结构及算法原理](https://link.segmentfault.com/?enc=Vs9P8l1A%2FD1PboOr%2BVAJtA%3D%3D.GV0uMrhEOoN0Cz2M8XQK%2B98KqYS4lQQ3iPteUaexN4M17cBbvcyhPtWXzz1W9KRqeukO7HpPIAopRl43lqJScQ%3D%3D)

> explain 查询计划
>
> Using where：表示优化器需要通过索引回表查询数据；
>
> Using index：表示直接访问索引就足够获取到所需要的数据，不需要通过索引回表，如覆盖索引；



作者：唐成勇

来源：https://segmentfault.com/a/1190000013672421