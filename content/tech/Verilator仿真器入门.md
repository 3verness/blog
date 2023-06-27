---
title: "Verilator仿真器入门"
slug: "verilator-tutorial"
subtitle: ""
defcription: ""
tags:
    - "fpga"
    - "verilator"
    - "systemverilog"
    - "tutorial"
date: 2023-06-27T20:58:24+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

## Why Verilator?

* 唯一支持SystemVerilog 2017标准的开源Verilog Simulator；
* 将Verilog编译为C++或SystemC类，更为方便的进行性能测试和仿真；
* 拥有严格的代码规范，仅支持`display()`, `$finish()`, `$fatal()`等少量不可综合的代码，在综合前排查错误；
* 快，非常快；

速度伴随着一定的代价，Verilator的仿真具有一定局限性，包括但不限于：

* 基于周期仿真，无时序信息，信号转变在瞬间完成，两时钟沿内的所有信号变化都会被忽略；
* 相比于实际电路中的“01xz”，Verilator中信号只有“01”两个值，且所有信号初值均为0，可能无法观察到电路初始化。

## How?

Verilator的仿真分为如下几步：

* 编写电路功能代码(Verilog/SystemVerilog)；
* 编写测试代码(C++/SystemC)；
* 由Verilog生成C++/SystemC工程；
* 编译可执行文件；
* 运行仿真。

具体命令如下：

* 转换代码，生成工程

  ```bash
  verilator -Wall --trace --cc alu.sv --exe tb_alu.cpp
  ```

  * `--cc`：转换为C++工程，若使用SystemC应使用`--sc`；
  * `--trace`：进行波形记录，仅当需要查看波形时使用；
  * `--exe`：工程`Makefile`生成可执行文件，如不使用会生成`.a`的静态链接库；
  * `-Wall`：显示所有格式警告。

  其他常用参数：

  * `--Mdir`：指定生成目录；
  * `--top-module`：指定top；
  * `-x-assign`：设定X对应值，可选`0`，`1`，`fast`（性能最优，默认），`unique`（随机值）；
  * `-x-initial`：设定初值，可选`0`，`unique`（随机值，默认，用于检查reset功能），`fast`（性能最优）。

  {{< note info >}}

  如果需要使用`unique`随机值，需要在运行可执行文件时传入runtime参数`+verilator+rand+reset+2`，其中可选`0`（全0），`1`（全1），`2`（随机），具体实现为：

  * 测试样例中，在初始化模块前向仿真器传入参数，如：

    ```c++
    int main(int argc, char** argv, char** env) {
        Verilated::commandArgs(argc, argv);
        Valu *dut = new Valu;
        ...
    }
    ```

  * 运行可执行文件时带参数运行：

    ```bash
    verilator -Wall --trace --x-assign unique --x-initial unique -cc alu.sv --exe tb_alu.cpp
    ```

  {{< /note >}}

  该步执行完成，会在指定目录（默认`obj_dir`）下生成工程文件，其中`Valu.h`为类头文件，包含所有可调用接口，`Valu.mk`为`Makefile`。

* 编译可执行文件

  ```bash
  make -C obj_dir -f Valu.mk Valu
  ```

  编译可执行文件，`-C`指定工程目录，`-f`选择工程文件，`Valu`为可执行文件名称。

* 运行仿真

  ```bash
  ./obj_dir/Valu
  ```

  运行仿真，会在当前目录生成波形或其他仿真文件。

## 如何编写Testbench

Verilator仿真器中，通过执行`eval()`步进仿真，并手动调用`dump()`记录，每次循环结构为:

```
+------------+   +------+   +-----------+   +------+
|invert clock+-->|eval()+-->|renew value+-->|dump()|
+------------+   +------+   +-----------+   +------+
```

具体直接看注释吧~

