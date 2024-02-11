# Lab2 Forwarding 及 AXI4-lite 总线内存模型





# 1 实验目的

- 完善流水线的基本功能，实现 Forwarding 机制
- 增强流水线性能，改善流水线的stall机制，decode机制。（修改了我自己的lab1的垃圾control模块）
- 将 Core 和 RAM 模块用总线加以连接，使得流水线的内存模型更接近真实的内存模型

# 2 实验原理

## 2.1 二段译码

![image-20240210210901344](./Lab2%20Forwarding%20%E5%8F%8A%20AXI4-lite%20%E6%80%BB%E7%BA%BF%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8Bimg/image-20240210210901344.png)

在很久很久以前 我和TA提到过一嘴这个事情，据说可以让码风看起来简洁清晰一些，编码效率高一点，包括后面在写到RaceControl的时候也需要用到这种编码技巧，于是我又开始找当时听明白了的同学学习。简单地说，就是取缔数不胜数的条件分支判断，不再通过对每一个条件下的每一个输出赋值来译码。而是将条件作为赋值的表达式，对每一个输出总结这个输出置一时的条件，将这些条件或在一起作为这个输出信号的表达式。

下面简单对比一下两种写法下的decode部分代码：



![image-20240210210906813](./Lab2%20Forwarding%20%E5%8F%8A%20AXI4-lite%20%E6%80%BB%E7%BA%BF%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8Bimg/image-20240210210906813.png)![image-20240210210937926](./Lab2%20Forwarding%20%E5%8F%8A%20AXI4-lite%20%E6%80%BB%E7%BA%BF%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8Bimg/image-20240210210937926.png)

第一种写法的代码篇幅明显长于第二种写法。而且对于每一种情况下还会需要为很多没有意义的信号赋0值。但是在第二种写法中只需要就行几次赋值就能解决。但是对于情况的分析要求相对比较高，对细节的要求比较苛刻。同样我也改写了RaceControl的部分，这样在以后增加flush和stall的时候只需要讲一条信号译码为哪几个信号需要置一然后或在后面就行。对于后续代码的维护和修改也方便很多。

## 2.2 假设全部不跳转的控制冲突解决

除了后续会提到的Forwarding用于解决一些数据冲突以外，我在这次实验中还做了假设不跳转来优化控制冲突的stall。原先我们解决控制冲突的方案是在ID阶段检测到控制冲突时将PC寄存器stall住，知道EX阶段计算得出是否跳转后再释放PC寄存器。如果跳转还需要先改写再释放。所以这里会有两拍的浪费。

但如果我们先假设全部不跳转，先让后续指令进入流水线，再在检测到需要跳转的时候再把额外载入了CPU流水线的指令flush掉，这样可以有效地减少stall的浪费拍数。

假设不跳转还在循环或者递归的代码中有良好的表现，因为实际上跳转指令的跳转只会在某一条出现，很大部分是不跳转的情况，这么一来相比单纯采用stall处理数据冲突，效率就会高很多。

## 2.3 Forwarding

在解决优化好控制冲突之后，我们要考虑的是如何进一步优化数据冲突。

正如实验指导中所说，在流水线的运行过程中，数据冲突是拖累流水线效率的一个重要的因素。 在顺序单发射处理器中，数据冒险只需要考虑RAW一种情况，本质上就是由读写同一个寄存器所导致的，所以数据冒险本身的检测只需对流水线中相应流水段中指令的寄存器编号进行比较，即可对数据冒险的发生进行判断。

我们也知道数据竞争本质上是因为当前需要的数据还没有到写回阶段，不论是在寄存器的读写发生冲突还是在内存的读写上发生冲突，归根结底就是你需要的数据实际上还在运行过程中。因此我们会需要在先前使用stall等待数据进入寄存器或进入内存，我们再开始读数据。

