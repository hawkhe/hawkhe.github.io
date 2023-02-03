---
title: "SimpleFS文件系统1: 初次使用"
date: 2020-05-07T00:25:43+08:00
tags: ["simplefs", "fs"]
draft: false
---



## 动机

文件系统是经常会接触的一个概念，但是习惯于在glibc之上编写程序的时候，文件系统又像是黑盒一样，了解的知识点也比较零散。

SimpleFS是一个设计和实现都十分简单的本地文件系统，因此非常适合作为入门的，学习文件系统相关知识的工具。



## 编译

### 下载源码

```bash
[root@node-8 simplefs]# git clone https://github.com/sysprog21/simplefs.git
```

### 编译

```bash
[root@node-8 simplefs]# cd simplefs
[root@node-8 simplefs]# make
make -C /lib/modules/4.18.0-348.2.1.el8_5.x86_64/build M=/work/simplefs modules
make[1]: Entering directory '/usr/src/kernels/4.18.0-348.2.1.el8_5.x86_64'
  CC [M]  /work/simplefs/fs.o
  CC [M]  /work/simplefs/super.o
...
  LD [M]  /work/simplefs/simplefs.ko
make[1]: Leaving directory '/usr/src/kernels/4.18.0-348.2.1.el8_5.x86_64'
```

编译的时候可能提示缺乏头文件，例如笔者CentOS-8.5上会有如下报错，

```
[root@node-8 simplefs]# make -j 4
cc -std=gnu99 -Wall -o mkfs.simplefs mkfs.c
In file included from /usr/include/linux/ioctl.h:5,
                 from /usr/include/asm/ioctls.h:5,
                 from /usr/include/bits/ioctls.h:23,
                 from /usr/include/sys/ioctl.h:26,
                 from mkfs.c:6:
/usr/include/asm/ioctl.h:5:10: fatal error: uapi/asm-generic/ioctl.h: No such file or directory
 #include <uapi/asm-generic/ioctl.h>
          ^~~~~~~~~~~~~~~~~~~~~~~~~~
compilation terminated.
make: *** [Makefile:15: mkfs.simplefs] Error 1
```

可以在include下面添加软连接解决

```bash
[root@node-8 simplefs]# ln -s  /usr/src/kernels/`(uname -r)`/include/uapi/asm-generic /usr/include/uapi/asm-generic
```

编译后主要生成两个文件，simplefs.ko和mkfs.simplefs。simplefs.ko是SimpleFS的内核模块，而mkfs.simplefs则是用来进行文件系统格式化的。



## 格式化验证

### 加载内核模块

```
[root@node-8 simplefs]# insmod simplefs.ko
[root@node-8 simplefs]# 
[root@node-8 simplefs]# cat /proc/filesystems | grep simplefs
        simplefs
```

### 格式化虚拟机硬盘

创建一个虚拟硬盘（为了方便分析，这里我们选择一个较小的容量），

```bash
[root@node-8 simplefs]# dd if=/dev/zero of=/tmp/image1 bs=4k count=128
128+0 records in
128+0 records out
524288 bytes (524 kB, 512 KiB) copied, 0.000995776 s, 527 MB/s
[root@node-8 simplefs]#
[root@node-8 simplefs]# ll -h /tmp/image1
-rw-r--r-- 1 root root 512K May  7 00:25 /tmp/image1 # 512KB大小的虚拟硬盘文件(RAW格式)
```

SimpleFS代码注释中给出的分区布局(每个block大小为4KB)，这里给出来方便大家理解:

```c++
/*
 * simplefs partition layout
 * +---------------+
 * |  superblock   |  1 block
 * +---------------+
 * |  inode store  |  sb->nr_istore_blocks blocks
 * +---------------+
 * | ifree bitmap  |  sb->nr_ifree_blocks blocks
 * +---------------+
 * | bfree bitmap  |  sb->nr_bfree_blocks blocks
 * +---------------+
 * |    data       |
 * |      blocks   |  rest of the blocks
 * +---------------+
 */
```



格式化

```bash
[root@node-8 simplefs]# ./mkfs.simplefs /tmp/image1
SIMPLEFS_BLOCK_SIZE: 4096
SIMPLEFS_INODES_PER_BLOCK: 56
Superblock: (4096) # 超级块占用4KB大小，以下为主要数据内容
        magic=0xdeadce # 标识SimpleFS的魔数
        nr_blocks=128 # 总的块数量, 512KB/4KB=128个。
        nr_inodes=168 (istore=3 blocks) # 总共允许创建168个inode, 168/(4KB/72B)=3,总共占用3个块存放inode信息。
        nr_ifree_blocks=1 # 使用1个块存放inode使用情况的bitmp(位图)
        nr_bfree_blocks=1 # 使用1个块存放data block(数据块)使用情况的bitmp
        nr_free_inodes=167 # 剩余inode的计数, 因为要把0号inode预留给文件系统的根, 故为168-1=167
        nr_free_blocks=121 # 剩余数据块的计数, 128 -1(sb) -3(inode store) -1(Ifree) -1(Bfree) -1(文件系统根目录需要保存的数据) = 121
Inode store: wrote 3 blocks
        inode size = 72 B
Ifree blocks: wrote 1 blocks
Bfree blocks: wrote 1 blocks
```

