---
title: "“芯”开始·2：cocotb仿真框架"
slug: "fpga-learn-2"
subtitle: ""
description: "搭建cocotb仿真环境，使用Python编写RTL仿真。"
tags:
    - "FPGA"
    - "oss-cad-suite"
    - "SystemVerilog"
    - "cocotb"
date: 2025-08-23T20:39:29+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

该系列的第二篇文章，想要分享的是RTL语言的仿真。我最早接触的RTL仿真工具是Vivado自带的仿真器，他拥有一个美观且实用的界面，并根据ZYNQ架构的特点提供了许多实用的工具，比如用于AXI总线仿真的VIP工具简直是神中神，并且可以在一套工具中完成行为级仿真、RTL仿真和时序仿真，对FPGA开发而言十分方便。

研究生阶段相当长的一段时间里，我只需要编写RTL代码实现功能，而无需FPGA部署。随着项目规模的膨胀，Vivado仿真器的速度渐渐无法满足需求。随后，我短暂使用了一段时间iverilog，它非常轻量，但遗憾的是，当时它对于SystemVerilog的支持非常局限，一些常用的状态类、接口功能都无法实现。之后我开始长期使用破解版的VCS、modelsim，这些工具确实专业且好用，许多公司都使用它们开发大型数字IC项目。

不过，我常常发现写测例是一件非常痛苦的事，问题大多来自于SystemVerilog语法，我的verilog水平并不高，SystemVerilog的仿真语法更是繁多复杂，此外，我也一直没有掌握UVM仿真的精髓，相比之下，我更倾向于使用编程中的“对拍”方法对模块进行仿真验证。

后来在“一生一芯”学习中，我接触到了Verilator这个开源仿真器，他可以将RTL语言转译成C++语言，并允许使用C++编写测例。这非常符合我的仿真习惯，我可以使用C++实现一个相同功能的模块，并在Testbench中测试RTL模块和C++模块的功能差异。

前短时间，我在oss-cad-suite中发现了一个更加方便的仿真工具，他就是今天的主角——cocotb。

## cocotb是什么？

cocotb是一个开源的、基于Python的硬件设计验证工具，它的核心思想是使用Python编写测例，从而替代传统的Verilog/SystemVerilog测试代码。它通过接口与HDL仿真器进行协同仿真，能够接入包括iverilog、Verilator、VCS在内的几乎所有主流仿真器。cocotb本质上提供了一个API，让你可以在Python中调用你的RTL设计。可以这样理解他的工作模式：被测模块的RTL模型在HDL仿真器中运行，测试逻辑、激励生成、结果检查则在Python中运行，cocotb充当“桥梁”，让Python测试代码可以与仿真器同步驱动、读取DUT的输入输出信号。

