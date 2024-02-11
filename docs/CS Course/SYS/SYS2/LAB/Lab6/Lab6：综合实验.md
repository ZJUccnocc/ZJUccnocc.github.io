# Lab6：综合实验

## 3220103648 吴梓聪

# 1 实验目的

- 学习 OS 在硬件层面的抽象
- 完善自己的 CPU Core，运行起自己编写的 binary 程序

# 2 实验课上提及的注意事项

## 2.1 `CSRModule` 接口

模仿 `Regfile` 写

## 2.2 异常检测

`pc_wb` 寄存器是写回的地址，也就是当前指令的地址。

`valid_wb` 表示是否有效，是否与 `CSR` 相关.

`time_int` 表示是否有时钟中断

`csr_ret[1:0]` 表示是否是ret指令，如果是 `sret` 则 `csr_ret[0] == 1` ，如果是 `mret` 则 `csr_ret[1] == 1` ，否则就都是0。

`except_commit` 里面是三条线，分别是 `epc` ，`ecause` ，`etval` 三个寄存器。

`priv[1:0]` 表示当前的特权态，3~0分别是MHSU四个态。

`switch_mode` 表示是否需要切换状态，有异常或者中断的时候会要切换状态。

`pc_csr` 表示如果要切换状态的话你切换的目标地址。

如果要 `switch_mode` 了就把五个街段寄存器内的指令全部 `flush` 掉，将pc寄存器更新为跳转的目标地址。

1. 首先需要再阶段中间检查是否有异常中断的发生。
   在每一个部分增加一个检查模块（虽然我们只检查 `ecall` 的异常），检查是否有异常产生。如果有异常就记录下当前的pc，cause，value。

2. 异常检测之后需要在阶段间进行传递。

   将之前记录下来的三个值存进对应的异常中间寄存器。而二路选择器是用来选择应该处理哪一个阶段的异常，具体应该是处理这条指令在越早阶段出现的异常。如果这个指令的前几个阶段没有异常，我们才会用这个阶段的。

3. 将异常指令的一些控制信号给清零

   指令发生异常之后，还要将异常指令的一些控制信号给清零，但因为流水线的原因，清零需要在下一个阶段执行，也就是给后面的下一阶段寄存器同步传一个 `flush` 信号

4. 特殊情况
   但这里又会遇到一个问题，就是你如果前一条指令被 `stall` 了，那么你现在这条指令也会 `stall` ，那么你的清空就执行到了异常的前一条指令上。这个需要在MEMWB中做更改：

   ![image-20240210211557860](./Lab6%EF%BC%9A%E7%BB%BC%E5%90%88%E5%AE%9E%E9%AA%8Cimg/image-20240210211557860.png)

5. 如果异常发生了，需要关闭内存访问等开关

   除了EXEMEM阶段的写内存信号以外还有PC阶段读内存的信号，像 `switch_mode` 发生之后读写信号也需要被关掉等等

6. 异常切换的时候碰上指令正在读内存

   因为在有了总线之后，读内存变成了一个比较漫长的过程，这个时候可能就会发生上述情况。即假如现在总线正在向内存取指令，这时突然发生了时钟中断，时钟中断发生后，我们会将 `pc` 改为 `pc_csr` ，但这时下一条从MEM中返回的指令是 `old_pc` 对应的，而不是跳转之后的。那么这个时候就需要将这个读到的内容舍弃，等待新读到的内容到达。

## 2.3 异常检测模块

模块都写好了，只需要连线即可

## 2.4 仿真

需要把 `clock.c` 文件中的time改为0x10000，然后还要把 `rand.c` 的正常内容注释掉，换成递增。

然后先在 `testcode` 里面 `make clean` ，然后再 `make` 然后再回到lab6的目录里make得到如下内容：

![image-20240210211600017](./Lab6%EF%BC%9A%E7%BB%BC%E5%90%88%E5%AE%9E%E9%AA%8Cimg/image-20240210211600017.png) 

另外有一种方式：`make 2> log` 可以看到一段文字显示的内容：

```shell
m
m
_
i
n
i
t


p
...
```

