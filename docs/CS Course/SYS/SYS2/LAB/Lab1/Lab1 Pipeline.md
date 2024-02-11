# Lab1 3220103648 吴梓聪

#  Pipeline CPU

## 一 实验目的

- **理解流水线的基本概念与思想：**流水线是一种将指令执行过程划分为多个阶段，并在不同阶段上并行执行的技术。通过实验，可以深入理解流水线的原理和工作方式，以及如何有效利用硬件资源来提高指令执行的效率。
- **基于在单周期 CPU 中已经实现的模块，实现 5 级流水线框架：**实验中需要设计和实现流水线CPU，这涉及到对指令集架构、流水线结构和控制信号的设计。通过实验，可以学习和掌握流水线设计的基本方法，包括流水线阶段划分、数据通路设计、控制单元设计等。
- **分析流水线性能与优化：**通过实验，可以对比分析不同流水线设计的性能差异，包括吞吐量、时钟周期、延迟等指标。通过对流水线的优化，例如通过增加流水线级数、改进流水线冲突处理等，可以了解优化对性能的影响，并学习如何提高流水线CPU的性能。
- **熟悉硬件描述语言和仿真工具：**在实验过程中，进一步学习使用硬件描述语言Verilog来描述和实现流水线CPU的硬件电路。通过实验，可以熟悉硬件描述语言的基本语法和使用方法。
- **理解数据竞争、控制竞争、结构竞争的原理和解决方法：**在流水线设计过程中，不可避免的会出现前后几条指令的数据冲突和控制冲突以及结构冲突，在本实验中主要采用flush和stall还有假设全不跳转的方法解决，结构冲突由于我们将 `InstRAM` 和 `DataRAM` 分开，因此暂时不会出现结构竞争的现象。
- **加入冲突检测模块和 stall 执行模块解决竞争问题：**实验中通过设计了 `RaceControler` 模块，用于检测流水线进行过程中存在的所有数据冲突和控制冲突，用stall停住后续指令的传入，用flush清空错误指令的刷新。
- **理解流水线设计在提高 CPU 的吞吐率，提升整体性能上的作用与优越性：**我们通过设计流水线将一个CPU的执行指令的过程划分为五个小过程，在每一个周期内，执行一个小过程的内容，因此将原先一个相对比较长的周期划分为好几个较短的周期，同时通过往每个周期塞入不同的指令，实现在一个瞬间同步执行多条指令。这样虽然会提高CPU的CPI，但是因为每个Circle的长度缩短了，所以总体的运行性能还是有显著提高的。

## 二 实验原理

1. 流水线概念：流水线是将指令执行过程划分为多个阶段，并在不同阶段上并行执行的技术。通过流水线，可以提高指令执行的并行性和效率。
2. 指令集架构：指令集架构定义了计算机体系结构中的指令集和对应的操作。在流水线CPU设计中，需要理解和遵循所采用的指令集架构，包括指令格式、操作码、寄存器等。
3. 流水线阶段划分：流水线需要将指令执行过程划分为多个阶段，常见的包括取指（Instruction Fetch）、译码（Instruction Decode）、执行（Execution）、访存（Memory Access）、写回（Write Back）等阶段。阶段划分需要合理安排，以保证指令之间的依赖和冲突能够正确处理。
4. 数据通路设计：数据通路是流水线中数据传输的路径，包括寄存器、ALU（算术逻辑单元）、数据存储器等。在设计中，需要确定数据通路的结构和组件之间的连接方式，以实现数据的正确传输和处理。
5. 流水线冲突处理：在流水线中，可能会出现各种类型的冲突，包括结构冲突（资源冲突）、数据冲突（数据相关）和控制冲突等。需要采取相应的冲突处理策略，如插入气泡（Bubble）、数据旁路（Data Forwarding）和分支预测（Branch Prediction）等，以保证指令的正确执行。
6. 控制单元设计：控制单元负责对流水线中的各个阶段进行协调和控制，包括控制信号的生成和时序控制。在设计中，需要合理设计控制单元的逻辑和状态机，以确保流水线的正常运行和指令的顺序执行。
7. 性能评估和优化：流水线CPU设计需要考虑性能评估和优化，包括吞吐量、时钟周期、延迟等指标。通过分析和优化流水线的结构、冲突处理策略和控制策略等，可以提高CPU的性能和效率。

