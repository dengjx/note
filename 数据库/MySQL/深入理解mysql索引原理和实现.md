# MySQL索引
MySQL索引的建立对于MySQL的高效运行是很重要的，索引可以大大提高MySQL的检索速度。

索引分单列索引和组合索引。单列索引，即一个索引只包含单个列，一个表可以有多个单列索引，但这不是组合索引。组合索引，即一个索引包含多个列。

创建索引时，你需要确保该索引是应用在	SQL 查询语句的条件(一般作为 WHERE 子句的条件)。

实际上，索引也是一张表，该表保存了主键与索引字段，并指向实体表的记录。

上面都在说使用索引的好处，但过多的使用索引将会造成滥用。因此索引也会有它的缺点：虽然索引大大提高了查询速度，同时却会降低更新表的速度，如对表进行INSERT、UPDATE和DELETE。因为更新表时，MySQL不仅要保存数据，还要保存一下索引文件。

建立索引会占用磁盘空间的索引文件。

**总的来说：** 索引在数据库中的作用是快速找出某个列中一个特定值得行，如果不用索引的话，MySQL必须从第一行记录开始遍历到相关行，表越大，时间越久，若采用索引，则可以快速定位某个位置去搜索数据文件，索引对于优化数据库查询有着不可替代的作用。但是，索引不是什么时候都需要建立，因为创建索引和维护索引都需要时间，而且，索引会占用磁盘空间。

## 索引的优点

1) 通过创建唯一索引，可以保证数据库每一行数据的唯一性
2) 可以大大提高查询速度
3) 可以加速表与表的连接
4) 可以显著的减少查询中分组和排序的时间。

## 索引的缺点

1) 创建索引和维护索引需要时间，而且数据量越大时间越长
2) 创建索引需要占据磁盘的空间，如果有大量的索引，可能比数据文件更快达到最大文件尺寸
3) 当对表中的数据进行增加，修改，删除的时候，索引也要同时进行维护，降低了数据的维护速度

## 索引的设计原则

1. 不是越多越好
2. 常更新的表越少越好
3. 数据量小的表最好不要建立索引
4. 不同的值比较多的列才需要建立索引
5. 某种数据本身具备唯一性的时候，建立唯一性索引，可以保证定义的列的数据完整性，以提高查询熟度
6. 频繁进行排序或分组的列(group by或者是order by)可以建立索引，提高搜索速度
7. 经常用于查询条件的字段应该建立索引

==**注意：索引需要占用磁盘空间，因此在创建索引时要考虑到磁盘空间是否足够；创建索引时需要对表加锁，因此实际操作中需要在业务空闲期间进行**==

## 常用索引的分类

1. **普通索引(Normal)：** 基本索引类型，允许在定义索引的列里插入空值或重复值。
2. **唯一索引(Unique)：** 索引列值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。主键索引是一种特殊的唯一索引，不允许有空值。
3. **单列索引：** 只包含一个列的索引，一个表中可以有多个。
4. **组合索引：** 包含多个列的索引，查询条件包含这些列的最左边的字段的时候，索引就会被引用，遵循最左缀原则。
5. **全文索引(Full Text)：** 在定义的值中支持全文查找，允许空值和重复值，可以在CHAR，VARCHAR或者TEXT字段类型上创建，仅支持MyISAM存储引擎。(注意：MySQL5.7开始支持InnoDB引擎)
6. **主键索引：** 是一种特殊的唯一索引，一个表只能有一个主键，不允许有空值。

### 普通索引
最基本的索引，没有任何限制。创建方式如下：

1. 直接创建索引

```sql
CREATE INDEX index_name ON table(column(length))

#如果是CHAR，VARCHAR类型，length可以小于字段实际长度；如果是BLOB和TEXT类型，必须指定 length。
```

2. 修改表结构的方式添加索引

```sql
ALTER TABLE table_name ADD INDEX index_name ON (column(length))
```

3. 创建表的时候同时创建索引

```sql
CREATE TABLE `table` (
    `id` int(11) NOT NULL AUTO_INCREMENT ,
    `title` char(255) CHARACTER NOT NULL ,
    `content` text CHARACTER NULL ,
    `time` int(10) NULL DEFAULT NULL ,
    PRIMARY KEY (`id`),
    INDEX index_name (title(length))
)
```

