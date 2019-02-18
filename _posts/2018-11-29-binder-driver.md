---
layout: post
title:  "Binder机制情景分析之深入驱动"
date:   2018-11-29
catalog:  true
tags:
    - android
    - binder
    - Linux
    - driver
---

# 一. 概述

看过上篇`C服务应用篇`内容你肯定已经了解binder的一个使用过程,但是肯定还会有很多疑问:  

- 1. service注册服务是怎么和ServiceManager联系上的?
- 2. client是怎么根据服务名找到的service进程?
- 3. client获取的handle和service注册到ServiceManager的handle是否相同?
- 4. client通过handle是怎么调用的服务?

这篇开始结合binder驱动进行数据交互的分析;

# 1.1 驱动中重要的数据结构 

| 数据结构 | 说明 |
| ------------ | ------- | 
| binder_proc |  每个使用open打开binder设备文件的进程都会在驱动中创建一个binder_proc的结构, 用来记录该<bar>进程的各种信息和状态.例如:线程表,binder节点表,节点引用表 |
| binder_thread | 每个binder线程在binder驱动中都有一个对应的binder_thread结构.记录了线程相关的信息,例如需要完成的任务等. |
| binder_node | bindder_proc 中有一张binder节点对象表,表项是binder_node结构. |
| binder_ref | binder_proc还有一张节点引用表,表象是binder_ref结构. 保存所引用对象的binder_node指针. |
| binder_buffer | 驱动通过mmap的方式创建了一块大的缓存区,每次binder传输数据,会在缓存区分配一个binder_buffer的结构来保存数据. |

# 1.2 说明

先讲解关于binder应用层调用`binder_open()`相关的公用流程的驱动代码;  


# 二. binder初始化-公共

# 2.1 binder_open

应用层open了binder驱动,对应驱动层代码如下:  

```
static int binder_open(struct inode *nodp, struct file *filp)
{
	struct binder_proc *proc;

	binder_debug(BINDER_DEBUG_OPEN_CLOSE, "binder_open: %d:%d\n",
		     current->group_leader->pid, current->pid);

	proc = kzalloc(sizeof(*proc), GFP_KERNEL);                                        ①
	if (proc == NULL)
		return -ENOMEM;
	get_task_struct(current);                                                         ②
	proc->tsk = current;
	INIT_LIST_HEAD(&proc->todo);                                                      ③
	init_waitqueue_head(&proc->wait);
	proc->default_priority = task_nice(current);

	binder_lock(__func__);

	binder_stats_created(BINDER_STAT_PROC);
	hlist_add_head(&proc->proc_node, &binder_procs);                                  ④
	proc->pid = current->group_leader->pid;
	INIT_LIST_HEAD(&proc->delivered_death);
	filp->private_data = proc;                                                        ⑤

	binder_unlock(__func__);

	if (binder_debugfs_dir_entry_proc) {
		char strbuf[11];

		snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
		proc->debugfs_entry = debugfs_create_file(strbuf, S_IRUGO,
			binder_debugfs_dir_entry_proc, proc, &binder_proc_fops);
	}

	return 0;
}
```

> ①: 为当前进程分配一个`struct binder_proc`宽度空间给proc;  
> ②: 获取当前进程的task结构;  
> ③: 初始化`binder_proc`中的todo链表;  
> ④: 将当前进程的`binder_proc`插入全局变量`binder_procs`中;  
> ⑤: 将proc保存到文件结构中,供下次调用使用;  


`binder_procs`这是一个全局的红黑树变量,该全局变量在binder驱动的最前方使用` static HLIST_HEAD(binder_procs);`进行的初始化;  
binder_open()函数的主要功能是打开binder驱动的设备文件,为当前进程创建和初始化`binder_proc`结构体proc.将proc插入到全局的红黑树`binder_procs`中,供将来查找用;  
同时变量proc还放到file结构的private_data字段中,调用驱动的其他操作时可从file结构中取出代表当前进程的`binder_proc`结构体使用;  


# 2.2 binder_ioctl  

应用层调用`ioctl`时对应的驱动层代码:  

```
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	struct binder_proc *proc = filp->private_data;
	struct binder_thread *thread;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;

	/*pr_info("binder_ioctl: %d:%d %x %lx\n",
			proc->pid, current->pid, cmd, arg);*/

	trace_binder_ioctl(cmd, arg);

	ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	if (ret)
		goto err_unlocked;

	binder_lock(__func__);
	thread = binder_get_thread(proc);                                                ①
	if (thread == NULL) {
		ret = -ENOMEM;
		goto err;
	}

	switch (cmd) {
	case BINDER_WRITE_READ:
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		if (ret)
			goto err;
		break;
	case BINDER_SET_MAX_THREADS:
		if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
			ret = -EINVAL;
			goto err;
		}
		break;
	case BINDER_SET_CONTEXT_MGR:
		ret = binder_ioctl_set_ctx_mgr(filp);
		if (ret)
			goto err;
		break;
	case BINDER_THREAD_EXIT:
		binder_debug(BINDER_DEBUG_THREADS, "%d:%d exit\n",
			     proc->pid, thread->pid);
		binder_free_thread(proc, thread);
		thread = NULL;
		break;
	case BINDER_VERSION: {
		struct binder_version __user *ver = ubuf;

		if (size != sizeof(struct binder_version)) {
			ret = -EINVAL;
			goto err;
		}
		if (put_user(BINDER_CURRENT_PROTOCOL_VERSION,
			     &ver->protocol_version)) {
			ret = -EINVAL;
			goto err;
		}
		break;
	}
	default:
		ret = -EINVAL;
		goto err;
	}
	ret = 0;
err:
	if (thread)
		thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
	binder_unlock(__func__);
	wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	if (ret && ret != -ERESTARTSYS)
		pr_info("%d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
err_unlocked:
	trace_binder_ioctl_done(ret);
	return ret;
}
```

获取binder版本号很简单,发`BINDER_VERSION`命令给驱动,驱动回复`BINDER_CURRENT_PROTOCOL_VERSION`给用户空间;  
这里讲下各个命令的意义:  

| 命令 | 含义 | 数据格式 |  
| ---- | ---- | ---- |
| BINDER_WRITE_READ | 向驱动读取和写入数据.可同时读和写 | struct binder_write_read |
| BINDER_SET_MAX_THREADS | 设置线程池的最大的线程数,达到上限后驱动将不会在通知应用层启动新线程  | size_t |
| BINDER_SET_CONTEXT_MGR | 将本进程设置为binder系统的管理进程,只有servicemanager进程才会使用这个命令且只能调用一次 | int |
| BINDER_THREAD_EXIT | 通知驱动当前线程要退出了,以便驱动清理该线程相关的数据 | int |
| BINDER_VERSION | 获取binder的版本号 | struct binder_version |


但是这有个注意点:  
> ①: 第一次调用`ioctl`时会为该进程创建一个线程;  

# 2.3 binder_get_thread

