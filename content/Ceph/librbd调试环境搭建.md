---
title: "Librbd调试环境搭建"
date: 2022-07-22T04:42:07Z
draft: false
---

# Librbd调试环境搭建


## 编译Ceph

编译环境

```bash
root@haosheng-dev2:~# cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=20.04
DISTRIB_CODENAME=focal
DISTRIB_DESCRIPTION="Ubuntu 20.04 LTS"
```

下载Ceph源码

```bash
root@haosheng-dev2:/work# git clone --branch v15.2.16 https://github.com/ceph/ceph.git
```

下载源码依赖

```
root@haosheng-dev2:/work# cd ceph
root@haosheng-dev2:/work/ceph# git submodule update --init --recursive
```

安装依赖包

```
root@haosheng-dev2:/work/ceph# ./install-deps.sh
```

因为后面需要使用gdb调试程序，故需要以debug模式进行编译

配置debug相关参数，并执行do_cmake.sh

```
root@haosheng-dev2:/work/ceph# ARGS="-DCMAKE_C_FLAGS=-O0 -g3 -gdwarf-4 -DCMAKE_CXX_FLAGS=-O0 -g3 -gdwarf-4" ./do_cmake.sh -DWITH_MGR_DASHBOARD_FRONTEND=off
```

cmake添加的参数的解释:

* *CMAKE_C_FLAGS=“-O0 -g3 -gdwarf-4”* ： c 语言编译配置
* *CMAKE_CXX_FLAGS=“-O0 -g3 -gdwarf-4”* ：c++ 编译配置
* *-O0* : 关闭编译器的优化，如果没有，使用GDB追踪程序时，大多数变量被优化,无法显示, 生产环境必须关掉
* *-g3* : 意味着会产生大量的调试信息
* *-gdwarf-4* : dwarf 是一种调试格式，dwarf-4 版本为4

执行编译

```bash
root@haosheng-dev2:/work/ceph# cd build
root@haosheng-dev2:/work/ceph/build# make -j4
```

查看编译后的库文件

```
root@haosheng-dev2:/work/ceph/build# ll -h lib/librbd.so*
lrwxrwxrwx 1 root root   11 Apr 20 09:56 lib/librbd.so -> librbd.so.1*
lrwxrwxrwx 1 root root   16 Apr 20 09:56 lib/librbd.so.1 -> librbd.so.1.12.0*
-rwxr-xr-x 1 root root 212M Apr 20 09:56 lib/librbd.so.1.12.0*
```



## 启动集群

使用vstart启动虚拟集群

```
root@haosheng-dev2:/work/ceph/build# make vstart
root@haosheng-dev2:/work/ceph/build# MON=1 OSD=3 MGR=1 ../src/vstart.sh --debug --new --localhost --filestore
```

查看虚拟集群状态

```
root@haosheng-dev2:/work/ceph/build# ./bin/ceph -s
*** DEVELOPER MODE: setting PATH, PYTHONPATH and LD_LIBRARY_PATH ***
2022-04-21T02:20:33.537+0000 7f6684d4e700 -1 WARNING: all dangerous and experimental features are enabled.
2022-04-21T02:20:33.561+0000 7f6684d4e700 -1 WARNING: all dangerous and experimental features are enabled.
  cluster:
    id:     54a63c07-ace5-44cf-9baf-7e0649c435d1
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum a (age 44s)
    mgr: x(active, since 39s)
    mds: a:1 {0=c=up:active} 2 up:standby
    osd: 3 osds: 3 up (since 31s), 3 in (since 31s)

  data:
    pools:   3 pools, 65 pgs
    objects: 22 objects, 2.2 KiB
    usage:   148 GiB used, 451 GiB / 600 GiB avail
    pgs:     65 active+clean

  io:
    client:   0 B/s wr, 0 op/s rd, 0 op/s wr
```

创建一个用于测试的存储池

```
root@haosheng-dev2:/work/ceph/build# ./bin/ceph osd pool create test 8 8 replicated
*** DEVELOPER MODE: setting PATH, PYTHONPATH and LD_LIBRARY_PATH ***
2022-04-21T02:43:58.265+0000 7ff89b461700 -1 WARNING: all dangerous and experimental features are enabled.
2022-04-21T02:43:58.305+0000 7ff89b461700 -1 WARNING: all dangerous and experimental features are enabled.
pool 'test' created
```



