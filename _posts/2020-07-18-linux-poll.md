---
layout: post
title:  "Linux内核之poll机制"
date:   2020-07-18
catalog:  true
tags:
    - kernel
    - poll

---

# 一. 概述

我们在应用层经常会用poll()去"轮询"文件描述符(Linux下一切皆文件); 相对于使用read()一遍遍的去读(这样才是真的轮询), 效率会高的多;
现在结合驱动来分析下poll机制的实现;

> 内核版本: linux4.1.15
> 主机: imx6ul

# 二. 应用层


应用层调用poll的代码片段大致如下：

```
struct pollfd fds[1];

fds[0].fd = fd;
fds[0].events = POLLIN;

poll(fds, 1, 1000);
```

查看过驱动的`struct file_operations`知道其中有个poll方法，所以在应用层调用poll函数，势必会调用驱动层对应的poll方法， 这是一个典型的系统调用；


# 三. 内核层

现在看下这个syscall的调用流程，下图是一次poll的调用流程:  

![](/images/kernel/20200716223313938_90104301.png)

在内核代码中搜索`SyS_poll`未搜索到，猜测这个函数被宏封装的(SYSCALL_DEFINE3)， 经过搜索查找到以下函数：

## 3.1 SyS_poll

> fs/select.c

```
SYSCALL_DEFINE3(poll, struct pollfd __user *, ufds, unsigned int, nfds,
		int, timeout_msecs)
{
	struct timespec end_time, *to = NULL;
	int ret;

	if (timeout_msecs >= 0) {
		to = &end_time;
		poll_select_set_timeout(to, timeout_msecs / MSEC_PER_SEC,
			NSEC_PER_MSEC * (timeout_msecs % MSEC_PER_SEC));                // 根据传送进来的超时时间计算出绝对时间给to变量
	}

	ret = do_sys_poll(ufds, nfds, to);                                      // 详解3.2

	......

}
```


## 3.2 do_sys_pol

```
int do_sys_poll(struct pollfd __user *ufds, unsigned int nfds,
		struct timespec *end_time)
{
	struct poll_wqueues table;
 	int err = -EFAULT, fdcount, len, size;
	long stack_pps[POLL_STACK_ALLOC/sizeof(long)];
	struct poll_list *const head = (struct poll_list *)stack_pps;
 	struct poll_list *walk = head;
 	unsigned long todo = nfds;

	if (nfds > rlimit(RLIMIT_NOFILE))
		return -EINVAL;

	len = min_t(unsigned int, nfds, N_STACK_PPS);                           // 可以看到len有个最小值
	for (;;) {
		walk->next = NULL;
		walk->len = len;
		if (!len)
			break;

		if (copy_from_user(walk->entries, ufds + nfds-todo,                 ①
					sizeof(struct pollfd) * walk->len))
			goto out_fds;

		todo -= walk->len;
		if (!todo)
			break;

		len = min(todo, POLLFD_PER_PAGE);
		size = sizeof(struct poll_list) + sizeof(struct pollfd) * len;      ②
		walk = walk->next = kmalloc(size, GFP_KERNEL);
		if (!walk) {
			err = -ENOMEM;
			goto out_fds;
		}
	}

	poll_initwait(&table);                                                  ③
	fdcount = do_poll(nfds, head, &table, end_time);                        ④
	poll_freewait(&table);

	for (walk = head; walk; walk = walk->next) {
		struct pollfd *fds = walk->entries;
		int j;

		for (j = 0; j < walk->len; j++, ufds++)
			if (__put_user(fds[j].revents, &ufds->revents))
				goto out_fds;
  	}

	err = fdcount;
out_fds:
	walk = head->next;
	while (walk) {
		struct poll_list *pos = walk;
		walk = walk->next;
		kfree(pos);
	}

	return err;
}
```

> ①: 将用户空间的fds挨个拷贝到内核空间的`struct poll_list`结构体中， 它将会以链表的形式串接在一起

```
struct poll_list {
	struct poll_list *next;
	int len;
	struct pollfd entries[0];
};
```

可以看到其中用于存储pollfd的变量是个零长度的数组；