```
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
	struct binder_thread *thread = NULL;
	struct rb_node *parent = NULL;
	struct rb_node **p = &proc->threads.rb_node;

	while (*p) {                                                                        ①
		parent = *p;
		thread = rb_entry(parent, struct binder_thread, rb_node);

		if (current->pid < thread->pid)
			p = &(*p)->rb_left;
		else if (current->pid > thread->pid)
			p = &(*p)->rb_right;
		else
			break;
	}
	if (*p == NULL) {
		thread = kzalloc(sizeof(*thread), GFP_KERNEL);                                    ②
		if (thread == NULL)
			return NULL;
		binder_stats_created(BINDER_STAT_THREAD);
		thread->proc = proc;
		thread->pid = current->pid;
		init_waitqueue_head(&thread->wait);                                               ③
		INIT_LIST_HEAD(&thread->todo);
		rb_link_node(&thread->rb_node, parent, p);
		rb_insert_color(&thread->rb_node, &proc->threads);                                ④
		thread->looper |= BINDER_LOOPER_STATE_NEED_RETURN;
		thread->return_error = BR_OK;
		thread->return_error2 = BR_OK;
	}
	return thread;
}
```

该函数的主要功能是从当前进程信息表中找到挂在下面的线程,`struct binder_thread`是用挂在进程信息表下`threads`节点的红黑树链表下;  

> ①: 先遍历`threads`节点的红黑树链表;  
> ②: 如果没有查找到,则分配一个`struct binder_thread`长度的空间;  
> ③: 初始化等待队列头节点和thread的todo链表;  
> ④: 将该线程插入到进程的`threads`节点;  

首先先遍历该链表,如果为找到线程信息则创建一个`binder_thread`线程,接着初始化线程等待队列和线程的`todo`链表,再将该线程节点挂在进程的信息表中的threads节点的红黑树中;  


# 2.4 binder_mmap

```
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
	int ret;
	struct vm_struct *area;
	struct binder_proc *proc = filp->private_data;                                   ①
	const char *failure_string;
	struct binder_buffer *buffer;

	if (proc->tsk != current)
		return -EINVAL;

	if ((vma->vm_end - vma->vm_start) > SZ_4M)
		vma->vm_end = vma->vm_start + SZ_4M;

	binder_debug(BINDER_DEBUG_OPEN_CLOSE,
		     "binder_mmap: %d %lx-%lx (%ld K) vma %lx pagep %lx\n",
		     proc->pid, vma->vm_start, vma->vm_end,
		     (vma->vm_end - vma->vm_start) / SZ_1K, vma->vm_flags,
		     (unsigned long)pgprot_val(vma->vm_page_prot));

	if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
		ret = -EPERM;
		failure_string = "bad vm_flags";
		goto err_bad_arg;
	}
	vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;

	mutex_lock(&binder_mmap_lock);
	if (proc->buffer) {
		ret = -EBUSY;
		failure_string = "already mapped";
		goto err_already_mapped;
	}

	area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);                     ②
	if (area == NULL) {
		ret = -ENOMEM;
		failure_string = "get_vm_area";
		goto err_get_vm_area_failed;
	}
	proc->buffer = area->addr;                                                       ③
	proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
	mutex_unlock(&binder_mmap_lock);

#ifdef CONFIG_CPU_CACHE_VIPT
	if (cache_is_vipt_aliasing()) {
		while (CACHE_COLOUR((vma->vm_start ^ (uint32_t)proc->buffer))) {
			pr_info("binder_mmap: %d %lx-%lx maps %p bad alignment\n", proc->pid, vma->vm_start, vma->vm_end, proc->buffer);
			vma->vm_start += PAGE_SIZE;
		}
	}
#endif
	proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL); ④
	if (proc->pages == NULL) {
		ret = -ENOMEM;
		failure_string = "alloc page array";
		goto err_alloc_pages_failed;
	}
	proc->buffer_size = vma->vm_end - vma->vm_start;

	vma->vm_ops = &binder_vm_ops;
	vma->vm_private_data = proc;

	if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)) {    ⑤
		ret = -ENOMEM;
		failure_string = "alloc small buf";
		goto err_alloc_small_buf_failed;
	}
	buffer = proc->buffer;
	INIT_LIST_HEAD(&proc->buffers);
	list_add(&buffer->entry, &proc->buffers);                                                ⑥
	buffer->free = 1;
	binder_insert_free_buffer(proc, buffer);
	proc->free_async_space = proc->buffer_size / 2;
	barrier();
	proc->files = get_files_struct(current);
	proc->vma = vma;
	proc->vma_vm_mm = vma->vm_mm;

	/*pr_info("binder_mmap: %d %lx-%lx maps %p\n",
		 proc->pid, vma->vm_start, vma->vm_end, proc->buffer);*/
	return 0;

err_alloc_small_buf_failed:
	kfree(proc->pages);
	proc->pages = NULL;
err_alloc_pages_failed:
	mutex_lock(&binder_mmap_lock);
	vfree(proc->buffer);
	proc->buffer = NULL;
err_get_vm_area_failed:
err_already_mapped:
	mutex_unlock(&binder_mmap_lock);
err_bad_arg:
	pr_err("binder_mmap: %d %lx-%lx %s failed %d\n",
	       proc->pid, vma->vm_start, vma->vm_end, failure_string, ret);
	return ret;
}
```

> ①: `filp->private_data`保存了我们open设备时创建的`binder_proc`信息;  
> ②: 为用户进程分配一块内核空间作为缓冲区;  
> ③: 把分配的缓冲区指针存放到`binder_proc`的buffer字段;  
> ④: 分配pages空间;  
> ④: 在内核分配一块同样页数的内核空间,并把它的物理内存和前面为用户进程分配的内存地址关联;  
> ⑤: 将刚才分配的内存块加入用户进程内存链表;  


`binder_mmap`函数首先调用`get_vm_area()`分配一块地址空间,这里创建的为虚拟内存,位于用户进程空间,接着调用`binder_update_page_range()`建立虚拟内存到物理内存的映射,这样用户空间和内核空间就能共享一块空间了;  

binder运用了mmap机制,在进程间的数据传输时就减小了拷贝次数;  如果不用mmap,从发送进程拷贝到内核空间调用一次`copy_from_user`,从内核空间到目标进程又需要调用`copy_to_user`,这样就发生两次数据拷贝.但运用了mmap后,只需要把发送的进程用户空间数据拷贝到发送进程的内核空间调用一次`copy_from_user`,因为目标进程内核空间缓存区和发送进程内核空间的缓冲区是共享;  


# 三. ServiceManager 


# 3.1 注册为Manager

## 3.1.1 binder_ioctl_set_ctx_mgr