## 三 实验过程

### 1 添加中间寄存器

每一个中间寄存器都应该接入 `clk` 信号和 `rst` 信号，因为流水线能起效果正是因为每一个阶段都被划分开来，所以不能使用组合电路实现，而需要有时钟信号控制有效信号的传入。以此来实现不同阶段的划分，这也正是多周期 CPU 的原理。通过时钟信号将一个指令的执行划分成多个环节后，减小周期长度，从而达到流水线同时处理多条指令，提高效率的目的。

中间寄存器的写法一般都如下：

```verilog
`timescale 1ns / 1ps
module XX_YY_Regs (
    input clk,
    input rstn,
    /*---stall 和 flush 信号---*/
    /*---XX阶段的输入信号---*/
    /*---YY阶段的输出信号---*/
);
    /*---定义在always块里赋值用的寄存器们---*/
    always @(posedge clk) begin
        if(~rstn||XX_flush) begin 
            /*---对寄存器清0---*/
        end
        else if(~XX_stall)begin
            /*---将XX阶段的指令传入寄存器，让数值进入下一阶段---*/
        end
    end
    /*---assign将寄存器的值连线到输出信号---*/
endmodule
```

### 2 实现stall

通过一个多端译码的操作，将每一条线需要输出高电平的情况整理出来，节省篇幅和if的判断时间。

```verilog
`timescale 1ns / 1ps

module RaceControler(
        input [31:0] inst,          //inst4
        input [31:0] IF_ID_inst,    //inst3
        input IF_ID_rs1_in_alu,
        input IF_ID_rs2_in_alu,
        input [31:0] ID_EX_inst,    //inst2
        input ID_EX_we_reg,
        input [31:0] EX_ME_inst,    //inst1
        input EX_ME_we_reg,
        input [ 3:0] EX_br_taken,
 
        output wire pc_stall,
        output wire IF_ID_stall,
        output wire ID_EX_stall,
        output wire EX_ME_stall,
        output wire ME_WB_stall,

        output wire IF_ID_flush,
        output wire ID_EX_flush,
        output wire EX_ME_flush,
        output wire ME_WB_flush
    );
    wire [ 4:0] ID_rs1,ID_rs2,EX_rd,ME_rd;
    wire [ 6:0] ID_inst_Type,EX_inst_Type;
    wire IF_ID_rs1_used,IF_ID_rs2_used;
    wire Data_Conflict_with_EX,Data_Conflict_with_ME,Control_Conflict_in_ID,Control_Conflict_in_EX;
    wire Data_Hazard,Control_Hazard_when_1,Control_Hazard_when_2;
    assign ID_rs1 = IF_ID_inst[19:15];
    assign ID_rs2 = IF_ID_inst[24:20];
    assign EX_rd  = ID_EX_inst[11: 7];
    assign ME_rd  = EX_ME_inst[11: 7];
    assign ID_inst_Type = IF_ID_inst[6:0];
    assign EX_inst_Type = ID_EX_inst[6:0];

parameter
JAL   = 7'b1101111,
JALR  = 7'b1100111,
Load  = 7'b0000011,
Store = 7'b0100011,
BType = 7'b1100011;

    assign IF_ID_rs1_used = IF_ID_rs1_in_alu | ID_inst_Type == BType | ID_inst_Type == Load | ID_inst_Type == Store;
    assign IF_ID_rs2_used = IF_ID_rs2_in_alu | ID_inst_Type == BType | ID_inst_Type == Load | ID_inst_Type == Store;
    assign Data_Conflict_with_EX = (EX_rd == ID_rs1 && ID_EX_we_reg && IF_ID_rs1_used && ID_rs1 != 0) |
                                   (EX_rd == ID_rs2 && ID_EX_we_reg && IF_ID_rs2_used && ID_rs2 != 0);
    assign Data_Conflict_with_ME = (ME_rd == ID_rs1 && EX_ME_we_reg && IF_ID_rs1_used && ID_rs1 != 0) |
                                   (ME_rd == ID_rs2 && EX_ME_we_reg && IF_ID_rs2_used && ID_rs2 != 0);
    assign Control_Conflict_in_ID = (ID_inst_Type == BType);
    assign Control_Conflict_in_EX = (EX_inst_Type == BType);

    assign Data_Hazard = Data_Conflict_with_EX | Data_Conflict_with_ME;
    assign Control_Hazard_when_1 = Control_Conflict_in_ID | (Control_Conflict_in_EX && EX_br_taken == 0) | (ID_inst_Type == JAL) | (ID_inst_Type == JALR);
    assign Control_Hazard_when_2 = (Control_Conflict_in_EX && EX_br_taken == 1)  | | (EX_inst_Type == JAL) | (EX_inst_Type == JALR);

    assign pc_stall = Data_Hazard || (Data_Hazard && Control_Hazard_when_1);
    assign IF_ID_stall = Data_Hazard || (~Data_Hazard && Control_Hazard_when_2);
    assign ID_EX_stall = 0;
    assign EX_ME_stall = 0;
    assign ME_WB_stall = 0;
    assign IF_ID_flush = (~Data_Hazard & Control_Hazard_when_1) || (~Data_Hazard & ~Control_Hazard_when_1 & Control_Hazard_when_2);
    assign ID_EX_flush = Data_Hazard;
    assign EX_ME_flush = 0;
    assign ME_WB_flush = 0;

endmodule

```