```c++
#include <cstdlib>
#include <iostream>

// verilator仿真库文件
#include <verilated.h>
// VCD波形库
#include <verilated_vcd_c.h>

// 导入模型类
#include "Valu.h"
// 导入模型中的enum
#include "Valu___024unit.h"

// 终止时间
#define MAX_SIM_TIME 200
// 全局仿真时间
vluint64_t sim_time = 0;

// 记录时钟周期
vluint64_t posedge_cnt = 0;

// 初始化函数
void dut_reset(Valu *dut, vluint64_t &sim_time)
{
    dut->rst = 0;
    if (sim_time >= 3 && sim_time < 6)
    {
        dut->rst = 1;
        dut->a_in = 0;
        dut->b_in = 0;
        dut->op_in = Valu___024unit::nop;
        dut->in_valid = 0;
    }
}

int main(int argc, char **argv, char **env)
{
    // 初始化C++随机数生成器
    srand(time(NULL));
    // 向仿真器传入参数
    Verilated::commandArgs(argc, argv);
    // 初始化电路模型
    Valu *dut = new Valu;

    // 启用波形记录
    Verilated::traceEverOn(true);
    // 创建波形
    VerilatedVcdC *m_trace = new VerilatedVcdC;
    // 指定波形和深度
    dut->trace(m_trace, 5);
    // 打开波形文件
    m_trace->open("waveform.vcd");

    while (sim_time < MAX_SIM_TIME)
    {
        // 初始化
        dut_reset(dut, sim_time);

        // 时钟翻转
        dut->clk ^= 1;
        // 计算波形
        dut->eval();

        // 检测上升沿 0->1
        if (dut->clk == 1)
        {
            // 测试用例延迟，等待初始化结束
            if (sim_time >= 10)
                posedge_cnt++;

            dut->op_in = Valu___024unit::nop;
            dut->in_valid = 0;
            // 输入加法测试用例
            if (posedge_cnt == 2)
            {
                dut->a_in = rand();
                dut->b_in = rand();
                dut->op_in = Valu___024unit::add;
                dut->in_valid = 1;
            }
            // 检查加法结果
            if (posedge_cnt == 4)
            {
                if (dut->out_valid != 1 || dut->out != uint8_t(dut->a_in + dut->b_in))
                    std::cout << "ERROR: add mismatch, "
                              << "input: a=" << int(dut->a_in) << ", "
                              << "b=" << int(dut->b_in) << ", "
                              << "exp: " << int(uint8_t(dut->a_in + dut->b_in)) << ", "
                              << "recv:" << int(dut->out) << ", "
                              << "simtime:" << int(sim_time) << std::endl;
            }
            // 输入减法测试用例
            if (posedge_cnt == 6)
            {
                dut->a_in = rand();
                dut->b_in = rand();
                dut->op_in = Valu___024unit::sub;
                dut->in_valid = 1;
            }
            // 检查减法结果
            if (posedge_cnt == 8)
            {
                if (dut->out_valid != 1 || dut->out != uint8_t(dut->a_in - dut->b_in))
                    std::cout << "ERROR: sub mismatch, "
                              << "input: a=" << int(dut->a_in) << ", "
                              << "b=" << int(dut->b_in) << ", "
                              << "exp: " << int(uint8_t(dut->a_in - dut->b_in)) << ", "
                              << "recv:" << int(dut->out) << ", "
                              << "simtime:" << int(sim_time) << std::endl;

                posedge_cnt = 0;
            }
        }

        // 记录当前时刻波形
        m_trace->dump(sim_time);
        // 全局时钟计时
        sim_time++;
    }

    // 关闭波形文件
    m_trace->close();
    // 释放内存
    delete dut;
    exit(EXIT_SUCCESS);
}
```

仿真结果如下图所示，注意76ps时钟上升沿处，由于输入值更新在步进仿真后，`in_valid`值尚未由0变1，仿真不读入输入数据，同理，78ps时钟上升沿处，`in_valid`值尚未从1变0，读入数据，一周期后输出有效。

