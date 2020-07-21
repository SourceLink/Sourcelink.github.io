---
layout: post
title:  "Linux内核之异步通知"
date:   2020-07-20
catalog:  true
tags:
    - fasync
    - kernel

---

# 一. 概述

异步通知机制：一旦设备就绪，则主动通知应用程序，这样应用程序根本就不需要查询设备状态，是一种“信号驱动的异步I/O”。


# 二. 应用层

```
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/select.h>
#include <unistd.h>
#include <signal.h>

void sig_func(int signum)
{
    printf("receive a signal from io,signalnum:%d\n", signum);
}

int main(void)
{
    int fd, flag;

    signal(SIGIO, sig_func);                        //sig_func()处理SIGIO信号

    fd = open("/dev/mmap_drv", O_RDWR);
    if(fd < 0)
    {
        perror("open");
        return -1;
    }


    fcntl(fd, F_SETOWN, getpid());                  //设置文件的所有权进程
    flag = fcntl(fd, F_GETFL);                      //获取状态标志
    fcntl(fd, F_SETFL, flag | FASYNC);              //设置FASYNC标志

    while(1) {
        sleep(10);
    }
    return 0;
}
```

主要步骤：

- 1. 先注册信号处理函数  
- 2. 设置设备的所有权进程(驱动层需要知道将信号发给谁)  
- 3. 设置fasync标志, 让设备可以发送信号  

当设备做了上面三个步骤，只要等驱动层发来`SIGIO`信号，`sig_func()`则会被调用执行；

缺点: 当你同时开启了多个设备的异步通知方式，当有信号来临时，你无法区分是哪个设备发来的信号；



# 三. 内核层


其实在看了前面的应用层的代码，相信大家可以看出异步通知是依赖信号实现的，但是这篇笔记主要讲异步通知部分，
现在来看下应用层是如何告诉驱动层该设备所有权进程和设置fasync标志；

## 3.1 SyS_fcntl

> fs/fcntl.c

内核中有个32位机器和64位机器，根据自己机器选择:

```
SYSCALL_DEFINE3(fcntl64, unsigned int, fd, unsigned int, cmd, unsigned long, arg)
SYSCALL_DEFINE3(fcntl, unsigned int, fd, unsigned int, cmd, unsigned long, arg)
```


```
SYSCALL_DEFINE3(fcntl64, unsigned int, fd, unsigned int, cmd,
		unsigned long, arg)
{
	struct fd f = fdget_raw(fd);
	....
	switch (cmd) {
    ....
	default:
		err = do_fcntl(fd, cmd, arg, f.file);
		break;
	}
    ....
}
```

其中核心函数是调用了`do_fcntl()`, 但是要注意这里的f.file是根据传进来的文件描述符fd获取的；

## 3.2 do_fcntl

```
static long do_fcntl(int fd, unsigned int cmd, unsigned long arg,
		struct file *filp)
{
	long err = -EINVAL;

	switch (cmd) {
	case F_DUPFD:
		err = f_dupfd(arg, filp, 0);
		break;
	case F_DUPFD_CLOEXEC:
		err = f_dupfd(arg, filp, O_CLOEXEC);
		break;
	case F_GETFD:
		err = get_close_on_exec(fd) ? FD_CLOEXEC : 0;
		break;
	case F_SETFD:
		err = 0;
		set_close_on_exec(fd, arg & FD_CLOEXEC);
		break;
	case F_GETFL:
		err = filp->f_flags;
		break;
	case F_SETFL:
		err = setfl(fd, filp, arg);
		break;
#if BITS_PER_LONG != 32
	/* 32-bit arches must use fcntl64() */
	case F_OFD_GETLK:
#endif
	case F_GETLK:
		err = fcntl_getlk(filp, cmd, (struct flock __user *) arg);
		break;
#if BITS_PER_LONG != 32
	/* 32-bit arches must use fcntl64() */
	case F_OFD_SETLK:
	case F_OFD_SETLKW:
#endif
		/* Fallthrough */
	case F_SETLK:
	case F_SETLKW:
		err = fcntl_setlk(fd, filp, cmd, (struct flock __user *) arg);
		break;
	case F_GETOWN:
		err = f_getown(filp);
		force_successful_syscall_return();
		break;
	case F_SETOWN:
		f_setown(filp, arg, 1);
		err = 0;
		break;
	case F_GETOWN_EX:
		err = f_getown_ex(filp, arg);
		break;
	case F_SETOWN_EX:
		err = f_setown_ex(filp, arg);
		break;
	case F_GETOWNER_UIDS:
		err = f_getowner_uids(filp, arg);
		break;
	case F_GETSIG:
		err = filp->f_owner.signum;
		break;
	case F_SETSIG:
		/* arg == 0 restores default behaviour. */
		if (!valid_signal(arg)) {
			break;
		}
		err = 0;
		filp->f_owner.signum = arg;
		break;
	case F_GETLEASE:
		err = fcntl_getlease(filp);
		break;
	case F_SETLEASE:
		err = fcntl_setlease(fd, filp, arg);
		break;
	case F_NOTIFY:
		err = fcntl_dirnotify(fd, filp, arg);
		break;
	case F_SETPIPE_SZ:
	case F_GETPIPE_SZ:
		err = pipe_fcntl(filp, cmd, arg);
		break;
	case F_ADD_SEALS:
	case F_GET_SEALS:
		err = shmem_fcntl(filp, cmd, arg);
		break;
	default:
		break;
	}
	return err;
}
```