实现 Forwarding 的基本功能需要在流水线的基础上添加冲突检测逻辑和相应的数据通路，这样就能够在 WB 阶段之前，将最新的寄存器的数据回传给后续的指令，减少 stall 的周期数，进而提高流水线的效率。

## 2.4 AXI4-lite总线

在此前的所有lab里面，我们与内存空间的交互都是通过直接连接RAM实现的，内存模块用寄存器数组来实现，并集成到了处理器内部，并且我们使用了一些 trick 来对其进行初始化，让其上电后就加载好待测试的内容。 但在真实的CPU中并不可能通过这种方式实现，真实的内存为了能够储存更多的数据，通常是有一块独立于处理器的ASIC芯片，例如内存条上的储存颗粒或是嵌入式开发板上的SRAM颗粒。访存请求传到内存颗粒需要时间，数据从内存颗粒传回处理器也需要时间，并且内存容量增大以后，寻址的时间也会增加，因此在真实的系统中，单次内存读写并不可能在单一的周期内完成，有的甚至需要几百甚至几万的周期来寻址和访存。

而且因为不同的RAM内存有着不同的响应效率和响应时间，我们无法设计固定的访存时间来改进我们的流水线CPU。由此可见我们之前实验的 RAM.v 是多么的理想化和不真实了。 因此我们需要将我们现在手上的Core部分拓展为一个AXI4-lite接口，其内部的实现原理大致如下：

> AXI4-lite 总线协议分为 master（主设备）和 slave（从设备）两部分接口，它规定了 master 设备和 slaver 设备如何握手和进行数据交换。简单来说，master 向 slave 发送地址数据和读写请求，然后进入等待，直到 slave 处理完请求后，向 master 发送完成信号和数据等信息，然后 master 接收数据、关闭请求，slave 也关闭应答和数据发送、等待下次事务的到来。

其整体框架也列出如下：

```
PipelineCPU
├── Axi_lite_Core
│    ├── Core    
│    ├── Core2Mem_FSM    
│    └── CoreAxi_lite    
└── Axi_lite_Mem
	 ├── MemAxi_lite
	 └── RAM
```

而模块间的交互也可表示在下面的这张图中：

![image-20240210210954607](./Lab2%20Forwarding%20%E5%8F%8A%20AXI4-lite%20%E6%80%BB%E7%BA%BF%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8Bimg/image-20240210210954607.png)

也就是说我们需要将我们Core中与RAM交互所需要用到的信号传输给Core2Mem_FSM的有限状态机。有限状态机再根据拿到的读取指令或读取数据的信号来判断将哪些数据传给CoreAxi_lite，再由CoreAxi_lite与Axi_liteMem之间进行交互。需要说明的是在本实验中我们并不需要关心Axi_liteMem的内部实现以及Axi_liteMem与CoreAxi_lite之间的交互方式，我们只需要知道的是，当内存读取结束返回数据时，还会额外返回一个valid信号作为表示读取成功的信号，有限状态机也正是根据Valid信号来执行状态转移，传递数据的。

因此我们在本次实验中需要完成的就是Core2Mem_FSM有限状态机内部的实现，以及Core和Core2Mem_FSM之间数据信号的传递和交互。

对于Core模块，在原有基础上需要额外获得来自Core2Mem_FSM模块的if_stall，mem_stall两个信号。前者表示在IF阶段的访问内存阶段遇到了问题需要stall，后者表示 MEM 阶段数据读写遇到了问题需要 stall，然后我们修改 Core 的 RaceController 模块，应对这两个信号，当这两种 stall 发生时，发送给各个阶段间寄存器对应的 stall、flush 信号。

对于Core2Mem_FSM模块，我们需要完成以下三个方面的目标：