```
static int binder_ioctl_set_ctx_mgr(struct file *filp)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	kuid_t curr_euid = current_euid();

	if (binder_context_mgr_node != NULL) {
		pr_err("BINDER_SET_CONTEXT_MGR already set\n");
		ret = -EBUSY;
		goto out;
	}
	ret = security_binder_set_context_mgr(proc->tsk);
	if (ret < 0)
		goto out;
	if (uid_valid(binder_context_mgr_uid)) {
		if (!uid_eq(binder_context_mgr_uid, curr_euid)) {
			pr_err("BINDER_SET_CONTEXT_MGR bad uid %d != %d\n",
			       from_kuid(&init_user_ns, curr_euid),
			       from_kuid(&init_user_ns,
					binder_context_mgr_uid));
			ret = -EPERM;
			goto out;
		}
	} else {
		binder_context_mgr_uid = curr_euid;                                    ①
	}
	binder_context_mgr_node = binder_new_node(proc, 0, 0);                   ②
	if (binder_context_mgr_node == NULL) {
		ret = -ENOMEM;
		goto out;
	}
	binder_context_mgr_node->local_weak_refs++;
	binder_context_mgr_node->local_strong_refs++;
	binder_context_mgr_node->has_strong_ref = 1;
	binder_context_mgr_node->has_weak_ref = 1;
out:
	return ret;
}
```

> ①: 保存当前进程的用户id到全局变量`binder_context_mgr_uid`;  
> ②: 为当前进程创建一个`binder_node`节点,保存到全局变量`binder_context_mgr_node`;  

# 3.2 进入循环

调用`ioctl`函数写入`BC_ENTER_LOOPER`命令给驱动,进入循环;  
当调用`ioctl`函数时命令为`BINDER_WRITE_READ`则调用下面函数:  

## 3.2.1 binder_ioctl_write_read

```
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;

	if (size != sizeof(struct binder_write_read)) {
		ret = -EINVAL;
		goto out;
	}
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {                                    ①
		ret = -EFAULT;
		goto out;
	}
	binder_debug(BINDER_DEBUG_READ_WRITE,
		     "%d:%d write %lld at %016llx, read %lld at %016llx\n",
		     proc->pid, thread->pid,
		     (u64)bwr.write_size, (u64)bwr.write_buffer,
		     (u64)bwr.read_size, (u64)bwr.read_buffer);

	if (bwr.write_size > 0) {                                                         ②
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
		trace_binder_write_done(ret);
		if (ret < 0) {
			bwr.read_consumed = 0;
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	if (bwr.read_size > 0) {
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp->f_flags & O_NONBLOCK);
		trace_binder_read_done(ret);
		if (!list_empty(&proc->todo))
			wake_up_interruptible(&proc->wait);
		if (ret < 0) {
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	binder_debug(BINDER_DEBUG_READ_WRITE,
		     "%d:%d wrote %lld of %lld, read return %lld of %lld\n",
		     proc->pid, thread->pid,
		     (u64)bwr.write_consumed, (u64)bwr.write_size,
		     (u64)bwr.read_consumed, (u64)bwr.read_size);
	if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
out:
	return ret;
}
```

> ①: 将用户空间的数据拷贝到内核空间;  
> ②: 这里可以看到驱动判断读写是根据读写buf的size来分辨且读写操作互不干扰;  

```
struct binder_write_read {
	binder_size_t		write_size;	/* bytes to write */
	binder_size_t		write_consumed;	/* bytes consumed by driver */
	binder_uintptr_t	write_buffer;
	binder_size_t		read_size;	/* bytes to read */
	binder_size_t		read_consumed;	/* bytes consumed by driver */
	binder_uintptr_t	read_buffer;
};
```
这个结构体数据很简单,一般只要填写size和buf指针就可以,buf的数据个数C服务应用篇有介绍;   

## 3.2.2 binder_thread_write

因为`binder_thread_write()`太长了所以每次用到了哪个命令再细讲;  

### 3.2.2.1 BC_ENTER_LOOPER

```
		case BC_ENTER_LOOPER:
			binder_debug(BINDER_DEBUG_THREADS,
				     "%d:%d BC_ENTER_LOOPER\n",
				     proc->pid, thread->pid);
			if (thread->looper & BINDER_LOOPER_STATE_REGISTERED) {
				thread->looper |= BINDER_LOOPER_STATE_INVALID;
				binder_user_error("%d:%d ERROR: BC_ENTER_LOOPER called after BC_REGISTER_LOOPER\n",
					proc->pid, thread->pid);
			}
			thread->looper |= BINDER_LOOPER_STATE_ENTERED;
			break;
```

这个命令有解释过告诉该线程进入循环状态,这个线程就是前面第一次调用ioctl创建的`struct binder_thread`结构的线程;  

## 3.2.3 binder_thread_read

接下来进入for循环后,又调用了一次ioctl进行读操作

