---
layout: post
title:  "【从零开始写RTOS】内核调度"
date:   2019-12-05
catalog:  true
author: Sourcelink
tags:
    - rtos
    - 内核调度

---

# 一. 临界区

在rtos中，进入临界区的实质是关闭中断（可配置优先级的），退出临界区就是开启中断（但是原先系统中断是关闭的情况下无需开启）

修改后：

```
unsigned int port_enter_critical(void)
{
  unsigned int ret = port_cpu_primask_get();

  port_cpu_int_disable();

  return ret;
}


void port_exit_critical(unsigned int _state)
{
  port_cpu_primask_set(_state);
}
```

exp:

```
void fun(void)
{
    unsigned char i = 0;

    unsigned int state = port_enter_critical();

    i++;
    ....

    port_exit_critical(state);
}
```


# 二. 初始化

会创建一个空闲进程， 当mcu没有进程需要运行的时候， 就调用空闲进程运行;


# 三. 启动


首先会更新`sl_ready_process`值，从就绪队列中获取一个进程;


- 1. pendsv优先级设置：

![](/images/rtos/pendsv_prio.png)

- 2. 设置启动flag


- 3. 触发一次上下文切换


# 四. 上下文切换

完整的上下文切换服务函数内容如下：

```
.thumb_func
.type PendSV_Handler, %function
PendSV_Handler:
    cpsid   i                                                      	/* Prevent interruption during context switch */
    mrs     r0, psp                                             	/* PSP is process stack pointer */
    cbz     r0, SWITCH_PROCESS                     					/* Skip register save the first time if R0=0 bl OS_CPU_PendSVHandler_nosave */
   
    subs    r0, r0, #0x20                                       	/* Save remaining regs r4-11 on process stack */
    stm     r0, {r4-r11}

    ldr     r1, =sl_current_process                                 /* sl_current_process->stack_pointer = SP; */
    ldr     r1, [r1]    
    str     r0, [r1]                                          		/* R0 is SP of process being switched out */
                                                            	    /* At this point, entire context of process has been saved */
SWITCH_PROCESS:	
    ldr     r0, =sl_current_process                                 /* sl_current_process  = sl_ready_process; */
    ldr     r1, =sl_ready_process
    ldr     r2, [r1]												/*  R2 = &tcb */
    str     r2, [r0]												/* sl_current_process = &tcb */

    ldr     r0, [r2]                                            	/* R0 is new process SP; */
  
    ldm     r0, {r4-r11}                                        	/* Restore r4-11 from new process stack */
    adds    r0, r0, #0x20
            
    msr     psp, r0                                             	/* Load PSP with new process SP */
    orr     lr, lr, #0x04                                       	/* Ensure exception return uses process stack */
    
    cpsie   i
    bx      lr                                                  	/* Exception return will restore remaining context */
```


假设下面两个指针的进程控制块结构为：

```
struct pcb {
    unsigned int *stack_point;
    struct list_head rq_list;
}
```


```
struct pcb *sl_current_process;
struct pcb *sl_ready_process;
```


假设现在创建了两个进程， 一个proc1, proc2, 定义方式如下：

```
struct pcb proc1;
struct pcb proc2;
```

假设`sl_ready_process`指向了`proc1`:

```
sl_ready_process = &proc1;
```

## 4.1 第一个进程装载并运行

```
    ldr     r0, =sl_current_process               # 把sl_current_process地址装载到r0寄存器中， 指针本身是有一个地址来存储本身
    ldr     r1, =sl_ready_process                  # 同上
    ldr     r2, [r1]											   # r2 = *r1 -->  r2 = sl_ready_process  -->  r2 = &proc1
    str     r2, [r0]                                           # *r0 = r2 --> sl_current_process = &proc1
```

就绪进程变换成当前进程的一个过程;


```
ldr     r0, [r2]                                           # 此时r2中是进程1的地址， r0 = *r2 --> r0 = proc1 --> r0 = proc1.stack_point(看下面解析)
```

首先，结构体变量的地址与其首个成员地址是相同的，即`&proc1 == &proc1.stack_point`, 推出`proc1 == proc1.stack_point`; 

每个进程堆栈使用下面函数完成初始工作：