- 由于在使用总线的过程中接口只有一个，一个时钟周期内只能传输一个数据信号。因此，无法同时处理IF阶段对指令的请求和MEM阶段对内存数据的请求。所以当两个事件同时发生时，我们需要裁决谁先谁后。
- 该模块需要将来自 Core 的请求发送给 CoreAxi_lite 模块，然后等待 CoreAxi_lite 模块完成总线交互返回结果，然后将结果返还给 Core；
- 当无法返回对应的任务或是正在访存寻址的过程中，该模块需要向 Core 发送对应的 stall 信号，告诉流水线的有关任务需要等待。

这个工作过程我们通过一个有限状态机实现，下面是这个有限状态机的状态转移表：

| 起始状态 | 目标状态 |               转移条件                |                           阶段任务                           |
| :------: | :------: | :-----------------------------------: | :----------------------------------------------------------: |
|   IDLE   |   IDLE   |               不会发生                |                           不会发生                           |
|   IDLE   |   DATA   |            mem阶段发送请求            | 将MEM阶段的访存信息请求发送给CoreAxi_lite<br />开启mem_stall信号 |
|   IDLE   |   INST   | mem阶段没有请求<br />但if阶段发送请求 | 将IF阶段的访存信息请求发送给CoreAxi_lite<br />开启if_stall信号 |
|   DATA   |   IDLE   |            接收到Valid信号            |        关闭mem_stall信号<br />关闭给CoreAxi_lite请求         |
|   DATA   |   DATA   |           未接收到Valid信号           |                             保持                             |
|   DATA   |   INST   |               不会发生                |                           不会发生                           |
|   INST   |   IDLE   |            接收到Valid信号            |         关闭if_stall信号<br />关闭给CoreAxi_lite请求         |
|   INST   |   DATA   |               不会发生                |                           不会发生                           |
|   INST   |   INST   |           未接收到Valid信号           |                             保持                             |

# 3 实验操作

## 3.1 二段译码

根据我一开始写好的将每一个条件表达式对应赋值的那个写法改成对每一个信号赋值条件的改法。在改的时候可以采取改好一条信号就再仿真一次的方法，这样可以减少每一次修改时候的可能出错的范围，减小调试的难度。

在改的过程中遇到了一些难度：

1. 如果不是case写法，而是if-else if的写法的话还是很有可能出现条件写的不严谨导致仿真报错的。尤其是有严格的if-else if关系的话需要将第一个条件的取反与在这个条件上，例如：
   ```verilog
   if( condition1 )begin
   	A=1;
   	B=0;
   end
   else if( condition2 )begin
   	A=1;
   	B=1;
   end
   //这种情况下就不能写成这种：
   A=condition1 | condition2;
   B=condition2;
   //而应该写成下面这种写法：
   A = condition1 | condition2;
   B = condition2 & ~condition1;
   ```

   因为condition1和condition2有不同优先级的区分，如果condition1是正确的，那么B就必须得是0。

2. 还有就是在二段译码的过程中，每新增一个需要译码的信号，就需要考虑这个信号和之前被译码过的每一个信号有没有牵连的关系。如果有的话需要参考第一条的情况。

## 3.2 假设不跳转的冲突解决

理论上理解起来很方便，检测控制冲突的过程放在EX阶段，如果是一个需要跳转的跳转指令，我们才会做出判断，需要flush掉所有读入的错误的信号，即flush掉IFID和IDEXE阶段间的寄存器即可。

```verilog
`timescale 1ns / 1ps