> ②: 结合前面可以分析出每个poll_list都存有一组pollfd， 而这个len大小都是更具前面min()结合剩余fd数量等计算出
> ③: 初始化struct poll_wqueues结构(等待队列)，主要为其poll_wait注册一个服务函数，后面再详细讲解
```
void poll_initwait(struct poll_wqueues *pwq)
{
	init_poll_funcptr(&pwq->pt, __pollwait);
	pwq->polling_task = current;
	pwq->triggered = 0;
	pwq->error = 0;
	pwq->table = NULL;
	pwq->inline_index = 0;
}
```

> ④: 详解3.2.1

总结：从应用层传过来的pollfd会被分成一组组poll_list, 每个poll_list中存有多个pollfd，长度由其中的成员变量len记录；最后每个poll_list块会串成一个链表；

### 3.2.1 do_poll

```
static int do_poll(unsigned int nfds,  struct poll_list *list,
		   struct poll_wqueues *wait, struct timespec *end_time)
{
	poll_table* pt = &wait->pt;
	ktime_t expire, *to = NULL;
	int timed_out = 0, count = 0;
	unsigned long slack = 0;
	unsigned int busy_flag = net_busy_loop_on() ? POLL_BUSY_LOOP : 0;
	unsigned long busy_end = 0;

	/* Optimise the no-wait case */
	if (end_time && !end_time->tv_sec && !end_time->tv_nsec) {
		pt->_qproc = NULL;
		timed_out = 1;
	}

	if (end_time && !timed_out)
		slack = select_estimate_accuracy(end_time);

	for (;;) {
		struct poll_list *walk;
		bool can_busy_loop = false;

		for (walk = list; walk != NULL; walk = walk->next) {                      // 先迭代poll_list
			struct pollfd * pfd, * pfd_end;

			pfd = walk->entries;                                                  // 这个entries上有一组fd， 而fd的数量为len
			pfd_end = pfd + walk->len;
			for (; pfd != pfd_end; pfd++) {                                       // 再迭代调用每个fd对应poll函数
				if (do_pollfd(pfd, pt, &can_busy_loop,
					      busy_flag)) {
					count++;
					pt->_qproc = NULL;
					/* found something, stop busy polling */
					busy_flag = 0;
					can_busy_loop = false;
				}
			}
		}
		pt->_qproc = NULL;                                                          // 清除poll队列的处理函数
		if (!count) {
			count = wait->error;
			if (signal_pending(current))
				count = -EINTR;
		}
		if (count || timed_out)
			break;

		/* only if found POLL_BUSY_LOOP sockets && not out of time */
		if (can_busy_loop && !need_resched()) {
			if (!busy_end) {
				busy_end = busy_loop_end_time();
				continue;
			}
			if (!busy_loop_timeout(busy_end))
				continue;
		}
		busy_flag = 0;

		if (end_time && !to) {
			expire = timespec_to_ktime(*end_time);
			to = &expire;
		}

		if (!poll_schedule_timeout(wait, TASK_INTERRUPTIBLE, to, slack))   // poll的超时处理与调度
			timed_out = 1;
	}
	return count;
}
```

总结：这里使用for循环迭代每个poll_list中存放的pollfd， 根据其中的成员len来迭代；

前面两个双从for循环，将应用层传进来的pollfd全部创建了对应的entry，并且将它们都加入了对应的等待队列；
最后调用了`poll_schedule_timeout()`执行力调度， 从而进程进入了休眠状态；


### 3.2.2 do_pollfd

```
static inline unsigned int do_pollfd(struct pollfd *pollfd, poll_table *pwait,
				     bool *can_busy_poll,
				     unsigned int busy_flag)
{
	unsigned int mask;
	int fd;

	mask = 0;
	fd = pollfd->fd;
	if (fd >= 0) {
		struct fd f = fdget(fd);
		mask = POLLNVAL;
		if (f.file) {
			mask = DEFAULT_POLLMASK;
			if (f.file->f_op->poll) {
				pwait->_key = pollfd->events|POLLERR|POLLHUP;
				pwait->_key |= busy_flag;
				mask = f.file->f_op->poll(f.file, pwait);
				if (mask & busy_flag)
					*can_busy_poll = true;
			}
			/* Mask out unneeded events. */
			mask &= pollfd->events | POLLERR | POLLHUP;            // 这里会过滤下， 看发生的事件是否是应用层需要的
			fdput(f);
		}
	}
	pollfd->revents = mask;

	return mask;
}
```

