---
title: "使用waitpid清理僵尸进程"
date: 2023-02-17T02:40:10Z
draft: false
tags: ["syscall"]
---

最近，发现公司某个agent进程运行时，会产生一些僵尸子进程的现象

使用ps命令进行查找，这些僵尸进程都来自于同一个父进程36791

```bash
[root@node-1 ~]# ps -ef | grep "[d]efunct"
root     37185 36791  0 11:50 ?        00:00:00 [sudo] <defunct>
root     56231 36791  0 13:50 ?        00:00:00 [sudo] <defunct>
root     56255 36791  0 13:50 ?        00:00:00 [sudo] <defunct>
root     56359 36791  0 13:50 ?        00:00:00 [sudo] <defunct>
```

使用top对它们进行观察，发现并没有占用内存和CPU，也没有被调度到

```bash
top - 17:41:38 up 14 days, 20:56,  1 user,  load average: 24.34, 37.25, 43.21
Tasks:   4 total,   0 running,   0 sleeping,   0 stopped,   4 zombie
%Cpu(s):  8.2 us,  4.2 sy,  0.0 ni, 86.6 id,  0.0 wa,  0.5 hi,  0.5 si,  0.0 st
MiB Mem : 257515.5 total,  80404.6 free, 169038.2 used,   8072.7 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.  82702.2 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
37185 root      20   0       0      0      0 Z   0.0   0.0   0:00.06 sudo
56231 root      20   0       0      0      0 Z   0.0   0.0   0:00.07 sudo
56255 root      20   0       0      0      0 Z   0.0   0.0   0:00.07 sudo
56359 root      20   0       0      0      0 Z   0.0   0.0   0:00.08 sudo
```

查了下资料，虽然一般来说，僵尸进程不影响系统运行，但是强迫症的我还是准备一探究竟。



## 验证