删除索引语法如下：

```sql
DROP INDEX index_name ON table
```

### 唯一索引
索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。创建方式如下：

1. 创建唯一索引

```sql
CREATE UNIQUE INDEX indexName ON table(column(length))
```

2. 修改表结构

```sql
ALTER TABLE table_name ADD UNIQUE indexName ON (column(length))
```

3. 创建表的时候直接指定

```sql
CREATE TABLE `table` (
    `id` int(11) NOT NULL AUTO_INCREMENT ,
    `title` char(255) CHARACTER NOT NULL ,
    `content` text CHARACTER NULL ,
    `time` int(10) NULL DEFAULT NULL ,
    UNIQUE indexName (title(length))
);
```

### 单列索引
只包含一个字段的索引叫做单列索引，包含两个或以上字段的索引叫做复合索引（或组合索引）。

单列索引的建立方法和只用单列作为索引字段的索引相同，在这不再细写。

### 组合索引
指多个字段上创建的索引，只有在查询条件中使用了创建索引时的第一个字段，索引才会被使用。使用组合索引时遵循最左前缀集合。

```sql
ALTER TABLE `table` ADD INDEX name_city_age (name,city,age); 
```

注意如果要使用建立的组合索引，查询条件需要这样使用：

```sql
SELECT * FROM `table` WHERE name='abc' AND city='xxx';

# 以下的语句不会使用组合索引
SELECT * FROM `table` WHERE city='xxx';
```

### 全文索引
主要用来查找文本中的关键字，而不是直接与索引中的值相比较。fulltext索引跟其它索引大不相同，它更像是一个搜索引擎，而不是简单的where语句的参数匹配。fulltext索引配合match against操作使用，而不是一般的where语句加like。它可以在 `CREATE TABLE`，`ALTER TABLE`，`CREATE INDEX` 使用，不过目前只有 `CHAR`、`VARCHAR`、`TEXT`  列上可以创建全文索引。值得一提的是，在数据量较大时候，现将数据放入一个没有全局索引的表中，然后再用CREATE index创建fulltext索引，要比先为一张表建立fulltext然后再将数据写入的速度快很多。

1. 创建表的适合添加全文索引

```sql
CREATE TABLE `table` (
    `id` int(11) NOT NULL AUTO_INCREMENT ,
    `title` char(255) CHARACTER NOT NULL ,
    `content` text CHARACTER NULL ,
    `time` int(10) NULL DEFAULT NULL ,
    PRIMARY KEY (`id`),
    FULLTEXT (content)
);
```

2. 修改表结构添加全文索引

```sql
ALTER TABLE article ADD FULLTEXT index_content(content)
```

3. 直接创建索引

```sql
CREATE FULLTEXT INDEX index_content ON article(content)
```

## 注意事项
使用索引时，有以下一些技巧和注意事项：

1. **索引不会包含有null值的列**  
只要列中包含有null值都将不会被包含在索引中，复合索引中只要有一列含有null值，那么这一列对于此复合索引就是无效的。所以我们在数据库设计时不要让字段的默认值为null。
2. **使用短索引**  
对串列进行索引，如果可能应该指定一个前缀长度。例如，如果有一个char(255)的列，如果在前10个或20个字符内，多数值是惟一的，那么就不要对整个列进行索引。短索引不仅可以提高查询速度而且可以节省磁盘空间和I/O操作。
3. **索引列排序**  
查询只使用一个索引，因此如果where子句中已经使用了索引的话，那么order by中的列是不会使用索引的。因此数据库默认排序可以符合要求的情况下不要使用排序操作；尽量不要包含多个列的排序，如果需要最好给这些列创建复合索引。
4. **like语句操作**  
一般情况下不推荐使用like操作，如果非使用不可，如何使用也是一个问题。like “%aaa%” 不会使用索引而like “aaa%”可以使用索引。
5. **不要在列上进行运算**  
这将导致索引失效而进行全表扫描，例如：

```sql
SELECT * FROM table_name WHERE YEAR(column_name)<2017;
```

6. **不使用not in和<>操作**

