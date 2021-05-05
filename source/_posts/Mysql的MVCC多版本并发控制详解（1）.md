---
title: Mysql的MVCC多版本并发控制详解（1）
date: 2021-05-04 18:12:50
tags:
 - Mysql
 - 校招
categories:
 - 计算机基础
 - Mysql
keywords: "MVCC"
top_img: /img/2.jpg
cover: /img/post/cover/4.jpg
---





## 目录

- 背景

- 前提回顾

- - 什么是MVCC
  - 什么是当前读和快照读
  - 当前读，快照读和MVCC关系
  - MVCC，乐观锁，悲观锁关系

- MVCC实现原理

- 整体流程

- 彩蛋

- RR和RC隔离级别下的InnoDB快照读有什么区别

- 闲聊

- 欢迎加入我的公众号 一起pk大厂

## 背景

> 写<<校招MySQL那些事>>系列文章，一方面帮助在校大学生可以提早知道大厂的面试过程，了解大厂究竟需要什么样人才，自己该如何进行准备，全力以赴；另一方面也是自我巩固知识，因为我也是从校招进入大厂的，自己也经历过痛并快乐的过程，也希望可以把我自己的经验，都吐露出来。
> 接下来我会写很多关于校招文章，比如<<校招Redis那些事>>，<<校招go那些事>>，<<校招kafka那些事>>等等，敬请期待。

## 前提回顾

### **什么是MVCC**

> 多版本并发控制(MVCC)，是一种用来解决读-写冲突的无锁并发控制，也就是为事务分配单向增长的时间戳，为每个修改保存一个版本，版本与事务戳关联，读操作只读事务开始前的数据库的快照。这样在读操作的时候不会阻塞写操作，写操作不会阻塞读操作的同时，也避免了脏读和不可重复读。

### 什么是当前读和快照读

在学MVCC多版本并发控制之前，先了解一下什么是mysql InnoDB下的当前读和快照读？

- 当前读

当前读指的就是它读取的记录是最新版本的。由于它要读取记录的版本是最新版本，所以读取时须保证其他事务不能修改当前记录，因此需要对读取的记录进行加锁。

- 快照读

快照读可以理解为不加锁的select操作就是快照读；快照读的前提是隔离级别不是串行级别，因为在串行隔离级别，快照读可以理解为当前读；快照读的出现，主要解决了在不加锁的情况下也可以进行读取，降低了锁开销；它的实现是基础多版本并发控制，即MVCC；由于它是基于多版本并发控制，所以使用快照读读取的记录并不一定是最新记录。

### 当前读，快照读和MVCC关系

- 准确的说，MVCC主要基于"维护一条数据的多个版本，进而保证在读操作的同时不会阻塞写操作，写操作的同时也不会阻塞读操作"
- 快照读其实就是MVCC的一种体现方式，进行非阻塞读。而相对而言，当前读就是悲观锁的体现，每次进行查询操作时，mysql都认为其是不安全操作，为其加锁保证安全，但每次读取的数据为最新数据
- MCC模型在mysql中的具体实现主要是由隐藏字段，undolog，read-view等去完成的，具体实现逻辑看下面的MVCC实现原理

### MVCC，乐观锁，悲观锁关系

- MVCC+乐观锁

MVCC解决读写冲突，乐观锁解决写写冲突

- MVCC+悲观锁

MVCC解决读写冲突，悲观锁解决写写冲突

这样组合的方式最大程度的提高数据库并发性能，解决读写冲突以及写写冲突所导致的问题。

## MVCC实现原理

MVCC就是多版本并发控制，用来解决读写冲突。它主要由隐藏字段，undolog日志，read-view来配合完成的。所以先看看这几个属性，理解其含义，便于理解MVCC实现原理。

### 隐藏字段

隐藏字段中除了咱们自定义的字段外，还隐含着其他属性字段，是系统默认给加上去的，比如roll_pointer，trx_id等字段。

- roll_pointer

回滚指针，指向这条记录的上一个版本

- trx_id

事务ID，记录创建/修改这条记录的事务ID，用于版本比较，从而找到快照

| name | age  | trx_id | roll_pointer(回滚指针) |
| ---- | ---- | ------ | ---------------------- |
| 迈莫 | 22   | 1      | 0x1654u                |

如表中所示，name和age属性为用户自定义属性，而trx_id和roll_pointer就表示隐藏属性，数据库默认添加。在这表中，trx_id表示操作这条记录的事务ID，roll_pointer是回滚指针，表示指向上一个版本，一般配合undolog日志使用

### undolog日志

undolog日志存储某条记录的所有操作，以链表方式将各个版本进行串联起来

### 举例

**第一步：**比如有个事务1向person表中插入一条数据，记录如下，name为迈莫，age为22，事务ID为1，回滚指针假设为null，如下图所示

