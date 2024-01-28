---
title: Lab3：RV64内核引导实验报告
date: 2023-11-12 23:13:25
categories:
- 大二
- SYS2
tags:
- SYS2
- Lab report
---

# Lab3：RV64内核引导实验报告

### 3220103648 吴梓聪



# 1 实验目的

- 学习 RISC-V 汇编，加强对RISC-V汇编代码的理解，了解RISC-V的三种特权模式，培养使用RISC-V汇编语言的能力，通过编写`head.S` 实现跳转到内核运行的第一个 C 函数。
- 了解 `SBI` 接口规范的定义，学习 `SBI` 的开源实现—— `OpenSBI` ，理解 `OpenSBI` 在本次实验中所起到的作用，并调用 `OpenSBI` 提供的接口完成字符的输出，即 `puts` 和 `puti` 函数。
- 学习 `Makefile` 相关知识，补充项目中的 `Makefile` 文件，来完成对整个工程的管理。
- 学习C语言中内联汇编的使用方法，在汇编层面实现系统调用和参数传递，进而了解C语言的一些底层操作逻辑与原理。
- 学习连接器的编写内容和编写规范，以及如何自己新建一个内存布局。

# 2 背景知识

## 2.1 RISC-V汇编

除了在SYS1中学过的一部分汇编知识，这次实验中还用到了以下几个方面的汇编知识：

- **常见的伪操作**

  - `.extern <symbolName>` : 表示声明了一个名为 <symbolName> 的外部符号

  - `.global <symbolName>` : 定义了一个全局的符号，使得链接器能够全局识别它，即一个程序文件中定义的符号能够被所有其他程序文件可见。

  - `.local <symbolName>` : 定义一个局部符号，使得此符号不能够被其他程序文件可见。

  - `.section <symbolName>` : 将接下来的代码汇编链接到名为 <symbolName> 的段当中，还可以指定可选的子段。常见的段如`.text` 、`.data` 、`.rodata` 、`.bss` 。

  - `.space <size>` : 用于指定符号的大小。在当前的代码段中分配 <size> 字节的空间。

  - ......

- **RISC-V语言架构**

  一个完整的 RISC-V 汇编 程序 由若干条语句组成

  一条典型的 RISC-V 汇编 语句 由 3 部分组成：`[ label: ]` `[ operation ]` `[ comment ]`

  - 标号：在GNU格式的汇编中，任何以冒号结尾的标识符都被认为是一个标号。表示当前指令的位置标记。

  - 操作符operation 可以有以下多种类型：

    - instruction（指令）: 直接对应二进制机器指令的字符串。例如addi指令、lw指令等。

    - pseudo-instruction（伪指令）: 为了提高编写代码的效率，可以用一 条伪指令指示汇编器产生多条实际的指令。

    - directive（指示/伪操作）: , 通过类似指令的形式(以“.”开头)，通知汇 编器如何控制代码的产生等，不对应具体的指令。

  - comment：是用'#'开始到整行结束的注释。

- **RISC-V的三种特权模式**

  在RISC-V中有三种特权模式：U（User），S（supervisor），M（machine）

  | Level | Encoding |         Name         | Abbreviation |
  | :---: | :------: | :------------------: | :----------: |
  |   0   |    00    |   User/Application   |      U       |
  |   1   |    01    |  Supervisor(监管者)  |      S       |
  |   2   |    10    | Hypervisor(管理系统) |      H       |
  |   3   |    11    |       Machine        |      M       |

  - 其中的Machine模式是在硬件层面，对硬件操作的抽象，是最高级别的权限。
  - Supervisor模式介于Machine模式和User模式之间，对应着操作系统中的内核态（Kernal），当用户需要内核资源向内核发送申请时，会切换到内核态进行处理。
  - User模式是执行用户程序的模式，在操作系统中对应用户态，是最低级别的权限。

