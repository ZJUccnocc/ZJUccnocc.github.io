# **Lab0-2实验报告**

## GUI模式和Batch模式对比

1.  化简原理图中的表达式，分析Logicsim生成的表达式是否符合你的预期

    ![image-20240210195804030](./Lab0-2%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240210195804030.png)

|       |     |     |        |
|-------|-----|-----|--------|
| Input |     |     | Output |
| I0    | I1  | I2  | O      |
| 0     | 0   | 0   | 0      |
| 0     | 0   | 1   | 1      |
| 0     | 1   | 0   | 1      |
| 0     | 1   | 1   | 0      |
| 1     | 0   | 0   | 1      |
| 1     | 0   | 1   | 0      |
| 1     | 1   | 0   | 0      |
| 1     | 1   | 1   | 1      |

符合

2.  学习Makefile的知识尝试理解每一步都进行了什么操作，谈谈你对FPGA开发流程的理解

    1.  Makefile：

        一个工程中的源文件不计其数，其按类型、功能、模块分别放在若干个目录中，makefile定义了一系列的规则来指定哪些文件需要先编译，哪些文件需要后编译，哪些文件需要重新编译，甚至于进行更复杂的功能操作。makefile带来的好处就是——“自动化编译”，一旦写好，只需要一个make命令，整个工程完全自动编译，极大的提高了软件开发的效率。

        := 是直接赋值语句

        ?= 是如果在此之前没被赋值，那么就赋值

        \$()是取括号内变量的值

    2.  FPGA开发流程：

        **方法一（Batch模式）：**

        1.  绘制原理图：

            用Logisim绘制想要的电路原理图依次选择上方状态栏FPGA \> Synthesize&Download，设置完参数（` Target board=FPGA4U,Settings \> Hardware description language used for FPGA-commander=Verilog` ）后,点击execute \> done生成一个Verilog文件。

        2.  仿真Verilog设计：

            用make指令通过运行makefile打开gtkwave，在gtkwave里查看仿真波形是否符合自己期望的原理图

        3.  综合与实现Verilog设计：

            用make指令通过运行makefile打开vivado，依次点击上方 `FLow \> Open Hardware Manager \> Open target \> Auto Connect \> Program device` 选择你生成的Bitstream最后点击Program。而Bitstream文件是在make环节生成的。

        **方法二（GUI模式）：**

        1.  绘制原理图：

            用Logisim绘制想要的电路原理图依次选择上方状态栏FPGA \> Synthesize&Download，设置完参数（ `Target board=FPGA4U,Settings \> Hardware description language used for FPGA-commander=Verilog` ）后,点击execute \> done生成一个Verilog文件。

        2.  新建Vivado工程文件:  
            选择无代码的RTL工程,开发板型号下方搜索并选择xc7a100tcsg324-1,点击`Add Source \> Add or create design sources`，将LogicSim之前生成的verilog文件添加进工程。
        
        3.  仿真Verilog设计：

            点击`Add Source \> Add or create simulation sources`，创建名为testbench的测试激励文件。

            编辑其内部指令：定义一个与Logisim生成的module名字相同的（main）格式的led实例并对不同变量赋值main中对应的接口。 在initial块中对reg类型变量进行赋值，其中#100是时间间隔，指经过100个时间单位后再执行后续操作，我们依次对输入的引脚进行调整。

            ![image-20240210195844869](./Lab0-2%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240210195844869.png)

            编写完成后，点击侧栏`SIMULATION \> Run Simulation \> Run Behavioral Simulation`，即可看到波形。

        4.  综合并实现Verilog设计：

            点击Add Source选择Add or create design sources，这次我们新建一个名为top的文件，编辑其内部内容，并将其设置为顶层模块。

            ![image-20240210195858120](./Lab0-2%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240210195858120.png)

            完成后点击左侧SYNTHESIS \> Run Synthesis对设计进行综合。综合完成后，点击上方Layout \> I/O Planning对引脚进行分配。 在下方I/O Ports窗口中按图示进行配置，对每个引脚的Package Pin和I/O Std属性进行调整，引脚顺序依次是I0、I1、I2、O。 如果没有显示Name，你可以根据I/O Port Properties中的Name来进行确认。

            ![image-20240210195903494](./Lab0-2%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240210195903494.png)

            完成后保存引脚.xdc文件，点击左侧PROGRAM AND DEBUG \> Generate Bitstream即可生成.bit文件，生成后同样Open Target和Program Device烧入到板子上去。

&nbsp;

3. 对比src/lab0-2/sim/src/testbench.cpp与你自己编写的testbench.v的区别

![](./Lab0-2%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240210195913683.png)![image-20240210195919976](./Lab0-2%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240210195919976.png)

Cpp文件对引脚的赋值是随机的，而testbench的赋值是人工设定的。

1.  对比src/lab0-2/syn/src/nexysa7.xdc和你生成的引脚分配的区别（仅比较I0、I1、I2和O使用的引脚

    ![](./Lab0-2%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240210200159115.png)![image-20240210200204053](./Lab0-2%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240210200204053.png)

    都是LVCMOS33类型引脚。

    我生成的（J15, L16, M13, H17）都在xdc文件末尾，而xdc之中还有很多其他引脚，应该是其他逻辑门的引脚。

    我自己在Vivado上生成的引脚更为简单直观。

&nbsp;

## Verilog练习

1.  化简下列卡诺图，写出对应的Verilog代码，并上板进行验证，其中将ABCD作为R15、M13、L16、J15，输出作为H17:

    ![c06dbe094fe6f18c85df310979e0fe0](./Lab0-2%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240210200213471.png)![image-20240210200217573](./Lab0-2%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8Aimg/image-20240210200217573.png)