```
static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, size_t size,
			      binder_size_t *consumed, int non_block)
{
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	int ret = 0;
	int wait_for_proc_work;

	if (*consumed == 0) {                                                        ①
		if (put_user(BR_NOOP, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
	}

retry:
	wait_for_proc_work = thread->transaction_stack == NULL &&                    ②
				list_empty(&thread->todo);

	if (thread->return_error != BR_OK && ptr < end) {
		if (thread->return_error2 != BR_OK) {
			if (put_user(thread->return_error2, (uint32_t __user *)ptr))
				return -EFAULT;
			ptr += sizeof(uint32_t);
			binder_stat_br(proc, thread, thread->return_error2);
			if (ptr == end)
				goto done;
			thread->return_error2 = BR_OK;
		}
		if (put_user(thread->return_error, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
		binder_stat_br(proc, thread, thread->return_error);
		thread->return_error = BR_OK;
		goto done;
	}


	thread->looper |= BINDER_LOOPER_STATE_WAITING;
	if (wait_for_proc_work)                                                     ③
		proc->ready_threads++;

	binder_unlock(__func__);

	trace_binder_wait_for_work(wait_for_proc_work,
				   !!thread->transaction_stack,
				   !list_empty(&thread->todo));
	if (wait_for_proc_work) {
		if (!(thread->looper & (BINDER_LOOPER_STATE_REGISTERED |                  ④
					BINDER_LOOPER_STATE_ENTERED))) {
			binder_user_error("%d:%d ERROR: Thread waiting for process work before calling BC_REGISTER_LOOPER or BC_ENTER_LOOPER (state %x)\n",
				proc->pid, thread->pid, thread->looper);
			wait_event_interruptible(binder_user_error_wait,
						 binder_stop_on_user_error < 2);
		}
		binder_set_nice(proc->default_priority);
		if (non_block) {
			if (!binder_has_proc_work(proc, thread))
				ret = -EAGAIN;
		} else
			ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));    ⑤
	} else {
		if (non_block) {
			if (!binder_has_thread_work(thread))
				ret = -EAGAIN;
		} else
			ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
	}

	binder_lock(__func__);

	if (wait_for_proc_work)                                                 ⑥
		proc->ready_threads--;
	thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

	if (ret)
		return ret;

	while (1) {
		uint32_t cmd;
		struct binder_transaction_data tr;
		struct binder_work *w;
		struct binder_transaction *t = NULL;

		if (!list_empty(&thread->todo)) {                                     ⑦
			w = list_first_entry(&thread->todo, struct binder_work,
					     entry);
		} else if (!list_empty(&proc->todo) && wait_for_proc_work) {          ⑧
			w = list_first_entry(&proc->todo, struct binder_work,
					     entry);
		} else {
			/* no data added */
			if (ptr - buffer == 4 &&
			    !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN))
				goto retry;
			break;
		}

		if (end - ptr < sizeof(tr) + 4)
			break;

		switch (w->type) {
		case BINDER_WORK_TRANSACTION: {
			t = container_of(w, struct binder_transaction, work);              ⑨
		} break;
		case BINDER_WORK_TRANSACTION_COMPLETE: {
			cmd = BR_TRANSACTION_COMPLETE;
			if (put_user(cmd, (uint32_t __user *)ptr))
				return -EFAULT;
			ptr += sizeof(uint32_t);

			binder_stat_br(proc, thread, cmd);
			binder_debug(BINDER_DEBUG_TRANSACTION_COMPLETE,
				     "%d:%d BR_TRANSACTION_COMPLETE\n",
				     proc->pid, thread->pid);

			list_del(&w->entry);                                               ⑩
			kfree(w);
			binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
		} break;
		case BINDER_WORK_NODE: {
			struct binder_node *node = container_of(w, struct binder_node, work);
			uint32_t cmd = BR_NOOP;
			const char *cmd_name;
			int strong = node->internal_strong_refs || node->local_strong_refs;
			int weak = !hlist_empty(&node->refs) || node->local_weak_refs || strong;

			if (weak && !node->has_weak_ref) {
				cmd = BR_INCREFS;
				cmd_name = "BR_INCREFS";
				node->has_weak_ref = 1;
				node->pending_weak_ref = 1;
				node->local_weak_refs++;
			} else if (strong && !node->has_strong_ref) {
				cmd = BR_ACQUIRE;
				cmd_name = "BR_ACQUIRE";
				node->has_strong_ref = 1;
				node->pending_strong_ref = 1;
				node->local_strong_refs++;
			} else if (!strong && node->has_strong_ref) {
				cmd = BR_RELEASE;
				cmd_name = "BR_RELEASE";
				node->has_strong_ref = 0;
			} else if (!weak && node->has_weak_ref) {
				cmd = BR_DECREFS;
				cmd_name = "BR_DECREFS";
				node->has_weak_ref = 0;
			}
			if (cmd != BR_NOOP) {
				if (put_user(cmd, (uint32_t __user *)ptr))
					return -EFAULT;
				ptr += sizeof(uint32_t);
				if (put_user(node->ptr,
					     (binder_uintptr_t __user *)ptr))
					return -EFAULT;
				ptr += sizeof(binder_uintptr_t);
				if (put_user(node->cookie,
					     (binder_uintptr_t __user *)ptr))
					return -EFAULT;
				ptr += sizeof(binder_uintptr_t);

				binder_stat_br(proc, thread, cmd);
				binder_debug(BINDER_DEBUG_USER_REFS,
					     "%d:%d %s %d u%016llx c%016llx\n",
					     proc->pid, thread->pid, cmd_name,
					     node->debug_id,
					     (u64)node->ptr, (u64)node->cookie);
			} else {
				list_del_init(&w->entry);
				if (!weak && !strong) {
					binder_debug(BINDER_DEBUG_INTERNAL_REFS,
						     "%d:%d node %d u%016llx c%016llx deleted\n",
						     proc->pid, thread->pid,
						     node->debug_id,
						     (u64)node->ptr,
						     (u64)node->cookie);
					rb_erase(&node->rb_node, &proc->nodes);
					kfree(node);
					binder_stats_deleted(BINDER_STAT_NODE);
				} else {
					binder_debug(BINDER_DEBUG_INTERNAL_REFS,
						     "%d:%d node %d u%016llx c%016llx state unchanged\n",
						     proc->pid, thread->pid,
						     node->debug_id,
						     (u64)node->ptr,
						     (u64)node->cookie);
				}
			}
		} break;
		case BINDER_WORK_DEAD_BINDER:
		case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
		case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: {
			struct binder_ref_death *death;
			uint32_t cmd;

			death = container_of(w, struct binder_ref_death, work);
			if (w->type == BINDER_WORK_CLEAR_DEATH_NOTIFICATION)
				cmd = BR_CLEAR_DEATH_NOTIFICATION_DONE;
			else
				cmd = BR_DEAD_BINDER;
			if (put_user(cmd, (uint32_t __user *)ptr))
				return -EFAULT;
			ptr += sizeof(uint32_t);
			if (put_user(death->cookie,
				     (binder_uintptr_t __user *)ptr))
				return -EFAULT;
			ptr += sizeof(binder_uintptr_t);
			binder_stat_br(proc, thread, cmd);
			binder_debug(BINDER_DEBUG_DEATH_NOTIFICATION,
				     "%d:%d %s %016llx\n",
				      proc->pid, thread->pid,
				      cmd == BR_DEAD_BINDER ?
				      "BR_DEAD_BINDER" :
				      "BR_CLEAR_DEATH_NOTIFICATION_DONE",
				      (u64)death->cookie);

			if (w->type == BINDER_WORK_CLEAR_DEATH_NOTIFICATION) {
				list_del(&w->entry);
				kfree(death);
				binder_stats_deleted(BINDER_STAT_DEATH);
			} else
				list_move(&w->entry, &proc->delivered_death);
			if (cmd == BR_DEAD_BINDER)
				goto done; /* DEAD_BINDER notifications can cause transactions */
		} break;
		}

		if (!t)
			continue;

		BUG_ON(t->buffer == NULL);
		if (t->buffer->target_node) {
			struct binder_node *target_node = t->buffer->target_node;

			tr.target.ptr = target_node->ptr;
			tr.cookie =  target_node->cookie;
			t->saved_priority = task_nice(current);
			if (t->priority < target_node->min_priority &&
			    !(t->flags & TF_ONE_WAY))
				binder_set_nice(t->priority);
			else if (!(t->flags & TF_ONE_WAY) ||
				 t->saved_priority > target_node->min_priority)
				binder_set_nice(target_node->min_priority);
			cmd = BR_TRANSACTION;                                              ①①
		} else {
			tr.target.ptr = 0;
			tr.cookie = 0;
			cmd = BR_REPLY;
		}
		tr.code = t->code;                                                   ①②
		tr.flags = t->flags;
		tr.sender_euid = from_kuid(current_user_ns(), t->sender_euid);

		if (t->from) {
			struct task_struct *sender = t->from->proc->tsk;

			tr.sender_pid = task_tgid_nr_ns(sender,
							task_active_pid_ns(current));
		} else {
			tr.sender_pid = 0;
		}

		tr.data_size = t->buffer->data_size;
		tr.offsets_size = t->buffer->offsets_size;
		tr.data.ptr.buffer = (binder_uintptr_t)(
					(uintptr_t)t->buffer->data +
					proc->user_buffer_offset);
		tr.data.ptr.offsets = tr.data.ptr.buffer +
					ALIGN(t->buffer->data_size,
					    sizeof(void *));

		if (put_user(cmd, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
		if (copy_to_user(ptr, &tr, sizeof(tr)))                               ①③
			return -EFAULT;
		ptr += sizeof(tr);

		trace_binder_transaction_received(t);
		binder_stat_br(proc, thread, cmd);
		binder_debug(BINDER_DEBUG_TRANSACTION,
			     "%d:%d %s %d %d:%d, cmd %d size %zd-%zd ptr %016llx-%016llx\n",
			     proc->pid, thread->pid,
			     (cmd == BR_TRANSACTION) ? "BR_TRANSACTION" :
			     "BR_REPLY",
			     t->debug_id, t->from ? t->from->proc->pid : 0,
			     t->from ? t->from->pid : 0, cmd,
			     t->buffer->data_size, t->buffer->offsets_size,
			     (u64)tr.data.ptr.buffer, (u64)tr.data.ptr.offsets);

		list_del(&t->work.entry);
		t->buffer->allow_user_free = 1;
		if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
			t->to_parent = thread->transaction_stack;
			t->to_thread = thread;
			thread->transaction_stack = t;
		} else {
			t->buffer->transaction = NULL;
			kfree(t);
			binder_stats_deleted(BINDER_STAT_TRANSACTION);
		}
		break;
	}

done:

	*consumed = ptr - buffer;                                             ①④
	if (proc->requested_threads + proc->ready_threads == 0 &&
	    proc->requested_threads_started < proc->max_threads &&
	    (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
	     BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
	     /*spawn a new thread if we leave this out */) {
		proc->requested_threads++;
		binder_debug(BINDER_DEBUG_THREADS,
			     "%d:%d BR_SPAWN_LOOPER\n",
			     proc->pid, thread->pid);
		if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
			return -EFAULT;
		binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
	}
	return 0;
}

```

