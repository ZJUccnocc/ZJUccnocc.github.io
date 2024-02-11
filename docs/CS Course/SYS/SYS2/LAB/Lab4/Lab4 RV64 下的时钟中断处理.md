# Lab4 RV64 下的时钟中断处理

## 3220103648 吴梓聪

# 1 实验目的

- 学习 RISC-V 的异常处理相关寄存器与指令，通过本次完成对异常处理的初始化
- 理解 CPU 上下文切换机制，并正确实现上下文切换功能
- 编写异常处理函数，完成对特定异常的处理
- 调用 OpenSBI 提供的接口，完成对时钟中断事件的设置

# 2 背景知识

## 2.1 控制状态寄存器

本实验只专注于 Supervisor 模式下的控制状态寄存器，包括：`sstatus` ，`sie` ，`stvec` ，`scause` ，`sepc` 等。

### 2.1.1 `sstatus`(Supervisor Status Register)

如下图是从《Risc-V Privileged》手册上扒下来的32位与64位系统下，`sstatus` 寄存器的各个位的分布：

![image-20240210211231085](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211231085.png)

在其中我们需要特别关注的是：SIE和SPP以及SPIE。

#### 2.1.1.1 SIE

> The SIE bit enables or disables all interrupts in supervisor mode. When SIE is clear, interrupts are not taken while in supervisor mode. When the hart is running in user-mode, the value in SIE is ignored, and supervisor-level interrupts are enabled. The supervisor can disable individual interrupt sources using the SIE CSR——《Risc-V Privileged》

意思是SIE主要作用为开启Supervisor模式下的所有中断的使能。如果是0的话则不会在Supervisor模式下接受任何中断。是1的话才会在Supervisor模式下执行这一层中产生的中断。

当在用户模式下运行时，SIE的值将不再重要，Supervisor模式下的中断将都被开启。

#### 2.1.1.2 SPP

> The SPP bit indicates the privilege level at which a hart was executing before entering supervisor mode. When a trap is taken, SPP is set to 0 if the trap originated from user mode, or 1 otherwise. When an SRET instruction (see Section 3.3.2) is executed to return from the trap handler, the privilege level is set to user mode if the SPP bit is 0, or supervisor mode if the SPP bit is 1; SPP is then set to 0.——《Risc-V Privileged》

SPP这一位是用于记录进入Supervisor模式前指令流（hart）的特权级别状态。当异常（来自软件或硬件的中断）发生时，如果这个异常是来自于用户态的话，SPP会被设置成0，否则被设置为1。

当遇到了SRET指令时，会根据SPP位上的数据来设置返回指令流时的特权级别状态。最后SPP还会被设置回0。

#### 2.1.1.3 SPIE

> The SPIE bit indicates whether supervisor interrupts were enabled prior to trapping into supervisor mode. When a trap is taken into supervisor mode, SPIE is set to SIE, and SIE is set to 0. When an SRET instruction is executed, SIE is set to SPIE, then SPIE is set to 1.——《Risc-V Privileged》

SPIE位指示在捕获到 Supervisor 模式之前是否启用了Supervisor的中断。当一个异常（来自软件或硬件）进入到Suporvisor模式时，SPIE被设置为SIE，SIE被设置为0。

当执行SRET指令时，SIE被设置回为SPIE，然后SPIE被设置为1。

### 2.1.2 `SIE` （Supervisor Interrupt Enable）

如下图是从《Risc-V Privileged》手册上扒下来的 `sie` 寄存器的各个位的分布：

![image-20240210211244836](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211244836.png)

由图可知，这个 `sie` 寄存器一共只有16位。其中的SSIE，STIE，SEIE分别指的是：Supervisor Software( Timer , External ) Interrupt enable，也就是三种中断（软件，时钟，终端）。

当 `sstatus` 中的 `sie` 位，也就是第二位是 0 时，`sie` 寄存器中的值就不重要了。但是当 `sstatus` 中的 `sie` 位是 1 时，中断是否生效会根据 `sie` 寄存器中的三个位上的值来决定。

### 2.1.3 `stvec` （Supervisor Trap Vector base address register）

如下图是从《Risc-V Privileged》手册上扒下来的 `stvec` 寄存器的各个位的分布：

![image-20240210211248320](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211248320.png)

![image-20240210211251178](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211251178.png)

