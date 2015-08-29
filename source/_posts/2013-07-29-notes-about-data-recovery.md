---
title: 记一次数据恢复过程
author: Jerry
comments: true
layout: post
permalink: /2013/07/notes-about-data-recovery/
categories:
  - Linux
  - Tech
tags:
  - Ext3
  - InnoDB
  - Linux
  - MySQL
---
上周因为停电（理由竟然是没交电费，无力吐槽）导致公司的一台测试服务器出现故障，无法启动。今天上午忙了半天，各种 Google 之后，终于搞定了，恢复了大部分数据，在这里记录一下。

<!--more-->

## 1. 文件系统的修复

一般出现掉电事故后，重启之后用各文件系统的 fsck （如 ext2 家族的 e2fsck） 命令检查一下文件系统就能搞定。但今天的情况更严重些，运行 e2fsck 直接报错：

<pre>root@livecd# e2fsck -f -v /dev/sda3
e2fsck 1.39 (29-May-2006)
Attempt to read block from filesystem resulted in short read while
trying to open /dev/sda3. Could this be a zero-length partition.
</pre>

各种 Google 之后发现造成这个问题的原因是文件系统的 Superblock 损坏了。解决办法是找到文件系统上的备份 Superblock 位置，具体做法有两种：用 dumpe2fs 或者 mke2fs -n。先试了下 dumpe2fs：

<pre>root@livecd# dumpe2fs /dev/sda3 | grep -i superblock
dumpe2fs 1.39 (29-May-2006)
Attempt to read block from filesystem resulted in short read while
trying to open /dev/sda3. Could this be a zero-length partition.
</pre>

还是报同样的错误，接着试了下 mke2fs -n，还好这下可以工作：

<pre>root@livecd# mke2fs -n /dev/sda3                           
mke2fs 1.39 (29-May-2006)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
17924096 inodes, 35822934 blocks
1791146 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=4294967296
1094 block groups
32768 blocks per group, 32768 fragments per group
16384 inodes per group
<span style="color: #0000ff;">Superblock backups stored on blocks: 32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 4096000, 7962624, 11239424, 20480000, 23887872 </span></pre>

其中标蓝色的部分就是文件系统上的 Superblock 备份，从中挑选一个作为 e2fsck -b 的参数，再进行一次 fsck 操作，结果顺利修复了文件系统。e2fsck 的 -b 选项是指定一个 Superblock 的位置，使用上面的命令中输出的任意一个都可以。可以看到 Superblock 的备份很多，一个不行可以用另一个，实践中所有 Superblock 都损坏的概率很小，因此还是比较安全的。

文件系统对于 Superblock 这类关键数据都会保留多个备份，出现故障的时候修复还是很方便的。我现在的疑问是所有这些 Superblock 备份都会保持和主 Superblock 同步吗？换句话说，备份是的时机是怎样的，如何才能知道哪份备份相对比较新点？先存疑，以后找到答案了再来更新。

## 2. MySQL 数据修复

文件系统抢救回来后能顺利进 OS 了，但试图启动服务的时候发现 MySQL 起不来了，一查日志，各种错误：

<pre>130729 11:18:51 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
130729 11:18:51 [Warning] You need to use --log-bin to make --binlog-format work.
130729 11:18:52 [Note] Plugin 'FEDERATED' is disabled.
130729 11:18:52 InnoDB: The InnoDB memory heap is disabled
130729 11:18:52 InnoDB: Mutexes and rw_locks use GCC atomic builtins
130729 11:18:52 InnoDB: Compressed tables use zlib 1.2.3
130729 11:18:52 InnoDB: Using Linux native AIO
130729 11:18:52 InnoDB: Initializing buffer pool, size = 2.0G
130729 11:18:52 InnoDB: Completed initialization of buffer pool
130729 11:18:52 InnoDB: highest supported file format is Barracuda.
InnoDB: Log scan progressed past the checkpoint lsn 492348697625
130729 11:18:52  InnoDB: Database was not shut down normally!
InnoDB: Starting crash recovery.
InnoDB: Reading tablespace information from the .ibd files...
InnoDB: Restoring possible half-written data pages from the doublewrite
InnoDB: buffer...
InnoDB: Doing recovery: scanned up to log sequence number 492353894193
130729 11:18:53  InnoDB: Error: page 45 log sequence number 492353898371
InnoDB: is in the future! Current system log sequence number 492353894193.
InnoDB: Your database may be corrupt or you may have copied the InnoDB
InnoDB: tablespace but not the InnoDB log files. See
InnoDB: http://dev.mysql.com/doc/refman/5.5/en/forcing-innodb-recovery.html
InnoDB: for more information.
130729 11:18:53  InnoDB: Error: page 147468 log sequence number 492353909330
InnoDB: is in the future! Current system log sequence number 492353894193.
InnoDB: Your database may be corrupt or you may have copied the InnoDB
InnoDB: tablespace but not the InnoDB log files. See
InnoDB: http://dev.mysql.com/doc/refman/5.5/en/forcing-innodb-recovery.html
InnoDB: for more information.
..... 省去错误日志若干
130729 11:18:53  InnoDB: Page checksum 1750256748, prior-to-4.0.14-form checksum 2917175473
InnoDB: stored checksum 1750256748, prior-to-4.0.14-form stored checksum 1125759973
InnoDB: Page lsn 114 2727622952, low 4 bytes of lsn at page end 2723860344
InnoDB: Page number (if stored to page already) 98337,
InnoDB: space id (if created with &gt;= MySQL-4.1.1 and stored already) 0
InnoDB: Page may be an update undo log page
InnoDB: Database page corruption on disk or a failed
InnoDB: file read of page 98337.
InnoDB: You may have to recover from a backup.
InnoDB: It is also possible that your operating
InnoDB: system has corrupted its own file cache
InnoDB: and rebooting your computer removes the
InnoDB: error.
InnoDB: If the corrupt page is an index page
InnoDB: you can also try to fix the corruption
InnoDB: by dumping, dropping, and reimporting
InnoDB: the corrupt table. You can use CHECK
InnoDB: TABLE to scan your table for corruption.
InnoDB: See also http://dev.mysql.com/doc/refman/5.5/en/forcing-innodb-recovery.html
InnoDB: about forcing recovery.
<span style="color: #ff0000;">InnoDB: Ending processing because of a corrupt database page.</span>
130729 11:18:53 mysqld_safe mysqld from pid file /var/lib/mysql/fly2save07.pid ended
</pre>