到这里才是最终调用驱动层所对应的poll处理函数

下图是一个内核层从被应用层调用再到驱动层的一些关键代码段截图：

![](/images/kernel/20200717235048216_1641889516.png)



# 四. 驱动层


驱动层的poll函数关键代码段如下：

```
wait_queue_head_t		test_pollwq;	

static unsigned int test_poll(struct file *file, struct poll_table_struct *poll)
{
    ...
	poll_wait(file, &test_pollwq, poll);
	...
}
```

## 4.1 poll_wait

```
static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
{
	if (p && p->_qproc && wait_address)
		p->_qproc(filp, wait_address, p);
}
```

这个函数只是做了些安全性判断，则调用了poll_table的服务函数，接下来看下前面(3.2节)注册的服务函数

前面已经分析过是使用`poll_initwait()`初始化了poll_wqueues并且注册了poll_table的服务函数：

```
void poll_initwait(struct poll_wqueues *pwq)
{
	init_poll_funcptr(&pwq->pt, __pollwait);   // 为poll队列结构注册了__pollwait处理函数
	pwq->polling_task = current;
	pwq->triggered = 0;
	pwq->error = 0;
	pwq->table = NULL;
	pwq->inline_index = 0;
}
```

注册过程：

```
static inline void init_poll_funcptr(poll_table *pt, poll_queue_proc qproc)
{
	pt->_qproc = qproc;
	pt->_key   = ~0UL; /* all events enabled */
}

```

实际的处理函数：

## 4.2 __pollwait

```
static void __pollwait(struct file *filp, wait_queue_head_t *wait_address,
				poll_table *p)
{
	struct poll_wqueues *pwq = container_of(p, struct poll_wqueues, pt);   // 根据poll_table找到wqueues
	struct poll_table_entry *entry = poll_get_entry(pwq);                  //  为poll_wqueues分配一个table_page, 并从其中获取一个table_entry
	if (!entry)
		return;
	entry->filp = get_file(filp);                    // 记录下是哪个进程
	entry->wait_address = wait_address;
	entry->key = p->_key;
	init_waitqueue_func_entry(&entry->wait, pollwake); // 初始化等待队列并且设置了唤醒函数
	entry->wait.private = pwq;                // 记录poll的wq 待观察
	add_wait_queue(wait_address, &entry->wait);  // 将构造好的等待队列 加入 等待队列链表
}
```

### 4.2.1 poll_get_entry

```
static struct poll_table_entry *poll_get_entry(struct poll_wqueues *p)
{
	struct poll_table_page *table = p->table;

	if (p->inline_index < N_INLINE_POLL_ENTRIES)
		return p->inline_entries + p->inline_index++;

	if (!table || POLL_TABLE_FULL(table)) {
		struct poll_table_page *new_table;

		new_table = (struct poll_table_page *) __get_free_page(GFP_KERNEL);
		if (!new_table) {
			p->error = -ENOMEM;
			return NULL;
		}
		new_table->entry = new_table->entries;
		new_table->next = table;
		p->table = new_table;
		table = new_table;
	}

	return table->entry++;
}
```

可以看到该函数先位poll_wqueues分配了一个table_page， 再从page上返回一个entry供后面的流程使用；

现在看下这个entry数据结构：

```
struct poll_table_entry {
	struct file *filp;                // 记录了文件信息，其中包括进程信息
	unsigned long key;                // poll返回类型
	wait_queue_t wait;                // poll等待队列
	wait_queue_head_t *wait_address;  // 记录poll等待队列的链表头
};
```


所以当驱动程调用了poll_wait后，整个数据结构关系会变成下面这种情况：

![](/images/kernel/20200718114302668_617350654.png)


每个需要poll的文件文件描述符，都会对应一个`poll_table_entry`，它会记录对应的文件信息， 等待队列和需要挂载等待队列头(驱动层知道需要挂载在哪个队列头)；

