# Lab5：RV64 内核线程调度

## 吴梓聪 3220103648

# 1 实验目的

- 了解线程概念，并学习线程相关结构体，并实现线程的初始化功能
- 了解如何使用时钟中断来实现线程的调度
- 了解线程切换原理，并实现线程的切换
- 掌握简单的线程调度算法，并完成一种简单调度算法的实现



# 2 背景知识

## 2.1 程序，进程与线程

**程序**是我们将源代码经过一系列的处理（编译，链接，优化等等）后得到的一个可执行文件。

**进程**就是目前正在CPU中运行，并且占用着计算机资源的**程序**。**进程**是动态的**程序**，它需要将运行中的**程序**的代码/数据加载到内存空间，并且还需要拥有一块属于这个进程的进程**运行栈**。

**进程**可以对应一个或多个**线程**，统一进程的不同**线程**往往具有相同的代码，共享一块内存，但是有不同的 CPU 执行状态。

## 2.2 线程的属性

在不同的操作系统中，为每个线程所保存的信息都不同。在本实验中采用一种比较基础的实现方式，每个线程会包括：

- **线程 (Thread) ID**：用于唯一确认一个线程。
- **运行栈**：每个线程都必须有一个独立的运行栈，保存运行时的数据。
- **执行上下文**：当线程不在执行状态时，我们需要保存其上下文（其实就是**状态寄存器**的值），这样之后才能够将其恢复，继续运行。
- **运行时间片**：为每个线程分配的运行时间。
- **优先级**：在优先级相关调度时，配合调度算法，来选出下一个执行的线程。

## 2.3 线程切换

在一个线程运行时，发生一个时钟中断，这时由操作系统获得 CPU 的使用权限，并依次完成以下任务：

1. 将当前线程的剩余运行时间减少一个单位。
2. 根据一个特定的调度算法来判定是继续执行还是调度其他的线程进入CPU 执行。这个算法是遍历了所有可执行的线程，并选出一个在特定调度算法下具有最高优先级的线程。
3. 保存当前线程的上下文到对应的 PCB 中，将目标线程的上下文载入到 CPU 中.
   其中的PCB也就是进程控制块，包含进程状态，进程编号，PC寄存器，上下文（状态寄存器），内存界限，打开文件列表......

# 3 实验过程

## 3.1 准备工程

- 本次实验额外增加了 `rand.h/rand.c` , `string.h/string.c` , `mm.h/mm.c` 三组文件，其中 `mm.h\mm.c` 提供了简单的物理内存管理接口，`rand.h\rand.c` 提供了 `rand()` 接口用以提供伪随机数序列，`string.h/string.c ` 提供了 `memset` 接口用以初始化一段内存空间。

- 在本次实验中我们需要一些物理内存管理的接口，于是提供了 `kalloc` 接口（见`mm.c`）。我们可以用 `kalloc` 来申请 4KB 的物理页。由于引入了简单的物理内存管理，我们需要在 `_start` 的适当位置调用 `mm_init` , 来初始化内存管理系统，并且在初始化时需要用一些自定义的宏，需要修改`defs.h` , 在 `defs.h` 添加如下内容：
  ```c
  #define PHY_START 0x0000000080000000// 物理内存起始地址
  #define PHY_SIZE  128 * 1024 * 1024 // 128MB， QEMU 默认内存大小 
  #define PHY_END   (PHY_START + PHY_SIZE)// 物理内存结束地址
  
  #define PGSIZE 0x1000 // 4KB物理页
  #define PGROUNDUP(addr) ((addr + PGSIZE - 1) & (~(PGSIZE - 1)))// 自动对齐，******
  #define PGROUNDDOWN(addr) (addr & (~(PGSIZE - 1)))	//自动对齐，************
  ```

## 3.2 `proc.h` 部分

将下列代码复制到 `proc.h` 文件中即可，这个文件应该位于 `arch/riscv/include/proc.h` 路径下。