- **从开机通电到OS运行**

  在机器上电运行后，硬件会先进行一些基础的初始化任务，比如将CPU的PC移动到内存中的一个叫 Bootloader 的程序段的起始地址。而这个 Bootloader 是操作系统内核运行之前，用于初始化硬件，加载操作系统内核的一个程序段。在我们使用的 RISC-V 架构里，Bootloader 运行在 Machine 模式下。Bootloader 运行完毕后就会把当前模式切换到 S 模式下，随后机器会开始运行 Kernel 。

附赠三个实验指导上的链接（三个都是github上的资源，可能需要科学上网才能打开）：

- [RISC-V Assembly Programmer's Manual](https://github.com/riscv-non-isa/riscv-asm-manual/blob/master/riscv-asm.md)
- [RISC-V Unprivileged Spec](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf)
- [RISC-V Privileged Spec](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)

## 2.2 SBI和OpenSBI

因为存在Supervisor模式里的Kernal与Machine模式的执行环境之间交互的需求，所以有了SBI（Supervisor Binary Interface）这个两级模式之间交互的接口规范。而OpenSBI则是一个RISC-V规范下实现的开源应用。软件基础研发平台（如RISC-V）和硬件研发商家（如SoC供应商）可以通过自主扩展OpenSBI的实现来适应特定的硬件配置。

也就是说为了使得我们千奇百怪的操作系统能够和我们千奇百怪的硬件相匹配，OpenSBI提出了一系列的规范，对Machine模式下的硬件进行了统一的定义，这样一来Supervisor模式下的内核就可以通过这些规范对硬件进行不同的操作。下面这张图清晰的表现了不同模式间的交互方法和交互渠道。

![image-20231105221745223](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231105221745223.png)

> 实际上我们在实现SBI的时候发现，其实也是在C函数中使用了一段内联汇编的代码，去执行一个叫ecall的调用标号，这个ecall应该就是openSBI部分给出的与硬件交互的一个接口。

## 2.3 Makefile

话不多说，先丢[学习链接](https://seisman.github.io/how-to-write-makefile/introduction.html)，跟我一起写Makefile！

摘要：

```
<target> : <prerequisites>
    <recipe>
    ...
    ...
```

**target**

​	可以是一个目标文件，也可以是一个可执行文件，还可以是一个标签。对于标签这种特性，在后续的“伪目标”章节中会有叙述。

**prerequisites**

​	生成该target所依赖的文件和/或target。

**recipe**

​	该target要执行的命令，可以是任意的shell命令。

Makefile的执行逻辑是：prerequisites中如果有一个以上的文件比target文件要新的话，recipe部分的命令就会被执行。

反斜杠（ `\` ）是换行符的意思，如果一行行末有 " \ " ，说明这个命令没有结束，还有下一行也是跟在这一行后面的，只是为了便于makefile的阅读，我们加个换行符后换行，显得整齐一些。

变量的声明：<变量名> = <表达式>

变量的使用：$(变量名)

GNU的make很强大，可以自动推导文件以及文件依赖关系后面的命令，于是我们就没必要去在每一个 `.o` 文件后都写上类似的命令，因为，make会自动识别，并推导命令。只要make看到一个 `.o` 文件，它就会自动的把 `.c` 文件加在依赖关系中，并且 `cc -c whatever.c` 也会被推导出来。

甚至可以用类似二段译码的方法改写如下内容：

```makefile
objects = main.o kbd.o command.o display.o \
    insert.o search.o files.o utils.o

edit : $(objects)
    cc -o edit $(objects)

main.o : defs.h
kbd.o : defs.h command.h
command.o : defs.h command.h
display.o : defs.h buffer.h
insert.o : defs.h buffer.h
search.o : defs.h buffer.h
files.o : defs.h buffer.h command.h
utils.o : defs.h

.PHONY : clean
clean :
    rm edit $(objects)
```

改写后：

```makefile
objects = main.o kbd.o command.o display.o \
    insert.o search.o files.o utils.o

edit : $(objects)
    cc -o edit $(objects)

$(objects) : defs.h
kbd.o command.o files.o : command.h
display.o insert.o search.o files.o : buffer.h

.PHONY : clean
clean :
    rm edit $(objects)
```

确实简洁了不少：）这种风格能让我们的makefile变得很短，但我们的文件依赖关系就显得有点凌乱了。

另外再贴一个很有用的东西：

![image-20231111223818840](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231111223818840.png)

## 2.4 内联汇编

需要学习的其实就是一个基本框架：

```c
    __asm__ volatile (
        "instruction1\n"
        "instruction2\n"
        ......
        ......
        "instruction3\n"
        : [out1] "=r" (v1),[out2] "=r" (v2)
        : [in1] "r" (v1), [in2] "r" (v2)
        : "memory"
    );
```

其中，三个`:`将汇编部分分成了四部分：

- 第一部分是汇编指令，指令末尾需要添加 '\n'。
- 第二部分是输出操作数部分。
- 第三部分是输入操作数部分。
- 第四部分是可能影响的寄存器或存储器，用于告知编译器当前内联汇编语句可能会对某些寄存器或内存进行修改，使得编译器在优化时将其因素考虑进去。如果不添加影响的寄存器或内存，目标寄存器可能同时用于参数传递，导致最终结果不符合预期。

这四部分中后三部分不是必须的。

`[in] "r" (in)` 代表着将 `()` 中的变量 `in` 放入寄存器中，并将绑定数据到`[]`中命名的符号 `in` 中去。

`[out] "=r" (out)`代表着将汇编指令中`%[out]`的值更新到变量`out`中。 

`[reg] "+r" (reg)`代表着既将 `()` 中的变量 `reg` 放入寄存器中，又将汇编指令中`%[reg]`的值更新到变量`reg`中。 

剩下的就是一些很简单的汇编知识了。

示例二中的代码其实可以换一种方式写：

```C
#define write_csr(reg, val) ({
    __asm__ volatile (
    	"csrw " #reg ", %0" 
    	:
    	: "r"(val)
    ); 
})
```

示例二定义了一个宏，其中`%0`代表着输出输入部分的第一个符号，即`val`。

`#reg`是 c 语言的一个特殊宏定义语法，相当于将 reg 进行宏替换并用双引号包裹起来。