如图我们可以看到 `stvec` 是由BASE和MODE两个部分组成的。因为我们在本次实验中只用Direct模式，也就是MODE位为1。然后BASE部分需要设置成处理异常信号的指令的地址处。

### 2.1.4 `scause`（Supervisor Cause Register）

如下图是从《Risc-V Privileged》手册上扒下来的 `scause` 寄存器的各个位的分布：

![image-20240210211254090](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211254090.png)

这个是 `Exception Code` 的参数表：

![image-20240210211256064](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211256064.png)

同样可以看到 `scause` 由两个部分组成：最高位 `Interrupt` 部分和 `Exception Code` 部分。其中Interrupt部分表示是否是由 `Interrupt` 产生的，如果是0表示是由软件引起的异常。

本次实验需要处理的是Supervisor timer interrupt。也就是 `Exception Code = 5` 的情况。那么我们只需要把当前这个 `scause` 寄存器的值设置为 `0x80000005` 即可。

### 2.1.5 `sepc` （Supervisor Exception PC Register）

用于储存中断发生处的地址，在中断处理结束后返回这个地址，恢复执行位置。 

## 2.2 特权指令

控制状态寄存器需要特定的特权指令才能进行操作。

### 2.2.1 CSR读写指令（`csrr`，`csrw`等）

所有CSR指令都以原子方式读取、修改、写入一个CSR。

- `csrr`
  - 用于读取一个 CSR 的值到通用寄存器，例如：
    `csrr rd, sstatus` 是将 `sstatus` 的值写入到 `rd` 寄存器中。

-  `csrw`
  - 把一个通用寄存器中的值写入 CSR 中，例如：
    `csrw sstatus, rs1` 是将 `rs1` 的值写入 `sstatus`。

- `csrs`
  - 把 CSR 中指定的 bit 置 1，例如：
    
    ```asm
    li    a0, 1 << n
    csrs  sstatus, a0
    #==完全等价于==>
    csrsi sstatus, (1 << n)
    	#这是将 sstatus 的右起第 n+1 位置 1。
    ```
    
  - `csrc` 基本一致，只是起到置 0 的效果。
  
- `csrrw` 
  - 读取一个 CSR 的值到通用寄存器，然后把另一个值写入该 CSR，例如：
    `csrrw rd, sstatus, rs1` 是将 `sstatus` 的值与存入到 `rd` 中，然后将 `rs1` 的值放入 `sstatus` 中。
  - 同理由 `csrrs` 和 `csrrc` 两条指令。

### 2.2.2 控制流转移指令（`sret`，`mret`）

此处的 `sret` 和 `mret` 分别是用于 S 态和 M 态的异常返回，在异常程序处理结束后，执行 `sret` 或 `mret` ，会将 `sepc` 或 `mepc` 再放回 `pc` 中， 返回到之前程序继续运行。

## 2.3 异常处理流程图

最后附一张晶晶姐直播回放上扒下来的异常处理流程图，据说最后会有用（）

![image-20240210211302202](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211302202.png)

## 2.4 上下文处理

由于在处理异常时，有可能会改变系统的状态。所以在真正处理异常之前，我们有必要对系统的当前状态进行保存，在异常处理完成之后，我们再将系统恢复至原先的状态，就可以确保之前的程序继续正常运行。

这里的系统状态通常是指寄存器，这些寄存器也叫做 CPU 的上下文 (`Context` ).

## 2.5 异常处理程序

异常处理程序会根据 `scause` 的值， 进入不同的处理逻辑，在本次试验中我们需要关心的只有 `Superviosr Timer Interrupt`。也就是在2.1.4中提到的部分。

# 3 实验步骤

## 3.1 准备工程

先根据手册给的示意图，将Lab3中对应的文件复制到这个文件夹下，形成一个与手册相一致的文件架构。

![image-20240210211308598](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211308598.png) ![image-20240210211305849](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211305849.png) 

然后再根据手册里的提示，对 `vmlinux.lds` 和 `head.S` 以及 `init/test.c` 做出对应的修改：

- 对 `vmlinux.lds` 的修改：在text段增加一个段标号 `.text.init` ，放在 `.text.entry` 之前，用于执行初始化的代码。

- 对 `head.S` 的修改：将 `_start` 这一标号的位置调整至新的段 `.text.init` 之中。