![](https://docs.cocotb.org/en/stable/_images/cocotb_overview.svg)

相较传统手动编写Verilog测例，我目前感受到cocotb有以下几点优势：

1. Python语法比Verilog简单得多，上手门槛极低；
2. 可以轻松利用Python的第三方库快速实现一个复杂算法的参考模型，进行“对拍”测试；
3. Python更容易对结果进行可视化显示，验证结果更加直观；
4. cocotb的测试脚本是和仿真器无关的，同一套仿真代码可以无须修改在所有主流仿真器上运行。

因此，对于绝大多数 FPGA 设计来说，cocotb 提供了一种远比手动编写Verilog测例更高效、灵活、强大的验证方法，它极大地提升了我的工作效率。

## 如何编写cocotb测例？

{{<note info>}}

本文基于`cocotb 2.0.0.dev1`版本编写，由于2.0版本尚处于开发阶段，部分接口可能存在调整，如运行时出现错误，请参照cocotb项目仓库提供的对应版本[示例](https://github.com/cocotb/cocotb/tree/master/examples)进行修改。

{{</note>}}

让我们先从一个移位寄存器的例子开始，我的习惯是将一个FPGA项目分成`hdl`和`tb`两个文件夹管理，`hdl`中放置模块源文件、约束文件，`tb`文件用于存放验证文件和仿真结果，仅仿真的文件结构如下所示：

```tree
shift_register
├── hdl
│   └── shift_register.sv
└── tb
    ├── Makefile
    └── shift_register_tb.py
```

其中，移位寄存器的硬件设计写在`shift_register.sv`文件中，测例则存放于`shift_register_tb.py`内，`Makefile`用于管理cocotb编译配置。

简单展示一下`shift_register.sv`源码，一个经典的异步复位、同步恢复移位寄存器：

```systemverilog
module shift_register #(
    parameter int DATA_WIDTH = 8
) (
    input logic clk,
    input logic rst_n,
    input logic d_i,
    output logic [DATA_WIDTH-1:0] q_o
);

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      q_o <= '0;
    end else begin
      q_o <= {q_o[DATA_WIDTH-2:0], d_i};
    end
  end

endmodule
```

我们重点关注测例文件`shift_register_tb`：

```py
import cocotb
from cocotb._extended_awaitables import ClockCycles
from cocotb.clock import Clock
from cocotb.triggers import RisingEdge, FallingEdge, Timer

import random

def shift_register_model(q, d, width):
    mask = (1 << width) - 1
    return ((int(q) << 1) | int(d)) & mask

@cocotb.test()
async def test_basic_shift(dut):
    # Start clock
    cocotb.start_soon(Clock(dut.clk, 10, unit="ns").start())

    # Apply reset
    dut.d_i.value = 0
    dut.rst_n.value = 0
    await Timer(20, unit="ns")
    dut.rst_n.value = 1
    await RisingEdge(dut.clk)

    # Save current q_o
    last_q = dut.q_o.value

    WIDTH = len(dut.q_o)
    for _ in range(WIDTH):
        dut.d_i.value = 1
        expected_q = shift_register_model(last_q, dut.d_i.value, WIDTH)
        await RisingEdge(dut.clk)
        assert dut.q_o.value == expected_q, f"Test basic shift failed, Last Q is {last_q}, D is {dut.d_i.value}, Expected Q is {expected_q}, but Actual Q is {dut.q_o.value}"
        last_q = expected_q
```

由于Python简单的语法和良好的可读性，哪怕你从没接触过数字仿真，你应该也能理解测例语句法的功能。

上文中提到，cocotb的一大特点是可以编写python模块与仿真硬件进行对拍，在这个例子中，我们编写了一个`shift_register_model`用于检查仿真结果的正确性，这在复杂模块调试时将会非常实用。

首先，Python代码通过`cocotb`包与hdl仿真器交互，包中提供了一些`Triggers`用于时序控制：

1. 时钟周期：`ClockCycles`，替代`always #n clk = ~clk`；
2. 定时任务：`Timer`，替代`#n`；
3. 边沿检测：`RisingEdge`、`FallingEdge`

仿真文件中测例以使用`@cocotb.test`修饰的函数形式存在，测例函数通过传参`dut`与硬件电路进行交互，使用`dut.信号名.value`对内部信号进行读取和激励。每个测例都是一个异步函数，同一个文件内可包含多条测例，并且这些测例可以并发运行，缩短仿真时间。当测例运行到`await`处时，便会暂停当前测例的运行，直到触发某一条件，通过`await`与`Trigger`的配合，可以方便的控制激励信号时序。测例中通过调用`assert`语句对结果进行检查，并输出错误信息。

如果你对详细的测例编写方式感兴趣，可以进一步阅读["Writing Testbench - cocotb"](https://docs.cocotb.org/en/stable/writing_testbenches.html)或查看[更多示例](https://github.com/cocotb/cocotb/tree/master/examples)。

## 如何运行cocotb？

Cocotb提供Makefile和Python runner两种运行方式，我个人比较倾向于使用Makefile，只需要导入cocotb提供的`Makefile.sim`，并做一些简单的配置，即可通过`make`运行。项目中的`Makefile`大概长这个样子：

```makefile
# Define default simulator
SIM ?= verilator
# Define hdl language
TOPLEVEL_LANG ?= verilog

# Define hdl source file
VERILOG_SOURCES = $(PWD)/../hdl/shift_register.sv

# Define Top
TOPLEVEL = shift_register

# Define Testbench
COCOTB_TEST_MODULES = shift_register_tb

# include cocotb's Makefile
include $(shell cocotb-config --makefiles)/Makefile.sim
```

`Makefile`中需要指定仿真器、hdl语言、源文件位置、TOP名称、仿真文件位置，并导入cocotb预置的`Makefile`文件。

准备好上述文件，我们便可以进入`tb`目录，运行`make`，cocotb会自动根据我们选择的仿真器编译项目，并输出结果：

![image-20250824151646055](https://img.ioyoi.me/20250824151648.webp)

上图所示是在Verilator仿真器的结果，我们并没有对Verilator进行任何配置，cocotb就自动帮我们生成并编译了整个项目，这实在是太方便了！从表格中可以看到，我们的`tb`文件中包含三项测例，均能顺利通过验证。

当验证出现错误时，则会输出`assert`断言中的信息并通过trace提供测例文件的具体位置/

![image-20250824161645183](https://img.ioyoi.me/20250824161647.webp)

如果我们想要换用其他仿真器运行，只需要输入：

```bash
make SIM=icarus
```

传入参数覆盖了Makefile中的仿真器选项，cocotb会自动帮我们调用iverilog执行本次仿真。

![image-20250824152136713](https://img.ioyoi.me/20250824152138.webp)

## 如何记录波形？

上一章节中，我们在仿真时只看到了最终的结果，如果我们想要记录波形该怎么做呢？

目前cocotb对仿真波形记录的支持尚不完善，对于不同的仿真器需要单独进行额外的配置，我目前只使用了Verilator和iverilog。

对于Verilator，只需要在Makefile中加入调用仿真器时传入的额外的`--trace`参数，在Makefile中加入下列内容可以记录fst格式波形文件。

```makefile
ifeq ($(SIM), verilator)
	EXTRA_ARGS += --trace --trace-fst --trace-structs
endif
```

而对于iverilog，则需要对hdl文件进行修改，加入`dump`相关语句，并在运行`make`时传入`make WAVES=1`。

```systemverilog
  // Dump waves
  initial begin
    $dumpfile("dump.vcd");
    $dumpvars(1, adder);
  end
```

![image-20250824161214304](/home/ioyoi/.config/Typora/typora-user-images/image-20250824161214304.png)

从波形文件可以看到仿真共经历3次复位，这是由于不做特殊配置的情况下，cocotb的测例是顺序依次进行的。

## 结语

本篇文章，我们介绍了FPGA项目开发中重要的仿真与验证步骤，所使用的cocotb库是一个方便、快速的仿真框架，Python与仿真器的交互大大降低了测例编写的难度和仿真器的配置，相信cocotb会为你带来一个与众不同的验证体验。

在下一篇文章中，我们将介绍如何使用开源工具链完成FPGA项目的网表综合、布局布线、比特流生成和下载烧录。