```
unsigned int *port_proc_stack_init(void *entry, void *arg, unsigned int *stack_addr, size_t stack_szie)
{ 
    unsigned int    *user_stack;

    user_stack      = &stack_addr[stack_szie -1];
    user_stack      = (unsigned int *)((unsigned int)(user_stack) & 0xFFFFFFF8ul);
    
    *(--user_stack) = (unsigned int)0x01000000ul;                          //xPSR
    *(--user_stack) = (unsigned int)entry;                                         // Entry Point
    *(--user_stack) = (unsigned int)0xFFFFFFFEul;                           // R14 (LR)
    *(--user_stack) = (unsigned int)0x12121212ul;                          // R12
    *(--user_stack) = (unsigned int)0x03030303ul;                          // R3
    *(--user_stack) = (unsigned int)0x02020202ul;                          // R2
    *(--user_stack) = (unsigned int)0x01010101ul;                          // R1
    *(--user_stack) = (unsigned int)arg;                                            // R0
    
    *(--user_stack) = (unsigned int)0x11111111ul;                          // R11
    *(--user_stack) = (unsigned int)0x10101010ul;                          // R10
    *(--user_stack) = (unsigned int)0x09090909ul;                          // R9
    *(--user_stack) = (unsigned int)0x08080808ul;                          // R8
    *(--user_stack) = (unsigned int)0x07070707ul;                          // R7
    *(--user_stack) = (unsigned int)0x06060606ul;                          // R6
    *(--user_stack) = (unsigned int)0x05050505ul;                          // R5
    *(--user_stack) = (unsigned int)0x04040404ul;                          // R4
    
    return user_stack;
}
```

每个进程在创建的时候会进程控制块会保存堆栈信息： 

```
unsigned int proc_stack[128];
proc1.stack_point = port_proc_stack_init(fun, null, proc_stack, 128);
```



```
    ldm     r0, {r4-r11}                             # 把堆栈中的数据装载到寄存器中
    adds    r0, r0, #0x20                          # 堆栈地址增加0x20  == proc1.stack_point + 0x20
```


```
    msr     psp, r0                                     # 此时r0中的堆栈指针已经偏移到了     *(--user_stack) = (unsigned int)arg;                                   // R0
    orr     lr, lr, #0x04                               # 确保异常返回后， 新任务使用psp指针
```

异常返回后值对应的描述如下：

![](/images/rtos/exc_return.png)



执行`bx lr`后， 因为使用的是psp指针， 此时psp指针会自动出栈0x20字节（类似`ldm     r0, {r4-r11} ` ）， 既8个寄存器，如下：

```
    *(--user_stack) = (unsigned int)0x01000000ul;                          //xPSR
    *(--user_stack) = (unsigned int)entry;                                         // PC
    *(--user_stack) = (unsigned int)0xFFFFFFFEul;                            // R14 (LR)
    *(--user_stack) = (unsigned int)0x12121212ul;                          // R12
    *(--user_stack) = (unsigned int)0x03030303ul;                          // R3
    *(--user_stack) = (unsigned int)0x02020202ul;                          // R2
    *(--user_stack) = (unsigned int)0x01010101ul;                          // R1
    *(--user_stack) = (unsigned int)arg;                                             // R0
```


这样就完成第一个进程装载并运行过程;


## 4.2 第一次进程切换过程

假设进程1里面执行了一次调度切换`port_os_ctxsw`， 又再次触发`PendSV_Handler`, 接着分析;


```
    subs    r0, r0, #0x20                # 堆栈指针偏移0x20
    stm     r0, {r4-r11}                    # 将r4-r11寄存器中的值保存堆栈中
```

这其实是完成了一个压栈的过程， 保存现场;


```
    ldr     r1, =sl_current_process                              # r1 = &sl_current_process
    ldr     r1, [r1]                                                          # r1 = sl_current_process --> r1 = &proc1
    str     r0, [r1]                                                          # proc1 = r0
```

r0的值是psp的值，执行` str     r0, [r1]   ` 相当于`proc1 = psp --> proc1.stack_point = psp`


mcu进入一场后会自动压栈， 部分关键寄存器，然后需要手动压栈`r4-r11`寄存器， 接着保存psp指针到进程控制块, 再接着执行切换进程相关操作(参考4.1);