*在[类UNIX系统](https://zh.wikipedia.org/wiki/类UNIX系统)中，**僵尸进程**是指完成执行（通过`exit`[系统调用](https://zh.wikipedia.org/wiki/系统调用)，或运行时发生[致命错误](https://zh.wikipedia.org/wiki/致命错误)或收到终止[信号](https://zh.wikipedia.org/wiki/信号_(计算机科学))所致），但在操作系统的进程表中仍然存在其[进程控制块](https://zh.wikipedia.org/wiki/进程控制块)，处于"[终止状态](https://zh.wikipedia.org/w/index.php?title=终止状态&action=edit&redlink=1)"的进程。这发生于[子进程](https://zh.wikipedia.org/wiki/子进程)需要保留表项以允许其[父进程](https://zh.wikipedia.org/wiki/父进程)读取子进程的退出状态 --- 维基百科*

以下是个例子:

```C++
#include <iostream>
#include <sys/wait.h>

using namespace std;

int main()
{
    pid_t pid = fork(); // 调用fork()生成子进程
    if (pid < 0)
    {
        cout << "Error creating";
        exit(1);
    }
    else if (pid == 0) // 如果是子进程
    {
        exit(1); // 直接退出，造成zombie状态
    }
    else // 如果是父进程
    {
        int a;
        cin >> a; // 造成等待
    }
    return 0;
}
```

运行后，子进程成为僵尸进程

```bash
root@hao6:~# ps aux | grep demo
root       64497  0.0  0.0   6048  1888 pts/1    S+   10:19   0:00 ./demo
root       64498  0.0  0.0      0     0 pts/1    Z+   10:19   0:00 [demo] <defunct>
root       64520  0.0  0.0   7004  2112 pts/3    S+   10:19   0:00 grep --color=auto demo
```

我们对上面的示例代码进行改造，父进程中使用waitpid()对子进程进行收割(reap)

```C++
int main()
{
    ...
	else // 如果是父进程
    {
        waitpid(pid, nullptr, 0); // 等待子进程退出
        int a;
        cin >> a;
    }
    ...
}
```

运行后，子进程正常退出

```bash
root@hao6:~# ps aux | grep demo
root       64577  0.0  0.0   6048  1924 pts/1    S+   10:31   0:00 ./demo
root       64616  0.0  0.0   7004  2128 pts/3    S+   10:32   0:00 grep --color=auto demo
```



## 一探究竟

*一旦退出态通过`wait`[系统调用](https://zh.wikipedia.org/wiki/系统调用)读取，僵尸进程条目就从进程表中删除，称之为"回收"（reaped）。正常情况下，进程直接被其父进程`wait`并由系统回收。 --- 维基百科*

那么问题来了，到底是如何触发父进程回收子进程的呢？

sys/wait.h中有着waitpid函数的定义

```C
extern __pid_t waitpid (__pid_t __pid, int *__stat_loc, int __options);
```

我们使用strace工具看一下，waitpid最终指向了哪个系统调用

```bash
root@hao6:~# strace ./demo
...
wait4(64730, NULL, 0, NULL)             = 64730
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=64730, si_uid=0, si_status=1, si_utime=0, si_stime=0} ---
...
```

根据关键字进行查找，发现程序执行了wait4这个系统调用。在内核源码中，寻找wait4函数，我们来一探究竟。

```bash
root@hao6:/work/linux-5.15.94# grep -nr 'SYSCALL_DEFINE'| grep 'wait4'
arch/alpha/kernel/osf_sys.c:1111:SYSCALL_DEFINE4(osf_wait4, pid_t, pid, int __user *, ustatus, int, options,
kernel/exit.c:1773:SYSCALL_DEFINE4(wait4, pid_t, upid, int __user *, stat_addr,
kernel/exit.c:1800:COMPAT_SYSCALL_DEFINE4(wait4,
```

定义在kernel/exit.c的1773行，主要调用了kernel_wait4()函数

```c
SYSCALL_DEFINE4(wait4, pid_t, upid, int __user *, stat_addr,
		int, options, struct rusage __user *, ru)
{
	struct rusage r;
	long err = kernel_wait4(upid, stat_addr, options, ru ? &r : NULL);

	if (err > 0) {
		if (ru && copy_to_user(ru, &r, sizeof(struct rusage)))
			return -EFAULT;
	}
	return err;
}
```

kernel_wait4()函数中，根据pid传入的大小进行了判断处理，对wait_opts进行填充，再调用了do_wait()函数

```c
long kernel_wait4(pid_t upid, int __user *stat_addr, int options,
		  struct rusage *ru)
{
	struct wait_opts wo;
	struct pid *pid = NULL;
	enum pid_type type;
	long ret;

	if (options & ~(WNOHANG|WUNTRACED|WCONTINUED|
			__WNOTHREAD|__WCLONE|__WALL))
		return -EINVAL;

	/* -INT_MIN is not defined */
	if (upid == INT_MIN)
		return -ESRCH;

	if (upid == -1)
		type = PIDTYPE_MAX;
	else if (upid < 0) {
		type = PIDTYPE_PGID;
		pid = find_get_pid(-upid);
	} else if (upid == 0) {
		type = PIDTYPE_PGID;
		pid = get_task_pid(current, PIDTYPE_PGID);
	} else /* upid > 0 */ { // 我们demo程序指定了子进程的PID，会走到这个分支
		type = PIDTYPE_PID; // 设置类型
		pid = find_get_pid(upid); // 获取指向pid类型的结构体的指针
	}

	wo.wo_type	= type;
	wo.wo_pid	= pid;
	wo.wo_flags	= options | WEXITED;
	wo.wo_info	= NULL;
	wo.wo_stat	= 0;
	wo.wo_rusage	= ru;
	ret = do_wait(&wo);
	put_pid(pid);
	if (ret > 0 && stat_addr && put_user(wo.wo_stat, stat_addr))
		ret = -EFAULT;

	return ret;
}
```

do_wait()函数的实现，我们这个示例中，最终会调用到do_wait_pid()函数上

```C
static long do_wait(struct wait_opts *wo)
{
	int retval;

	trace_sched_process_wait(wo->wo_pid);

	init_waitqueue_func_entry(&wo->child_wait, child_wait_callback);
	wo->child_wait.private = current;
	add_wait_queue(&current->signal->wait_chldexit, &wo->child_wait);
repeat:
	/*
	 * If there is nothing that can match our criteria, just get out.
	 * We will clear ->notask_error to zero if we see any child that
	 * might later match our criteria, even if we are not able to reap
	 * it yet.
	 */
	wo->notask_error = -ECHILD;
	if ((wo->wo_type < PIDTYPE_MAX) &&
	   (!wo->wo_pid || !pid_has_task(wo->wo_pid, wo->wo_type)))
		goto notask;

	set_current_state(TASK_INTERRUPTIBLE);
	read_lock(&tasklist_lock);

	if (wo->wo_type == PIDTYPE_PID) { // 上层函数置wo_type为PIDTYPE_PID，故会走到这个分支
		retval = do_wait_pid(wo);
		if (retval)
			goto end;
	} else {
		struct task_struct *tsk = current;

		do {
			retval = do_wait_thread(wo, tsk);
			if (retval)
				goto end;

			retval = ptrace_do_wait(wo, tsk);
			if (retval)
				goto end;

			if (wo->wo_flags & __WNOTHREAD)
				break;
		} while_each_thread(current, tsk);
	}
	read_unlock(&tasklist_lock);

notask:
	retval = wo->notask_error;
	if (!retval && !(wo->wo_flags & WNOHANG)) {
		retval = -ERESTARTSYS;
		if (!signal_pending(current)) {
			schedule();
			goto repeat;
		}
	}
end:
	__set_current_state(TASK_RUNNING);
	remove_wait_queue(&current->signal->wait_chldexit, &wo->child_wait);
	return retval;
}
```

do_wait_pid()函数实现

```c
/*
 * Optimization for waiting on PIDTYPE_PID. No need to iterate through child
 * and tracee lists to find the target task.
 */
static int do_wait_pid(struct wait_opts *wo)
{
	bool ptrace;
	struct task_struct *target;
	int retval;

	ptrace = false;
	target = pid_task(wo->wo_pid, PIDTYPE_TGID);
	if (target && is_effectively_child(wo, ptrace, target)) {
		retval = wait_consider_task(wo, ptrace, target);
		if (retval)
			return retval;
	}

	ptrace = true;
	target = pid_task(wo->wo_pid, PIDTYPE_PID); // 找到子进程的task_struct
	if (target && target->ptrace &&
	    is_effectively_child(wo, ptrace, target)) {
		retval = wait_consider_task(wo, ptrace, target); // 调用至此
		if (retval)
			return retval;
	}

	return 0;
}
```

wait_consider_task()函数实现比较长，我们只取关键部分

```c
static int wait_consider_task(struct wait_opts *wo, int ptrace,
				struct task_struct *p)
{
	/*
	 * We can race with wait_task_zombie() from another thread.
	 * Ensure that EXIT_ZOMBIE -> EXIT_DEAD/EXIT_TRACE transition
	 * can't confuse the checks below.
	 */
	int exit_state = READ_ONCE(p->exit_state); // 先获取子进程的退出状态
	int ret;
    ...
	/* slay zombie? */
	if (exit_state == EXIT_ZOMBIE) { // 匹配到子进程成为僵尸进程，走入这个分支
		/* we don't reap group leaders with subthreads */
		if (!delay_group_leader(p)) {
			/*
			 * A zombie ptracee is only visible to its ptracer.
			 * Notification and reaping will be cascaded to the
			 * real parent when the ptracer detaches.
			 */
			if (unlikely(ptrace) || likely(!p->ptrace))
				return wait_task_zombie(wo, p); // 最终调用至此
		}
    ...
```

wait_task_zombie()函数实现，只取了关心的部分

```c
static int wait_task_zombie(struct wait_opts *wo, struct task_struct *p)
{
	int state, status;
    ...
    state = (ptrace_reparented(p) && thread_group_leader(p)) ?
		EXIT_TRACE : EXIT_DEAD;
    ...
	if (state == EXIT_DEAD)
		release_task(p); // 最终在这里释放了该子进程的剩余资源
```

在成功匹配子进程的状态之后，就使用release_task函数对子进程进行了回收，这也是为什么使用waitpid之后，就没有找到前一个示例中所发现的defunct的僵尸进程了。具体的内核如何通过release_task()函数释放进程占用的剩余资源，这里就不继续跟踪了。



## 引用

[wikipedia-僵尸进程](https://zh.wikipedia.org/wiki/%E5%83%B5%E5%B0%B8%E8%BF%9B%E7%A8%8B)
[kernel源码](https://elixir.bootlin.com/linux/v5.10.168/source/kernel/exit.c)


