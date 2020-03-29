---
title: MySQL：buffer_size相关配置
date: 2020-03-01 21:02:48
tags:
    - MySQL
categories:
    - MySQL
---

MySQL中涉及到size的缓冲配置项有不少，有些配置对性能的影响还是较大的，这里简单介绍一下。

<!-- more -->

首先查一下带{% label info@buffer_size%}相关的配置项，如下：

```
mysql> SHOW VARIABLES LIKE '%buffer_size%';
+-------------------------+----------+
| Variable_name           | Value    |
+-------------------------+----------+
| bulk_insert_buffer_size | 8388608  |
| innodb_log_buffer_size  | 16777216 | 
| innodb_sort_buffer_size | 1048576  |
| join_buffer_size        | 262144   |
| key_buffer_size         | 8388608  |
| myisam_sort_buffer_size | 8388608  |
| preload_buffer_size     | 32768    |
| read_buffer_size        | 131072   |
| read_rnd_buffer_size    | 262144   |
| sort_buffer_size        | 262144   |
+-------------------------+----------+
10 rows in set (0.00 sec)
```

<div style="width:100px">Variable_name</div> | <div style="width:40px">范围</div> | <div style="width:30px">是否动态参数</div> | <div style="width:35px">默认值</div> | <div style="width:35px">最小值</div> | <div style="width:55px">最大值</div> | <div style="width:150px">说明</div> | 备注
--- |--- |--- |--- |--- |--- |--- |---
bulk_insert_buffer_size<br>(<span style="color:red">MyISAM专用</span>) | Global<br>Session | Yes | 8MB | 0 | 4GB(32 bit)<br>-(64 bit) | MyISAM批量插入非空表数据时使用的高速树状缓存区大小（以每个线程为单位） | -
innodb_log_buffer_size<br>(<span style="color:red">InnoDB专用</span>) | Global | Yes | 16MB | 1MB | 4GB | InnoDB用来写入磁盘日志文件的缓冲区的字节大小 | 如果业务代码中比较多较大的事务处理，最好将该值调大一些，避免事务提交前频繁写入磁盘，节省磁盘I/O
innodb_sort_buffer_size<br>(<span style="color:red">InnoDB专用</span>) | Global | No | 1MB | 64KB | 64MB | 指定在创建InnoDB索引期间用于排序数据的排序缓冲区的大小 | (1) 此排序区域{% label info@仅用于创建索引期间的合并排序%}，而不用于以后的索引维护操作。在索引创建完成时释放缓冲区；<br>(2) 此选项的值还控制在联机DDL操作期间扩展临时日志文件以记录并发DML的数量；<br>(3) 在{% label info@创建索引%}的{% label info@ALTER TABLE%}或{% label info@CREATE TABLE%}语句中，将分配{% label info@3个%}缓冲区，每个缓冲区的大小由该选项定义。另外，将辅助指针分配给排序缓冲区中的行，以便排序可以在指针上运行(而不是在排序操作期间移动行)
join_buffer_size | Global<br>Session | Yes | 256KB | 128B | 4GB(Windows)<br>4GB(32 bit)<br>-(64 bit) | 用于普通索引(plain index)扫描、范围索引(range index)扫描和不使用索引执行全表扫描的联接(join)的缓冲区的最小大小 | (1) 每两个表之间的全联接(full join)被分配1个join buffer，如果是多个表的复杂连接，需要多个join buffer。<br>(2) {% label info@最好保持全局设置较小%}，如果全局大小大于使用它的大多数查询所需的大小，那么内存分配时间会导致显著的性能下降。<br>(3) 当使用{% label info@块嵌套循环%}(Block Nested-Loop)时，较大的联接缓冲区可以在第一个表中所有行中的所有必需列都存储在联接缓冲区中的情况下发挥有益的作用。<br>(4) 当使用{% label info@批处理密钥访问%}(Batched Key Access)时，join_buffer_size的值定义了向存储引擎发出的每个请求中密钥的批处理大小。缓冲区越大，对联接操作的右表的顺序访问就越多，这可以显著提高性能
key_buffer_size<br>(<span style="color:red">MyISAM专用</span>) | Global | Yes | 8MB | 8B | 4GB(32 bit)<br><div style="width:55px">OS_PER_PROCESS_LIMIT(64 bit)</div> | <div style="width:150px">索引缓冲区（索引缓存），设置的最大值不要超过机器总内存的25%。可以通过<span style="font-size:0.6em">Key_reads/Key_read_requests</span>（应<0.01），<span style="font-size:0.6em;">Key_writes/Key_write_requests</span>，检查索引缓冲的性能</div> | 使用中的缓冲区比例（近似值）：<br>1 - ((Key_blocks_unused * key_cache_block_size) / key_buffer_size)
myisam_sort_buffer_size<br>(<span style="color:red">MyISAM专用</span>) | Global<br>Session | Yes | 8MB | 4KB | 4GB(32 bit)<br>-(64 bit) | 在REPAIR TABLE期间对MyISAM索引排序或使用CREATE INDEX或ALTER TABLE创建索引时分配的缓冲区大小 | - 
preload_buffer_size | Global,<br>Session | Yes | 32KB | 1KB | 1GB | 预加载索引时分配的缓冲区大小 | -
read_buffer_size | Global<br>Session | Yes | 128KB | 8200B | 2GB | 1. 对MyISAM表进行顺序扫描的每个线程都会为其扫描的每个表分配此大小（以字节为单位）的缓冲区。<br>2. 对于所有存储引擎：<br>(1) 在为ORDER BY排序行时，用于将索引缓存在临时文件（而不是临时表）中。<br>(2) 对于批量插入分区。<br>(3) 用于缓存嵌套查询的结果。 | 该值为4kb的倍数（如果不是会四舍五入取最近的）
read_rnd_buffer_size | Global,<br>Session | Yes | 256KB | 1B | 2GB | 该值用于从MyISAM表进行读取，并且对于任何存储引擎均用于多范围读取优化 | 当在键排序操作之后按排序顺序从MyISAM表中读取行时，将通过此缓冲区读取这些行以避免磁盘查找（将变量设置为较大的值可以大大提高ORDER BY性能。但是这是为每个客户端分配的缓冲区，因此不应将全局变量设置为较大的值，而是仅在需要运行大型查询的那些客户端中更改会话变量）
sort_buffer_size | Global<br>Session | Yes | 256KB | 32KB | 4GB(32 bit)<br>-(64 bit) | 每个必须执行排序的会话都会分配此大小的缓冲区 | (1) 该参数是会话级的（每个session用到排序时都会分配），所以不应设置过大；<br>(2) SHOW GLOBAL STATUS时如果看到每秒有很多Sort_merge_passes，则可以考虑增加sort_buffer_size值来加快ORDER BY或GROUP BY操作，这些操作无法通过查询优化或改进的索引来改善