## 编写客户端程序

### 使用librados库的程序

```c
// -*- mode:C++; tab-width:8; c-basic-offset:2; indent-tabs-mode:t -*-
// vim: ts=8 sw=2 smarttab
/*
 * Ceph - scalable distributed file system
 *
 * This is free software; you can redistribute it and/or
 * modify it under the terms of the GNU Lesser General Public
 * License version 2.1, as published by the Free Software
 * Foundation. See file COPYING.
 */

// install the librados-dev and librbd package to get this
#include <rados/librados.hpp>
#include <rbd/librbd.hpp>
#include <iostream>
#include <string>
#include <sstream>

const char *pool_name = "test";
const std::string image_name = "rbd1";
const char test_data[] = "hello rbd!\0";

int main(int argc, const char **argv)
{
    int ret = 0;

    // we will use all of these below
    librados::IoCtx io_ctx;

    // first, we create a Rados object and initialize it
    librados::Rados rados;
    {
        ret = rados.init("admin"); // just use the client.admin keyring
        if (ret < 0)
        { // let's handle any error that might have come back
            std::cerr << "couldn't initialize rados! error " << ret << std::endl;
            ret = EXIT_FAILURE;
            goto out;
        }
        else
        {
            std::cout << "we just set up a rados cluster object" << std::endl;
        }
    }

    /*
     * Now we need to get the rados object its config info. It can
     * parse argv for us to find the id, monitors, etc, so let's just
     * use that.
     */
    {
        ret = rados.conf_parse_argv(argc, argv);
        if (ret < 0)
        {
            // This really can't happen, but we need to check to be a good citizen.
            std::cerr << "failed to parse config options! error " << ret << std::endl;
            ret = EXIT_FAILURE;
            goto out;
        }
        else
        {
            std::cout << "we just parsed our config options" << std::endl;
            // We also want to apply the config file if the user specified
            // one, and conf_parse_argv won't do that for us.
            for (int i = 0; i < argc; ++i)
            {
                if ((strcmp(argv[i], "-c") == 0))
                {
                    ret = rados.conf_read_file(argv[i + 1]);
                    if (ret < 0)
                    {
                        // This could fail if the config file is malformed, but it'd be hard.
                        std::cerr << "failed to parse config file " << argv[i + 1]
                                  << "! error" << ret << std::endl;
                        ret = EXIT_FAILURE;
                        goto out;
                    }
                    break;
                }
            }
        }
    }

    /*
     * next, we actually connect to the cluster
     */
    {
        ret = rados.connect();
        if (ret < 0)
        {
            std::cerr << "couldn't connect to cluster! error " << ret << std::endl;
            ret = EXIT_FAILURE;
            goto out;
        }
        else
        {
            std::cout << "we just connected to the rados cluster" << std::endl;
        }
    }

    /*
     * create an "IoCtx" which is used to do IO to a pool
     */
    {
        ret = rados.ioctx_create(pool_name, io_ctx);
        if (ret < 0)
        {
            std::cerr << "couldn't set up ioctx! error " << ret << std::endl;
            ret = EXIT_FAILURE;
            goto out;
        }
        else
        {
            std::cout << "we just created an ioctx for our pool" << std::endl;
        }
    }

    /*
     * open an rbd image and write data to it
     */
    {
        librbd::RBD rbd;
        librbd::Image image;

        ret = rbd.open(io_ctx, image, image_name.c_str(), NULL);
        if (ret < 0)
        {
            std::cerr << "couldn't open the rbd image! error " << ret << std::endl;
            ret = EXIT_FAILURE;
            goto out;
        }
        else
        {
            std::cout << "we just opened the rbd image" << std::endl;
        }

        size_t len = strlen(test_data);
        ceph::bufferlist bl;
        bl.append(test_data, len);

        ret = image.write(0, len, bl);
        if (ret < 0)
        {
            std::cerr << "couldn't write to the rbd image! error " << ret << std::endl;
            ret = EXIT_FAILURE;
            goto out;
        }
        else
        {
            std::cout << "we just wrote data to our rbd image " << std::endl;
        }

        image.close();
    }

    ret = EXIT_SUCCESS;
out:
    rados.shutdown();

    return ret;
}
```