> ①: 先判断readbuf结构传进来的值是否为0,为0则返回一个`BR_NOOP`到用户空间,但是此时用户空间线程是睡眠的;
> ②: 如果当前线程的todo链表为空且传送数据栈无数据时,则表示当前进程空闲;  
> ③: 如果当前进程在write`BC_REGISTER_LOOPER or BC_ENTER_LOOPER`前就开始执行读操作,则进入休眠;  
> ④: 进入休眠,唤醒后会检测`binder_has_proc_work`,当前进程是否工作(判断当前进程的todo链表是否为空和thread的looper状态),未工作则继续休眠;  
> ⑤: 进入休眠,唤醒后会检测当前线程是否工作;   
> ⑥: 能走到这说明已经唤醒了,进程的ready线程计数减一,且线程的looper等待状态清除;  
> ⑦: 先查看线程的todo链表中是否有要执行的工作;  
> ⑧: 再检查进程的todo链表中是否有需要执行的工作;  
> ⑨: 通过binder_work节点找到`struct binder_transaction`结构体地址;  
> ⑩:如果工作类型是`TRANSACTION_COMPLETE`则表示工作已经执行完了,可以将此工作从线程或进程的todo链表中删除;  
> ①①: 回复命令`BR_TRANSACTION`或`BR_REPLY`;  
> ①②: 填充回复的数据;  
> ①③: 将`tr`的数据拷贝到用户空间,`ptr`指针指向的是用户空间的那个readbuf;  
> ①④: 最后`consumed`中保存了此次回复数据的长度;  

注意看第十三点:  

```
		tr.data.ptr.buffer = (binder_uintptr_t)(
					(uintptr_t)t->buffer->data +
					proc->user_buffer_offset);
```
拷贝个体用户空间的仅仅是数据buf地址,因为使用了mmap,用户空间可以直接使用这块内存,这里也体现了拷贝一次的效率;  

这里你可以发现read到的数据格式一般都是`BR_NOOP+CMD+数据+CMD+数据....`;  
ServiceManager在刚进入循环开始第一次读操作时,没有其他线程就绪,此时只是返回一个`BR_NOOP`就开始休眠了;


#  3.3 总结流程

![](/images/binder/binder_serviceManager_start.jpg)


# 四. led_control_service

# 4.1 注册服务

调用`led_control_server.c`中:
```
svcmgr_publish(bs, svcmgr, LED_CONTROL_SERVER_NAME, led_control);
```
注册为一个服务者,这里面主要调用了`binder_call()`写入了`BC_TRANSACTION`命令,详细的流程`C服务应用`篇已经写过了,现在就主要结合驱动分析;  
这里需要注意的是`binder_call()`调用是同时读写binder驱动的,先看下写操作再看读操作;  

## 4.1.1 binder_thread_write

### 4.1.1.1 BC_TRANSACTION

```
		case BC_TRANSACTION:
		case BC_REPLY: {
			struct binder_transaction_data tr;

			if (copy_from_user(&tr, ptr, sizeof(tr)))
				return -EFAULT;
			ptr += sizeof(tr);
			binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
			break;
		}

```

先将用户空间`write_buffer`数据拷贝到内核的`struct binder_transaction_data tr`中, 再调用`binder_transaction()`处理这些数据;  


### 4.1.1.2 binder_transaction

这个函数巨长....   