也就是将原先在lab5输出的内容放到这里每一个字母换行一次。还可以在lab6下面找到一个叫log的文件，里面记录了输出的所有记录，包括所有汇编指令，还有现在进行到哪个函数了。

## 2.5 下板

`make board_sim` 用于生成下板的代码，完成之后会有很多新的文件产生。然后再接上 `make bitstream` 就可以得到一份 bit 流文件，后面还有一些配置，可以见实验手册。但是在综合之前还需要配置一下 `ip` 核，具体操作详见 [zyygg的1:34:33开始的视频教学](https://classroom.zju.edu.cn/livingpage?course_id=54753&sub_id=1027221&tenant_code=112&sub_public=1) 或者是 [实验指导](https://zju-sys.pages.zjusct.io/sys2/sys2-fa23/ipCore/) 

# 3 实验仿真准备

## 3.1 环境配置

### 3.1.1 文件配置

将lab2的硬件代码全部拷贝到 `/submit` 文件夹下，并且将需要合并的代码做一个简单的合并。

将lab5的软件代码全部拷贝到 `/testcode/kernel` 文件夹下。`sbi.c` 和 `mini_sbi.c` 中给出了重复的函数，删去sbi内的部分即可。

### 3.1.1 统一label

将一些出现的 `_ekernel` 的地方改成 `_end` 确保不再出现相关的报错。

### 3.1.2 更新 `Verilator`

首先安装点必要的软件包：

```shell
sudo apt-get install git perl python3 make autoconf g++ flex bison ccache
sudo apt-get install libgoogle-perftools-dev numactl perl-doc
sudo apt-get install libfl-dev
sudo apt-get install zlibc zlib1g zlib1g-dev
```

然后从 `github` 获取源文件，准备安装环境

```shell
git clone https://github.com/verilator/verilator
cd verilator
autoconf
./configure
make
```

在一段漫长的编译过程后终于可以开始安装：

```shell
sudo make install
```

最后可以通过指令查看安装情况：

```shell
verilator --version
```

### 3.1.3 调整 `Makefile` 

- 更改常量

  ```makefile
  In ”/lab6/testcode/Makefile“
  - -march=rv64i-zicsr
  + =march=rv64imafd
  ```

- 便捷化调试

  ```
  In "/lab6/Makefile"
  verilate:testcode/testcase.elf
  + 	make -C testcode clean
  + 	make -C testcode sim
  	mkdir -p $(DIR_BUILD)
  ```

  这样一来只需要在lab6下执行make就可以顺带把软件部分的也重新清空并且编译一次。省时省力小妙招~

## 3.2 硬件部分代码调整

### 3.2.0 搬运

在搬运的过程中还要理解 `zjgg` 给的代码的变量和自己之前代码变量的相同和不同，而且大部分是看名字猜，只能先搬着，然后慢慢debug，如果有下辈子希望可以有点注释555。

### 3.2.1 `.v` 文件的升级

因为现在用的是最新的 `Verilator` ，我们需要改掉一些之前写的不规范的代码，比如位宽对齐问题。（虽然很无脑，但是改了好久...）

### 3.2.2 循环赋值问题

我编译着编译着，把位宽还有变量名之类的弱智问题改完之后，就只剩下了两个循环组合逻辑的警告：

![image-20240210211612203](./Lab6%EF%BC%9A%E7%BB%BC%E5%90%88%E5%AE%9E%E9%AA%8Cimg/image-20240210211612203.png)

这个bug也算是老熟人了（真的不想再遇到了），说明应该是有在连线上的问题。回头检查了一下数据通路，发现一个问题。可以看到两个问题都在 `Memmap` 上，而这个模块实际上的作用就是判断访存的地址是属于Mem还是IO，并且对应开启一些使能信号。

![image-20240210211614401](./Lab6%EF%BC%9A%E7%BB%BC%E5%90%88%E5%AE%9E%E9%AA%8Cimg/image-20240210211614401.png)

我们可以看到 `Axi_lite_Core` 在和 `Memmap` 以及 `MemFSM` 做交互。而我们写的Core模块输出的信号除了模拟信号以外只有 `pc` `address` `we_mem` `wdata_mem` `wmask_mem` `re_mem` `if_request` `switch_mode` `time_int` 九个信号。

![image-20240210211617214](./Lab6%EF%BC%9A%E7%BB%BC%E5%90%88%E5%AE%9E%E9%AA%8Cimg/image-20240210211617214.png)

将其整理后发现 `Core` 传给 `memmap` 的信号是`we_mem` `re_mem` `address` ，不难猜测两个使能信号应该问题不大，主要是这个 `address` 信号。在Lab2中，我们的 `address` 是ALU的计算结果直接传递到访问内存的过程中，也就是说在EXE阶段之后我们的address就已经传过去了，而不是在MEM阶段。之前这么干的原因主要在于省略访问内存时的⼀拍stall。

但现在去看 `Memmap.v` 的内容:

![image-20240210211619633](./Lab6%EF%BC%9A%E7%BB%BC%E5%90%88%E5%AE%9E%E9%AA%8Cimg/image-20240210211619633.png)

发现我们报错的两个变量的赋值都是和我们可爱的 `address` 有关。但实际上 `Memmap` 这个模块接受计算的 `address` 应该是在MEM阶段的指令的 `address` 而不是为了节省 `stall` 而前递的EXE阶段的 `address` 。如果传入的是EXE阶段的指令的 `address` 的话，计算得到的所有结果对应也都是EXE阶段的结果，但是这个又会返回去对MEM及其以后的所有阶段产生影响，也就形成了回环。

因此我们如果要解决这个 `bug` 就得把之前lab2的前递代码取缔掉（在不动 TAgg 代码的前提下）。

我的解决方法是回归MEM阶段的本真，将所有读写内存的操作放回到MEM阶段执行，而不是像Lab2一样提前把一些东西传进总线。具体其实就是改一下和内存交互的那几个信号，将原先与EXE相关的都延迟到MEM即可。

但这也会需要我们对我们的状态机做出一定更改。为了保障我们做的调整不会影响到我们对指令的读取，我们需要将if_stall延长一拍。

## 3.3 软件部分代码调整

### 3.3.1 根据 `math.h` 调整

将所有软件代码中的乘除模号分别替换为 `int_mul()` ，`int_div()` ，`int_mod()` 三个函数。此处不一一展示了。

# 4 实验过程

## 4.1 `CSRModule` 

由于本次实验的目标中要求我们写的CPU可以识别和处理CSR特权级别指令，所以我们需要将这些指令增加到我们可以接受的指令集中，对其进行解码译码，并通过 `CSRModelu` 进行寄存器的读取和写入。

### 4.1.1 解码译码

好的这个时候不得不掏出我们祖传的手册，可以看到六位特权指令的 `opcode` 都是 `1110011` ![image-20240210211622871](./Lab6%EF%BC%9A%E7%BB%BC%E5%90%88%E5%AE%9E%E9%AA%8Cimg/image-20240210211622871.png)

```verilog
parameter:
...
CSRR  = 7'b1110011;

CSRR:begin
	case(funct3)
		3'b000:  decode = 22'b0010_1000_0000_0000_0000_0000;//mret,sret
		3'b001:  decode = 22'b0010_0000_0000_0000_1000_1000;//csrrw
		3'b010:  decode = 22'b0010_0000_0000_0000_1000_1000;//csrrs
		3'b011:  decode = 22'b0010_0000_0000_0000_1000_1000;//csrrc
		3'b101:  decode = 22'b0010_0111_0000_0000_0100_1000;//csrrwi
		3'b110:  decode = 22'b0010_0111_0000_0000_0100_1000;//csrrsi
		3'b111:  decode = 22'b0010_0111_0000_0000_0100_1000;//csrrci
		default: decode = 22'b0000_0000_0000_0000_0000_0000;
	endcase
end
```

根据数据通路图，我们知道在IDEXE阶段还需要完成对特权寄存器的读操作，那么我们就可以根据我们更多见到的伪指令分为三类，只读不写，只写不读，又读又写。

**只读不写**

形如 `csrr <reg>,<priv_reg>` 的伪指令的意思是将 `<priv_reg>` 的值取出来放到 `<reg>` 中。其真实写法应该是：`csrrw <reg>, <priv_reg>, x0` 但实际上，如果直接这样执行会导致 `<priv_reg>` 被 `x0` 写入，因此我们需要额外对是否写入 `<priv_reg>` 做判断。

```Verilog
assign IfCsrOnlyRead = (op_code == CSRR && inst[19: 15] = 5'b0) ? 1 : 0;
```

**只写不读**

同样，形如 `csrw <priv_reg>,<reg>` 的伪指令的真实写法应该是：`csrrw x0, <priv_reg>, <reg>` 。这样除了我们像达成的效果以外还会额外导致 `x0` 有一个被 `<priv_reg>` 的尝试，但是因为x0无法被写入，所以这个地方不需要额外操作。

**又读又写**

形如 `csrrw <reg0>,<priv_reg>,<reg1>` 就是字面意思上的完成将 `<priv_reg>` 放入 `<reg0>` ，再将 `<reg1>`  放入 `<priv_reg>` 的过程。也不需要做额外的判断。

对于这三种不带立即数的指令，我们解码译码的时候只需要令 `alu_bsel=0` 并且 `alu_op = ADD` 即可保证最后的 `alu_result` 是我们要的结果。再令 `wb_sel=2'b01` 保证在写回阶段选择写回  `alu_result` 。	

### 4.1.2 模块实现

根据实验指导，`CSRModule` 模块应该是在ID阶段和EXE阶段之间，和Reg的位置相同，用来实现提取特权级别的指令。

#### 4.1.2.1 阶段间寄存器修改

为了让这个模块能够正常运行，我们需要在一些阶段间增加传递 `IfCsrOnlyRead` ，`Except` 信号，但是因为这个 `Except` 是之后在异常检测模块使用的，所以我们这里不展开描述，并且这个寄存器实际上也不是增加在阶段间寄存器里的。而这个 `IfCsrOnlyRead` 在上一节也讲过了，主要是表征这条指令是不是 `CSR` 指令以及是不是只读的类型。然后这个作为指令的属性，需要用阶段间寄存器，跟着一条指令同步往下传。

#### 4.1.2.2 `csr_ret` 指令

TAgg 在课上曾提到 `csr_ret` 是作为判定这个指令是不是 `sret` 或者 `mret` 指令，在具体的实现代码中可以看到：

```verilog
assign s_ret=csr_ret[0]&~s_trap&~m_trap;
assign m_ret=csr_ret[1]&~s_trap&~m_trap;
```

`csr_ret[0]` 表征当前这条指令是不是 `sret` 指令，`csr_ret[1]` 表征当前这条指令是不是 `mret` 指令。所以在传入的时候，我们对 `csr_ret` 的赋值应该如下：

```verilog
assign mret = (inst_WB == 32'h30200073 )? 1 : 0;
assign sret = (inst_WB == 32'h10200073 )? 1 : 0;
csr_ret = {mret,sret}
```

#### 4.1.2.3 连线结果

```Verilog
	wire mret = (ME_WB_inst == 32'h30200073)?1:0;
    wire sret = (ME_WB_inst == 32'h10200073)?1:0;
    CSRModule csrmodule(
        .clk(clk),
        .rst(~rstn),
        .csr_we_wb(ME_WB_IfCsrOnlyRead),
        .csr_addr_wb(ME_WB_inst[31: 20]),
        .csr_val_wb(ME_WB_mem_rdata),         //.......
        .csr_addr_id(IF_ID_inst[31:20]),
        .csr_val_id(csr_val_id),    // out

        .pc_wb(ME_WB_pc),
        .valid_wb(ME_WB_valid),
        .time_int(time_int),
        .csr_ret({mret,sret}),
        .except_commit(ME_WB_except_commit),

        // out
        .priv(priv),
        .switch_mode(switch_mode),
        .pc_csr(pc_csr),

        .cosim_interrupt(cosim_interrupt),
        .cosim_cause(cosim_cause),
        .cosim_csr_info(cosim_csr_info)
    );
```

## 4.2 `ExceptExamine`

本实验需要处理的异常指令只有一类—— `ecall` 指令。通过分析代码可以知道：

- 检测模块接受外部ID阶段的 `inst` ，`pc` ，`stall` ，`flush` 等信号。

- 用 `InstExamine` 检查当前的代码是否是一条异常代码，检测结果会由 `InstExamine` 模块打包输出。

- 如果有新的异常在当前阶段产生，会将 `except_happen_id` 置1。

  ```Verilog
  //先判断这条指令在前几个阶段是否有异常发生，如果有则选择前几个阶段检测到的结果，否则用新的结果
  assign except=except_id.except?except_id:except_new;
  //except_happen_id信号主要用于检测当前阶段是否有产生异常信号
  assign except_happen_id=except_new.except&~except_id.except;
  ```

理论上在每一个阶段都需要做一次异常检测，因为不同的异常可能会发生在不同的阶段，但是因为我们只检测一种异常—— `ecall` 指令，而这个异常只会在ID阶段就可以发现，所以我们在这个实验中只需要一个异常检测，剩下的异常检测可以先不写。

但是因为最后这个信号需要通过阶段间寄存器传递到WB阶段，在传到WB阶段后再给到 `CSRModule` ，`CSRModule` 根据是否有异常来做出对应的反应。所以我们还是需要在阶段间寄存器上做手脚。

但是因为 `IDExceptExamine.sv` 中本身自带了一个时序逻辑，所以可以将他看作和ID_EX_Regs阶段间寄存器同级的模块：

```
|-----IF------|------ID------|------EX------|------ME------|------WB------|
|--------IF_ID_Regs--+--ID_EX_Regs-----EX_ME_Regs-----ME_WB_Regs----------|
		 		     |-IDExceptExam
```

因为我们没有在IF阶段对指令进行检查，所以我们需要给 `IF_ID_except_commit` 赋一个初始值。

```Verilog
ExceptStruct::ExceptPack IF_ID_except_commit={except: 1'b0, epc:64'b0, ecause:64'b0,etval: 64'b0};
```

除此之外还需要在两个阶段间寄存器传递Except结构体。需要注意的是，如果用到结构体需要把文件后缀改成 `sv` ，并且还要引用结构体的声明文件。

## 4.3 异常（Exception）处理

我们常见的异常主要有以下几种情况：

1. 非法指令：在低特权态试图运行高特权态的指令，或者试图操作高特权态才能访问的寄存器。或者一条无法识别的指令或者不支持的指令或者空指令也会出发非法异常。

2.  `ecall` 异常：调用 `ecall` 时，处理器会判定此条指令为异常，并传递此时的一系列异常信息（通过

   `sbi_ecall` 函数），并根据 `mcause/scause` 的值不同去执行不同的异常处理操作。

判断是否异常的指令已经由心地善良的 TAgg 完成了，我们只需要在 `CSRModule` 接收到异常信息时对其做出相应的处理方法即可。具体 `CSRModule` 会发生哪些输出信号的变化呢：

1. 将 `switch_mode` 置 `1` ，告诉流水线要准备切换特权态了。
2. `priv` 如果原来为 `2'b11` (M) 要变为 `2'b00` (U)，如果是 `2'b00` (U) 要切换为 `2'b11` (M) 
3. `pc_csr` 输出异常指令的 `pc`

我们要做的是：

1. 将IF阶段的 `pc` 切换为 `pc_csr` 
2. 由于 `CSRModule` 收到异常信号时，异常指令已经到了WB阶段，其他阶段的指令是异常指令之后的指令，很可能存在一些冲突，所有我们需要将所有阶段间寄存器 `flush` 掉，在处理完异常后在回到异常指令的pc处执行后续指令。

### 4.3.1 IF阶段pc处理

```Verilog
always @(posedge clk or negedge rstn) begin
    if(~rstn) IF_pc <= 0;
+   else if(switch_mode) IF_pc <= pc_csr;
    else if(~pc_stall) IF_pc <= npc;
end
```

对于 `npc` 的处理在 `mux_PC` 组合电路模块下完成。虽然理论上我们也可以把 `switch_mode` 输入到 `Mux_PC` 模块下完成对 `npc` 的选择。

### 4.3.2 flush处理

```verilog
module RaceController{
    ...
+   input switch_mode,
    ...
}
- assign IF_ID_flush = (Control_Hazard_need_flush||if_stall)&~IF_ID_stall;
+ assign IF_ID_flush =(Control_Hazard_need_flush||if_stall)&~IF_ID_stall|switch_mode;
- assign ID_EX_flush = Control_Hazard_need_flush & ~ID_EX_stall;
+ assign ID_EX_flush = Control_Hazard_need_flush & ~ID_EX_stall | switch_mode;
- assign EX_ME_flush = Control_Hazard_need_flush&~EX_ME_stall&if_stall;
+ assign EX_ME_flush = Control_Hazard_need_flush&~EX_ME_stall&if_stall|switch_mode;
- assign ME_WB_flush = mem_stall;
+ assign ME_WB_flush = mem_stall | switch_mode;
```

其实就是在每一个flush信号后面或上一个 `switch_mode` 信号即可。

## 4.4 中断（Interrupt）处理

同样本实验简化成只需要我们实现时钟中断即可。在Lab5中，我们也已经在软件层面设置了第一次时钟中断到达时 `I/O Device` 的部分中的 `Timer` 外设会给 `Core` 输入一个 `time_int` 信号来触发硬件层面的时钟中断。

实际上，当遇到时钟中断时，我们可以把它看成是一个异常的发生，只不过这个异常并不是从流水线CPU内部在执行代码的时候出现，而是由CPU以外的硬件产生的。

但是这里又有一个需要注意的地方，也就是老师上课时候提到的：异常切换的时候碰上指令正在读内存。因为在有了总线之后，读内存变成了一个比较漫长的过程，这个时候可能就会发生上述情况。但是这个地方有需要分情况讨论：

1. `valid_mem` 没有返回，但又有了读指令的需求，这个时候得到的指令就是应该舍弃的。
2. `valid_mem` 返回，

即假如现在总线正在向内存取指令，这时突然发生了时钟中断，时钟中断发生后，我们会将 `pc` 改为 `pc_csr` ，但这时下一条从MEM中返回的指令是 `old_pc` 对应的，而不是跳转之后的。那么这个时候就需要将这个读到的内容舍弃，等待新读到的内容到达。

## 4.5 软件有关 `rdtime` 指令的修改

如图，因为 `rdtime` 指令实现太过复杂，TAgg 没有再进一步实现在 `S` 态下读写 `mtime` 的办法，所以我们很多在Lab5中涉及到rdtime指令的内容都要重写或者删除。

![image-20240210211633895](./Lab6%EF%BC%9A%E7%BB%BC%E5%90%88%E5%AE%9E%E9%AA%8Cimg/image-20240210211633895.png)

具体修改如下：

```c
// "/init/test.c" 直接删完，让他一直while(1)就行
  |void test() {
- |   unsigned long record_time = 0; 
  |  while (1) {
- |       unsigned long present_time;
- |       __asm__ volatile("rdtime %[t]" : [t] "=r" (present_time) : : "memory");
- |       present_time = int_div(present_time,10000000);
- |       if (record_time < present_time) {
- |           record_time = present_time; 
- |       }
  |  }
  |}
// "/kernel/arch/riscv/kernel/trap.c"
+ |#include "sbi.h"
+ |unsigned long TIMECLOCK = 0x30000;
  |...
  |  case 5: //Supervisor Mode Timer Interrupt
- |		clock_set_next_event();
+ |		sbi_set_timer(TIMECLOCK);
  |		do_timer();
  |	break;
  |	default: break;
  |...
// "/kernel/arch/riscv/kernel/clock.c && clock.h && clock.o" delete
// "/kernel/arch/riscv/kernel/head.S"
...
- |  rdtime a0
- |  addi a0,a0,0x30000
+ |	 li a0, 0x30000 
- |  la sp, _end+0x1000
+ |  la sp, boot_stack_top+0x1000
	 ......
+ |  boot_stack_top:
+ |  .align 4
...
```

## 4.6 软件有关sbi的修改

主要原因是老师又实现了一个mini_sbi，所以有些调用得有点调整。

```asm
// "/kernel/arch/riscv/kernel/head.S"
...
     li a0, 0x30000
+ |  li a1, 0
+ |  li a2, 0
+ |  li a3, 0
+ |  li a4, 0
+ |  li a5, 0
+ |  li a6, 0
+ |  li a7, 0
	 la sp, boot_stack_top+0x1000
+ |  ecall
```

## 4.6 实验结果

![image-20240210211637850](./Lab6%EF%BC%9A%E7%BB%BC%E5%90%88%E5%AE%9E%E9%AA%8Cimg/image-20240210211637850.png)

# 5 思考题

## 5.1 `putc` 的特权切换

**题目描述**

使用 `putc` 函数输出一个字符 'a' 前后需要发生几次特权态切换，请将切换的状态和切换的原因一一列举出来。

**解答**

我们可以在一次输出的log信息中查找 `putc` 函数，并跟踪他的函数调用链：

![image-20240210211641140](./Lab6%EF%BC%9A%E7%BB%BC%E5%90%88%E5%AE%9E%E9%AA%8Cimg/image-20240210211641140.png)

```
Supervisor (priv == 01)							Machine (priv == 11)
	putc()
	  |
  sbi_ecall()
  	  |
   	  +-----------------------ecall-----------------------+
   	  													  |
   	  											  _sbi_trap_entry()
   	  											  		  |
   	  										     _sbi_trap_handler()
   	  										    		  |
   	  										     _sbi_scall_handler()
   	  										     		  |
   	  										    sbi_console_putchar()
   	  										     		  |
   	  +-----------------------mret------------------------+
   	  |
    finish
```

可以看到一共是有两次进程切换。一次通过 `ecall` 从S态到M态，一次通过 `mret` 从M态到S态。

## 5.2 异常选择

**题目描述**

如果流水线的 IF、ID、MEM 阶段都检测到了异常发生，应该选择哪个流水级的异常作为 trap_handler 处理的异常？请说说为什么这么选择。

**解答**

如果是在**同一个时间**的三个阶段检查到了异常发生，那么应该选择MEM阶段的。因为MEM阶段的指令在执行顺序上是最前面的，理论上当一条指令发生异常就会先处理异常，后面的指令都要等异常处理结束再运行，所以我们要选取三条指令中最早出现异常的指令，也就是MEM阶段的指令。

如果是在**同一条指令**的三个阶段检测到了异常发生，那么应该处理IF阶段的。因为理论上一条指令在发生异常的时候就不会再运行了，需要先处理异常，但是因为流水线的设计，它还是会经历后面的阶段以及每个阶段的异常检测。但实际上我们需要处理的那个异常应该是最早发生的异常，也就是IF阶段的异常。

## 5.3 CSR特权指令

**题目描述**

CSR 寄存器的读写操作如 `csrrw` 、`csrrwi`会不会引入新的 `stall` ？如果会，在你的实现中引入了哪些 `stall` ？可以用 `forward` 技术来减少这部分 `stall` 吗？

**解答**

读操作因为是组合电路，所以不会有 `stall` 产生。

```verilog
module CSRModule{
	...
    input [11:0] csr_addr_id,
    output [63:0] csr_val_id,
    ...
};

wire [4:0] compress_index={csr_addr_id[9],csr_addr_id[6],csr_addr_id[2:0]};
assign csr_val_id=compress[compress_index];
```

写操作因为是和 `Regs` 模块一样的时钟下降沿的时序逻辑，所以会有一拍的 `stall` 产生的。但因为特权指令的格式和一般指令的格式是保持一致的，所以我们在译码正确之后就可以让他这条指令当作一条普通指令来跑，如果像 `csrrw` 、`csrrwi` 指令需要写回，也可以像普通的写回指令一样共用一套 `stall` 的检测和执行机制。所以不需要额外增加 `stall` 的处理。

至于处理的话也可以像Regs一样用旁路的前递机制。检测数据冲突是否发生并且可以前递。

## 5.4 临别赠礼

**题目描述**

可以为系统 II 的课改留下任何宝贵的心得体会和建议吗？（不记录分数，纯属用于吐槽）

**解答**

千言万语汇作五个字：”还是太简单。“

另外实验指导里面好多地方都是直接复制去年的，痕迹很明显哈，记得找时间改改（比如今年的lab4是去年的lab5，导致很多地方出现指代不明的情况）