## 其他
MySQL支持诸多存储引擎，而各种存储引擎对索引的支持也各不相同，因此MySQL数据库支持多种索引类型，如BTree索引，B+Tree索引，哈希索引，全文索引等等。

### 哈希索引
只有memory（内存）存储引擎支持哈希索引，哈希索引用索引列的值计算该值的hashCode，然后在hashCode相应的位置存执该值所在行数据的物理位置，因为使用散列算法，因此访问速度非常快，但是一个值只能对应一个hashCode，而且是散列的分布方式，因此哈希索引不支持范围查找和排序的功能。

### 全文索引
FULLTEXT（全文）索引，仅可用于MyISAM和InnoDB，针对较大的数据，生成全文索引非常的消耗时间和空间。对于文本的大对象，或者较大的CHAR类型的数据，如果使用普通索引，那么匹配文本前几个字符还是可行的，但是想要匹配文本中间的几个单词，那么就要使用LIKE %word%来匹配，这样需要很长的时间来处理，响应时间会大大增加，这种情况，就可使用时FULLTEXT索引了，在生成FULLTEXT索引时，会为文本生成一份单词的清单，在索引时及根据这个单词的清单来索引。FULLTEXT可以在创建表的时候创建，也可以在需要的时候用ALTER或者CREATE INDEX来添加

==全文索引的查询也有自己特殊的语法，而不能使用LIKE %查询字符串%的模糊查询语法：==

```sql
SELECT * FROM table_name MATCH(ft_index) AGAINST('查询字符串');
```

注意：

+ 对于较大的数据集，把数据添加到一个没有FULLTEXT索引的表，然后添加FULLTEXT索引的速度比把数据添加到一个已经有FULLTEXT索引的表快。
+ 5.6版本前的MySQL自带的全文索引只能用于MyISAM存储引擎，如果是其它数据引擎，那么全文索引不会生效。5.6版本之后InnoDB存储引擎开始支持全文索引
+ 在MySQL中，全文索引支队英文有用，目前对中文还不支持。5.7版本之后通过使用ngram插件开始支持中文。
+ 在MySQL中，如果检索的字符串太短则无法检索得到预期的结果，检索的字符串长度至少为4字节，此外，如果检索的字符包括停止词，那么停止词会被忽略。
+ 更深入的理解参考文章：[全文索引的深入理解](https://www.cnblogs.com/dreamworlds/p/5462018.html)

### BTree索引和B+Tree索引

+ BTree的查询、插入、删除过程可以参考：https://blog.csdn.net/endlu/article/details/51720299
+ B+Tree是BTree的一个变种

**B+Tree对比BTree的优点：**

1、磁盘读写代价更低

一般来说B+Tree比BTree更适合实现外存的索引结构，因为存储引擎的设计专家巧妙的利用了外存（磁盘）的存储结构，即磁盘的最小存储单位是扇区（sector），而操作系统的块（block）通常是整数倍的sector，操作系统以页（page）为单位管理内存，一页（page）通常默认为4K，数据库的页通常设置为操作系统页的整数倍，因此索引结构的节点被设计为一个页的大小，然后利用外存的“预读取”原则，每次读取的时候，把整个节点的数据读取到内存中，然后在内存中查找，已知内存的读取速度是外存读取I/O速度的几百倍，那么提升查找速度的关键就在于尽可能少的磁盘I/O，那么可以知道，每个节点中的key个数越多，那么树的高度越小，需要I/O的次数越少，因此一般来说B+Tree比BTree更快，因为B+Tree的非叶节点中不存储data，就可以存储更多的key。

2、查询速度更稳定

由于B+Tree非叶子节点不存储数据（data），因此所有的数据都要查询至叶子节点，而叶子节点的高度都是相同的，因此所有数据的查询速度都是一样的。

更多操作系统内容参考：

[硬盘结构](https://blog.csdn.net/william_munch/article/details/84347788)

[扇区、块、簇、页的区别](http://www.cnblogs.com/johnnyflute/archive/2013/12/28/3495442.html)

[操作系统层优化（进阶，初学不用看）](https://www.jianshu.com/p/f9ac7d87f656)

# 参考
[深入理解MySQL索引原理和实现](https://blog.csdn.net/tongdanping/article/details/79878302#3、BTree索引和B%2BTree索引)


