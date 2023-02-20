---
title: "使用系统调用fork创建子进程"
date: 2023-02-20T06:23:37Z
draft: false
tags: ["syscall"]
---





上一篇文章中，在使用strace跟踪demo程序系统调用时，不知道小伙伴们有没有发现一个小细节。<!--more-->

```bash
root@hao6:~# strace ./demo
...
brk(0x560d8b593000)                     = 0x560d8b593000
futex(0x7f763abc977c, FUTEX_WAKE_PRIVATE, 2147483647) = 0
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f763a66ea10) = 99202
wait4(99202, NULL, 0, NULL)             = 99202
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=99202, si_uid=0, si_status=1, si_utime=0, si_stime=0} ---
newfstatat(0, "", {st_mode=S_IFCHR|0600, st_rdev=makedev(0x88, 0x4), ...}, AT_EMPTY_PATH) = 0
...
```



## 问题

那就是我们的demo程序，使用的是fork函数来创建子进程，但是系统调用里面并没有使用fork，而是使用的clone。

我们去对应版本glibc中一探究竟，首先找到glibc的版本，

```bash
root@hao6:~# ldd --version | grep -i glibc
ldd (Ubuntu GLIBC 2.35-0ubuntu3.1) 2.35
```

在nptl/sysdeps/unix/sysv/linux/fork.c文件中，找到了fork在linux系统下的实现，

```c
pid_t
__libc_fork (void)
{
...
#ifdef ARCH_FORK  // 如果定义了ARCH_FORK，则调用它
  pid = ARCH_FORK ();
#else  // 否则，调用fork的系统调用
# error "ARCH_FORK must be defined so that the CLONE_SETTID flag is used"
  pid = INLINE_SYSCALL (fork, 0);
#endif
...
}
weak_alias (__libc_fork, fork)
```

ARCH_FORK有多个操作系统版本的定义，这里找到了测试环境所使用的x86_64中的定义，

```c
// nptl/sysdeps/unix/sysv/linux/x86_64/fork.c

#define ARCH_FORK() \
  INLINE_SYSCALL (clone, 4,						      \
		  CLONE_CHILD_SETTID | CLONE_CHILD_CLEARTID | SIGCHLD, 0,     \
		  NULL, &THREAD_SELF->tid)
```

可以看出，ARCH_FORK实际上是在这里最终调用了clone的系统调用。而这里设置的3个flag，也与strace中看到的一致，那这次静态分析翻车的概率应该就不大了。。



## 调用fork

如果小伙伴们要问，我就是要调用fork，不用clone，该怎么办呢？

参考网上大神给出的方法，绕过glibc，直接嵌入汇编代码。

这里要先找到fork系统调用的编号，可见其为57号，

```bash
root@hao6:/work# grep 'fork' linux-5.15.94/arch/x86/entry/syscalls/syscall_64.tbl
57      common  fork                    sys_fork
58      common  vfork                   sys_vfork
```

这里更新一下demo代码，

```c
#include <iostream>
#include <sys/wait.h>

using namespace std;

int main()
{
	pid_t pid;

	asm("mov $57, %rax");  // 设置rax寄存器为57
	asm("syscall");  // 调用fork()生成子进程
	asm("movl %%eax, %0": "=m" (pid));  // 赋值返回值到pid

    if (pid < 0)
    {
        cout << "Error creating";
        exit(1);
    }
    else if (pid == 0) // 如果是子进程
    {
        printf("This is child: %i\n", getpid());
        exit(1); // 直接退出，造成zombie状态
    }
    else // 如果是父进程
    {
        printf("This is parent: %i\n", getpid());
        waitpid(pid, nullptr, 0);
        int a;
        cin >> a;
    }
    return 0;
}
```

这里用到了asm的扩展格式，查看资料简单梳理一下，其指令格式为:

```c
asm [volatile] ("汇编指令" : "输出操作数列表" : "输入操作数列表" : "改动的寄存器");
```

其中，

* 输出操作数列表：汇编代码如何把处理结果传递到 C 代码中
* 输入操作数列表：C 代码如何把数据传递给内联汇编代码
* 改动的寄存器：告诉编译器，在内联汇编代码中，我们使用了哪些寄存器。指定这个是为了避免编译器在使用寄存器的时候，与我们的asm指令冲突了。

其中，输出和输入操作数列表的格式也是固定的，

```c
"[输出修饰符]约束"(寄存器或内存地址)
```

输出修饰符是可选的，主要是用来对输出寄存器或内存进行额外的说明，

* +：被修饰的操作数可以读取，可以写入；
* =：被修饰的操作数只能写入；
* %：被修饰的操作数可以和下一个操作数互换；
* &：在内联函数完成之前，可以删除或者重新使用被修饰的操作数；

约束就是告诉编译器，使用约束了范围的寄存器或内存，

* a: 使用 eax/ax/al 寄存器；
* b: 使用 ebx/bx/bl 寄存器；
* c: 使用 ecx/cx/cl 寄存器；
* d: 使用 edx/dx/dl 寄存器；
* r: 使用任何可用的通用寄存器；
* m: 使用变量的内存位置；

写到这里，也就不难理解demo中的asm扩展用法了。



## 验证

我们再执行下demo程序，

```bash
root@hao6:~# ./demo 
This is parent: 105000
This is child: 105001

```

发现了子进程中对其pid的打印，与父进程不一样，可以确定我们已经成功使用嵌入的汇编代码，执行了fork的系统调用。



## 引用

[内联汇编很可怕吗？看完这篇文章，终结它！](https://www.cnblogs.com/sewain/p/14707347.html)