看来是若干数据页损坏导致数据库无法启动，根据日志中提到的 See also，去 <a href="http://dev.mysql.com/doc/refman/5.5/en/forcing-innodb-recovery.html" target="_blank">MySQL 官方文档</a>看了一下，发现了 innodb\_force\_recovery 这样一个选项。根据其说明，这个选项是为了解决当出现数据损坏时无法启动服务，或者无法执行指令 Dump 数据的问题。它的原理是停止 InnoDB 引擎的后台操作，从而避免这些操作可能触发的错误，进而尽量保证数据能被 Dump 出来。这个选项的取值为0（默认）到 6，其中 0 是关闭该选项，其他则由低到高，表示停掉的操作越来越多，可能丢失的数据也可能越来越多，同时能让服务起来以及 Dump 成功的可能性也更大。

*   `1` (`SRV_FORCE_IGNORE_CORRUPT`)
    
    Let the server run even if it detects a corrupt [page][1]. Try to make `SELECT * FROM <em><code>tbl_name`</em></code> jump over corrupt index records and pages, which helps in dumping tables.

*   `2` (`SRV_FORCE_NO_BACKGROUND`)
    
    Prevent the [master thread][2] from running. If a crash would occur during the [purge][3] operation, this recovery value prevents it.

*   `3` (`SRV_FORCE_NO_TRX_UNDO`)
    
    Do not run transaction [rollbacks][4] after [crash recovery][5].

*   `4` (`SRV_FORCE_NO_IBUF_MERGE`)
    
    Prevent [insert buffer][6] merge operations. If they would cause a crash, do not do them. Do not calculate table[statistics][7].

*   `5` (`SRV_FORCE_NO_UNDO_LOG_SCAN`)
    
    Do not look at [undo logs][8] when starting the database: `InnoDB` treats even incomplete transactions as committed.

*   `6` (`SRV_FORCE_NO_LOG_REDO`)
    
    Do not do the [redo log][9] roll-forward in connection with recovery.

根据文档里的描述，开到 4 能恢复绝大部分的数据，但更高的级别就可能会一定的数据丢失。很可惜，我开到 4 依然无法正常启动服务，所以改到了 5，之后正常启动了服务。然后用 mysqldump -A 把所有的数据库都备份了下来，之后 drop 了所有的 database，删掉了 ib 开头的若干数据和日志文件，然后重启服务，用 mysql 命令把之前导出的数据再次导入就搞定了。

肯定有部分数据在这个过程中丢失了，但还好我们这只是个测试服务器。

## 3. 参考

*   <a style="line-height: 1.714285714; font-size: 1rem;" href="http://forums.linuxmint.com/viewtopic.php?f=49&t=74362" target="_blank">Hard disk failure &#8211; Short Read &#8211; Superblock Issue</a>
*   <a style="line-height: 1.714285714; font-size: 1rem;" href="http://www.cyberciti.biz/tips/surviving-a-linux-filesystem-failures.html" target="_blank">Surviving a Linux Filesystem Failures</a>
*   <a style="line-height: 1.714285714; font-size: 1rem;" href="http://dev.mysql.com/doc/refman/5.5/en/forcing-innodb-recovery.html" target="_blank">Forcing InnoDB Recovery</a>
*   [MySQL Won’t Start – InnoDB Corruption and Recovery][10]
*   <a style="line-height: 1.714285714; font-size: 1rem;" href="http://www.percona.com/software/mysql-innodb-data-recovery-tools" target="_blank">Percona Data Recovery Tool for InnoDB</a><span style="line-height: 1.714285714; font-size: 1rem;"> Percona 出品的看起来很牛叉的 InnoDB 恢复工具，貌似是在上面说的办法都不工作的情况下使用的</span>

&nbsp;

 [1]: http://dev.mysql.com/doc/refman/5.5/en/glossary.html#glos_page "page"
 [2]: http://dev.mysql.com/doc/refman/5.5/en/glossary.html#glos_master_thread "master thread"
 [3]: http://dev.mysql.com/doc/refman/5.5/en/glossary.html#glos_purge "purge"
 [4]: http://dev.mysql.com/doc/refman/5.5/en/glossary.html#glos_rollback "rollback"
 [5]: http://dev.mysql.com/doc/refman/5.5/en/glossary.html#glos_crash_recovery "crash recovery"
 [6]: http://dev.mysql.com/doc/refman/5.5/en/glossary.html#glos_insert_buffer "insert buffer"
 [7]: http://dev.mysql.com/doc/refman/5.5/en/glossary.html#glos_statistics "statistics"
 [8]: http://dev.mysql.com/doc/refman/5.5/en/glossary.html#glos_undo_log "undo log"
 [9]: http://dev.mysql.com/doc/refman/5.5/en/glossary.html#glos_redo_log "redo log"
 [10]: http://chepri.com/mysql-innodb-corruption-and-recovery/