- 对 `init/test.c` 的修改：
  ```C
  <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< 原先的 test.c
  ...
  while(1) {}
  >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> 修改之后的 test.c
  ...
  unsigned long record_time = 0; 
  while (1) {
      unsigned long present_time;
      __asm__ volatile("rdtime %[t]" : [t] "=r" (present_time) : : "memory");
      present_time /= 10000000;
      if (record_time < present_time) {
          printk("kernel is running! Time: %lus\n", present_time);
          record_time = present_time; 
      }
  }
  ```

## 3.2 开启异常处理

从代码顺序上应该是先设置好 `stvec` ，将异常处理函数的入口地址存入 `stvec` 寄存器之中，再开启时钟中断，然后设置好第一次的时钟中断后再开启S态下的中断响应。但是为了调试每一个分步骤是否正确，我们选择先编写开启时钟中断和中断响应的部分，再完成后续的部分。

- 开启时钟中断：

  在2.1.2 对 `SIE` 的介绍中，我们知道SIE的第六位是对应的时钟中断开启，因此我们的代码就应该是：

  ````asm
  li a0, 1 << 5
  csrs sie, a0	#表示使得SIE第六位被置为1。
  ````

- 开启中断相应：

  在2.1.1对 `sstatus` 的介绍中，我们知道 `sstatus` 的第二位为1时表示可以 Supervisor 阶段的 interrupt 的开启。那么同样的我们的代码就应该是：

  ```asm
  li a0, 1 << 1
  csrs sstatus, a0 	#表示 sstatus 的第二位被置1。
  ```

- 对应的debug：

  因为无论是使用 `spike` 还是 `qemu` 来 debug，我们都需要先获得Image镜像文件，那么我们首先需要对我们的工程文件进行编译。也就是首先需要对我们的 `Makefile` 体系做出对应的修改，这一部分在3.6中会详细的阐述，此处不作赘述。

  除此之外还因为在编译前需要先完成 `entry.S` 的部分，所以我们后面的内容先按兵不动。

- 设置 `stvec` 即异常处理函数的入口：

  因为一条指令的长度是32位，那么每一个pc的跨度就是4，也就是0b100，所以一个地址的低两位一定是0b00。因为我们这时候用的是Direct模式，所以不需要做额外处理。实现代码如下：

  ```asm
  la a0, _traps
  csrw stvec, a0
  ```

- 设置第一次的时钟中断：

  我们参考3.5的实现逻辑来完成这一部分代码，但是需要额外注意的事情是，因为在这个部分出现了函数的调用，所以我们需要在这个部分前开辟好一块栈空间，代码如下：

  ```asm
  rdtime a0
  li t0, 10000000
  add a0, a0, t0
  la  sp, _end+0x1000
  call sbi_set_timer
  ```

## 3.3 实现上下文切换

我们要使用汇编实现上下文切换机制，需要修改 `./arch/riscv/kernel/` 目录下的 `entry.S` 文件，共包含以下几个步骤：

1. 保存 CPU 的寄存器（上下文）到内存中（栈上）。

2. 将 `scause` 和 `sepc` 中的值传入异常处理函数 `trap_handler`。

   （`trap_handler` 在 3.4 中实现，我们将会在 `trap_handler` 中实现对异常的处理。）

3. 在完成对异常的处理之后， 我们从内存中（栈上）恢复 CPU 的寄存器（上下文）。

4. 从 trap 中返回。

实现代码如下，注释如下：

```asm
    .section .text.entry
    .align 2
    .globl _traps 
_traps:
    # YOUR CODE HERE
    # -----------
        # 1. save 32 registers and sepc to stack
        # sp,ra,gp,x4,t0~t6,s0~s11,a0~a7,fp 共计32个指针依次存入栈，每一个占用8位地址。
	sd sp, -08(sp)
	sd ra, -16(sp)
	... ...
	sd a7, -248(sp)
	sd fp, -256(sp)
	addi sp,sp,-256
    # -----------
        # 2. call trap_handler
		# 要将 scause 和 sepc 的值依次传入作为参数传入函数
		# 需要注意的是，不论是栈传参还是用a0，a1传参，都需要和后面的使用参数的方式统一
	csrr a0, scause
	csrr a1, sepc
	call trap_handler
    # -----------
        # 3. restore sepc and 32 registers (x2(sp) should be restore last) from stack
		# 和第一部分一样，但是反向依次传回
	ld fp, 00(sp)
	ld a7, 08(sp)
	... ...
	ld ra, 240(sp)
	ld sp, 248(sp)
    # -----------
        # 4. return from trap
        # 因为是从s态启动的一个trap，所以返回也需要用sret
	sret
    # -----------
```

