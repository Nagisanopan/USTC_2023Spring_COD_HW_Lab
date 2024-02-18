# Lab 1 Report
<center> 姓名：张郑飞扬  学号：PB21071416  日期：2023.3.28</center>

## 实验目的与内容
---
* 目的
    * 掌握算术逻辑单元 (ALU) 的功能
    * 掌握数据通路和有限状态机的设计方法
    * 掌握组合电路和时序电路，以及参数化、结构化的Verilog描述方法
* 内容
    * 算术逻辑单元（ALU）及测试
    * ALU应用：计算斐波那契—卢卡斯数列（Fibonacci Lucas Series）

## 逻辑设计、仿真结果、下载测试
---

1. ALU
 
    本实验设计了一个具有7种运算功能（加法、减法、相等判断、无符号比较判断、逻辑与、逻辑或、逻辑异或）的ALU,其中加减法功能带有溢出判断。下面给出ALU及其测试电路的代码设计。

    文件结构如下：

    ```plaintext
    ALU_test.v
    --- ALU.v
    --- decoder.v
    ```
   
   ALU板块如下：

    ```verilog
    module ALU 
    #(  parameter WIDTH = 6,
        parameter ADD = 4'b0000,
        parameter SUB = 4'b0001,
        parameter EQL = 4'b0010,
        parameter ULE = 4'b0011,
        parameter AND = 4'b0101,
        parameter OR  = 4'b0110,
        parameter XOR = 4'b1000
    ) 
    (
        input [WIDTH-1:0] a, b,     //两操作数（ 对于减运算， a是被减数）
        input [3:0] func,           //操作功能（ 加、 减、 与、 或、 异或等）
        output reg [WIDTH-1:0] y,   //运算结果（ 和、 差 …）
        output reg of               //溢出标志of， 加减法结果溢出时置1
    );

    reg c;  
    reg [5:0] d;   

    always @(*) 
    begin
        case(func)
        ADD:
        begin
            if((a[5] ^ b[5]))   //a,b 异号时不存在溢出
            begin
                y <= a + b;
                of <= 0;
            end    
            else                //a,b 同号时可能溢出
            begin 
                {c ,y[4:0]} <= a[4:0] + b[4:0];
                y[5] <= a[5];
                of <= (a[5] & b[5]) ^ c; //符号位进位异或最高数值位进位
            end
        end

         SUB:
        begin
            d <= ~b + 6'b000001;    // 取反+1
            if((a[5] ^ d[5]))   
            begin
                y <= a + d;
                of <= 0;
            end    
            else                
            begin 
                {c ,y[4:0]} <= a[4:0] + d[4:0];
                y[5] <= a[5];
                of <= (a[5] & d[5]) ^ c; 
            end
        end

        EQL:
        begin
            of <= 0;
            if(a == b)  y <= 6'b000001;
            else        y <= 6'b000000;
        end

        ULE:
        begin
            of <= 0;
            if(a < b)   y <= 6'b000001;
            else        y <= 6'b000000;
        end

        AND:
        begin
            of <= 0;
            y <= a & b;
        end

        OR:
        begin
            of <= 0;
            y <= a | b;
        end

        XOR:
        begin
            of <= 0;
            y <= a ^ b;
        end

        default:
        begin
            of <= 0;
            y <= 6'b000000;
        end

        endcase
    end
    endmodule
    ```

    其中比较重要的部分是对溢出的判断。这里利用了两操作数同号时，最高数值位进位和符号位进位进行“异或”逻辑运算的判断方法。若值为1，则溢出，若值不为1，则没有溢出。

    另外，减法操作只需要对减数进行“取反+1”，再利用上述逻辑进行溢出判断即可。

    其他操作只需要利用Verilog中的运算符一步到位。

    <br/>

    decoder板块如下：

    ```verilog
    module decoder
    (
    input en,
    input [1:0]sel,
    output ena, enb, enf    
    );

    assign ena = en & (sel == 2'b00);
    assign enb = en & (sel == 2'b01);
    assign enf = en & (sel == 2'b10);

    endmodule
    ```

    由于FPGAOL外设资源有限，因此端口需要分时复用：操作数a, b和功能f 复用开关输入x[5:0]。复用方法：通过sel和en ，译码生成寄存器使能信号ena，enb，enf，将开关输入x[5:0]分时存入寄存器 F(x[3:0])，A(x[5:0])，B(x[5:0])。这是设计上述译码器的原因。

    <br/>

    顶层模块ALU_test：
    ```verilog
    module ALU_test
    #(parameter WIDTH = 6)
    (
    input clk,
    input en,
    input [1:0]sel,
    input [5:0] x,
    output reg [5:0]y,
    output reg of
    );

    wire ena, enb, enf;
    wire [WIDTH - 1:0] alu_y;
    wire alu_of;
    reg  [3:0] f;
    reg  [WIDTH - 1:0] a, b;

    decoder decoder1(
    .en(en),
    .sel(sel),
    .ena(ena),
    .enb(enb),
    .enf(enf)
    );

    ALU ALU1(
    .a(a),
    .b(b),
    .func(f),
    .y(alu_y),
    .of(alu_of)
    );

    always @(posedge clk)
    begin
        if (enf) f <= x[3:0];
        if (ena) a <= x;
        if (enb) b <= x;
        y <= alu_y;
        of <= alu_of;
    end

    endmodule
    ```

    <br/>
    仿真文件如下：

    ```verilog
    module simulation();
    parameter PERIOD = 100;
    parameter HALFCYCLE = 10;
    parameter WIDTH = 6;

    reg clk, en;
    reg [1:0] sel;
    reg [WIDTH-1:0] x;

    wire [WIDTH-1:0] y;
    wire of;

    ALU_test #(.WIDTH(WIDTH)) testbench(
        .clk(clk),
        .en(en),
        .sel(sel),
        .x(x),
        .y(y),
        .of(of)
    );

    always #(HALFCYCLE) clk = ~clk;
    initial begin
        clk = 1'b0;
        en = 1'b0;
        sel = 2'b11;
        x = 6'h00;
        #(PERIOD)
        en = 1'b1;
        #(PERIOD)
        sel = 2'b10;    // 选择加法功能
        x = 6'b000000;  
        #(PERIOD)
        sel = 2'b00;    // 输入a的值：12
        x = 6'b001100;
        #(PERIOD)
        sel = 2'b01;    // 输入b的值：2
        x = 6'b000010;
        #(PERIOD)
        x = 6'b010100;  // 换一个b的值：20
        #(PERIOD)
        sel = 2'b10;    
        x = 6'b000001;  //选择减法功能
        #(PERIOD)
        x = 6'b000010;  //选择判断等于功能
        #(PERIOD)
        x = 6'b000011;  //选择判断无符号小于功能
        #(PERIOD)
        x = 6'b000101;  //选择逻辑与功能
        #(PERIOD)
        x = 6'b000110;  //选择逻辑或功能 
        #(PERIOD)
        x = 6'b001000;  //选择逻辑异或功能
        #(PERIOD)
        $finish;
    end
    endmodule
    ```

    得到的仿真波形为：![](ALU_TEST仿真.png)

    RTL分析：![](RTL.png)
    
    综合分析: ![](Synthesis.png)

    下载测试：在FPGOL上完成12+2运算 ![](ALU_FPGA_12+2.png)