```c
#include "types.h"

#define NR_TASKS  (1 + 3) // 用于控制 最大线程数量 （idle 线程 + 3 内核线程）

#define TASK_RUNNING 0 // 为了简化实验，所有的线程都只有一种状态
//后续还可以添加其他的线程状态。
#define PRIORITY_MIN 1
#define PRIORITY_MAX 5

/* 用于记录 线程 的 内核栈与用户栈指针 */
/* (lab6中无需考虑，在这里引入是为了之后实验的使用) */
struct thread_info {
    uint64 kernel_sp;
    uint64 user_sp;
};

/* 线程状态段数据结构 */
struct thread_struct {
    uint64 ra;
    uint64 sp;
    uint64 s[12];
};

/* 线程数据结构 */
struct task_struct {
    struct thread_info* thread_info;//线程信息
    uint64 state;    // 线程状态
    uint64 counter;  // 运行剩余时间 
    uint64 priority; // 运行优先级
    uint64 pid;      // 线程id
    struct thread_struct thread;	//线程状态段
};

void task_init(); /* 线程初始化 创建 NR_TASKS 个线程 */ 
void do_timer(); /* 在时钟中断处理中被调用 用于判断是否需要进行调度 */
void schedule();/* 调度程序 选择出下一个运行的线程 */
void switch_to(struct task_struct* next);/* 线程切换入口函数*/
void dummy();/* dummy funciton: 一个循环程序，循环输出自己的 pid 以及一个自增的局部变量*/
```

## 3.3 `proc.c` 部分

### 3.3.1 `task_init()` 函数的实现

这个函数实现的**主要功能**如下：

1. 调用 `kalloc()` 为包括 `idle线程` 在内的多个线程分配一个4KB的物理页，将 `task_struct` 存放在该页的低地址部分， 将线程的栈指针 `sp` 指向该页的高地址。
2. 为多个线程设置 `state` 、`pid` 和 `counter` 以及 `priority` 等属性。
3.  这里和 `idle` 设置的区别在于，要为这些线程设置 `thread_struct` 中的 `ra` 和`sp` ，`ra` 应该是 `__dummy`（见 4.3.2）的地址，`sp` 设置为该线程申请的物理页的高地址。

**具体步骤：**

1. 因为操作系统本身就是一个 `idle线程` ，但我们并没有为他设计好他的 `task_struct` 。所以第一步应该是为 `idle` 设置 `task_struct`。并将`current` , `task[0]`都指向 `idle`。
   1. 调用 `kalloc()` 为 idle 分配一个物理页 
   2. 设置 `state` 为 `TASK_RUNNING`; 
   3. 由于 `idle` 不参与调度 可以将其 `counter` / `priority` 设置为 0
   4. 设置 `idle` 的 `pid` 为 0
   5. 将 `current` 和 `task[0]` 指向 `idle`
2. 然后再完成对其余进程的设置。
   1. 其中每个线程的 `state` 为 `TASK_RUNNING`; 
   2. `counter` 为 0, `priority` 使用 `rand()` 来设置
   3. `pid` 为该线程在线程数组中的下标。
   4. 为 `task[1] ~ task[NR_TASKS - 1]` 设置 `thread_struct` 中的 `ra` 和 `sp` ，其中 `ra` 设置为 __dummy （见 4.3.2）的地址， `sp` 设置为 该线程申请的物理页的高地址

**代码实现：**

```c
extern void __dummy();

struct task_struct* idle;           // idle process
struct task_struct* current;        // 指向当前运行线程的 `task_struct`
struct task_struct* task[NR_TASKS]; // 线程数组，所有的线程都保存在此

void task_init() {
    idle = (struct task_struct*)kalloc();
    idle->state = TASK_RUNNING;
    idle->counter = 0;
    idle->priority = 0;
    idle->pid = 0;
    current = idle;
    task[0] = idle;

    for(int i = 1;i < NR_TASKS;i ++){
        task[i] = (struct task_struct*)kalloc();
        task[i]->state = TASK_RUNNING;
        task[i]->counter = 0;
        task[i]->priority = rand() % (PRIORITY_MAX - PRIORITY_MIN + 1) + PRIORITY_MIN;
        task[i]->pid = i;
        task[i]->thread.ra = __dummy;
        task[i]->thread.sp = task[i] + PGSIZE;
    }

    printk("...proc_init done!\n");
}
```

### 3.3.2 `dummy()` 函数的理解与在 `entry.s` 中设置 `dummy()` 

**先贴代码：**