所以看到这里，捋捋思路， 在驱动层调用了poll_wait函数， 最终会 调用上面这个函数`__pollwait`，
在这个函数创建了等待队列，并且记录一些关键信息， 然后设置唤醒函数， 并且将自身进程加入对应的等待队列链表，
但此时进程并没有进入休眠状态， 因为还没有发生调度；


为什么这里需要设置唤醒函数，可以看看前面linux内核之休眠与唤醒一篇的笔记；

## 4.3 pollwake

现在来详细看下调用了`wake_up`等函数，最终调用我们前面注册的pollwake服务函数:

```
static int pollwake(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
	struct poll_table_entry *entry;

	entry = container_of(wait, struct poll_table_entry, wait);
	if (key && !((unsigned long)key & entry->key))    // 判断下key
		return 0;
	return __pollwake(wait, mode, sync, key);
}
```

接着调用`__pollwake()`:

```
static int __pollwake(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
	struct poll_wqueues *pwq = wait->private;
	DECLARE_WAITQUEUE(dummy_wait, pwq->polling_task);

	smp_wmb();
	pwq->triggered = 1;

	return default_wake_function(&dummy_wait, mode, sync, key);
}
```

这里基本和休眠唤醒笔记讲解的是相同的，同样调用的事default_wake_function() 实现的唤醒；

# 五. 总结

从内核层调用到驱动层时(f.file->f_op->poll)，因为调用poll_wait，实质是调用`__pollwait`服务函数将进程加入了等待队列中，在全部的pollfd加入队列后， 
在`do_poll()`中调用`poll_schedule_timeout()`进行调度，使对应的进程进入休眠状态； 

以下面驱动层代码为例：

```
wait_queue_head_t		test_pollwq;	

static unsigned int test_poll(struct file *file, struct poll_table_struct *poll)
{
    ...
	poll_wait(file, &test_pollwq, poll);
    printk("poll wait return");
	...
}
```

第一次poll_wait被调用时，只是将其加入等待队列，但执行完不会立刻进入休眠，而是会继续执行完printk打印；
在返回内核层，调用了`poll_schedule_timeout()`时才会进行休眠；


等待被唤醒，可以使用`wake_up`等函数进行唤醒， 唤醒后，继续会到前面休眠的地方`poll_schedule_timeout()`，因为do_poll()是个for循环函数，
所以会继续执行驱动层对应的poll函数， 然后再次执行printk打印函数，可以看到do_poll()函数的退出条件：

```
	if (count || timed_out)
			break;
```

一个表示有事件发生， 一个表示超时；

再看do_poll()中的一段代码：

```
		if (do_pollfd(pfd, pt, &can_busy_loop,
					      busy_flag)) {
					count++;
					pt->_qproc = NULL;
					/* found something, stop busy polling */
					busy_flag = 0;
					can_busy_loop = false;
				}
```

可以看到count的变化依赖与`do_pollfd()`的返回值mask, 即驱动层poll函数返回的mask在非零情况下，表示有事件发生，所以应用层是否能被唤醒，主要看驱动层poll的返回的mask；
执行poll_wait仅仅是为了将进程加入等待队列； 

所以驱动层的函数般会这样写：


```
wait_queue_head_t		test_pollwq;	

static unsigned int test_poll(struct file *file, struct poll_table_struct *poll)
{
    unsigned int mask = 0;
	poll_wait(file, &test_pollwq, poll);
    if (event) {
        mask |= POLLIN | POLLRDNORM;
    }
    printk("poll wait return");

	return mask;
}
```


# 附录: 关键数据结构

```
struct pollfd {
	int fd;
	short events;
	short revents;
};
```

```
struct poll_list {
	struct poll_list *next;
	int len;
	struct pollfd entries[0];
};
```

```
struct poll_table_entry {
	struct file *filp;
	unsigned long key;
	wait_queue_t wait;
	wait_queue_head_t *wait_address;
};
```


```
struct poll_wqueues {
	poll_table pt;
	struct poll_table_page *table;
	struct task_struct *polling_task;
	int triggered;
	int error;
	int inline_index;
	struct poll_table_entry inline_entries[N_INLINE_POLL_ENTRIES];
};
```