### 3 debug

看起来整个实验到第二步已经结束了，但实际上这才刚刚开始，我又遇到了数不胜数的各种类型的bug，包括 `RaceControl` 部分实现逻辑上的漏洞，包括实验接线上的一些困难。也包括一些控制冲突的处理，以及BRAM的实现，等等。

### 4 总结

最终实现的CPU也比较粗糙，没有实现更多的实际优化部分。包括没有实现预判全部跳转，如果不跳转再flush多放入的指令；或者是没有实现forwarding来进一步改善数据冲突等等。但是在后续的CPU中应该会尝试改善。另外，我在阶段间传参的时候可能还传递了很多可能会多余的信号。

## 三 思考题

### 1 利用 `fibonacci` 函数，进行多周期CPU与单周期CPU比较

#### 题目描述

1. 对于 syn.asm 的 `fibonacci`（13 - 19 行 ），请计算该 loop 在流水线 CPU 的 CPI，再用 lab0 的单周期 CPU 运行 test1，对比二者的 CPI
2. 对于 test2（21-47 行），请计算你的 CPU 的 CPI，再用 lab0 的单周期 CPU 运行 test2，对比二者的 CPI；

#### 解答

##### 第一题

```asm
00  addi ra,zero,1
04  addi sp,zero,1
08  addi tp,zero,5

;code begin
0c  add  gp,ra,sp
10  add  ra,sp,gp
14  add  sp,ra,gp
18  addi tp,tp,-1
1c  bne  zero,tp,0xc<fibbonacci>
20  addi t0,zero,1597
24  bne sp,t0,<fail>
```

**冲突分析：**0c与10之间存在数据冲突，14与10之间也存在数据冲突，18与1c之间存在数据冲突，1c与20之间存在控制冲突20与24之间又存在数据冲突。因此一共需要的拍数示意图如下：

![image-20240210210807508](./Lab1%20Pipelineimg/image-20240210210807508.png)

因此`PCPU_CPI=21/7=3` 而 `SCPU_CPI=7/7=1` ，但是由于 `Pipeline` 的一个周期比单周期的一个周期更短，所以效率会更高。

##### 第二题