## 3.4 实现异常处理函数

需要修改 `arch/riscv/kernel/` 目录下的 `trap.c` 文件，完成以下目标：

1. 在 `trap.c` 中实现异常处理函数`trap_handler()` , 其接收的两个参数分别是 `scause` 和 `sepc` 两个寄存器中的值。
2. 通过 `scause` 判断 `trap` 类型，如果是 `interrupt` 判断是否是 `timer interrupt`
3. 如果是 `timer interrupt` 则打印输出相关信息（即 4.6 节中输出的 `[S] Supervisor Mode Timer Interrupt `），并通过 `clock_set_next_event()` 设置下一次时钟中断 
4. `clock_set_next_event()` 见 4.5 节 
5. 其他interrupt / exception 可以直接忽略

```c
void trap_handler(unsigned long scause, unsigned long sepc) {
    switch(scause>>63){	//判断最高位
        case 1:	//Interrupt
            switch(scause & 0x7FFFFFFFFFFFFFFF){//去除最高位
                case 5:
                    printk("[S] Supervisor Mode Timer Interrupt\n");
                    clock_set_next_event();
                    break;
                default:
                    break;
			}
            break;
        case 0:	//Exception
            break;
    }
	return;
}
```

## 3.5 实现时钟中断相关函数

需要修改 `arch/riscv/kernel/` 目录下的 `clock.c` 文件，并完成以下目标：

1. 在 `clock.c` 中实现 `get_cycles()`：使用 `rdtime` 汇编指令获得当前 `time` 寄存器中的值。
2. 在 `clock.c` 中实现 `clock_set_next_event()`：调用 `sbi_ecall` 的衍生函数 `sbi_set_timer` ，设置下一个时钟中断事件。
   ![image-20240210211317591](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211317591.png) 

```c
unsigned long TIMECLOCK = 10000000;
// QEMU中时钟的频率是10MHz, 也就是1秒钟相当于10000000个时钟周期。
unsigned long get_cycles() {
	unsigned long ret_val;
    __asm__ volatile (
        "rdtime %[ret_val]"
        : [ret_val] "=r" (ret_val)
        : : "memory"
    );
    return ret_val;
}

void clock_set_next_event() {
    unsigned long next = get_cycles() + TIMECLOCK;
	sbi_set_timer(next);
} 
```

## 3.6 编译与测试

由于加入了一些新的 `.c` 文件，可能需要修改或添加一些 `Makefile` 或 `.h` 文件，请同学自己尝试修改，使项目可以编译并运行。

我们可以看到在最外层的 `Makefile` 文件并没有需要修改的地方，加上我们的debug和run都是使用spike完成，所以这个run和debug甚至可以被注释掉，完全用不着了。

然后我们依次检查 `./lib` `./init` `./arch/riscv` 三个目录下的 `Makefile` 文件，并做出如下修改：

```makefile
#./lib
    ...
        ${GCC} ${CFLAG} -c print.c
    ...
        #改为==>
    ...
        ${GCC} ${CFLAG} -c printk.c
    ...
#./init
	#无修改
#./arch/riscv
	#无修改
```

首先证明一下再用spike：

![image-20240210211327529](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211327529.png)![image-20240210211320993](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211320993.png) 

我们在_start位置插入断点，当执行到指令 `csrw stvec, a0` 时，检查寄存器的值看看是否正确地写入了：

![image-20240210211330864](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211330864.png) 

![image-20240210211335393](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211335393.png) 

可以看到我们的写入是正确的 `_traps` 的地址（0x80200040）。

继续执行，可以看到 `sie` 的值也正确的赋值了。

![image-20240210211337386](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211337386.png) 

这里也成功地将 `sstatus` 的第二位置为一。

![image-20240210211339595](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211339595.png) 

等到执行到 `j start_kernal` 的时候，程序就正常执行了。执行结果如下：

![image-20240210211341438](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211341438.png) 

# 4 思考题

## 4.1 理解 `medeleg` 和` mideleg` 

