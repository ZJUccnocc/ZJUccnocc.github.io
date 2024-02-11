# Lab1实验报告

## 第一部分

### Verilog简单学习

对于最终烧录到开发板的top module，一种简化的Verilog代码描述如下：

```verilog
module top(
input Input0,
input Input1,
input Input2,
input Input3,
input Input4,
output S1,
output S2
);
wire S;
MUX4T1_1 mux(
.S({Input1, Input0}), .I0(Input2), .I1(Input3), .I2(Input4), .I3(S), .O(S1)
);
Adder_1 add(
.A(Input2), .B(Input3), .CI(Input4), .S(S), .CO(S2)
);
endmodule
```



其中I0-I4对应板子从右到左的前五个开关，S1-S2对应从右到左的前两个LED。请给出该Verilog所描述的电路，并在板子上验证其正确性。

![](./Lab1%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240121151821236.png)![](./Lab1%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240121151825813.png)![image-20240121151837004](./Lab1%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240121151837004.png)



### 多路复用器拓展

在实验中我们已经拥有了MUX4(控制的信号数)T1_1(输入的位数)，实际上它是由如3.1图中展示的结构组成

![image-20240121151855607](./Lab1%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240121151855607.png)

1.  MUX2T1_1是由哪种decoder和AND-OR结构组成的？

    1.  1-to-2¹-line decoder

    2.  2 \* 2AND-OR

2.  MUX4T1_2是怎么组成MUX8T1_1的？

    1.  取两个MUX4T1_2（M1，M2）

    2.  两个MUX的信息输入端前分别接一个2-to-1 encoder

    3.  将MUX8T1_1 的3个1位输入接到encoder的3个输入端，另一个接0

3.  MUX2^(m)T1-n是怎么构成的？（m\<=n）

    1.  1-to-n Splitter![image-20240121152612626](./Lab1%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240121152612626.png)

    2.  N-to-2^(n)-line decoder

    3.  2^(m) \* 2 AND-OR

### 全加器拓展

1.  如何利用多个全加器构成支持64位加法的全加器

    将两个加数从低位开始每一位各自输入一个全加器，其中第一个Cin为0，而后的Cin连至上一级的Cout，最后得到的S从低到高位排序后输入一个encoder得到两个加数的和。

2.  如何通过增加一个控制加减法的信号CTRL和一个异或门实现减法![image-20240121151916836](./Lab1%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240121151916836.png)

    在CTRL=0时是加法，在CTRL=1时是减法

    S’=S， 观察知Cout和Cout’上下对称关系与A一致

    将A与CTRL异或后输入A得到的Cout，输出的S还需与CTRL异或得到S

    一个异或门我认为不够

## 第二部分

### 行为描述方式学习

1.  用行为描述方式设计七段SegDecoder模块，并上板验证

```verilog
`timescale 1ns / 1ps
module SegDecoder (
input wire [3:0] data,
input wire point,
input wire LE,
output wire a,
output wire b,
output wire c,
output wire d,
output wire e,
output wire f,
output wire g,
output wire p
);
reg ai,bi,ci,di,ei,fi,gi,pi;