![image-20230627204815218](https://img.ioyoi.me/20230627204823.webp)

{{< note info >}}

Testbench中使用了`Valu___024unit.h`中的`enum operation_t`，若想生成对应的C++ `enum`，需要在SystemVerilog代码中添加辅助注释：

```systemverilog
typedef enum logic [1:0] {
  add = 2'h1,
  sub = 2'h2,
  nop = 2'h0
} operation_t /*verilator public*/;
```

{{< /note >}}

## 再送一个Makefile

```makefile
MODULE=alu

# 运行仿真
.PHONY:sim
sim: waveform.vcd

# 仅生成工程文件
.PHONY:verilate
verilate: .stamp.verilate

# 生成工程文件并编译
.PHONY:build
build: ./obj_dir/V$(MODULE)

# 仿真并查看波形
.PHONY:waves
waves: waveform.vcd
	@echo
	@echo "### WAVES ###"
	gtkwave waveform.vcd

waveform.vcd: ./obj_dir/V$(MODULE)
	@echo
	@echo "### SIMULATING ###"
	@./obj_dir/V$(MODULE) +verilator+rand+reset+2

./obj_dir/V$(MODULE): .stamp.verilate
	@echo
	@echo "### BUILDING SIM ###"
	make -C obj_dir -f V$(MODULE).mk V$(MODULE)

.stamp.verilate: $(MODULE).sv tb_$(MODULE).cpp
	@echo
	@echo "### VERILATING ###"
	verilator -Wall --trace --x-assign unique --x-initial unique -cc $(MODULE).sv --exe tb_$(MODULE).cpp
	@touch .stamp.verilate

# 格式化代码
.PHONY:lint
lint: $(MODULE).sv
	verilator --lint-only $(MODULE).sv

.PHONY:clean
clean:
	rm -rf .stamp.*;
	rm -rf ./obj_dir
	rm -rf waveform.vcd
```

* `make sim`：直接运行仿真；
* `make build`：生成工程并编译可执行文件；
* `make verilate`：仅转换Verilog代码生成工程；
* `make waves`：仿真并通过`gtkwave`查看波形；
* `make lint`：格式化Verilog代码；
* `make clean`：清理。

# 附录：使用的算术逻辑单元

```systemverilog
typedef enum logic [1:0] {
  add = 2'h1,
  sub = 2'h2,
  nop = 2'h0
} operation_t /*verilator public*/;

module alu #(
    parameter WIDTH = 8
) (
    input clk,
    input rst,

    input operation_t op_in,
    input [WIDTH-1:0] a_in,
    input [WIDTH-1:0] b_in,
    input in_valid,

    output logic [WIDTH-1:0] out,
    output logic out_valid
);

  operation_t op_in_r;
  logic [WIDTH-1:0] a_in_r;
  logic [WIDTH-1:0] b_in_r;
  logic in_valid_r;
  logic [WIDTH-1:0] result;

  always_ff @(posedge clk, posedge rst) begin
    if (rst) begin
      op_in_r <= nop;
      a_in_r <= '0;
      b_in_r <= '0;
      in_valid_r <= '0;
    end else begin
      op_in_r <= op_in;
      a_in_r <= a_in;
      b_in_r <= b_in;
      in_valid_r <= in_valid;
    end
  end

  always_comb begin
    result = '0;
    if (in_valid_r) begin
      case (op_in_r)
        add: result = a_in_r + b_in_r;
        sub: result = a_in_r + (~b_in_r + 1'b1);
        default: result = '0;
      endcase
    end
  end

  always_ff @(posedge clk, posedge rst) begin
    if (rst) begin
      out <= '0;
      out_valid <= '0;
    end else begin
      out <= result;
      out_valid <= in_valid_r;
    end
  end
endmodule
```

## 参考资料

1. [Verilator Doc](https://verilator.org/guide/latest/index.html)
2. [Verilator Pt.1: Introduction](https://itsembedded.com/dhd/verilator_1/)
3. [Verilator Pt.2: Basics of SystemVerilog verification using C++](https://itsembedded.com/dhd/verilator_2/)
4. [Verilator Pt.3: Traditional style verification example](https://itsembedded.com/dhd/verilator_3/)
