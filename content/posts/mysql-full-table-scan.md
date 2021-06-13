+++
title = "Mysql 全表扫描"
date = 2021-06-12T23:15:49+08:00
images = []
tags = ["mysql"]
categories = []
draft = false

+++

## 背景

​		技术栈是Django ORM + MySQL

​		已有的项目，历史原因，Django orm的model里充满了`index=True, null=True`,  并且各种操作导致全表扫描特别多。

## 从InnoDB的索引说起

​		在 InnoDB 中，表都是根据主键顺序以索引的形式存放的，这种存储方式的表称为索引组织表。InnoDB 使用了 B+ 树索引模型，所以数据都是存储在 B+ 树中的。

 		每一个索引在InnoDB里对应一棵B+树，所以索引就是在利用树的结构加速扫描，索引分为主键索引(聚簇索引)和普通索引（二级索引）。行为上的差别是，前者只需要搜索主键B+树，后者需要先搜索该二级索引B+树然后得到ID再搜索主键B+树，也就是普通索引需要回表（查第二次）。更多的细节就不再赘述，那个不是本文的重点。

## 索引的建立

​		基于回表这个操作，对于常用的值，为了减少可以利用覆盖索引的方式，建立联合索引，并且通过规划联合索引的顺序来减少索引数， 不需要回表来提高性能，当然会造成索引树的维护问题，这个需要权衡。

​		我们的业务表里的很多字段是字符串类型的，并且很多做了联合索引，这里就存在一个问题了。索引的值越长，占用的磁盘空间就越大，相同的数据页能放下的索引值就越少，搜索的效率也就会越低。

更糟糕的是，很多列是`null=True` 这就引申出另外一个问题了，索引以及联合索引是否能为null。

​		关于null是否能作为索引列的值，或者换个说法，索引列是否能为null，先上官方文档：

 ```
 You can add an index on a column that can have NULL values if you are using the MyISAM, InnoDB, or MEMORY storage engine. Otherwise, you must declare an indexed column NOT NULL, and you cannot insert NULL into the column.
 ```

文档说索引在引擎是InnoDB，MyISAM，MEMORY的情况下是可以为null的，然后还有如[这里所述](https://dev.mysql.com/doc/refman/5.7/en/is-null-optimization.html) 这种索引能够走的前提是你采用is null来判断。最后，如果不采用覆盖索引的方式，`select * from xxx` 的话，那么如果数据量少，mysql的优化器可能采用全表扫描。

 		这是我们代码里的另一类问题，因为各种原因导致的不走索引。

### 全表扫描的场景

* 在 where 子句中使用 !=  操作符，引擎放弃使用索引而进行全表扫描

* 在 where 子句中使用 or 来连接条件，如果一个字段有索引，一个字段没有索引，将导致引擎放弃使用索引而进行全表扫描

* in 和 not in 
* like模糊查询
* 在where子句中对字段进行函数操作
* select * from xx
* 联合索引中的没利用最左前缀匹配原则

由于情况太多。只列举了我们代码里出现情况。

## 总结

​		当发现你的代码里出现以上情况之一建议优化，但是不要盲目优化，应该先采profiler工具，以Django为例是增加Django-silk之后， 获取到数据，权衡利弊之后再去做调整。