```c
void dummy() {
    uint64 MOD = 1000000007;
    uint64 auto_inc_local_var = 0;
    int last_counter = -1; // 记录上一个counter
    int last_last_counter = -1; // 记录上上个counter
    while(1) {
        if (last_counter == -1 || current->counter != last_counter) {
            last_last_counter = last_counter;
            last_counter = current->counter;
            auto_inc_local_var = (auto_inc_local_var + 1) % MOD;
            printk("[PID = %d] is running. auto_inc_local_var = %d\n", current->pid, auto_inc_local_var); 
        } else if((last_last_counter == 0 || last_last_counter == -1) && last_counter == 1) {
            last_counter = 0; 
            current->counter = 0;
        }
    }
}
```

运行中的线程在遇到时钟中断时，会将它的上下文保存到栈中，当线程再次被调度运行时会从栈中恢复他的上下文。但如果被调度的线程是初次创建的，此时线程的栈为空，没有上下文，所以我们需要为第一次调度的线程提供一个特殊的返回函数 `__dummy()` 。

本次实验中使 `task[1] ~ task[NR_TASKS - 1]` 都运行同一段代码 `dummy()` ，我们可以在这里的输出中添加信息辅助调试和Debug。

在 `entry.s` 中，我们需要完成的任务是：

1. 设置 `sepc` 为 `dummy()` 的地址
2. 使用 `sret` 从中断中返回

因此代码也非常简单：

```asm
	.extern dummy
	.global __dummy
__dummy:
	la a0, dummy
	csrw sepc, a0
	sret
```

### 3.3.3 `switch_to()` 函数与 `__switch_to` 函数的实现

#### 3.3.3.1 `switch_to()` 函数部分

本函数应该实现的**主要功能**如下：

1. 判断下一个执行的线程 `next` 与当前的线程 `current` 是否为同一个线程
2. 如果是同一个则不做任何处理
3. 如果不是同一个则调用 `__switch_to` 函数进行进程切换

**实现代码：**

```c
void switch_to(struct task_struct* next) {
    if(next != current){
        printk("switch to [PID = %d, PRIORITY = %d, COUNTER = %d]\n", next->pid, next->priority, next->counter);//最后输出的时候需要的内容
        struct task_struct* prev = current;
        current = next;
        __switch_to(prev,next);//因为这里马上要保存当前的上下文，所以__switch_to一定要最后一步执行，因此跟更换进程的任务应该在进入__switch_to之前就完成。
    }
}
```

#### 3.3.3.2 `__switch_to` 函数实现

本函数应该实现的**主要功能**如下：

1. 保存当前线程的`ra` , `sp` , `s0~s11`到，传入的两个参数中当前线程的 `thread_struct` 中
2. 将下一个线程的 `thread_struct` 中的相关数据载 入到 `ra` , `sp` , `s0~s11`中

因为本函数接受的参数是两个指针，也就是说我们在这段汇编代码中需要用到 `ld` 和 `sd` 指令以及计算结构体中的变量长度。

对于如下的这个结构体，我们可以进行计算：

```c
struct task_struct {
    struct thread_info* thread_info;
    uint64 state;    // 线程状态
    uint64 counter;  // 运行剩余时间 
    uint64 priority; // 运行优先级
    uint64 pid;      // 线程id

    struct thread_struct thread;
};
struct thread_struct {
    uint64 ra;
    uint64 sp;
    uint64 s[12];
};
```

一个指向线程信息的指针是64位的，其余还有四个长度同样是64位的变量，那么在正式进入指定线程的结构体前会有 `5*64/8个字节` 的内存空间，也就是从偏移地址为40的地方开始是 `thread_struct` 的内容。再根据结构体 `thread_struct` 中的变量内容，我们可以得到如下代码：

```asm
.global __switch_to
__switch_to:
    sd ra, 40(a0)
    sd sp, 48(a0)
    sd s0, 56(a0)
    sd s1, 64(a0)
    sd s2, 72(a0)
    sd s3, 80(a0)
    sd s4, 88(a0)
    sd s5, 96(a0)
    sd s6, 104(a0)
    sd s7, 112(a0)
    sd s8, 120(a0)
    sd s9, 128(a0)
    sd s10, 136(a0)
    sd s11, 144(a0)

    ld ra, 40(a1)
    ld sp, 48(a1)
    ld s0, 56(a1)
    ld s1, 64(a1)
    ld s2, 72(a1)
    ld s3, 80(a1)
    ld s4, 88(a1)
    ld s5, 96(a1)
    ld s6, 104(a1)
    ld s7, 112(a1)
    ld s8, 120(a1)
    ld s9, 128(a1)
    ld s10, 136(a1)
    ld s11, 144(a1)

    ret
```