补充：
<div style="width:130px">Variable_name</div> | <div style="width:30px">范围</div> | <div style="width:30px">是否动态参数</div> | <div style="width:35px">默认值</div> | <div style="width:35px">最小值</div> | <div style="width:100px">最大值</div> | <div style="width:90px">说明</div> | 备注
--- |--- |--- |--- |--- |--- |--- |---
<div style="width:130px">innodb_buffer_pool_chunk_size</div> | Global | No | 128M | 1M | <div style="width:100px">innodb_buffer_pool_size / innodb_buffer_pool_instances（缓冲池大小/缓冲池实例个数）</div> | <div style="width:90px">对缓冲池分块以避免在调整缓冲池大小时复制缓冲池的全部页，这个值就是用来定义每块的大小</div> | (1) 初始化时如果innodb_buffer_pool_chunk_size*缓冲池实例个数比当前的缓冲池大小还大，innodb_buffer_pool_chunk_size会被调整为：缓冲池大小/缓冲池实例个数；<br>(2) ==innodb_buffer_pool_size== 一定是等于innodb_buffer_pool_chunk_size * innodb_buffer_pool_instances的，如果修改==innodb_buffer_pool_chunk_size==配置，初始化时也会自动调整innodb_buffer_pool_size大小为innodb_buffer_pool_chunk_size * innodb_buffer_pool_instances。（所以调整innodb_buffer_pool_chunk_size配置时需要留心对缓冲池大小的影响）；<br>(3) 为避免潜在性能问题，不要让innodb_buffer_pool_instances的值超过1000

> 链接：
https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html 
https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html 