## 2.5编译相关知识

GNU的ld也就是链接器，用于将多个 `.o` 文件和库文件链接成为可执行文件。在操作系统开发过程中，为了指定程序的内存布局，ld会使用连接脚本 (Link Script) 来控制，在 Linux Kernel 中链接脚本被命名为 vmlinux.lds。

![image-20231112014306158](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231112014306158.png)

链接脚本中有 `.` 、`*` 两个重要的符号。

单独的 `.` 在链接脚本代表当前地址，它有赋值（`_start = . `）、被赋值（`. = BASE_ADDR`）、自增等操作。

而 `*` 有两种用法，其一是 `*()` 在大括号中表示**将所有文件中符合括号内要求的段放置在当前位置**，其二是作为通配符。

除了一开始的几个对链接地址和程序入口的声明，链接脚本的主体是 SECTIONS 部分，在这里链接脚本的工作是将程序的各个段按顺序放在各个地址上，在例子中就是从 0x80200000 地址开始放置了`.text`，`.rodata`，`.data`和`.bss`段。各个段的作用可以简要概括成 :

| 段名   | 主要作用                         |
| ------ | -------------------------------- |
| text   | 存放执行的代码                   |
| rodata | 存放常量等只读数据               |
| data   | 存放已初始化的静态变量、全局变量 |
| bss    | 存放未初始化的静态变量、全局变量 |

在链接脚本中可以自定义符号，例如以上所有`_s`与 `_e`开头的符号都是我们自己定义的。

# 3 实验步骤

## 3.1 完善head.S

在这一步我们需要完成的只有两个部分：

- 为即将运行的第一个C程序设置程序栈（4KB），位于 `_end` 之后的空间，也就是说我们只需要将sp指向_end以后4KB的位置，将其作为栈顶，防止后续过程中对 `.bss` 造成误改。
- 然后设置一条跳转指令，跳转至 `main.c` 中的 `start_Kernel` 函数。

