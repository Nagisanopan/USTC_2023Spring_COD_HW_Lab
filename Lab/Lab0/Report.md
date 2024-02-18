## <center>Lab0

<p align="right" >  PB21071416 张郑飞扬 </p>

### 实验要求
---
    在 Verilog oj47 中，我们实现了一个计数循环为 15 的计数器。
 
    事实上在此基础上，通过一些简单的修改，我们就可以实现一个任意位数的计数器。现在，我们希望完成一个时分秒时钟，该时钟在每次 clk 上升沿时秒位加 1（clk 信号不必以秒为周期），满 20 后清零，分位加 1；分位满 10 后清零，时位加 1；时位满 5 后三个位全部清零，如此循环。也就是说，我们设计的时钟一分钟只有 20 秒，一小时只有 10 分钟，一天只有五小时。*
 
    请尝试使用模块化的设计方法完成该时钟，并给出所有模块输入输出的仿真波形。


<br/>
<br/>

### 代码呈现

---
1. 顶层代码Clock
   
```Verilog
   module Clock(
    input clk,
    input rst,
    output [2:0] hour,
    output [3:0] min,
    output [4:0] sec);
    
    wire cos; //秒进位标志
    wire com; //分进位标志

    Sec sec1(
        .clk(clk),
        .rst(rst),
        .sec(sec),
        .co(cos)
    );

    Min min1(
        .clk(cos),
        .rst(rst),
        .min(min),
        .co(com)
    );

    Hour hour1(
        .clk(com),
        .rst(rst),
        .hour(hour)
        );
    endmodule 
```


2. Sec模块
   
```Verilog
   module Sec(
    input clk,
    input rst,
    output [4:0]sec,
    output co     //进位标志
    );

    wire [4:0]count;
    
    regiss regis_sec(
        .clk(clk),
        .rst(rst),
        .q(sec),
        .d(count),
        .co(co)
    );

    assign count = sec+1;

    endmodule
   
   module regiss(
    input clk,
    input rst,
    input [4:0]d,
    output reg [4:0]q,
    output reg co
    );

    always@(posedge clk or posedge rst) //计数器赋值
    begin
        if(rst)
            q <= 5'b0;
        else if( q == 5'b10011) 
        begin
            q <= 0;
        end
        else
            q <= d;
    end
    
    always@(posedge clk or posedge rst) //进位赋值
    begin
        if(rst)
          co <= 0;
        else if(q == 5'b10011)
          co <= 1;
        else co <= 0;
    end
    endmodule
```
3. Min模块
```Verilog
   module Min(
    input clk,
    input rst,
    output [3:0]min,
    output co     //进位标志
    );

    wire [3:0]count;
    
    regism regis_min(
        .clk(clk),
        .rst(rst),
        .q(min),
        .d(count),
        .co(co)
    );

    assign count = min+1;
   endmodule

   module regism(
    input clk,
    input rst,
    input [3:0]d,
    output reg [3:0]q,
    output reg co
    );

    always@(posedge clk or posedge rst)  //计数器赋值
    begin
        if(rst)
            q <= 4'b0;
        else if( q == 4'b1001) 
            q <= 4'b0;
        else
            q <= d;
    end

    always@(posedge clk or posedge rst) //进位赋值
    begin
        if(rst)
          co <= 0;
        else if(q == 4'b1001)
          co <= 1;
        else co <= 0;
    end
    endmodule
```

4. Hour模块
   
```Verilog
   module Hour (
    input clk,
    input rst,     
    output [2:0] hour);

    wire [2:0]count;

    regish  regis_hour(
        .clk(clk),
        .rst(rst),
        .q(hour),
        .d(count)
    );

    assign count = hour+1;

   endmodule
   
   module regish(
    input clk,
    input rst,
    input [2:0]d,
    output reg [2:0]q);

    always@(posedge clk or posedge rst) 
    begin
        if(rst)
            q <= 3'b0;
        else if (q == 3'b100)
            q <= 3'b0;
        else
            q <= d;
    end
   endmodule 
```

<br/>
<br/>

### 仿真波形

---

1. Clock
   
   ![仿真波形](clock.png)

2. Sec

   ![仿真波形](sec.png)

3. Min

   ![仿真波形](min.png)

4. Hour

   ![仿真波形](hour.png)

<br/>
<br/>

### Vivado生成的电路图
---
![电路图](RTL.png)

<br/>
<br/>

### 反馈
---
        感谢助教给出的非常清晰易懂的实验文档，通过这个实验，我把差不多忘干净的Verilog写法和Vivado使用方法又拾起来了，更重要的是现在学会了用VSCode编写Verilog代码，这样方便了许多。实验整体做下来比较顺畅，体验不错。
