# Lab3实验报告

1.  使用行为描述实现二位BCD码计数器，比较它们的资源使用情况。

    ```verilog
    module Counter_99 (
        input clk,
	    input rst,
        input en,
	    output [3:0] one,
        output [3:0] ten,
        output co
    );
    reg co_r,c;
    reg[3:0] one_r,ten_r;
	always @(posedge clk) begin
         if(rst == 1) one_r <= 0;
	     else if(en == 1)begin
             case (one)
	             4'b1000:begin
                     one_r <= 4'b1001;
                     c <= 1;
                 end 
                 4'b1001:begin
                     one_r <= 4'b0000;
                     c <= 0;
                 end 
                 default:begin
                     one_r <= one_r + 1;
                     c <= 0;
                 end
             endcase
         end
    end
    assign one = one_r;
    always @(posedge clk) begin
    if(rst == 1) ten_r <= 0;
    	else if(c == 1 && en == 1)begin
    		case (ten)
    			4'b1001:begin
                    ten_r <= 4'b0000;
                    co_r <= 1;
            	end
    			default:begin
                   ten_r <= ten_r + 1;
                   co_r <= 0;
    			end
    		endcase
    	end
    end
    assign co = co_r;
    assign ten = ten_r;
    endmodule

资源使用情况：结构描述中额外增加了一个reg用于传输进位，而行为描述额外增加了两个四位寄存器喝两个进位寄存器。结构描述的资源使用率更高。

2.  你的分频器中计数器的最大值是多少；如何实现一个占空比不是50%的分频器。

    1.  1Hz分频器Divider中计数器的最大值为99。

    2.  设计一个模150计数器，并附有两个output co1，co2，co1用于记录计数达到99。co2用于记录计数达到149，co1和co2都能让Divider取反，从而达到33%占空比分频器

3.  如果一个复杂功能的状态机的状态图和3.1节中给出的计数器状态图类似存在一部分闲置状态，你认为会导致什么问题？谈谈你的看法，可以从性能、成本、安全等方面展开，角度不限。

    1.  成本方面：由于状态闲置，导致硬件寄存器单元的使用率不同，甚至可能出现闲置寄存器单元的现象，导致成本的浪费。

    2.  安全方面：由于状态闲置，但复杂的功能会使得维护状态系统的难度上升。