### 3.3.4 `do_timer()` 函数的实现与调用

#### 3.3.4.1 实现

在这个函数中我们需要实现的**主要功能**如下：

1. 判断如果当前的进程原先就用完了时间片，则调度新进程。
2. 否则将当前进程的counter--。
3. 如果此时的进程的时间片被用完了也重新调度。

**代码实现**如下：

```c
void do_timer(void) {
    if(current->counter == 0) schedule();
    else if(--current->counter == 0) schedule(); 
}
```

#### 3.3.4.2 调用

本函数的调用是在发生时钟中断时调用。也就是说具体的调用应该发生在时钟中断的处理函数中。

也就是我们在这里需要回去修改 `lab4` 给出的 `trap.c` ：

```c
void trap_handler(unsigned long scause, unsigned long sepc) {
    switch(scause >> 63){ //Highest bit
        case 1: //Interrupt
            switch (scause & 0x7FFFFFFFFFFFFFFF){
                case 5: //Supervisor Mode Timer Interrupt
                    // printk("[S] Supervisor Mode Timer Interrupt\n");
                    clock_set_next_event();
                    do_timer();
                    break;
                default:
                    break;
            }
            break;
        case 0: //Exception
            break;
    }
    return;
}
```

### 3.3.5 `schedule()` 的实现