查看虚拟硬盘在格式化之后的内容

```bash
[root@node-8 simplefs]# hexdump -C /tmp/image1
00000000  ce ad de 00 80 00 00 00  a8 00 00 00 03 00 00 00  |................|
00000010  01 00 00 00 01 00 00 00  a7 00 00 00 79 00 00 00  |............y...|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001000  fd 41 00 00 00 00 00 00  00 00 00 00 00 10 00 00  |.A..............|
00001010  00 00 00 00 00 00 00 00  00 00 00 00 01 00 00 00  |................|
00001020  02 00 00 00 06 00 00 00  00 00 00 00 00 00 00 00  |................|
00001030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00004000  fe ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
00004010  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
00005000  80 ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
00005010  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
00006000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00080000
```

分析

* 0x0000~0x0FFF，4KB大小，为超级块的数据，其头部的4个字节为小端转换后的魔数，魔数的定义: \#define SIMPLEFS_MAGIC 0xDEADCELL
* 0x1000~0x3FFF，12KB大小，为存放inode信息所用的3个块，由之前mkfs输出我们得知一个inode占用72B大小，那有数据写入的地址范围(0x1000~0x102F)总共占用48B，可以判断当前已经写入了1个inode信息。
* 0x4000~0x4FFF，4KB大小，为inode空闲位图，1表示未使用，0表示已经使用，故0xfe(二进制:11111110)则表示index为0的inode已经被使用。
* 0x5000~0x5FFF，4KB大小，为数据块(data block)空闲位图，1表示未使用，0表示已经使用，0x80的二进制为10000000，表示前7个数据块已经被使用。
* 0x6000~0x80000，488KB大小，为剩余的用于保存数据块的地址空间，因为当前还没有数据写入，故全为0值。



## 挂载验证

使用已格式化的虚拟硬盘文件挂载到/mnt目录上

```bash
[root@node-8 simplefs]# mount -t simplefs -o loop /tmp/image1 /mnt
[root@node-8 simplefs]#
[root@node-8 simplefs]# df -h | grep /mnt
/dev/loop0           512K   28K  484K   6% /mnt
[root@node-8 simplefs]#
```

可见总容量512KB，已使用容量28KB，均与格式化环节中的分析一致。



## 文件写入验证

创建文件并写入数据

```bash
[root@node-8 simplefs]# echo "bar" > /mnt/foo.dat
```

查看新建文件元数据

```bash
[root@node-8 simplefs]# stat /mnt/foo.dat
  File: /mnt/foo.dat
  Size: 4               Blocks: 2          IO Block: 4096   regular file
Device: 700h/1792d      Inode: 1           Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2022-05-09 00:22:45.000000000 +0800
Modify: 2022-05-09 00:22:24.000000000 +0800
Change: 2022-05-09 00:22:24.000000000 +0800
 Birth: -
```

查看虚拟机硬盘中的内容

```bash
[root@node-8 simplefs]# sync
[root@node-8 simplefs]#
[root@node-8 simplefs]# hexdump -C /tmp/image1
00000000  ce ad de 00 80 00 00 00  a8 00 00 00 03 00 00 00  |................|
00000010  01 00 00 00 01 00 00 00  a6 00 00 00 70 00 00 00  |............p...|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001000  fd 41 00 00 00 00 00 00  00 00 00 00 00 10 00 00  |.A..............|
00001010  40 ee 77 62 52 ee 77 62  40 ee 77 62 01 00 00 00  |@.wbR.wb@.wb....|
00001020  02 00 00 00 06 00 00 00  26 fa 17 e7 a1 ee 20 81  |........&..... .|
00001030  58 41 4d cc aa 98 b4 fd  0a 84 00 00 00 00 00 00  |XAM.............|
00001040  00 00 00 00 00 00 00 00  a4 81 00 00 00 00 00 00  |................|
00001050  00 00 00 00 04 00 00 00  40 ee 77 62 55 ee 77 62  |........@.wbU.wb|
00001060  40 ee 77 62 02 00 00 00  01 00 00 00 07 00 00 00  |@.wb............|
00001070  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00004000  fc ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
00004010  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
00005000  00 00 ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
00005010  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
*
00006000  01 00 00 00 66 6f 6f 2e  64 61 74 00 00 00 00 00  |....foo.dat.....|
00006010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00007000  00 00 00 00 08 00 00 00  08 00 00 00 00 00 00 00  |................|
00007010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00008000  62 61 72 0a 00 00 00 00  00 00 00 00 00 00 00 00  |bar.............|
00008010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00080000
```

分析

* 存放inode信息的地址段中，有新的一条数据写入(0x1040~0x106F)，编号为1，结合stat之前命令的输出，应该为新建文件的inode信息。
* 数据块的地址空间内，出现了新建文件的文件名“foo.dat”，以及文件内容"bar"。



## 小结

* 通过本节的介绍，对SimpleFS有了初步的了解
* 验证中所经历的格式化、挂载、文件创建/数据写入等步骤的内部原理，将在后续的学习过程中一一展开分析，今天先到这里吧~

## 引用

[源代码](https://github.com/sysprog21/simplefs)