需要注意的是，我们这里必须有一个叫 `_start` 的标号，用作程序的入口，因为在 `vmlinux.lds` 中设置程序入口是 `_start` .

代码实现如下：

```asm
.extern start_kernel

.section .text.entry
.globl _start, _end
_start:
    la sp, _end+0x1000
    j start_kernel
```

## 3.2 完善Makefile文件

同样的，在这个部分我们实际上需要做到了能够在根文件夹输入make之后能够相应根文件夹下发的操作需求，那么我们可以仿照其他文件夹下的Makefile，包括 lab3/init/Makefile 和 lab3/arch/riscv/kernal/Makefile 两个Makeflie文件下能看到

```makefile
####-------------------------------lab3/arch/riscv/kernal/Makefile--------------------------------####
ASM_SRC		= $(sort $(wildcard *.S))
C_SRC       = $(sort $(wildcard *.c))
OBJ		    = $(patsubst %.S,%.o,$(ASM_SRC)) $(patsubst %.c,%.o,$(C_SRC))

all:$(OBJ)


%.o:%.S
	${GCC}  ${CFLAG} -c $<

%.o:%.c
	${GCC}  ${CFLAG} -c $<

clean:
	$(shell rm *.o 2>/dev/null)
####------------------------------------lab3/init/Makefile---------------------------------------####
C_SRC       = $(sort $(wildcard *.c))
OBJ		    = $(patsubst %.c,%.o,$(C_SRC))

file = main.o
all:$(OBJ)

%.o:%.c
	${GCC} ${CFLAG} -c $<
clean:
	$(shell rm *.o 2>/dev/null)
```

因此我们比较省事情的一种方法是直接复制 lab3/init/Makefile 的内容到 lab3/lib/Makefile 下。甚至我们看到在助教给的代码中有一个根本没用到的file变量，非常多余。

另外我们也可以自己写一份Makefile。因为我们知道在这个Makefile里我们需要做的只有在参数为all的时候用gcc生成 `.c` 文件的 `.o` 文件，在参数为clean的时候删掉所有生成的 `.o` 文件。

代码如下：

```makefile
all:
	${GCC} ${CFLAG} -c
clean:
	$(shell rm *.o)
```

其中的变量 `{GCC}` 和变量 `{CFLAG}` 虽然不在这个文件中定义了，但是在 lab3/Makefile 中定义了，也就是 lab3/lib/Makefile 的上级Makefile中有的定义可以直接继承。

在其他makefile里有的 `$<` 符号表示依赖项中的第一个文件名。除此之外还有类似的：

`$^` 符号表示所有的依赖文件列表，也就是当前规则所依赖的所有文件，以空格为分隔。

`$@` 符号表示目标文件名，在一个规则中，目标文件通常是由规则的第一个目标定义的。

另外，在我们的Makefile文件中也频繁的出现一些通配符：

> 1. *(星号)：表示任意长度的任意字符（包括空字符），可以出现在文件名中的任意位置。
> 2. ? (问号)：表示一个任意字符，且只能替代一个字符。
> 3. [] (中括号)：可匹配其中某个指定字符，可以出现在文件名中的任意位置，如 [abc] 表示可以匹配 a、b、c 中任意一个字符。
> 4. {} (花括号)：可使用逗号隔开的多个字符串中的一个。例如 {a,b,c} 表示可以匹配字符串 a、b、c 中的任意一个。

除此之外我们还看到一个 `%` 百分号，在Linux中，**`%`不是通配符**，而是一种特殊的替换符号，在一些命令或脚本中可以使用。

在 `make` 命令中，`%` 可以匹配任意字符序列，用于表示规则中的通配符，如 `%.o` 可以匹配所有以 `.o` 为后缀的文件名。这样就可以很方便地使用单一规则来生成多个目标文件。

需要注意的是，`%.o` 只能匹配在当前目录下的 `.o` 文件，不能匹配所有的`.o`文件。在这种情况下，需要使用相应的通配符来匹配不同目录下的`.o`文件。

## 3.3 补充sbi.c

为了补充sbi.c，我们只需要完成 `sbi_ecall` ，在这个函数中，我们需要完成以下内容：

