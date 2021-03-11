---
title: Swap介绍与应用
date: 2020-02-11 01:46:38
tags:
    - Linux
    - Swap
    - 文档翻译
categories:
    - Coding
---

{% cq %} Linux将其物理RAM（随机访问内存）划分为称为页的内存块。Swap是将内存页复制到硬盘上预先配置的空间（称为{% label info@交换空间 %}）以释放该内存页的过程。物理内存和交换空间的总大小就是可用的虚拟内存量。{% endcq %}

<!-- more -->

<!-- TOC -->

- [1. 交换空间（Swap space）](#1-交换空间swap-space)
- [2. 交换分区（Swap partition）](#2-交换分区swap-partition)
- [3. 交换文件（Swap file）](#3-交换文件swap-file)
- [4. 性能](#4-性能)
  - [4.1. 两个影响swap性能的参数](#41-两个影响swap性能的参数)
  - [4.2. 优先级](#42-优先级)
  - [4.3. 使用zswap或zram](#43-使用zswap或zram)
  - [4.4. Striping (?)](#44-striping-)

<!-- /TOC -->

### 1. 交换空间（Swap space）

- 交换空间可以采用{% label warning@磁盘分区%}或{% label warning@文件%}的形式。交换空间可用于两个目的，即将虚拟内存扩展到已安装的物理内存（RAM）之外（也称为“enable swap”），也可用于磁盘挂起支持（suspend-to-disk support）。
- 启用交换是否有益取决于已安装的物理内存量以及运行所有所需程序所需的内存量。如果物理内存量小于所需的量，则启用交换是有益的。这样可以避免内存不足的情况，Linux内核的{% label warning@OOM killer%}机制将通过杀死进程来自动尝试释放内存。要将虚拟内存量增加到所需的数量，请添加必要的差异作为交换空间。启用交换的最大缺点是{% label warning@性能较低%}。
- 检查Swap状态：

```bash
swapon -s
```

或者

```bash
free -m #free还指示内存是否不足，可以通过启用或增加Swap来补救。
```

### 2. 交换分区（Swap partition）

- 将分区设置为Linux交换区域（{% label danger@注：指定分区上的所有数据将丢失%}）：

```bash
mkswap / dev / sd xy
```

- 启用device进行分页：

```bash
swapon /dev/sdxy
```

- 要在启动时启用此交换分区，添加以下内容到/etc/fstab：

```bash
UUID=device_UUID none swap defaults 0 0 
#device_UUID是swap的UUID
```

- 通过systemd激活
systemd基于两种不同的机制激活交换分区。两者都是的可执行文件 {% label warning@/usr/lib/systemd/system-generators%}。生成器在启动时运行，并创建用于安装的本机systemd单元。首先 {% label warning@systemd-fstab-generator%}读取fstab，生成单元（包括用于交换的单元）。第二步，{% label warning@systemd-gpt-auto-generator%}检查根磁盘以生成单元。它仅在GPT磁盘上运行，并且可以通过分区类型GUID识别交换分区。
- 禁用交换

```bash
swapoff / dev / sd xy
swapoff -a  #禁用所有交换空间
```

### 3. 交换文件（Swap file）

<u>*作为创建整个分区的替代方案，交换文件提供了动态更改大小的功能，并且更容易完全删除。*</u>

- 手动创建

```bash
#对于Btrfs这种写时复制（copy-on-write）的文件系统

#首先创建一个长度为零的文件
truncate -s 0 /swapfile
#使用chattr对其设置 No_COW 属性
chattr +C /swapfile
#禁用压缩
btrfs property set /swapfile compression none
```

<i>{% label danger@注：自Linux内核版本5.0起，Btrfs开始支持交换文件，但有一定限制:<br>
(1) 交换文件不能位于快照子卷上。正确的过程是创建一个新的子卷来放置交换文件。<br>
(2) 它不支持跨多个设备的文件系统上的交换文件。%}</i>

```bash
#使用fallocate创建一个交换文件（M = Mebibytes，G = Gibibytes）
fallocate -l 512M /swapfile
#注意： Fallocate可能会导致某些文件系统（例如F2FS）出现问题。作为替代，使用dd更可靠，但更慢
dd if=/dev/zero of=/swapfile bs=1M count=512 status=progress

#设置正确的权限（world-readable swap file是一个巨大的本地漏洞）：
chmod 600 /swapfile

#格式化
mkswap /swapfile

#激活
swapon /swapfile

#编辑fstab配置，添加以下条目
vim /etc/fstab
/swapfile none swap defaults 0 0
```

<i>{% label danger@注：交换文件必须由其在文件系统上的位置指定，而不是由其UUID或LABEL指定%}</i>

- 删除交换文件

```bash
#先关闭
swapoff /swapfile
#再删除
rm -f /swapfile
#最后从/etc/fstab中删除相关条目
vim /etc/fstab
...
```

- 自动创建
<u>*systemd-swap是一个脚本，用于从zram交换、交换文件和交换分区创建混合交换空间。它与systemd项目无关。*</u>

```bash
#安装systemd-swap包
git clone https://github.com/Nefelim4ag/systemd-swap.git
cd systemd-swap
sudo make install
#设置/etc/systemd/swap.conf的交换文件分块配置
swapfc_enabled=1
? swapfc_force_preallocated=1 (如果日志一直报这个错误systemd-swap[..]: WARN: swapFC: ENOSPC，就开启)
#启动systemd-swap服务
sudo service systemd-swap start
```

### 4. 性能

<u>*交换操作通常比直接访问RAM中的数据要慢得多。完全禁用交换以提高性能有时会导致性能下降，因为这会减少可用于VFS缓存的内存，从而导致更频繁、更昂贵的磁盘I/O。*</u>

#### 4.1. 两个影响swap性能的参数

- **Swappiness**
swappiness sysctl参数表示内核对交换空间的偏好(或避免)。Swappiness的值可以在0到100之间，默认值是60。低值导致内核避免交换，高值导致内核尝试使用交换空间。在足够的内存上使用较低的值可以提高许多系统的响应能力。
- **VFS cache pressure**
它控制内核回收用于缓存VFS caches的内存的趋势，而不是pagecache和swap。增加这个值会增加回收VFS缓存的速度。

```bash
#检查当前swappiness|vfs_cache_pressure值（或者可以查看文件/sys/fs/cgroup/memory/memory.swappiness或/proc/sys/vm/swappiness）
sysctl vm.swappiness
sysctl vm.vfs_cache_pressure

#临时设置（由于/proc组织性很差，仅出于兼容性目的而保留，因此建议使用/sys代替）
sysctl -w vm.swappiness=10
sysctl -w vm.vfs_cache_pressure=50

#永久设置
vim /etc/sysctl.d/99-swappiness.conf
vm.swappiness=10
vm.vfs_cache_pressure=50
```

#### 4.2. 优先级

如果有多个交换文件或交换分区，则应考虑为每个交换区域分配一个优先级值（0到32767）。在使用优先级较低的交换区之前，系统将使用优先级较高的交换区。

```bash
#例，有一个较快的磁盘（/dev/sda）和一个较慢的磁盘（/dev/sdb），为最快的设备上的交换区域分配更高的优先级（pri）
/dev/sda1 none swap defaults,pri=100 0 0
/dev/sdb2 none swap defaults,pri=10  0 0

#或通过swapon的--priority参数：
swapon --priority 100 /dev/sda1
```

#### 4.3. 使用zswap或zram

Zswap是Linux内核功能，为交换的页面提供压缩的回写缓存。这样可以提高性能并减少IO操作。ZRAM在内存中创建虚拟压缩的交换文件，以替代磁盘上的交换文件。

#### 4.4. Striping (?)

出于交换性能的原因，没有必要使用RAID。内核本身可以在多个设备上进行stripe swapping，只要在/etc/fstab文件中赋予它们相同的优先级即可。

> 链接：
<https://wiki.archlinux.org/index.php/Swap>
<https://www.linux.com/news/all-about-linux-swap-space>