```
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply)
{
	struct binder_transaction *t;
	struct binder_work *tcomplete;
	binder_size_t *offp, *off_end;
	struct binder_proc *target_proc;
	struct binder_thread *target_thread = NULL;
	struct binder_node *target_node = NULL;
	struct list_head *target_list;
	wait_queue_head_t *target_wait;
	struct binder_transaction *in_reply_to = NULL;
	struct binder_transaction_log_entry *e;
	uint32_t return_error;

	e = binder_transaction_log_add(&binder_transaction_log);
	e->call_type = reply ? 2 : !!(tr->flags & TF_ONE_WAY);
	e->from_proc = proc->pid;
	e->from_thread = thread->pid;
	e->target_handle = tr->target.handle;
	e->data_size = tr->data_size;
	e->offsets_size = tr->offsets_size;

	if (reply) {
    ......
	} else {
		if (tr->target.handle) {
			struct binder_ref *ref;

			ref = binder_get_ref(proc, tr->target.handle);                                ①
			if (ref == NULL) {
				binder_user_error("%d:%d got transaction to invalid handle\n",
					proc->pid, thread->pid);
				return_error = BR_FAILED_REPLY;
				goto err_invalid_target_handle;
			}
			target_node = ref->node;
		} else {
			target_node = binder_context_mgr_node;                                        ②
			if (target_node == NULL) {
				return_error = BR_DEAD_REPLY;
				goto err_no_context_mgr_node;
			}
		}
		e->to_node = target_node->debug_id;
		target_proc = target_node->proc;                                                ③
		if (target_proc == NULL) {
			return_error = BR_DEAD_REPLY;
			goto err_dead_binder;
		}
		if (security_binder_transaction(proc->tsk,
						target_proc->tsk) < 0) {
			return_error = BR_FAILED_REPLY;
			goto err_invalid_target_handle;
		}
		if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {                   ④
			struct binder_transaction *tmp;

			tmp = thread->transaction_stack;
			if (tmp->to_thread != thread) {
				binder_user_error("%d:%d got new transaction with bad transaction stack, transaction %d has target %d:%d\n",
					proc->pid, thread->pid, tmp->debug_id,
					tmp->to_proc ? tmp->to_proc->pid : 0,
					tmp->to_thread ?
					tmp->to_thread->pid : 0);
				return_error = BR_FAILED_REPLY;
				goto err_bad_call_stack;
			}
			while (tmp) {
				if (tmp->from && tmp->from->proc == target_proc)                             ⑤
					target_thread = tmp->from;
				tmp = tmp->from_parent;
			}
		}
	}
	if (target_thread) {
		e->to_thread = target_thread->pid;
		target_list = &target_thread->todo;
		target_wait = &target_thread->wait;
	} else {
		target_list = &target_proc->todo;
		target_wait = &target_proc->wait;
	}
	e->to_proc = target_proc->pid;

	/* TODO: reuse incoming transaction for reply */
	t = kzalloc(sizeof(*t), GFP_KERNEL);
	if (t == NULL) {
		return_error = BR_FAILED_REPLY;
		goto err_alloc_t_failed;
	}
	binder_stats_created(BINDER_STAT_TRANSACTION);

	tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
	if (tcomplete == NULL) {
		return_error = BR_FAILED_REPLY;
		goto err_alloc_tcomplete_failed;
	}
	binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);

	t->debug_id = ++binder_last_id;
	e->debug_id = t->debug_id;

	if (reply)
		binder_debug(BINDER_DEBUG_TRANSACTION,
			     "%d:%d BC_REPLY %d -> %d:%d, data %016llx-%016llx size %lld-%lld\n",
			     proc->pid, thread->pid, t->debug_id,
			     target_proc->pid, target_thread->pid,
			     (u64)tr->data.ptr.buffer,
			     (u64)tr->data.ptr.offsets,
			     (u64)tr->data_size, (u64)tr->offsets_size);
	else
		binder_debug(BINDER_DEBUG_TRANSACTION,
			     "%d:%d BC_TRANSACTION %d -> %d - node %d, data %016llx-%016llx size %lld-%lld\n",
			     proc->pid, thread->pid, t->debug_id,
			     target_proc->pid, target_node->debug_id,
			     (u64)tr->data.ptr.buffer,
			     (u64)tr->data.ptr.offsets,
			     (u64)tr->data_size, (u64)tr->offsets_size);

	if (!reply && !(tr->flags & TF_ONE_WAY))
		t->from = thread;
	else
		t->from = NULL;
	t->sender_euid = task_euid(proc->tsk);
	t->to_proc = target_proc;
	t->to_thread = target_thread;
	t->code = tr->code;
	t->flags = tr->flags;
	t->priority = task_nice(current);

	trace_binder_transaction(reply, t, target_node);

	t->buffer = binder_alloc_buf(target_proc, tr->data_size,                      　⑥
		tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
	if (t->buffer == NULL) {
		return_error = BR_FAILED_REPLY;
		goto err_binder_alloc_buf_failed;
	}
	t->buffer->allow_user_free = 0;
	t->buffer->debug_id = t->debug_id;
	t->buffer->transaction = t;
	t->buffer->target_node = target_node;
	trace_binder_transaction_alloc_buf(t->buffer);
	if (target_node)
		binder_inc_node(target_node, 1, 0, NULL);

	offp = (binder_size_t *)(t->buffer->data +
				 ALIGN(tr->data_size, sizeof(void *)));

	if (copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)
			   tr->data.ptr.buffer, tr->data_size)) {
		binder_user_error("%d:%d got transaction with invalid data ptr\n",
				proc->pid, thread->pid);
		return_error = BR_FAILED_REPLY;
		goto err_copy_data_failed;
	}

	if (copy_from_user(offp, (const void __user *)(uintptr_t)
			   tr->data.ptr.offsets, tr->offsets_size)) {
		binder_user_error("%d:%d got transaction with invalid offsets ptr\n",
				proc->pid, thread->pid);
		return_error = BR_FAILED_REPLY;
		goto err_copy_data_failed;
	}
	if (!IS_ALIGNED(tr->offsets_size, sizeof(binder_size_t))) {
		binder_user_error("%d:%d got transaction with invalid offsets size, %lld\n",
				proc->pid, thread->pid, (u64)tr->offsets_size);
		return_error = BR_FAILED_REPLY;
		goto err_bad_offset;
	}
	off_end = (void *)offp + tr->offsets_size;
	for (; offp < off_end; offp++) {
		struct flat_binder_object *fp;

		if (*offp > t->buffer->data_size - sizeof(*fp) ||
		    t->buffer->data_size < sizeof(*fp) ||
		    !IS_ALIGNED(*offp, sizeof(u32))) {
			binder_user_error("%d:%d got transaction with invalid offset, %lld\n",
					  proc->pid, thread->pid, (u64)*offp);
			return_error = BR_FAILED_REPLY;
			goto err_bad_offset;
		}
		fp = (struct flat_binder_object *)(t->buffer->data + *offp);                  　 ⑦
		switch (fp->type) {
		case BINDER_TYPE_BINDER:
		case BINDER_TYPE_WEAK_BINDER: {
			struct binder_ref *ref;
			struct binder_node *node = binder_get_node(proc, fp->binder);                　⑧

			if (node == NULL) {
				node = binder_new_node(proc, fp->binder, fp->cookie);
				if (node == NULL) {
					return_error = BR_FAILED_REPLY;
					goto err_binder_new_node_failed;
				}
				node->min_priority = fp->flags & FLAT_BINDER_FLAG_PRIORITY_MASK;
				node->accept_fds = !!(fp->flags & FLAT_BINDER_FLAG_ACCEPTS_FDS);
			}
			if (fp->cookie != node->cookie) {
				binder_user_error("%d:%d sending u%016llx node %d, cookie mismatch %016llx != %016llx\n",
					proc->pid, thread->pid,
					(u64)fp->binder, node->debug_id,
					(u64)fp->cookie, (u64)node->cookie);
				return_error = BR_FAILED_REPLY;
				goto err_binder_get_ref_for_node_failed;
			}
			if (security_binder_transfer_binder(proc->tsk,
							    target_proc->tsk)) {
				return_error = BR_FAILED_REPLY;
				goto err_binder_get_ref_for_node_failed;
			}
			ref = binder_get_ref_for_node(target_proc, node);                            ⑨
			if (ref == NULL) {
				return_error = BR_FAILED_REPLY;
				goto err_binder_get_ref_for_node_failed;
			}
			if (fp->type == BINDER_TYPE_BINDER)                                          ⑩
				fp->type = BINDER_TYPE_HANDLE;
			else
				fp->type = BINDER_TYPE_WEAK_HANDLE;
			fp->handle = ref->desc;                                                      ①①
			binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
				       &thread->todo);

			trace_binder_transaction_node_to_ref(t, node, ref);
			binder_debug(BINDER_DEBUG_TRANSACTION,
				     "        node %d u%016llx -> ref %d desc %d\n",
				     node->debug_id, (u64)node->ptr,
				     ref->debug_id, ref->desc);
		} break;

	.........

if (reply) {
		BUG_ON(t->buffer->async_transaction != 0);
		binder_pop_transaction(target_thread, in_reply_to);
	} else if (!(t->flags & TF_ONE_WAY)) {
		BUG_ON(t->buffer->async_transaction != 0);
		t->need_reply = 1;
		t->from_parent = thread->transaction_stack;
		thread->transaction_stack = t;
	} else {
		BUG_ON(target_node == NULL);
		BUG_ON(t->buffer->async_transaction != 1);
		if (target_node->has_async_transaction) {
			target_list = &target_node->async_todo;
			target_wait = NULL;
		} else
			target_node->has_async_transaction = 1;
	}
	t->work.type = BINDER_WORK_TRANSACTION;
	list_add_tail(&t->work.entry, target_list);                                     ①②
	tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
	list_add_tail(&tcomplete->entry, &thread->todo);
	if (target_wait)
		wake_up_interruptible(target_wait);
	return;

.....

}
```

