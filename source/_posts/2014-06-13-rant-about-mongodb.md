---
layout: post
title: 对 MongoDB 的一些吐槽
date: 2014-06-13 17:32:02 +0800
comments: true
categories:
  - Tech
tags:
  - MongoDB
keywords: MongoDB, NoSQL, Database
---

首先声明，我们对 MongoDB 的使用谈不上深入，我更是没有太多经验，所以下
文的这些吐槽不一定都是对的。下面提到的问题，有的可能是我们的使用方式不
对；有的可能是没找到合适的解决方案；有的可能根本就不是适合 MongoDB 的
使用场景。总之，希望这些不会成为你使用/不使用 MongoDB 的参考，如果你是
MongoDB 高手，也请不吝赐教。

## 0. 背景

我们使用 MongoDB 是为了存储一些机票检索的历史数据以便于后期的分析，是
典型的 OLAP 类的使用方式。数据一旦写入就不会有 Update，只有读取。单条
很庞大，经过精简后的纯 JSON 大小也有几十上百 K（我怀疑 MongoDB 并不适
合存储太大的 Document ）。

我们初期使用了三个节点部署 MongoDB，组成一个 Replication Set，其中两个
实际承载数据，另一个充当 Arbiter，只投票。

后来其中的 Secondary 因为严重的硬件故障送修了，Replication Set 实际上变
成了单节点在运行。修复好了以后 Secondary 跟不上 Primary 了，重建
Secondary 也因为一些原因（后面详述）没能搞定，只能作罢。

现在的状况是原 Master 单节点运行（Arbiter 等于没用），但磁盘分区快被占
满，急需进行迁移；Secondary 是全新安装的实例，我们打算直接切换，跑两个
单实例，不再做 Replication Set。

<!--more-->

## 1. 可怕的磁盘空间占用

前面提到我们的单个 Document 就有几十上百 K，而每天有 100 到 200 万这样
的数据产生。目前数据库里已经有了上亿条记录，把一个 6T 的分区占用了 5T
多。

如果说 MongoDB 因为其基于 Memory Mapped File 的工作原理，占用磁盘空间
大，那没有简单的办法释放磁盘空间是什么情况？

MongoDB 从操作系统申请的磁盘空间是不会主动释放的，唯一的办法是执行
`repairDatabase`，但 `repairDatabase` 需要有
[相当于数据量的空闲磁盘空间才能进行](http://docs.mongodb.org/manual/reference/command/repairDatabase/#dbcmd.repairDatabase)
：

> `repairDatabase` requires free disk space equal to the size of your
> current data set plus 2 gigabytes.

如果你像我们这么倒霉，MongoDB 占用了 5T，实际数据占用了 3.5T，但所有分
区的剩余空间加起来都不到 3T 的话，那么恭喜，你没法通过 `repairDatabase`
把它多占用的那 1个多 T 空间释放出来了。

什么，你说 `compact` ？人家说了，`compact` 根本就
[不会释放磁盘空间](http://docs.mongodb.org/manual/reference/command/compact/#dbcmd.compact)
：

> `compact` requires up to 2 gigabytes of additional disk space while
> running. Unlike `repairDatabase`, compact does not free space on the
> file system.

现在我们已经切换到另一台服务器了，所以老的实例再也不会增加数据了，但这
1T 的空间，我要怎么才能要回来啊我靠，同服务器上的 MySQL 还要用呢……

## 2. 蛋疼的 Initial Sync

前面提到了我们的 Secondary 机器曾被送去维修，回来之后数据差别太大导致跟
不上 Primary 了，需要重新初始化。这个也十分蛋疼，因为它实在太慢了，慢到
官方都
[*不推荐*](http://docs.mongodb.org/manual/tutorial/restore-replica-set-from-backup/)
这么干。因为 Initial Sync 太慢，所以可能 Sync 完了，从还是落后主太多。

> If your database is large, initial sync can take a long time to
> complete. For large databases, it might be preferable to copy the
> database files onto each host.

官方推荐的方法是直接拷贝数据文件，因为要保证数据文件是一致的。最好是结
合 LVM Snapshot 之类的机制，先做个快照，然后再从快照上拷贝，这样可以保
证数据的一致性。但我们很不幸地 *没有* 使用 LVM，而是直接使用的 ext4，我
Google 了半天也没找到直接对 ext4 做快照的办法（难道是不支持？）。

好吧，看来我只有停掉服务来拷贝了。算了一下，接近 6T 的数据通过千兆网卡
可能要传 24 小时左右，Fuck…… 仔细一想，这些数据文件有很多应该是还未
被使用的吧，实际数据不应该有那么大，有没有别的办法呢，比如用 `mongodump`？

然后发现，`mongodump` 只能用来初始化一个主，而不能用来初始化 Repl Set 里
的从……我就呵呵了。

所以我到现在也不知道，像我们这样一个目前只有一个 Primary，且数据量巨大
的 Repl Set，要怎么添加一个从。

## 3. 极差的性能

在我们的这种使用场景下，MongoDB 的性能明显达不到网上各种 Benchmark 宣
称的那么好。从日志里看插入一条数据的耗时，从 100ms 到 1000ms 不等。服
务器上的日志为证：

```
[conn205885] insert queryhistory.queryDetail ninserted:1 keyUpdates:0 locks(micros) w:18173 1930ms
[conn205882] insert queryhistory.queryDetail ninserted:1 keyUpdates:0 locks(micros) w:5385 1841ms
[conn205879] insert queryhistory.queryDetail ninserted:1 keyUpdates:0 locks(micros) w:37265 1869ms
[conn205902] insert queryhistory.queryDetail ninserted:1 keyUpdates:0 locks(micros) w:566 1870ms
[conn205900] insert queryhistory.queryDetail ninserted:1 keyUpdates:0 locks(micros) w:71031 294ms
[conn205880] insert queryhistory.queryDetail ninserted:1 keyUpdates:0 locks(micros) w:15367 335ms
[conn205911] insert queryhistory.queryDetail ninserted:1 keyUpdates:0 locks(micros) w:1464 337ms
[conn205894] insert queryhistory.queryDetail ninserted:1 keyUpdates:0 locks(micros) w:20006 348ms
```

查询的性能则更是让人蛋疼，一个很简单的都能命中索引的查询也经常会耗时很
久，极端的情况下几分钟都出不来结果。

当然我怀疑性能差的原因与我们的单个文档过大有关，可能 MongoDB 并不适合
这种大 Document 的用法。

## 4. 总结

还是那句话，我的这些吐槽不一定都对，只是我在使用过程中发现的一些不爽之
处而已。回过头来，我觉得当初不应该选择 MongoDB，它并不适合我们的使用场
景。如果重新选择，我觉得 MySQL 加上某种分表的机制都可能足够应付我们的数
据量了，或者直接研究一下 Spark + Shark 这类分布式方案。有时间和机会我真
想在这个方向好好探索一下。