![img](https://pic1.zhimg.com/80/v2-f1b2a06e885b3014ee8f80dd0c4955a8_720w.png)

**第二步：**此时又来一个事务2，对该记录的age值进行修改，修改为23

- 首先将事务1的操作记录添加到undolog日志中
- 将事务2的操作记录作为数据的最新记录
- 将事务2中隐藏字段roll_pointer(回滚指针)指向事务1，进行串联

![img](https://pic3.zhimg.com/80/v2-8e51a58627ab48a08405cbcd114d9d1a_720w.jpg)

**第三步：**又有一个事务3，对该记录的name值进行操作，修改为memolei

- 首先将事务2的操作记录迁移到undolog日志中
- 将事务3的操作记录作为数据的最新记录
- 将事务3中隐藏字段roll_pointer(回滚指针)指向事务2，进行串联

![img](https://pic2.zhimg.com/80/v2-13407997de5f6f0694fe4a7f31bfc291_720w.jpg)

从上面图可以看到，每当有事务对该数据进行操作时，首先会将操作前最新数据迁移到undolog日志中，将当前事务操作的记录作为最新数据，并且将隐藏字段roll_pointer指向上一个版本

### read-view(一致性视图)

当事务第一次执行查询sql时会生成一致性视图read-view，它由执行查询时所有未提交事务ID数组(数组里最小的ID为min_id)和已创建的最大事务id(max_id)组成，查询的数据结果需要跟read-view做比较从而得到快照结果

**版本链比对规则：**

- 如果落在绿色部分(trx_id<min_id)，表示这个版本是已提交的事物生成的，这个数据是可见的
- 如果落在红色部分(trx_id>max)，表示这个版本是由将来启动的事务生成的，是肯定不可见的
- 如果落在黄色部分(min_id<=trx_id<=max_id)，那就包括两种情况

\1. 若row的trx_id在数组中，表示这个版本是由还没提交的事务生成的，不可见，当前自己的事务是可见的

\2. 若row的trx_id不在数组中，表示这个版本是已经提交的事务生成的，是可见的

## 整体流程

### 前提条件

![img](https://pic3.zhimg.com/80/v2-59ffcdf659ad8df88f10f82936c91a7e_720w.jpg)

如图所示，有四个事务，分别为translation2，translation3，translation4，translation5；translation2，translation3，translation4分别对数据库进行修改操作，translation5进行查询操作。

### 执行过程

\1. 当translation2对persion表中ID为1的数据进行修改时，首先会将该记录操作作为最新记录，并且roll_pointer指向上一个版本，但尤于translation2未commit，所以translation2仍然为未提交事务

![img](https://pic1.zhimg.com/80/v2-9dab91109e2d3001dc24d0f8c308cff8_720w.jpg)

\2. 当translation3对persion表中ID为1的数据进行修改时，首先会将该记录操作作为最新记录，并且roll_pointer指向上一个版本，但尤于translation3未commit，所以translation3仍然为未提交事务

![img](https://pic3.zhimg.com/80/v2-5bb36e144f837743ad5753a0d7ecdd5e_720w.jpg)

\3. 当translation4对persion表中ID为1的数据进行修改时，首先会将该记录操作作为最新记录，并且roll_pointer指向上一个版本，但尤于translation4已commit，所以translation4为已提交事务



![img](https://pic3.zhimg.com/80/v2-94156aa707ca922f04c03e69fb98c096_720w.jpg)



\4. translation5进行select语句查询时，并且由于其是第一次创建查询语句，所以会创建read-view一致性视图。read-view一致性视图是由未提交事务ID数组和最大事务ID组成，由表中可知，当translation5进行查询时，translation2和translation3都为未提交事务，translation4为已提交事务，所以，read-view中未提交事务ID数组由translation2和translation3组成，最大事务ID为translation4，所以max_id为translation5，min_id为translation2

![img](https://pic3.zhimg.com/80/v2-9684831facdd335f95c6d698a9d26d76_720w.jpg)

\5. 接下来的话，就需要进行版本链比较(若不记得版本链比较规则回去温习一下)，由于read-view由未提交事务数组[1,2]和最大事务ID3组成。由于当前最新记录事务ID为4,4大于最大事务ID3，所以无法查看；进行回溯，到undolog日志查询，事务3在未提交事务数组中，也就是中间这一段，由于事务三不在未提交事务数组中，说明事务3为已提交事务，因此是可见的，最终结果为name="memolei"

![img](https://pic3.zhimg.com/80/v2-94156aa707ca922f04c03e69fb98c096_720w.jpg)

到此，MVCC整个流程处理完了，一遍生，二遍熟，温故而知新。

## 彩蛋

### RR和RC隔离级别下的InnoDB快照读有什么区别

- RR隔离级别下，当事务第一次进行快照读，仅此一次创建read-view视图，所以read-view中的未提交事务数组和最大事务ID始终保持不变，因此每次读取时只会读取事务之前的数据
- 而在RC隔离级别下，每次事务进行快照读时，都会生成read-view视图，导致在RC隔离级别下事务可以看到其他事务修改后的数据，这也是导致不可重复的原因

总之在RC隔离级别下，是每个快照读都会生成并获取最新的Read View；而在RR隔离级别下，则是同一个事务中的第一个快照读才会创建Read View, 之后的快照读获取的都是同一个Read View。