> ①: 目标handle不为0时说明是客户端调用服务端的情况;  
> ②: 目标handle为0,说明是请求ServiceManager服务,保存目标节点`struct binder_node`;  
> ③: 根据`binder_node`获取到`binder_proc`;  
> ④: 判断此次调用是否需要reply;  
> ⑤: 根据`transaction_stack`找到目标线程(第一次传输不会进来);  
> ⑥: 目标进程的在mmap空间分配一块buf,接着调用`copy_from_use`将用户空间数据拷贝进刚分配的buf中,这样目标进程可以直接读取数据;  
> ⑦: 获取`struct flat_binder_object`的首地址, `offp`保存的是object距数据头的偏移值,详细可以看下`C服务应用`的3.2节;  
> ⑧: 为新传进来的binder实体构造一个`binder_node`;  
> ⑨: 查看目标进程的`refs_by_node`红黑树上是否有ref指向该节点,如果没有则为该目标进程创建一个ref指向该node;   
> ⑩: 原来的类型是binder实体,现在要传给ServiceManager就需要改变为handle引用类型;  
> ①①: 把刚才创建的ref中的des赋值给等下要串给应用层的数据, 接着给该节点增加一次引用标记且会将一些事务加入自身线程的todo链表;  
> ①②: 将需要处理的事务加入目标进程或目标线程的todo链表,并唤醒它;  

现在`ServiceManager`进程的`binder_proc`中的`refs_by_node`红黑树上挂有一个新的`binder_ref`指向了传进来的binder实体;
且这个新挂上去的`binder_ref`中`desc`成员为1(即传给应用层的handle),因为这是第一个指向该节点的引用,以后会递增;  

## 4.1.2 binder_thread_read

- 第一次read:  
在写操作完后就开始读操作了,因为刚开始进程和线程的todo链表中没有需要处理的事务,再回复了`BR_NOOP`后就开始睡眠了;  

写操作的时候有为创建的binder实体的node增加引用并加入了todo链表,这时`led_control_service`进程被唤醒;  
开始处理`BINDER_WORK_NODE`事务,命令为`BR_INCREFS`, `BR_ACQUIRE`等;  

- 第二次read:  
这次read是在处理完`BR_INCREFS`, `BR_ACQUIRE`等命令以后,又一次读数据,并进入睡眠;  

PS: 这可以先不看,继续往下看ServiceManager被唤醒的流程;  

好了,回到`led_control_service`进程了,被`ServiceManager`唤醒了;   
这也没做啥事,就是构造数据后将其传回了用户空间;  
接着发送释放buf的命令给内核空间,让它释放了内核mmap分配的数据;  

## 4.1.3 ServiceManager被唤醒

### 4.1.3.1 binder_thread_read

该函数解析详细看3.2.3,这里说下ServiceManager进程被唤醒后做了哪些事情,
> ①: 先取出进程todo链表中需要处理的事务;  
> ②: 再找到处理事务的`struct binder_transaction`结构体地址,取出刚才从发送进程拷贝进mmap分配的空间中数据进行处理;  
> ③: 发送给ServiceManager的用户空间;  

接着用户空间就会开始处理数据,数据格式如下:  

![](/images/binder/binder_transaction_format.png)

用户空间在在收到数据后解析到`BR_TRANSACTION`命令后做了的流程分析见`C服务应用`篇2.4节;  


### 4.1.3.2 binder_thread_write

- 1. 应用层在调用do_add_serivice时最后还向驱动写入了两个命令(`BC_ACQUIRE`和`BC_REQUEST_DEATH_NOTIFICATION`);  
- 2. 执行完fun后写入reply,又写入两个命令(`BC_FREE_BUFFER`和`BC_REPLY`); 

#### 4.1.3.2.1 BC_ACQUIRE

```
		switch (cmd) {
		case BC_INCREFS:
		case BC_ACQUIRE:
		case BC_RELEASE:
		case BC_DECREFS: {
			uint32_t target;
			struct binder_ref *ref;
			const char *debug_string;

			if (get_user(target, (uint32_t __user *)ptr))
				return -EFAULT;
			ptr += sizeof(uint32_t);
			if (target == 0 && binder_context_mgr_node &&                                    ①
			    (cmd == BC_INCREFS || cmd == BC_ACQUIRE)) {
				ref = binder_get_ref_for_node(proc,
					       binder_context_mgr_node);
				if (ref->desc != target) {
					binder_user_error("%d:%d tried to acquire reference to desc 0, got %d instead\n",
						proc->pid, thread->pid,
						ref->desc);
				}
			} else
				ref = binder_get_ref(proc, target);
			if (ref == NULL) {
				binder_user_error("%d:%d refcount change on invalid ref %d\n",
					proc->pid, thread->pid, target);
				break;
			}
			switch (cmd) {
			case BC_INCREFS:
				debug_string = "IncRefs";
				binder_inc_ref(ref, 0, NULL);
				break;
			case BC_ACQUIRE:                                                                ②
				debug_string = "Acquire";
				binder_inc_ref(ref, 1, NULL);
				break;
			case BC_RELEASE:
				debug_string = "Release";
				binder_dec_ref(ref, 1);
				break;
			case BC_DECREFS:
			default:
				debug_string = "DecRefs";
				binder_dec_ref(ref, 0);
				break;
			}
			binder_debug(BINDER_DEBUG_USER_REFS,
				     "%d:%d %s ref %d desc %d s %d w %d for node %d\n",
				     proc->pid, thread->pid, debug_string, ref->debug_id,
				     ref->desc, ref->strong, ref->weak, ref->node->debug_id);
			break;
		}
```

> ①: 这个是根据传进来的handle获取`binder_ref`;  
> ②: 对刚才获取到ref增加强引用;  


#### 4.1.3.2.2 BC_REQUEST_DEATH_NOTIFICATION