`do_fcntl`可以对应很多cmd， 这里我们只重点关注`F_SETOWN`和`F_SETFL`命令;

### 3.2.1 F_SETOWN

```
	case F_SETOWN:
		f_setown(filp, arg, 1);
		err = 0;
		break;
```

上面是`F_SETOWN`所对应的代码片段， 现在看下`f_setown()`函数， 注意arg是传递进来的进程pid；

```
void f_setown(struct file *filp, unsigned long arg, int force)
{
	enum pid_type type;
	struct pid *pid;
	int who = arg;
	type = PIDTYPE_PID;
	if (who < 0) {
		type = PIDTYPE_PGID;
		who = -who;
	}
	rcu_read_lock();
	pid = find_vpid(who);
	__f_setown(filp, pid, type, force);
	rcu_read_unlock();
}
```

```
void __f_setown(struct file *filp, struct pid *pid, enum pid_type type,
		int force)
{
	security_file_set_fowner(filp);
	f_modown(filp, pid, type, force);
}
```

```
static void f_modown(struct file *filp, struct pid *pid, enum pid_type type,
                     int force)
{
	write_lock_irq(&filp->f_owner.lock);
	if (force || !filp->f_owner.pid) {
		put_pid(filp->f_owner.pid);
		filp->f_owner.pid = get_pid(pid);
		filp->f_owner.pid_type = type;

		if (pid) {
			const struct cred *cred = current_cred();
			filp->f_owner.uid = cred->uid;
			filp->f_owner.euid = cred->euid;
		}
	}
	write_unlock_irq(&filp->f_owner.lock);
}
```

请注意其中的代码片段：

```
		filp->f_owner.pid = get_pid(pid);
```

将进程的pid存放在filp->f_owner.pid变量中，而这个filp变量是通过fd获取到的(即fd与filp是一一对应的);
每个进程只有在打开设备或者文件的时候才会获取到fd, 而且不同的进程打开同一设备或文件获取的fd是不同的；

所以可以说filp是与进程一一对应的；


### 3.2.2 F_SETFL

```
	case F_SETFL:
		err = setfl(fd, filp, arg);
		break;
```

```
static int setfl(int fd, struct file * filp, unsigned long arg)
{
	.....
	/*
	 * ->fasync() is responsible for setting the FASYNC bit.
	 */
	if (((arg ^ filp->f_flags) & FASYNC) && filp->f_op->fasync) {
		error = filp->f_op->fasync(fd, filp, (arg & FASYNC) != 0);      // 执行对应驱动层的fasync服务函数
		if (error < 0)
			goto out;
		if (error > 0)
			error = 0;
	}
	spin_lock(&filp->f_lock);
	filp->f_flags = (arg & SETFL_MASK) | (filp->f_flags & ~SETFL_MASK);
	spin_unlock(&filp->f_lock);

 out:
	return error;
}
```

这个函数的主要功能是执行驱动层的fasync服务函数；


# 四. 驱动层

现在驱动层的代码片段如下：


```
static struct fasync_struct *test_async;
static int drv_fasync (int fd, struct file *filp, int on)
{
	return fasync_helper(fd, filp, on, &test_async);
}
```

接下来看下`fasync_helper()`中做了什么事情；

## 4.1 fasync_helper



```
int fasync_helper(int fd, struct file * filp, int on, struct fasync_struct **fapp)
{
	if (!on)
		return fasync_remove_entry(filp, fapp);
	return fasync_add_entry(fd, filp, fapp);
}
```

### 4.1.1 fasync_add_entry