实验指导告诉我们可以参考 [`Linux` 的调度算法](https://elixir.bootlin.com/linux/0.11/source/kernel/sched.c#L122)，也就是如下的这段代码：

```c
void schedule(){ //在文档中并没有单独定义这个函数，这里的定义是为了方便移植到我们的实验中
    int i,next,c;
    struct task_struct ** p;	//在Linux的文档中，p是一个二级指针，也就是一个指向task_struct结构体地址的指针。
//但实际上我感觉在和schedule()实现相关的后文中并没有用到二级指针，但是因为数据类型都是一个64位的数据，所以应该影响不大。
    //... ...
	while (1) {
		c = -1;
		next = 0;
		i = NR_TASKS;	//这个参数我们实验中也有出现，代表总线程数
		p = &task[NR_TASKS];	//这里是从最后一个task开始，p作为一个指向task[NR_TASKS]的指针。
		while (--i) {
			if (!*--p)	//p递减，如果p指向的不是一个有效的内容，则继续递减
				continue;
			if ((*p)->state == TASK_RUNNING && (*p)->counter > c)//调度算法相关的部分
				c = (*p)->counter, next = i;
            //不难发现，这就是一个遍历所有可运行的任务并且找到拥有最大counter的任务作为下一个调用的内容。
		}
		if (c) break;	//找完一遍后如果找到了counter不是0的可运行任务，就结束循环。
		for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)	//如果可运行的任务的counter都是0，则重置所有任务的counter
            	//但是实际上所有任务初始化时的counter都是0，所以第一次的调度一定会执行这段代码。
			if (*p)	//需要注意的是，这个地方的重置同样会重置那些状态不是可运行的任务，所以counter并不都是0。
				(*p)->counter = ((*p)->counter >> 1) + (*p)->priority;	//因此这里的第一项并不能去掉。
	}
	switch_to(next);
}
```

`Linux` 所使用的调度算法归纳如下：

1. 如果存在counter不为0的TASK_RUNNING态任务，则选出counter最大的任务。

2. 否则需要对所有任务（包括状态不是TASK_RUNNING的任务）执行一个操作：

   ```c
   task->counter = task->counter >> 1 + task->priority;
   ```

3. 然后再次寻找counter不为0且最大的TASK_RUNNING态任务。

根据最后给出的预期结果图，我们可以知道实际的调度算法是选取counter最小的任务先执行，而不是像Linux一样先执行counter最大的任务。那么如果想要移植为我们可以使用的代码，可以更改如下：

```c
void schedule(void) {
    struct task_struct* next;
	while (1) {
		int min_counter = PRIORITY_MAX+1;
		for(int i = 1;i < NR_TASKS;i ++)
            if(task[i]->state == TASK_RUNNING  && task[i]->counter != 0 && (int)task[i]->counter < min_counter) min_counter = task[i]->counter, next = task[i];
		if (min_counter != PRIORITY_MAX+1)
            break;
		for(int i = 1;i < NR_TASKS;i ++)
            if (task[i]){
                task[i]->counter = (task[i]->counter >> 1) + task[i]->priority;
                printk("SET [PID = %d, PRIORITY = %d, COUNTER = %d]\n",task[i]->pid,task[i]->priority,task[i]->counter);
            }
	}
	switch_to(next);
}
```

## 3.4 实验结果

最后得到的结果是：

```
Launch the qemu ......

OpenSBI v0.9
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name             : riscv-virtio,qemu
Platform Features         : timer,mfdeleg
Platform HART Count       : 1
Firmware Base             : 0x80000000
Firmware Size             : 100 KB
Runtime SBI Version       : 0.2

Domain0 Name              : root
Domain0 Boot HART         : 0
Domain0 HARTs             : 0*
Domain0 Region00          : 0x0000000080000000-0x000000008001ffff ()
Domain0 Region01          : 0x0000000000000000-0xffffffffffffffff (R,W,X)
Domain0 Next Address      : 0x0000000080200000
Domain0 Next Arg1         : 0x0000000087000000
Domain0 Next Mode         : S-mode
Domain0 SysReset          : yes

Boot HART ID              : 0
Boot HART Domain          : root
Boot HART ISA             : rv64imafdcsu
Boot HART Features        : scounteren,mcounteren,time
Boot HART PMP Count       : 16
Boot HART PMP Granularity : 4
Boot HART PMP Address Bits: 54
Boot HART MHPM Count      : 0
Boot HART MHPM Count      : 0
Boot HART MIDELEG         : 0x0000000000000222
Boot HART MEDELEG         : 0x000000000000b109
MAIN!
mm_initing...
...mm_init done!
task_initing...
...proc_init done!
2023 ZJU Computer System II
SET [PID = 1, PRIORITY = 1, COUNTER = 1]
SET [PID = 2, PRIORITY = 4, COUNTER = 4]
SET [PID = 3, PRIORITY = 5, COUNTER = 5]
switch to [PID = 1, PRIORITY = 1, COUNTER = 1]
[PID = 1] is running. auto_inc_local_var = 1
switch to [PID = 2, PRIORITY = 4, COUNTER = 4]
[PID = 2] is running. auto_inc_local_var = 1
[PID = 2] is running. auto_inc_local_var = 2
[PID = 2] is running. auto_inc_local_var = 3
[PID = 2] is running. auto_inc_local_var = 4
switch to [PID = 3, PRIORITY = 5, COUNTER = 5]
[PID = 3] is running. auto_inc_local_var = 1
[PID = 3] is running. auto_inc_local_var = 2
[PID = 3] is running. auto_inc_local_var = 3
[PID = 3] is running. auto_inc_local_var = 4
[PID = 3] is running. auto_inc_local_var = 5
SET [PID = 1, PRIORITY = 1, COUNTER = 1]
SET [PID = 2, PRIORITY = 4, COUNTER = 4]
SET [PID = 3, PRIORITY = 5, COUNTER = 5]
switch to [PID = 1, PRIORITY = 1, COUNTER = 1]
[PID = 1] is running. auto_inc_local_var = 2
switch to [PID = 2, PRIORITY = 4, COUNTER = 4]
[PID = 2] is running. auto_inc_local_var = 5
[PID = 2] is running. auto_inc_local_var = 6
[PID = 2] is running. auto_inc_local_var = 7
[PID = 2] is running. auto_inc_local_var = 8
switch to [PID = 3, PRIORITY = 5, COUNTER = 5]
[PID = 3] is running. auto_inc_local_var = 6
[PID = 3] is running. auto_inc_local_var = 7
[PID = 3] is running. auto_inc_local_var = 8
```

顺便自证一下用的spike调试，为了bonus()

![image-20240210211511781](./Lab5%EF%BC%9ARV64%20%E5%86%85%E6%A0%B8%E7%BA%BF%E7%A8%8B%E8%B0%83%E5%BA%A6img/image-20240210211511781.png)

# 4 思考题

## 4.1进程调度时的上下文切换

### 问题

在 RV64 中一 共用 32 个通用寄存器， 为什么 `context_switch` 中只保存了 14 个？

### 解答

因为在时钟中断发生时，时钟中断的处理函数已经把所有的通用寄存器的内容全部保存到当前的线程栈中，所以我们实际上可以根据 `sp` 的值来完成未被储存的寄存器的上下文的恢复。

## 4.2 `gdb` 调试观察 `ra` 的变化

### 问题

当线程第一次调用时， 其 `ra` 所代表的返回点是 `__dummy`。 那么在之后的线程调用中 `context_switch` 中，`ra` 保存 / 恢复的函数返回点是什么呢 ？ 请同 学用 `gdb` 尝试追踪一次完整的线程切换流程， 并关注每一次 `ra` 的变换。

### 解答

![image-20240210211515288](./Lab5%EF%BC%9ARV64%20%E5%86%85%E6%A0%B8%E7%BA%BF%E7%A8%8B%E8%B0%83%E5%BA%A6img/image-20240210211515288.png)

第一次执行switch_to的时候，在task[1~3]的内存空间，可以看到，第一个64位记载着前一个结构体的地址；第二个64位记载着任务状态，也就是 `TASK_RUNNING (0)` ；第三个64位记载着对应 `task` 的 `counter` 的值，分别是1，4，5；第四个64位记载着对应 `task` 的 `priority` 的值，分别是1，4，5；第五个64位记载着对应 `task` 的 `pid` 的值；第五个64位记载着对应 `task` 的 `ra` 的值；第六个64位记载了 `task` 的 `sp` ，也就是下一个 `task` 的地址。然后会是后面用到的s0~s11。现在运行着的进程应该是0号进程，也就是IDLE进程。

![image-20240210211517489](./Lab5%EF%BC%9ARV64%20%E5%86%85%E6%A0%B8%E7%BA%BF%E7%A8%8B%E8%B0%83%E5%BA%A6img/image-20240210211517489.png) 由图可知0x80200154是__dummy的位置。

代码第二次回到这个断点 `switch_to()` 时表明1号任务已经用完时间片了，这时候我们再看 `task[1]` 的内容：

![image-20240210211519424](./Lab5%EF%BC%9ARV64%20%E5%86%85%E6%A0%B8%E7%BA%BF%E7%A8%8B%E8%B0%83%E5%BA%A6img/image-20240210211519424.png)

可以看到counter确实是0了。再根据指令 `i r ra` 我们可以看到当前运行时的 `ra` 寄存器的值：

![image-20231214145531877](D:\Desktop\课程文件\计算机系统\Lab\Lab5\image-20231214145531877.png)

不难发现在正常的运行过程中，`ra` 的值就是当前这个函数的返回位置，当刚切换成下一个第一次被调用的进程时，`ra` 又会变成原先设置好的 `__dummy`。使得当前进程在 `__switch_to` 结束后进入到我们希望的 `__dummy` 中。

![image-20240210211521648](./Lab5%EF%BC%9ARV64%20%E5%86%85%E6%A0%B8%E7%BA%BF%E7%A8%8B%E8%B0%83%E5%BA%A6img/image-20240210211521648.png)

然后dummy再通过设置 `sepc` 使得 `sret` 返回后的位置是 `dummy()` 函数，并在 `dummy()` 函数中运行直到这个进程的时间片结束回到`switch_to()` 函数。

如果是之后的第二次调度某一个进程，我们的可以看到结构体中的 `ra` 已经不是原来的值 `__dummy` 了，而是：

![image-20240210211523685](./Lab5%EF%BC%9ARV64%20%E5%86%85%E6%A0%B8%E7%BA%BF%E7%A8%8B%E8%B0%83%E5%BA%A6img/image-20240210211523685.png)

这个位置实际上就是 `switch_to()` 函数的结尾处。再 `switch_to()` 函，数结束后返回到 `do_timer()` 函数中，而后回到 `trap_handler()` 中，最后是回到 `__trap` 中，然后返回正常的用户态执行，直到下一次时钟中断。