1. 将 ext(Extension ID) 放入寄存器 a7 中，fid(Function ID) 放入寄存器 a6 中，将 arg[0-5] 放入寄存器 a[0-5] 中。
2. 使用`ecall`指令。`ecall`之后系统会进入 M 模式，之后 OpenSBI 会完成相关操作。
3. OpenSBI 的返回结果会存放在寄存器 a0、a1 中，其中 a0 为 error code，a1 为返回值，我们用 sbiret 结构来接受这两个返回值。

再结合我们前面学的内联汇编和RISC-V的知识，我们可以得到以下代码：

```C
#include "types.h"
#include "sbi.h"

struct sbiret sbi_ecall(int ext, int fid, 
                        uint64 arg0,uint64 arg1, uint64 arg2,
                        uint64 arg3, uint64 arg4,uint64 arg5){
    struct sbiret ret;
    __asm__ volatile (
        "mv a0, %[arg0]\n"
        "mv a1, %[arg1]\n"
        "mv a2, %[arg2]\n"
        "mv a3, %[arg3]\n"
        "mv a4, %[arg4]\n"
        "mv a5, %[arg5]\n"
        "mv a6, %[fid]\n"
        "mv a7, %[ext]\n"
        "ecall"
        : [arg0] "+r" (arg0), [arg1] "+r" (arg1)
        : [arg2] "r" (arg2), [arg3] "r" (arg3), [arg4] "r" (arg4), [arg5] "r" (arg5), [fid] "r" (fid), [ext] "r" (ext)
        : "a0", "a1", "a2", "a3", "a4", "a5", "a6", "a7", "memory"
    );
    ret.error = arg0;
    ret.value = arg1;
    return ret;
}
```

前面的部分是给a0到a7依次赋值，最后在进行一次ecall的调用。然后我们需要做的还有完成输入输出的规定，以及对空间保护的说明。最后返回的时候也只需要将arg0和arg1作为一个结构体导出即可。

## 3.4 puts()和puti()

在完成了sbi_scalll后，我们只需要做好参数的传递工作即可。根据下表，我们可以知道如果我们要输出的话，ext的值应该是0x01。

| Function Name                    | Function ID | Extension ID |
| -------------------------------- | :---------- | ------------ |
| sbi_set_timer 设置时钟相关寄存器 | 0           | 0x00         |
| sbi_console_putchar 打印字符     | 0           | 0x01         |
| sbi_console_getchar 接受字符     | 0           | 0x02         |

也就是说我们可以再封装一个函数：void print_a_char_in_screen(int ch)；具体实现如下：

```c
 void print_a_char_in_screen(int ch){
	sbi_ecall(0x01, 0, ch, 0, 0, 0, 0, 0);
}
```

这样一来我们只需要传递一个参数ch(要打印的字符)即可。

对于puts，我们可以很简单的直接按一个个字符输出，从左往右顺序输出。

```C
void puts(char *s) {
  while (*s) print_a_char_in_screen(*s++);
}
```

对于puti，我们可以通过判断是否是0来减少循环次数，提高程序时间。另外我们还需要判断这个整数是否是正整数，如果不是我们还需要提前输出一个问号。如果我们要判断的对象是正数的话，那我们可以用一个字符数组来储存每一个位置的数值对应的char，代码实现如下:

```c
void puti(int x) {
  char tmp[16];     //to store the string of int
  int i = 0;
  if (x == 0) {
    print_a_char_in_screen('0');//if it's zero, we can finish early
    return;
  }
  if (x < 0) {
    print_a_char_in_screen('-');//if it's negative number,we need add a '-',before processing.
    x = -x;   //turn it into a postive number.
  }
  while (x) {
    tmp[i++] = '0' + x % 10;    
    x /= 10;
    // convert a number into a char. 12345 is stored as below:
    //  +-----------+-----+-----+-----+-----+-----+
    //  |   index   |  0  |  1  |  2  |  3  |  4  |
    //  +-----------+-----+-----+-----+-----+-----+
    //  |  content  | '5' | '4' | '3' | '2' | '1' |
    //  +-----------+-----+-----+-----+-----+-----+
  }
  while (i) print_a_char_in_screen(tmp[--i]);
}
```