```
static int fasync_add_entry(int fd, struct file *filp, struct fasync_struct **fapp)
{
	struct fasync_struct *new;

	new = fasync_alloc();                              // 分配一个fasync结构体内存
	if (!new)
		return -ENOMEM;

	if (fasync_insert_entry(fd, filp, fapp, new)) {    // 详见4.3
		fasync_free(new);
		return 0;
	}

	return 1;
}
```

### 4.1.2 fasync_insert_entry

```
struct fasync_struct *fasync_insert_entry(int fd, struct file *filp, struct fasync_struct **fapp, struct fasync_struct *new)
{
        struct fasync_struct *fa, **fp;

	spin_lock(&filp->f_lock);
	spin_lock(&fasync_lock);
	for (fp = fapp; (fa = *fp) != NULL; fp = &fa->fa_next) {
		if (fa->fa_file != filp)
			continue;

		spin_lock_irq(&fa->fa_lock);
		fa->fa_fd = fd;
		spin_unlock_irq(&fa->fa_lock);
		goto out;
	}

	spin_lock_init(&new->fa_lock);
	new->magic = FASYNC_MAGIC;
	new->fa_file = filp;            // 记录filp指针，该变量记录了pid
	new->fa_fd = fd;                // 记录所属的fd
	new->fa_next = *fapp;           // 将其挂载到fapp链表
	rcu_assign_pointer(*fapp, new);
	filp->f_flags |= FASYNC;

out:
	spin_unlock(&fasync_lock);
	spin_unlock(&filp->f_lock);
	return fa;
}
```

分析到这里可以知道， 前面传进来的`test_async`变量拥有一个单向链表，这样就可以注册多个进程拥有者， 即在发送信号的时候可以同时通知多个进程；

## 4.2 kill_fasync

```
kill_fasync(&test_async, SIGIO, POLL_IN);
```

驱动中是使用上述函数进行信号的发送，这里面涉及到了信号相关的知识， 这里暂且不管，先关注异步通知部分；


```
void kill_fasync(struct fasync_struct **fp, int sig, int band)
{
	/* First a quick test without locking: usually
	 * the list is empty.
	 */
	if (*fp) {
		rcu_read_lock();
		kill_fasync_rcu(rcu_dereference(*fp), sig, band);           // 调用该函数发送信号
		rcu_read_unlock();
	}
}
```

```
static void kill_fasync_rcu(struct fasync_struct *fa, int sig, int band)
{
	while (fa) {
		struct fown_struct *fown;
		unsigned long flags;

		if (fa->magic != FASYNC_MAGIC) {
			printk(KERN_ERR "kill_fasync: bad magic number in "
			       "fasync_struct!\n");
			return;
		}
		spin_lock_irqsave(&fa->fa_lock, flags);
		if (fa->fa_file) {
			fown = &fa->fa_file->f_owner;                              // 从fa变量中获取进程的拥有者，进程的pid
			/* Don't send SIGURG to processes which have not set a
			   queued signum: SIGURG has its own default signalling
			   mechanism. */
			if (!(sig == SIGURG && fown->signum == 0))
				send_sigio(fown, fa->fa_fd, band);                    // 发送信号
		}
		spin_unlock_irqrestore(&fa->fa_lock, flags);
		fa = rcu_dereference(fa->fa_next);                            // 获取下一个fasync_struct变量
	}
}
```

这个函数会迭代fa中的单向链表，依次给其进程拥有者发送信号；

# 五. 关键数据结构

```c
struct pid
{
	atomic_t count;
	unsigned int level;
	/* lists of tasks that use this pid */
	struct hlist_head tasks[PIDTYPE_MAX];
	struct rcu_head rcu;
	struct upid numbers[1];
};
```


```c
struct fd {
	struct file *file;
	unsigned int flags;
};
```

```c
struct file {
	union {
		struct llist_node	fu_llist;
		struct rcu_head 	fu_rcuhead;
	} f_u;
	struct path		f_path;
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;

	/*
	 * Protects f_ep_links, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t		f_lock;
	atomic_long_t		f_count;
	unsigned int 		f_flags;
	fmode_t			f_mode;
	struct mutex		f_pos_lock;
	loff_t			f_pos;
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;

	u64			f_version;
#ifdef CONFIG_SECURITY
	void			*f_security;
#endif
	/* needed for tty driver, and maybe others */
	void			*private_data;

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct list_head	f_ep_links;
	struct list_head	f_tfile_llink;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space	*f_mapping;
} __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */
```