**题目描述：**

通过查看 `RISC-V Privileged Spec` 中的 `medeleg` 和 `mideleg` 解释上面 `MIDELEG` 值的含义，如果实验中 `mideleg` 没有设定为正确的值结果会怎么样呢？

**答：**

**MIDELEG： **Machine Interrupt Delegation Register 中断委派寄存器

**MEDELEG：** Machine Exception Delegation Register 异常委派寄存器

这两个寄存器都是用于配置处理器如何处理异常和中断的寄存器。

而其中的 `mideleg` 寄存器用于配置哪些中断可以被委托给机器级别处理，而不是通过异常委托给特权模式处理。`mideleg`的每个位对应一个中断，当对应位被设置为1时，相应的中断将会被委托给机器模式处理。

本次实验中，`MIDELEG` 输出值为 0x222（0b1000100010），第 5 位为 1，对应中断 `Supervisor timer interrupt` 。根据手册的说明，如果其为 0，则不会进入我们编写的处理程序，也就不会输出对应消息。

## 4.2 `time` 与 `cycle` 寄存器

**题目描述：**

机器启动后 time、cycle 寄存器分别是从 0 开始计时的吗，从 0 计时是否是必要的呢？（有关 `mcycle` 寄存器的内容可以参考手册）

**答：**

在 `Spike` 下，调试查看对应的 `time` 和 `cycle` 寄存器的值。

![image-20240210211420967](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211420967.png)

在 `Qemu` 下，调试查看对应的 `time` 和 `cycle` 寄存器的值。

![image-20240210211433252](./Lab4%20RV64%20%E4%B8%8B%E7%9A%84%E6%97%B6%E9%92%9F%E4%B8%AD%E6%96%AD%E5%A4%84%E7%90%86img/image-20240210211433252.png)

发现两者得到的结果居然不一样！在 `qemu` 中，time是0，cycle不是0，而在 `spike` 中，time和cycle都不是0。

因为time寄存器主要用于记录时间信息，它通常用于测量代码执行的相对时间，以便评估程序的运行时间和性能。通过读取time寄存器的值，可以得到代码执行的时间片或持续时间。

而cycle寄存器主要用于记录处理器内部时钟周期的数量。它通常用于测量代码执行的精确时钟周期数，以评估程序的性能和指令的执行效率。通过读取cycle寄存器的值，可以得到代码执行期间经过的时钟周期数。

我们实际上都可以通过记录开始时候的寄存器值和记录某一时刻的寄存器值，通过相减来得到这段时间内经过的时间或者周期数，因此实际上都不是很有必要从0开始。

## 4.3 指令集扩展

**题目描述：**

阅读 [The RISC-V Instruction Set Manual Volume I: Unprivileged ISA (V20191213)](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf) 中第 1.2 章节 `RISC-V Software Execution Environments and Harts` ，谈谈如何在一台不支持乘除法指令扩展的处理器上执行乘除法指令。

**答：**

如果在一台不支持乘法指令扩展的处理器上遇到乘法指令会触发 `Illegal Instruction` 异常，会有以下几个相关的异常处理寄存器可以供我们使用：

1. `mcause`寄存器：`mcause`寄存器用于存储异常的原因。它有一个字段用于表示当前异常的原因代码。对于非法指令异常，其原因代码将指示为非法指令。`mcause`寄存器的位布局根据RISC-V架构的特定实现而有所不同。
2. `mepc`寄存器：`mepc`寄存器用于存储导致异常的指令的地址。当异常发生时，处理器会将导致异常的指令的地址存储在`mepc`寄存器中，以便在异常处理程序中确定异常发生的位置。
3. `mtval`寄存器：`mtval`寄存器用于存储导致异常的值。对于非法指令异常，`mtval`寄存器存储了引发异常的非法指令本身。

因此我们可以从 `mcause` 中看到这个异常的原因，在 `mepc` 拿到异常指令的地址，可以从这里获取指令的内容，也可以直接从 `mtal` 寄存器中获取指令。在判断出异常后会进入到对应的异常处理函数中，我们可以在异常处理函数中根据上述获取到指令的方法，判断是否是乘除法，如果是乘法可以用连加来代替，如果是除法可以用连减来代替。最后处理结束后返回回对应的 `mepc` 处即发生异常的指令处即可。