```
case BC_REQUEST_DEATH_NOTIFICATION:
		case BC_CLEAR_DEATH_NOTIFICATION: {
			uint32_t target;
			binder_uintptr_t cookie;
			struct binder_ref *ref;
			struct binder_ref_death *death;

			if (get_user(target, (uint32_t __user *)ptr))
				return -EFAULT;
			ptr += sizeof(uint32_t);
			if (get_user(cookie, (binder_uintptr_t __user *)ptr))
				return -EFAULT;
			ptr += sizeof(binder_uintptr_t);
			ref = binder_get_ref(proc, target);                                    ①
			if (ref == NULL) {
				break;
			}

			if (cmd == BC_REQUEST_DEATH_NOTIFICATION) {
				if (ref->death) {
					break;
				}
				death = kzalloc(sizeof(*death), GFP_KERNEL);
				if (death == NULL) {
					thread->return_error = BR_ERROR;
					break;
				}
				binder_stats_created(BINDER_STAT_DEATH);
				INIT_LIST_HEAD(&death->work.entry);
				death->cookie = cookie;
				ref->death = death;                                                  ②
				if (ref->node->proc == NULL) {
					ref->death->work.type = BINDER_WORK_DEAD_BINDER;
					if (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)) {
						list_add_tail(&ref->death->work.entry, &thread->todo);
					} else {
						list_add_tail(&ref->death->work.entry, &proc->todo);
						wake_up_interruptible(&proc->wait);
					}
				}
			} else {
				if (ref->death == NULL) {
					break;
				}
				death = ref->death;
				if (death->cookie != cookie) {
					break;
				}
				ref->death = NULL;
				if (list_empty(&death->work.entry)) {
					death->work.type = BINDER_WORK_CLEAR_DEATH_NOTIFICATION;
					if (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)) {
						list_add_tail(&death->work.entry, &thread->todo);
					} else {
						list_add_tail(&death->work.entry, &proc->todo);
						wake_up_interruptible(&proc->wait);
					}
				} else {
					BUG_ON(death->work.type != BINDER_WORK_DEAD_BINDER);
					death->work.type = BINDER_WORK_DEAD_BINDER_AND_CLEAR;
				}
			}
		} break;
```

> ①: 根据handle获取ref;  
> ②: 讲死亡通知挂到ref的death节点上;  

这样操作后在这个ref指向的node节点的进程(这个场景为led_control_service),在死亡时会反馈给ServiceManager;  


#### 4.1.3.2.3 BC_FREE_BUFFER

```
		case BC_FREE_BUFFER: {
			binder_uintptr_t data_ptr;
			struct binder_buffer *buffer;

			if (get_user(data_ptr, (binder_uintptr_t __user *)ptr))
				return -EFAULT;
			ptr += sizeof(binder_uintptr_t);

			buffer = binder_buffer_lookup(proc, data_ptr);                            ①
			if (buffer == NULL) {
				break;
			}
			if (!buffer->allow_user_free) {
				break;
			}

			if (buffer->transaction) {
				buffer->transaction->buffer = NULL;
				buffer->transaction = NULL;
			}
			if (buffer->async_transaction && buffer->target_node) {
				BUG_ON(!buffer->target_node->has_async_transaction);
				if (list_empty(&buffer->target_node->async_todo))
					buffer->target_node->has_async_transaction = 0;
				else
					list_move_tail(buffer->target_node->async_todo.next, &thread->todo);
			}
			trace_binder_transaction_buffer_release(buffer);
			binder_transaction_buffer_release(proc, buffer, NULL);                   ②
			binder_free_buf(proc, buffer);
			break;
		}
```

> ①: 根据`data.ptr.buffer`的地址找到前面为拷贝`led_control_service`写入内核的数据而分配的mmap缓存区地址(详见4.1.1.2);
> ②: 释放那块buf;  

这里需要注意`data_ptr`虽然是用户空间传来的,但是这也是由内核空间拷贝给用户空间的且该值在用户空间未改变;  

#### 4.1.3.3.4 BC_REPLY

```
		case BC_TRANSACTION:
		case BC_REPLY: {
			struct binder_transaction_data tr;

			if (copy_from_user(&tr, ptr, sizeof(tr)))
				return -EFAULT;
			ptr += sizeof(tr);
			binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
			break;
		}
```

这个流程在前面将`BC_TRANSACTION`为讲解,这里我们单独讲解下;  


```
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply)
{

    .......

    if (reply) {
        in_reply_to = thread->transaction_stack;
    if (in_reply_to == NULL) {
        binder_user_error("%d:%d got reply transaction with no transaction stack\n",
        proc->pid, thread->pid);
        return_error = BR_FAILED_REPLY;
        goto err_empty_call_stack;
    }
        binder_set_nice(in_reply_to->saved_priority);
        if (in_reply_to->to_thread != thread) {
            return_error = BR_FAILED_REPLY;
        in_reply_to = NULL;
        goto err_bad_call_stack;
    }
        thread->transaction_stack = in_reply_to->to_parent;                    ①
        target_thread = in_reply_to->from;
        if (target_thread == NULL) {
            return_error = BR_DEAD_REPLY;
        goto err_dead_binder;
    }
        if (target_thread->transaction_stack != in_reply_to) {
            return_error = BR_FAILED_REPLY;
            in_reply_to = NULL;
            target_thread = NULL;
            goto err_dead_binder;
        }
        target_proc = target_thread->proc;                                     ②
    } else {
    ......
    }

    .....

    if (target_thread) {                                                       ③
        e->to_thread = target_thread->pid;
        target_list = &target_thread->todo;
        target_wait = &target_thread->wait;
    } else {
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }
    ......

```

> ①: 从线程的传输栈上找到目标线程(当前进程为ServiceManager进程);  
> ②: 通过目标线程查找到目标进程;  
> ③: 这里可以看出reply是用线程的来完成的,因为是将要处理事情的是线程的todo链表; 

接下来从用户空间拷贝数据,然后在将事务挂到目标线程的todo链表,再唤醒目标线程;  
这样又回到了`led_control_service`进程了,请看4.1.2;  


# 4.2 设置线程上限

```
	case BINDER_SET_MAX_THREADS:
		if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
			ret = -EINVAL;
			goto err;
		}
		break;
```

将上限值拷贝到proc的max_threads成员中保存;  


# 4.3 总结流程

![](/images/binder/binder_server_register.jpg)


# 五. test_client

# 5.1 获取服务

client获取服务和service注册服务的流程几乎一样,流程中涉及到的驱动相关代码,上一篇都有讲解,这里放一个具体流程;  
具体流程如下图:  
![](/images/binder/binder_server_get.png)

# 5.2 调用服务

调用服务的时候,是根据刚才获取的handle去调用和client获取服务的流程相同,只是目标进程变成了`led_control_service`,这里不再重复讲解了,请参考上图;  

# 六.  mmap用点

以下进程都是在内核态的描述;  

# 6.1 内核态

在查看驱动源码时,发现注册服务时`led_control_service`进程将用户空间数据拷贝到内核后,再唤醒`ServiceManager`进程后,`ServiceManager`进程内核空间可以直接使用;  

# 6.2 用户态

还有一点,在`ServiceManager`将内核空间数据拷贝到用户空间时,仅仅只是把刚才在`led_control_service`进程分配的mmap空间的地址传给了` ServiceManager`的用户空间,而用户空间可以通过该地址直接访问数据了;  


以上为mmap在binder的两点用法,跨进程,内核;  

PS: 篇幅太长了,binder的知识点很多,还有好多需要更新;   