### 执行编译

```
root@haosheng-dev2:/work/ceph_test# g++ -g hello_rbd.cc -lrados -lrbd -I/work/ceph/src/include -L/work/ceph/build/lib -o hello_rbd -Wl,-rpath,/work/ceph/build/lib
```

gcc各个参数的意义:

* *-g* : 允许gdb调试
* *-lrados* : -l 指定依赖库的名字为rados
* -I : 指定编译时依赖的librados.h头文件的路径
* *-L* : 指定编译时依赖库的的路径， 如果不指定将在系统目录下寻找
* *-o* : 编译的二进制文件名
* *-Wl* : 指定编译时参数
* *-rpath* : 指定运行时依赖库的路径， 如果不指定将在系统目录下寻找

### 运行并查看结果

创建存储池与镜像

```
root@haosheng-dev2:/work/ceph/build# ./bin/ceph osd pool create test 8 8 replicated
root@haosheng-dev2:/work/ceph/build#
root@haosheng-dev2:/work/ceph/build# ./bin/rbd create test/rbd1 --size 32MB
root@haosheng-dev2:/work/ceph/build#
root@haosheng-dev2:/work/ceph/build# ./bin/rbd ls -p test
rbd1
```

运行程序

```bash
root@haosheng-dev2:/work/ceph_test# ./hello_rbd -c /work/ceph/build/ceph.conf             we just set up a rados cluster object
we just parsed our config options
we just connected to the rados cluster
we just created an ioctx for our pool
we just opened the rbd image
we just wrote data to our rbd image
```

查看结果

```bash
root@haosheng-dev2:/work/ceph/build# ./bin/rados ls -p test
test-write
root@haosheng-dev2:/work/ceph/build#
root@haosheng-dev2:/work/ceph/build# ./bin/rbd export test/rbd1 /tmp/test_rbd1
Exporting image: 100% complete...done.
root@haosheng-dev2:/work/ceph/build#
root@haosheng-dev2:/work/ceph/build# hexdump -C /tmp/test_rbd1
00000000  68 65 6c 6c 6f 20 72 62  64 21 00 00 00 00 00 00  |hello rbd!......|
00000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
02000000
```

## 调试运行客户端程序

### 安装配置GDB

```
root@haosheng-dev2:/work/ceph_test# apt-get install gdb -y
root@haosheng-dev2:/work/ceph_test# cat ~/.gdbinit
set history save on
set print pretty on
```

### 编写调试的配置文件

```bash
root@haosheng-dev2:/work/ceph_test# cat run_hello_rbd.gdb
set breakpoint pending on
handle SIGUSR2 noprint nostop
handle SIGUSR1 noprint nostop

file /work/ceph_test/hello_rbd # 设置调试的程序

break hello_rbd.cc:139 # 设置初始断点

run -c /work/ceph/build/ceph.conf # 设置运行参数
```

### 执行调试

```
root@haosheng-dev2:/work/ceph_test# gdb -x run_hello_rbd.gdb
we just connected to the rados cluster
we just created an ioctx for our pool
[New Thread 0x7fffb77fe700 (LWP 1342822)]
[New Thread 0x7fffb6ffd700 (LWP 1342823)]
[New Thread 0x7fffb67fc700 (LWP 1342824)]
[New Thread 0x7fffb5ffb700 (LWP 1342825)]
we just opened the rbd image
Thread 1 "hello_rbd" hit Breakpoint 1, main (argc=3, argv=0x7fffffffe468) at hello_rbd.cc:139
warning: Source file is more recent than executable.
139             ret = image.write(0, len, bl);
(gdb)
```

发现程序暂停在预设置的断点处。



## 参考

* https://blog.csdn.net/XingKong_678/article/details/51590459
* https://github.com/ceph/ceph/blob/master/examples/librbd/hello_world.cc