```asm
28  addi sp,zero,272
2c  addi gp,zero,336
30  lb   ra,0(sp)
34  sb   ra,0(gp)
38  lb   ra,1(sp)
3c  sb   ra,1(gp)
40  lh   ra,2(sp)
44  sh   ra,2(gp)
48  lw   ra,4(sp)
4c  sw   ra,4(gp)
50  ld   ra,8(sp)
54  sd   ra,8(gp)
58  lbu  ra,0(sp)
5c  sd   ra,16(gp)
60  lb   ra,0(sp)
64  sd   ra,24(gp)
68  lhu  ra,0(sp)
6c  sd   ra,32(gp)
70  lh   ra,0(sp)
74  sd   ra,40(gp)
78  lwu  ra,0(sp)
7c  sd   ra,48(gp)
80  lw   ra,0(sp)
84  sd   ra,56(gp)
88  addi ra,zero,8
8c  addi t0,zero,8
```

**冲突分析：**28与30之间有一个数据冲突，需要空一拍，从30到88之间每条L类指令和每条S类指令间全部存在数据冲突。因此每隔两条空两拍。因此一共应该有26+4+11*2+1=53拍。

因此`PCPU_CPI=53/26=2.04` 而 `SCPU_CPI=26/26=1` ，但是由于 `Pipeline` 的一个周期比单周期的一个周期更短，所以效率会更高。

### 2 总结数据冲突和控制冲突

#### 题目描述

1. 请你对数据冲突 , 控制冲突情况进行分析归纳，试着将他们分类列出；
2. 如果 EX/MEM/WB 段中不止一个段的写寄存器与 ID 段的读寄存器发生了冲突，该如何处理？
3. 如果数据冲突和控制冲突同时发生应该如何处理呢？

#### 解答

1. 数据冲突的检测一般在ID进行，可以分为与现在EX阶段的指令存在冲突或与现在ME阶段的指令存在冲突。只要存在冲突就需要stall现在的IF_ID阶段间寄存器，和pc寄存器和npc寄存器（有关BRAM的实现），并flush掉ID_EX阶段间寄存器，知道冲突解决之后再继续运行。

   控制冲突，一般而言我们在ID阶段解码当前指令发现是跳转指令后，我们都需要先stall住pc寄存器和npc寄存器（有关BRAM的实现），再flush掉IF_ID阶段间寄存器。等到经过EX阶段返回确认后是否跳转再根据是否跳转决定是否需要stall。

2. 因为我们只需要给出一个条件式，所以实际上就优先哦按段EX阶段如果存在冲突即可安排stall，依次判断MEM，WB阶段即可。

3. 如果数据冲突和控制冲突同时存在，我们需要优先处理数据冲突，在数据冲突结束后如果还有控制冲突再去解决控制冲突。

### 3 寄存器组的BRAM的设计

#### 题目描述

能否使用 BRAM 来实现寄存器组，如果不行是出于什么原因，如果可以需要怎么修改？

#### 解答

我认为可以，因为BRAM需要你提前一个时钟周期传入你的地址，读/写使能等等参数，否则就会造成额外的一个时钟周期的浪费。如果非要用BRAM实现寄存器组的话，我们可以将这个寄存器组调整为类似阶段间寄存器的位置。

如果是读取内容的话我们可以把IF阶段的 `inst` 的[19:15]和[24:20]作为rs1和rs2直接传入BRAM型的寄存器组，这样就可以在EX阶段前得到rs1_data和rs2_data。

如果是写入内容的话我们也同样可以将ME阶段的 `rd_data` 和ME阶段的 `inst` 的 [11:7] 作为写入的目标 `rd` 寄存器这样也可以在WB阶段成功写入。

但是这样会多一些额外的连线，导致额外的开销。

最重要的是，这会导致关键路径，也就是从ME_WB阶段间寄存器传到Regs的那条线被提前到直接从ME阶段的 `rd_data` 传入Regs。关键路径提前到ME阶段，这也会导致一个Circle的周期变长，等于是起到了时间换空间的效果。

### 4 分析关键路径

#### 题目描述

尝试分析整体设计的关键路径，也就是最长路径，可以参考自己的 implement 报告。

#### 解答：

![image-20240210210823021](./Lab1%20Pipelineimg/image-20240210210823021.png)

最长路径应该是从ME_WB的 `inst` 的[11:7]表示的目标寄存器连线到寄存器组中对应的位置。