<br/>
<br/>


2. FLS

    本设计的数据通路如下: ![](data%20path.jpg)

    本设计的状态转换图如下：![](State%20Switch.jpg)

    本设计的文件结构如下：
    ```plaintext
    FLS.v
    ---getedge.v
    ---FSM.v
    ---ALU.v
    ```

    getedge为取边沿信号模块：
    ```verilog
    module getedge(
    input clk,
    input button,
    input rst,
    output button_edge
    );

    reg button_r1, button_r2;
    always @(posedge clk) button_r1 <= button & ~rst;
    always @(posedge clk) button_r2 <= button_r1;
    assign button_edge = button_r1 & (~button_r2);
    endmodule
    ``` 

    FSM为有限状态机模块，有三个状态（INIT,LOAD_A,LOAD_B)，代码描述模式采用三段式:

    ```verilog
    module FSM(
    input clk,
    input rst,
    input en,
    output [1:0] state
    );
    reg [1:0] cs;
    reg [1:0] ns;
    
    parameter INIT = 2'b00;
    parameter LOAD_A = 2'b01;
    parameter LOAD_B = 2'b10;

    // FSM Part 1: Current State(CS) Description
    always @(posedge clk) 
        begin
            if (rst) cs <= INIT;
            else if (en) cs <= ns;
        end

    // FSM Part 2: Next State(NS) Description
    always @(*) begin
        case (cs)
            INIT: ns = LOAD_A;
            LOAD_A: ns = LOAD_B;
            LOAD_B: ns = LOAD_B;
            default: ns = INIT;
        endcase
    end

    // FSM Part 3: Output Description
    assign state = cs;
    endmodule
    ```

    ALU模块直接实例化上面已经做好的ALU，并且选择其加法模式。需要注意的是输入输出位宽有变化。

    FLS顶层模块：

    ```verilog
    module FLS(
    input clk,
    input rst,
    input en,
    input [6:0] d,
    output reg [6:0] f
    );
    //getedge
    wire en_edge;
    getedge get_en_edge(
        .clk(clk),
        .button(en),
        .rst(rst),
        .button_edge(en_edge)
    );

    // ALU
    reg [6:0] a;
    wire [6:0] alu_out;
    ALU #(.WIDTH(7)) ALU_adder(
        .a(a),
        .b(f),
        .func(4'b0000),      //选择加法功能
        .y(alu_out)
    );
    
    // FSM
    wire [1:0] sel;
    FSM FSM1(
        .clk(clk),
        .rst(rst),
        .en(en_edge),
        .state(sel)
    );

    // other details
    always @(posedge clk) 
    begin
        if (rst) a <= 7'h00;
        else if (en_edge) 
        begin
            case (sel)
                2'b00: a <= d;
                2'b10: a <= f;
                default: a <= a;
            endcase
        end
    end
    always @(posedge clk) 
    begin
        if (rst) f <= 7'h00;
        else if (en_edge) 
        begin
            case (sel)
                2'b00: f <= d;
                2'b01: f <= d;
                2'b10: f <= alu_out;
                default: f <= f;
            endcase
        end
    end
    endmodule
    ```

    仿真文件如下：
    ```verilog
    module sim();
    reg clk;
    reg rst;
    reg en;
    reg [6:0] d;
    wire [6:0] f;

    FLS testbench(
        .clk(clk),
        .rst(rst),
        .en(en),
        .d(d),
        .f(f)
    );

    parameter HALFCYCLE = 1;

    always #(HALFCYCLE) clk = ~clk;

    initial begin
        clk = 1'b0;
        #200 $finish;
    end

    initial begin
        rst = 1'b1;
        #7 rst = 1'b0;
    end

    initial begin
        en = 1'b0;
        #2  en = 1'b1;
        #25 en = 1'b0;
        #10 en = 1'b1;
        #10 en = 1'b0;
        #10 en = 1'b1;
        #10 en = 1'b0; 
        #10 en = 1'b1;
        #10 en = 1'b0;
        #10 en = 1'b1;
        #10 en = 1'b0;
        #10 en = 1'b1;
    end

    initial begin
        d = 7'h02;
        #32 d = 7'h03;
        #20 d = 7'h04;
        #20 d = 7'h05;
        #20 d = 7'h06;
    end
    endmodule
    ```

    仿真波形如下:![](FLS_仿真.png)

    下载测试如下：  
    输入第一项：2 ![](FLS%20step1_input2.png)
    输入第二项：3 ![](FLS%20step2_input3.png)
    产生第三项：5 ![](FLS%20step3_5.png)
    产生第四项：8 ![](FLS%20step4_8.png)

<br/>
<br/>

## 总结
---
本次实验分为两部分，第一部分实现了一个基本的ALU和其测试电路。第二部分通过实例化该ALU，实现了一个计算斐波那契数列的有限状态机。实验难度适中，但任务量较大，写仿真代码、调试代码都比较花时间。