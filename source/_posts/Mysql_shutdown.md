---
title: mysql频繁重启 问题解决
date: 2020-04-01 02:46:01
tags: 
    - Linux
    - 网站开发
    - 实战总结
categories:
    - 问题解决
---

最近每天晚上的定时任务都会跑失败，业务错误日志都是{% label danger@MySQL server has gone away%}。去查MySQL的日志也没有对应的错误信息，只有启动日志：

```Log
2020-01-19T07:04:07.434344Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.18) starting as process 10717
2020-01-19T07:04:09.716710Z 0 [System] [MY-010229] [Server] Starting crash recovery...
2020-01-19T07:04:09.786959Z 0 [System] [MY-010232] [Server] Crash recovery finished.
```

<!-- more -->

查看内核日志（dmesg |grep mysqld），发现是内存不够用直接被kill掉了。

```Log
[5974675.178431] [32531]  1000 32531   343724   120571     343        0             0 mysqld
[5974675.178504] Out of memory: Kill process 32531 (mysqld) score 476 or sacrifice child
[5974675.180641] Killed process 32531 (mysqld) total-vm:1374896kB, anon-rss:482284kB, file-rss:0kB, shmem-rss:0kB
```

我的服务器配置如下，可见资源非常紧张，然而运行一个几乎无人访问的[{% label info@小网站%}](https://prettycrazyjoey.cn/)应该也够了。

操作系统 | CPU | 内存 | 公网带宽 | 硬盘
---|--- | --- | --- | ---
CentOS 7.5 64位 | 1核 | 1GB | 1Mbps | 50GB

但问题还是要解决的，不然每天<i>free -m</i>提心吊胆的实在是太折磨。我最初想到的方法是调整MySQL配置和定期清理缓存，经过一番查询和亲自实践才发现最有效的办法其实是使用交换文件作为虚拟内存。总之，这里还是记录和总结一下具体的解决方法吧。

#### 1. 调整MySQL配置

MySQL刚安装的时候就已经把innodb_buffer_pool_size调整到了80M，只好继续调低，目前配置64M（__注意__：要同时修改{% label primary@innodb_buffer_pool_chunk_size%}的大小，该值和innodb_buffer_pool_size一样都是默认128M。它相当于缓冲池的最小单位，如果缓冲池总大小比该值还小，在初始化时实际上不会改变）。但感觉这应该节省不了多少内存，只好继续从别的配置项着手。有关buffer_size相关的各配置项的详细说明请参见 [{% label info@《MySQL：buffer_size相关配置》%}](/2020/03/01/Mysql_buffer_size/)。

实际上，以我当前的服务器条件，在修改配置上节省出更多的内存，优化的空间已经非常小，且大部分MySQL的默认配置已经是相对最优的了。所以还是应该换个思路。

#### 2. 使用Swap虚拟内存

<u>*Linux将其物理RAM（随机访问内存）划分为称为页的内存块。Swap是将内存页复制到硬盘上预先配置的空间（称为交换空间）以释放该内存页的过程。物理内存和交换空间的总大小就是可用的虚拟内存量。*</u>

更详细一些的介绍可以参考这篇，[{% label info@Swap介绍与应用%}](/2020/02/11/Swap/)。

最快捷的方法就是安装systemd-swap来使用交换空间，这是一个用来管理swap的脚本，只要简单几步就可以将复杂的创建和配置过程搞定，如下：

```Shell
# 安装systemd-swap包
git clone https://github.com/Nefelim4ag/systemd-swap.git
cd systemd-swap
sudo make install

# 设置/etc/systemd/swap.conf的交换文件分块配置
swapfc_enabled=1
swapfc_force_preallocated=1 (如果日志一直报这个错误systemd-swap[..]: WARN: swapFC: ENOSPC，就开启)

# 启动systemd-swap服务
sudo service systemd-swap start
```

#### 【后记】
在使用Swap之前基本上MySQL每天都会重启一两次，使用之后就没再发生过重启的情况了，到目前也已经将近两个月过去了，而数据也是在不断增加的，实践证明，在内存空间不是十分富裕的情况下使用Swap交换空间作为虚拟内存，是非常有效的措施。

当前的内存使用情况如下：

-| total | used | free | shared | buff/cache | available
---|--- |--- |--- |--- |--- |---
Mem: | 991 | 533 | 66 | 0 | 392 | 278
Swap:| 1023 | 566 |457
                            