## 3.5 修改def.h

补充def.h 的代码，完成read_csr的宏定义。实现代码如下：

```makefile
#define csr_read(csr)                       \
({                                          \
    register uint64 __v;                    \
    asm volatile ("csrr %0, " #csr          \
                    : "=r" (__v) :          \
                    : "memory");            \
    __v;                                    \
})
```

## 3.6 实验结果

![image-20231112030336017](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231112030336017.png)

# 4 思考题

## 4.1 System.map

### 题目描述

编译之后，通过 System.map 查看 vmlinux.lds 中自定义符号的值，比较他们的地址是否符合你的预期。

### 操作

![image-20231112114918140](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231112114918140.png)

整理到一个Excel表格中按地址排序后得到：

| 地址       | 自定义符号名           |
| ---------- | ---------------------- |
| 0x80200000 | BASE_ADDR              |
| 0x80200000 | _start                 |
| 0x80200000 | _stext                 |
| 0x8020000c | sbi_ecall              |
| 0x80200100 | start_kernel           |
| 0x80200140 | test                   |
| 0x80200150 | puts                   |
| 0x802001c0 | print_a_char_in_screen |
| 0x80200210 | puti                   |
| 0x80200308 | _etext                 |
| 0x80201000 | _srodata               |
| 0x80201019 | _erodata               |
| 0x80202000 | _sdata                 |
| 0x80202000 | _edata                 |
| 0x80202000 | _sbss                  |
| 0x80202000 | _ebss                  |
| 0x80202000 | _end                   |

可以看到在及地址之后是程序入口，然后是一个text段的头标记，然后是在程序段中依次实现的一些函数标号，再之后是text段的尾标记。之后的rodata段有19字节的大小，data，bss段没有大小，所以和_end的标记在同一个位置。rodata段中应该还储存了一些只读的变量，所以会有19字节的大小。可以说是基本符合预期。

## 4.2 特权态和中断

### 题目描述

在你的第一条指令处添加断点，观察你的程序开始执行时的特权态是多少，中断的开启情况是怎么样的？

**提示**：可以尝试在第一条指令处插入一些特权操作，如`csrr a0, mstatus`，观察调试现象，进行当前特权态的判断

### 操作

![image-20231112195503439](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231112195503439.png)

 可以看到特权级别为1，也就是Supervisor级别。

终端开启情况如图。

## 4.3 各内存段的内容

### 题目描述

在你的第一条指令处添加断点，观察内存中 text、data、bss 段的内容是怎样的？

### 操作

按照上一题的方法进行插入断点的操作。再敲c运行到第一条指令停住，通过 `x/<n>xw <lable>` 可以看到 <lable> 处的内存空间中的内容。那么我们只需要依次看每一个段的内容即可。

#### text段：

![image-20231112182651571](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231112182651571.png)

#### rodata段

可以看到rodata段里面存放了" ZJU Computer System II\n"这个字符串。

![image-20231112195943106](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231112195943106.png)

#### data段和bss段

根据我们前面system.map可以知道这两个段是空的，没有实际上的内容，在gdb中也可以看到：

![image-20231112200119484](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231112200119484.png)

## 4.4 用汇编代码传递参数

### 题目描述

尝试从汇编代码中给 C 函数 start_kernel 传递参数

### 操作

还好学了小白老师的汇编，这道题其实很简单，只需要知道RISC-V是怎么做传递参数的约定的就行。

![A](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/A.png)

我们给a0设置一个常数，然后再让start_kernal设置一个参数x并在head.S中按照约定做好参数传递即可。

代码实现如下：

![image-20231112190705703](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231112190705703.png)
![image-20231112190723839](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231112190723839.png)

最后的输出结果是：

![image-20231112190840780](./Lab3%EF%BC%9ARV64%E5%86%85%E6%A0%B8%E5%BC%95%E5%AF%BC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20231112190840780.png)