module RaceControler(
        input [31:0] ID_EX_inst,
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

parameter
JAL   = 7'b1101111,
JALR  = 7'b1100111,
BType = 7'b1100011;

    wire [ 6:0] EX_inst_Type;
    wire Control_Hazard_need_flush,flag;
    assign EX_inst_Type = ID_EX_inst[6:0];
    
    assign Control_Hazard_need_flush = ((EX_inst_Type == BType) && EX_br_taken == 1) | (EX_inst_Type == JAL) | (EX_inst_Type == JALR);

    assign pc_stall = 0;
    assign IF_ID_stall = 0;
    assign ID_EX_stall = 0;
    assign EX_ME_stall = 0;
    assign ME_WB_stall = 0;
    assign IF_ID_flush = Control_Hazard_need_flush;
    assign ID_EX_flush = Control_Hazard_need_flush;
    assign EX_ME_flush = 0;
    assign ME_WB_flush = 0;
    
endmodule
```

## 3.3 Forwording

正如先前所说，我们只需要做两件事，首先是添加检测冲突的逻辑，第二个是构建传递信号的数据通路。

### 3.3.1 检测冲突的逻辑

如果我们在即将取出某一个寄存器内的值时，也就是在EX阶段，进行检测。如果检测到EX阶段的rs1或者rs2和后面两个阶段的某一条指令的rd是一致的话，说明存在数据冲突。大致表达表如下：

|                    表达式                    |    取值     | 优先级 |
| :------------------------------------------: | :---------: | ------ |
| EX->rs1 == ME->rd && ME->wen && EX->rs1 != 0 | rs1取ME->rd | 1      |
| EX->rs1 == WB->rd && WB->wen && EX->rs1 != 0 | rs1取WB->rd | 2      |
|                     else                     |  rs1取rs1   | 3      |
| EX->rs2 == ME->rd && ME->wen && EX->rs2 != 0 | rs2取ME->rd | 1      |
| EX->rs2 == WB->rd && WB->wen && EX->rs2 != 0 | rs2取WB->rd | 2      |
|                     else                     |  rs2取rs1   | 3      |

这里我们可以使用一个信号来作为后续选择器的输入，那么ForwadingUnit的实现代码就应该如下：

```Verilog
module ForwardingUnit(
    input [63:0] ID_EX_rs1,
    input [63:0] ID_EX_rs2,
    input [63:0] EX_ME_rd,
    input [63:0] ME_WB_rd,
    input EX_ME_we_reg,
    input ME_WB_we_reg,
    output [1:0] Forwarding_1,
    output [1:0] Forwarding_2
);
reg [1:0] Forwarding_1_reg,Forwarding_2_reg;
always @(*) begin
    if(ID_EX_rs1 == EX_ME_rd && EX_ME_we_reg && ID_EX_rs1 != 0) Forwarding_1_reg = 2'b01;
    else if(ID_EX_rs1 == ME_WB_rd && ME_WB_we_reg && ID_EX_rs1 != 0) Forwarding_1_reg = 2'b10;
    else Forwarding_1_reg = 2'b00;
    if(ID_EX_rs2 == EX_ME_rd && EX_ME_we_reg && ID_EX_rs2 != 0) Forwarding_2_reg = 2'b01;
    else if(ID_EX_rs2 == ME_WB_rd && ME_WB_we_reg && ID_EX_rs2 != 0) Forwarding_2_reg = 2'b10;
    else Forwarding_2_reg = 2'b00;
end
assign Forwarding_1 = Forwarding_1_reg;
assign Forwarding_2 = Forwarding_2_reg;
endmodule
```

### 3.3.2 搭建数据通路

其实实现起来非常简单，一部分是将判断是否存在冲突所需要的信号传入Unit，也就是几个rs和几个rd以及写使能，然后输出的信号是控制选择器的信号，代码如上。

另一部分是需要在原先的rs数据和后面两个阶段可能写回的rd数据之间做选择，可以写一个MUX3_1的选择器，需要注意的是和Unit单元的约定，什么值表示选什么。实现代码如下：

```verilog
module MUX_3_1 (
    input [63:0] ID_EX_rs_data,
    input [63:0] EX_ME_rd_data,
    input [63:0] ME_WB_rd_data,
    input [ 1:0] sel_signal,
    output [63:0] Real_rs_data
);
reg [63:0] res;
always @(*) begin
    case (sel_signal)
    2'b00: res = ID_EX_rs_data;
    2'b01: res = EX_ME_rd_data;
    2'b10: res = ME_WB_rd_data;
    2'b11: res = 0;
    endcase
end
assign Real_rs_data = res;
endmodule
```

## 3.4 AXI4-lite总线

比起前面的几个实验部分，这个部分可以说是相当大。

但是实际上也可以分为两个部分，一个是对Core进行修改，使其能够符合现在需要的接口，并且确保能够传出正确的数据。一个是对RaceControl进行修改。另一个是用代码实现有限状态机，使之能够与后续已经完成的部分进行交互。

### 3.4.1 修改Core

这个部分其实也还好，首先是你需要知道每个信号在表示什么意思。但是因为今年助教并没有写对应的意思，所以有点像填空题，需要你根据上下文猜测（当然也可以根据他的名字来猜），下面是对照表：

| 名字                    | 对应的意思                                                   |
| ----------------------- | ------------------------------------------------------------ |
| output [63:0] pc        | pc寄存器的值，用于访问内存获得指令                           |
| input  [31:0] inst      | 32位的指令，表示在Core中不需要对一个64位的inst做截取操作     |
| output [63:0] address   | 地址，在RAM模块中，我们可以看到还是进行了一个截取了[11：3]，所以我们这里实际上只需要传入alu_result即可<br />![image-20240210211003119](./Lab2%20Forwarding%20%E5%8F%8A%20AXI4-lite%20%E6%80%BB%E7%BA%BF%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8Bimg/image-20240210211003119.png) |
| output we_mem           | 写使能信号                                                   |
| output [63:0] wdata_mem | 写入内存的值                                                 |
| output [ 7:0] wmask_mem | 写入内存时的掩码                                             |
| output re_mem           | 读使能                                                       |
| input  [63:0] rdata_mem | RAM返回的读到的内存的数据                                    |
| output if_request       | 是否有IF阶段的请求                                           |
| input if_stall          | IF阶段是否出现了问题需要处理                                 |
| input mem_stall         | MEM阶段是否出现了问题需要处理                                |

那么对应连线如下：

```verilog
	assign pc = (pc_stall)?IF_pc:(((ID_EX_npc_sel)&&(br_taken))?alu_result:IF_pc+4);
    assign address = alu_result;
    assign wdata_mem = data_pkg;
    assign we_mem = ID_EX_we_mem;
    assign wmask_mem = rw_wmask;
    assign re_mem = ID_EX_inst[6:0] == 7'b0000011;
    assign if_request = ~pc_stall;
```

需要声明的是我这里使用的还是一个BRAM类型的Core，所以后续在处理FSM的时候也会有相应的操作。

### 3.4.2 修改RaceControl

正如之前所说的，这里的RaceControl除了解决预判不跳转可能会出现的flush需求，还要处理状态机给出的if_stall和mem_stall的需求，实现代码如下：

```verilog
`timescale 1ns / 1ps

module RaceControler(
        input [31:0] inst,          //inst4
        input [31:0] IF_ID_inst,    //inst3
        input [31:0] ID_EX_inst,    //inst2
        input [31:0] EX_ME_inst,    //inst1
        input [ 3:0] EX_br_taken,
        input if_stall,
        input mem_stall,

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

parameter
JAL   = 7'b1101111,
JALR  = 7'b1100111,
BType = 7'b1100011;

    wire [ 6:0] EX_inst_Type;
    wire Control_Hazard_need_flush,flag;
    assign EX_inst_Type = ID_EX_inst[6:0];
    
    assign Control_Hazard_need_flush = ((EX_inst_Type == BType) && EX_br_taken == 1) | (EX_inst_Type == JAL) | (EX_inst_Type == JALR);

    assign pc_stall = if_stall | mem_stall;
    assign IF_ID_stall = mem_stall;
    assign ID_EX_stall = mem_stall | (Control_Hazard_need_flush & if_stall);
    assign EX_ME_stall = mem_stall;
    assign ME_WB_stall = 0;
    assign IF_ID_flush = (Control_Hazard_need_flush || if_stall) & ~IF_ID_stall;
    assign ID_EX_flush = Control_Hazard_need_flush & ~ID_EX_stall;
    assign EX_ME_flush = Control_Hazard_need_flush & ~EX_ME_stall & if_stall;
    assign ME_WB_flush = mem_stall;
    
endmodule
```

需要注意的是：

- 理论上MEM阶段的读使能信号和EXE阶段的写使能信号理论上不会同时为1，我们仅仅会在MEM的读使能生效时出发stall。
- stall和flush不会同时发生，也就是在等待RAM返回数据的过程中，所有阶段都需要等待，即便有控制冲突需要fluhs，也得优先处理stall信号。

### 3.4.3 FSM状态机实现

这个部分其实才是这个实验的关键，我们可以根据之前完成的状态转移表来实现，实现代码如下：

```verilog
module Core2Mem_FSM (
    input wire clk,
    input wire rstn,
    input wire [63:0] pc,
    input wire if_request,

    input wire [63:0] address_cpu,
    input wire wen_cpu,
    input wire ren_cpu,
    input wire [63:0] wdata_cpu,
    input wire [7:0] wmask_cpu,

    output [31:0] inst,
    output [63:0] rdata_cpu,
    output if_stall,
    output mem_stall,
    

    output reg [63:0] address_mem,
    output reg ren_mem,
    output reg wen_mem,
    output reg [7:0] wmask_mem,
    output reg [63:0] wdata_mem,
    input wire [63:0] rdata_mem,
    input wire valid_mem
);
parameter
IDLE = 2'b00,
DATA = 2'b01,
INST = 2'b10;
wire flag = ren_cpu | wen_cpu;
reg open_IFstall,open_MEMstall,open_Inst,open_Rdata;

reg Maintain_ren,Maintain_wen,Maintain_state;
reg [63:0] Maintain_address,Maintain_wdata,Maintain_inst;
reg [ 7:0] Maintain_wmask;

reg [1:0] state;

always @(posedge clk or negedge rstn) begin
    if(~rstn) begin
        state <= IDLE;

        address_mem <= 0;
        ren_mem <= 0;
        wen_mem <= 0;
        wmask_mem <= 0;
        wdata_mem <= 0;

        open_IFstall <= 0;
        open_MEMstall <= 0;
        open_Inst <= 0;
        open_Rdata <= 0;

        Maintain_address <= 0;
        Maintain_inst <= 0;
        Maintain_ren <= 0;
        Maintain_state <= 0;
        Maintain_wdata <= 0;
        Maintain_wen <= 0;
        Maintain_wmask <= 0;
    end
    else begin
        case(state)
            IDLE:begin
                if(Maintain_state)begin
                    ren_mem <= Maintain_ren;
                    wen_mem <= Maintain_wen;
                    address_mem <= Maintain_address;
                    wmask_mem <= Maintain_wmask;
                    wdata_mem <= Maintain_wdata;

                    open_MEMstall <= 1;
                    open_IFstall <= 1;
                    state <= DATA;
                end
                else if(flag)begin
                    ren_mem <= ren_cpu;
                    wen_mem <= wen_cpu;
                    address_mem <= address_cpu;
                    wmask_mem <= wmask_cpu;
                    wdata_mem <= wdata_cpu;

                    open_IFstall <= 0;
                    open_MEMstall <= 1;
                    state <= DATA;
                end
                else if(if_request)begin
                    address_mem <= pc;
                    ren_mem <= 1;
                    wen_mem <= 0;
                    wmask_mem <= 8'b1111_1111;
                    wdata_mem <= wdata_cpu;

                    open_IFstall <= 1;
                    open_MEMstall <= 0;
                    state <= INST;
                end
            end
            DATA:begin
                if(valid_mem) begin
                    ren_mem <= 0;
                    wen_mem <= 0;
                    if(Maintain_state)address_mem <= pc;
                    open_MEMstall <= 0;
                    open_IFstall <= 0;
                    open_Rdata <= 1;
                    if(Maintain_state)open_Inst <= 1;
                    state <= IDLE;
                    Maintain_state <= 0;
                end
            end 
            INST:begin
                if(valid_mem) begin
                    ren_mem <= 0;
                    wen_mem <= 0;
                    Maintain_inst <= rdata_mem;
                    open_IFstall <= 0;
                    open_Inst <= 1;
                    if(~Maintain_state) open_MEMstall <= 0;
                    state <= IDLE;
                end
                else if(~if_request && flag)begin
                    Maintain_state <= 1;
                    Maintain_address <= address_cpu;
                    Maintain_ren <= ren_cpu;
                    Maintain_wdata <= wdata_cpu;
                    Maintain_wen <= wen_cpu;
                    Maintain_wmask <= wmask_cpu;

                    open_MEMstall <= 1;  
                end
            end
            default: state <= IDLE;
        endcase
    end
end

assign inst = (open_Inst) ? ((((pc-4)>>2)%2 == 0)?Maintain_inst[31:0]:Maintain_inst[63:32]) : 0;
assign rdata_cpu = (open_Rdata) ? rdata_mem : 0;
assign if_stall = open_IFstall;
assign mem_stall = open_MEMstall;

endmodule
```

首先，我在代码中处理了模拟BRAM的取法：

`assign inst = (inst_open ) ? (((pc-4 > 2)%2 = 0 ) ? inst_keep[31:0] : inst_keep[63: 32]) : 0;`

在我们原来的Core模块中，当Pipeline在执⾏当前inst指令的同时，要向RAM发送下⼀条inst的请求，所以这里处理的时候应该取pc-4。

其次，是数据的保持部分：

我引入了一组寄存器用于暂存不论是在取指令的时候遇到取内存的需求还是在取内存时遇到取指令的需求，将这个取指令的一些参数放入其中储存。并且在获取到RAM的返回数据后将这保持的信号传入总线进行处理。

```Verilog
reg Maintain_ren,Maintain_wen,Maintain_state;
reg [63:0] Maintain_address,Maintain_wdata,Maintain_inst;
reg [ 7:0] Maintain_wmask;
... ...
always @(posedge clk or negedge rstn) begin
    if(~rstn) begin
    ...
    end
	else begin
        case(state)
			IDLE:begin
                if(Maintain_state)begin
                    ren_mem <= Maintain_ren;
                    wen_mem <= Maintain_wen;
                    address_mem <= Maintain_address;
                    wmask_mem <= Maintain_wmask;
                    wdata_mem <= Maintain_wdata;

                    open_MEMstall <= 1;
                    open_IFstall <= 1;
                    state <= DATA;
                end
         	...
         	end
         	DATA:begin
                if(valid_mem) begin
                    ren_mem <= 0;
                    wen_mem <= 0;
                    if(Maintain_state)address_mem <= pc;
                    open_MEMstall <= 0;
                    open_IFstall <= 0;
                    open_Rdata <= 1;
                    if(Maintain_state)open_Inst <= 1;
                    state <= IDLE;
                    Maintain_state <= 0;
                end
            end
            INST:begin
                if(valid_mem) begin
                    ren_mem <= 0;
                    wen_mem <= 0;
                    Maintain_inst <= rdata_mem;
                    open_IFstall <= 0;
                    open_Inst <= 1;
                    if(~Maintain_state) open_MEMstall <= 0;
                    state <= IDLE;
                end
                else if(~if_request && flag)begin
                    Maintain_state <= 1;
                    Maintain_address <= address_cpu;
                    Maintain_ren <= ren_cpu;
                    Maintain_wdata <= wdata_cpu;
                    Maintain_wen <= wen_cpu;
                    Maintain_wmask <= wmask_cpu;

                    open_MEMstall <= 1;  
                end
            end
            defualt:... ...
        endcase
    end
end            
```

# 4 仿真验收

![image-20240210211033524](./Lab2%20Forwarding%20%E5%8F%8A%20AXI4-lite%20%E6%80%BB%E7%BA%BF%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8Bimg/image-20240210211033524.png)

![image-20240210211035867](./Lab2%20Forwarding%20%E5%8F%8A%20AXI4-lite%20%E6%80%BB%E7%BA%BF%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8Bimg/image-20240210211035867.png)

上板结果不再在这里贴图了。

# 5 思考题

## 5.1 Stall机制与Forwarding机制

### 题目描述

1. 在引入 Forwarding 机制后，是否意味着 stall 机制就不再需要了？为什么？

2. 你认为 Forwarding 机制在实际的电路设计中是否存在一定的弊端？和单纯的 stall 相比它有什么缺点？
3. 不考虑 AXI4-lite 总线的影响，引入 Forwarding 机制之后 cpi 和 stall 相比提升了哪些？

### 解答

1. 引入 Forwarding 机制后，并不意味着 stall 机制就不再需要了。因为我们在后面还会遇到一些需要暂停流水线的时候，而不是只有在遇到了数据冲突时才会考虑暂停流水线。比如在加入总线后，我们需要暂停等待内存空间返回数据。所以实际上还是需要stall机制的。而且就算是再Forwarding机制里，也存在需要stall的时候，比如当第一条指令和第三条指令之间存在数据冲突，第二条指令是sd指令也和第三条指令存在数据冲突，那么这个时候就需要stall住等待第二条指令再MEM阶段后写入内存，sd才能再EX阶段运行。
2. forwarding机制的缺点在于增加的数据通路是比较理想的。但在实际上，这会增加电路的复杂性，Forwarding机制需要在流水线中添加额外的硬件逻辑和控制电路，以实现数据的传递。除此之外还会增加电路的延迟，虽然Forwarding机制可以减少流水线中stall的拍数，但在某些情况下，数据旁路传递的延迟可能会增加。特别是在多级流水线中，当数据相关跨越多个流水线阶段时，需要多个Forwarding路径，每个路径都会增加一定的传递延迟，反而有可能拖慢cpu的效率。
3. cpi的提升体现在不再需要通过大面积的stall来解决两条指令之间的数据竞争了，所以大约平均每次发生数据冲突会少两拍的stall。

## 5.2 总线的影响和改进

### 题目描述

计算加入 AXI4-lite 总线之后的 cpi，思考 cpi 的值受到什么因素的制约，考虑可能的提升方法？

### 解答

![image-20240210211046594](./Lab2%20Forwarding%20%E5%8F%8A%20AXI4-lite%20%E6%80%BB%E7%BA%BF%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8Bimg/image-20240210211046594.png)

9a8一共是618条指令，一共使用了13023个周期，所以cpi应该是21.07

这时的cpi所受的主要制约是在RAM中取指令和在总线上传输数据造成的等待stall，如果我们需要提升的话可以考虑使用性能更优秀的RAM内存和总线。

## 5.3 关键路径分析

### 题目描述

尝试分析添加 Forwarding 后整体设计的关键路径。

### 解答

在增加了Forwarding之后可能会出现以下关键路径，具体会和载体的性能有关。

1. 旁路传递路径：Forwarding机制通过旁路传递数据，从而减少数据相关导致的停顿周期。在设计中，需要确保旁路路径能够及时传递数据。如果旁路路径的延迟较长，可能会增加关键路径的延迟。
2. 额外储存WB：因为我们在分析中发现，有可能会存在一种现象是第一条指令和第二条sd指令分别都和第三条指令有冲突，因为写回需要等一个周期，那么WB阶段的数据就会面临丢失的风险，因此我们就得额外增加一个寄存器用于暂存需要forwarding的那个WB阶段的数据。这个寄存器的前递可能会存在着一条关键路径。
