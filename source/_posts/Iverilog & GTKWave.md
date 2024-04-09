title: Iverilog & GTKWave
tags:
  - 软件
categories: [杂]
author: siegelion
date: 2019-09-14 15:05:00
---

---
###### 一直都因为太过懒惰，没有更新网站的文章。
正好最近学校小学期的任务是，进行VerilogHDL语言的学习，需要安装可以编译该语言的软件，课上学习使用的vivado，当时学的时候就是云里雾里，现在需要用到了，打算在自己的电脑上安装一个，看到软件大小`10.4G`，果断放弃。

一顿搜索引擎，发现大家与我意见一致，vivado对我们来说确实是屠龙刀了。

###### 更好的选择是 Iverilog 加上 GTKWave

直接从[Icarus Verilog for Windows](http://bleyer.org/icarus/ "Icarus Verilog for Windows")下载安装即可，这里提供的为Windows版

Icarus Verilog中已经包含了GTKWave，不必再额外下载

1. 注意安装时是默认全部安装，不要自作聪明自定义,反而弄巧成拙
2. 安装时最后有一步，可以自动将其需要用到的路径，加入自动变量，建议设置

安装之后就可以编辑一个verilog文件了

```verilog
/*****
**  文件名称：hello_world_tb.v
**  功能描述：一个iverilog和GTKWave使用方式介绍的hello world例子
*****/

// synopsys translate_off
`timescale 1 ns / 1 ps
// synopsys translate_on

module hello_world_tb;
    parameter PERI = 10;

    reg clk;
    reg rst_n;

    always #(PERI/2) clk = ~clk;

    initial
    begin
        $dumpfile("hello_world_tb.vcd");
        $dumpvars(0,hello_world_tb);
        $display("hello world!");
        clk = 0;
        rst_n = 0;
        repeat(10) @(posedge clk);
        rst_n = 1;
        repeat(100) @(posedge clk);
        $finish;
    end

endmodule
```
保存为 `hello_world_tb.v` 文件

调出命令行，以此输入以下命令


```verilog
iverilog -o hello hello_world_tb.v
vvp -n hello -lxt2
gtkwave hello_world_tb.vcd
```


就可以看到，.v文件以此被编译为.vcd文件，用gtkwave打开.vcd文件既可以看到波形了。