always@(*) begin
    case({LE,data[3:0]})
        5'b00000:begin ai = 0;bi = 0;ci = 0;di = 0;ei = 0;fi = 0;gi = 1;end
        5'b00001:begin ai = 1;bi = 0;ci = 0;di = 1;ei = 1;fi = 1;gi = 1;end
        5'b00010:begin ai = 0;bi = 0;ci = 1;di = 0;ei = 0;fi = 1;gi = 0;end
        5'b00011:begin ai = 0;bi = 0;ci = 0;di = 0;ei = 1;fi = 1;gi = 0;end
        5'b00100:begin ai = 1;bi = 0;ci = 0;di = 1;ei = 1;fi = 0;gi = 0;end
        5'b00101:begin ai = 0;bi = 1;ci = 0;di = 0;ei = 1;fi = 0;gi = 0;end
        5'b00110:begin ai = 0;bi = 1;ci = 0;di = 0;ei = 0;fi = 0;gi = 0;end
        5'b00111:begin ai = 0;bi = 0;ci = 0;di = 1;ei = 1;fi = 1;gi = 1;end
        5'b01000:begin ai = 0;bi = 0;ci = 0;di = 0;ei = 0;fi = 0;gi = 0;end
        5'b01001:begin ai = 0;bi = 0;ci = 0;di = 0;ei = 1;fi = 0;gi = 0;end
        5'b01010:begin ai = 0;bi = 0;ci = 0;di = 1;ei = 0;fi = 0;gi = 0;end
        5'b01011:begin ai = 1;bi = 1;ci = 0;di = 0;ei = 0;fi = 0;gi = 0;end
        5'b01100:begin ai = 0;bi = 1;ci = 1;di = 0;ei = 0;fi = 0;gi = 1;end
        5'b01101:begin ai = 1;bi = 0;ci = 0;di = 0;ei = 0;fi = 1;gi = 0;end
        5'b01110:begin ai = 0;bi = 1;ci = 1;di = 0;ei = 0;fi = 0;gi = 0;end
        5'b01111:begin ai = 0;bi = 1;ci = 1;di = 1;ei = 0;fi = 0;gi = 0;end
        default :begin ai = 1;bi = 1;ci = 1;di = 1;ei = 1;fi = 1;gi = 1;end
    endcase
    case(point)
        0:pi=1;
        1:pi=0;
    endcase
end
assign a=ai, b=bi,c=ci,d=di,e=ei,f=fi,g=gi,p=pi;

```

### Verilog语法学习

1.  wire型变量和reg型变量的区别：

    1.  从仿真角度来说，HDL语言面对的是编译器，相当于使用软件思路，此时：   
        wire对应于连续赋值，如 `assign` ； 
        reg对应于过程赋值，如 `always` ， `initial` ；

    2.  从综合角度，HDL语言面对的是综合器，相当于从电路角度来思考，此时：

        wire型变量综合出来一般情况下是一根导线。

        reg变量在 `always` 中有两种情况：

        `always @（a or b or c）` 形式的，即不带时钟边沿的，综合出来还是组合逻辑；

        `always @（posedge clk）` 形式的，即带有边沿的，综合出来一般是时序逻辑，会包含触发器（Flip-Flop）

2.  现有变量wire s，如何表示复制32次的变量wire\[31:0\] si

    `assign[31:0] si={s,s,s,s ,s,s,s,s ,s,s,s,s ,s,s,s,s ,s,s,s,s ,s,s,s,s ,s,s,s,s ,s,s,s,s}`

3.  请阐述assign =,=,\<=的区别和适用条件

    赋值符号左边的形式取决于是连续赋值还是过程赋值

    右边可以是获得值的任何表达式

    **连续赋值**：当右边的获得值发生变化就会进行赋值(对wire赋值)

    `assign [变量名] = [获得值]`

    **过程赋值**：没有期限，变量保存获得值直到下一次对该变量过程赋值

    `[变量名] = [获得值]`

    > Tips：变量声明赋值只允许在模块级别(initial,always)进行
    >
    > =是阻塞赋值，会阻塞下一个语句执行，属于顺序执行语句
    >
    > \<=是非阻塞赋值，属于并行执行语句
    >
    > 由于语句块是同时执行的，很多情况下用**阻塞赋值**难以判断执行顺序

4.  请解释 ``timescale 1ns/1ps` 的含义

    分别表示时间的单位和精度，比如 `#100` 表示 `100.000ns` 

5.  请解释过程控制模块 `always @(posedge clk or negedge rstn)` 的触发条件

    当时钟处在上升沿或下降沿的时候，语句块被执行

### 七段数码管显示优化

第一步：将需要输出的32位分为8组4位二进制数，即8个16进制数

第二步：将8组4位二进制数依次输入一个decoder并将他的输出在一个短时间间隔后依次接到下一个显示数码管。

按照这样的方法能够实现在短时间内用一个decoder循环接入8个显示数码管，利用视觉暂留达到显示8个数码管的目的。
