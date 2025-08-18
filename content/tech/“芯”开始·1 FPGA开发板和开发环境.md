---
title: "“芯”开始·1：聊聊FPGA开发板和开发环境"
slug: "fpga-learn-1"
subtitle: "基于嘉立创逻辑派和全开源工具链的FPGA开发环境"
description: "踏上FPGA之旅的第一步，选择一块合适的开发板，并搭建一套丝滑的开发环境"
tags:
    - "FPGA"
    - "oss-cad-suite"
    - "Gowin"
    - "逻辑派"
date: 2025-08-16T18:37:38+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

## 引子

你是否曾梦想过亲手打造一颗属于自己的CPU？不是在面包板上插满74系列芯片，也不是用软件模拟，而是用一行行代码创造出一颗能跑指令、能做运算的“芯”？如果你和我一样，也有这样的梦，那么，欢迎来到FPGA的世界！

距离上次更新博客已经一年有余，生活和工作的节奏总让人身不由己。最近，日渐枯燥的工作让我想在生活中找些挑战，以此为契机开启一个全新的系列。灵感最初来源于中科院计算所的[“一生一芯”](https://ysyx.oscc.cc/)项目，我曾在学生时代的尾巴与它擦肩而过。如今，我对数字电路的兴趣愈发浓厚，我决定以在FPGA上实现一个RISC-V处理器为目标，开启这段“芯”旅程，不仅是对过去的弥补，更是对自己硬件能力的一次系统性升级。

本系列会放慢脚步，从准新手的视角学习分享，一步步向着那颗RISC-V前进。

## 开发板：从硬件开始

工欲善其事，必先利其器。选择一块合适的开发板，是进行FPGA学习的第一步。

过去我做过一些FPGA开发，常用的是Xilinx家的ZYNQ系列。它功能强大，有众多的IP核供直接使用，开发体验如同搭乐高般顺滑。但它有两个缺点令我诟病：一是配套的Vivado软件是个巨无霸，动辄几十上百GB；二是开发板价格不菲，让学生党和爱好者望而却步。

去年底，我发现了嘉立创的[“逻辑派”](https://wiki.lckfb.com/zh-hans/fpga-ljpi/)。它搭载了一颗兆易GD32 MCU和一颗高云GW2A FPGA，是纯正的国产方案。板上附赠一颗2Gb DDR3和一些实用外设，搞活动到手仅138元。这个价格，还要什么自行车？买来当核心板都血赚。

![image-20250818221010124](/home/ioyoi/.config/Typora/typora-user-images/image-20250818221010124.png)

我把它和市面上其他几款高性价比FPGA拉出来对比了一下：

| 型号        | **Gowin GW2A-LV18** | Lattice ECP5 LFE5U-25 | AMD Artix 7 XC7A25T | Intel Cyclone 5CEA4 |
| ----------- | ------------------- | --------------------- | ------------------- | ------------------- |
| LUTs        | 20736 (LUT4)        | 24000 (LUT4)          | 23360 (LUT6)        | 25000 (ALM)         |
| BSRAM       | 828K                | 1008K                 | 1620K               | 1760K               |
| Multipliers | 48 (18x18)          | 28 (18x18)            | 80 (25x18 DSP)      | 50 (18x18)+25(DSP)  |
| PLLs        | 4                   | 2+2 (DLL)             | 3 (MMCM+PLL)        | 4                   |
| Price(RMB)  | 70 (speed C8/I7)    | 70 (speed -7)         | 100 (speed -2)      | 120 (EP4CE22)       |

主角高云GW2A，在逻辑资源上和主打性价比的Lattice打得有来有回，虽然LUT少了点，但慷慨地给了更多的乘法器和PLL。而由于AMD和Intel的LUT更高级（LUT6/ALM），在LUT数量相近的情况下，它们能实现更复杂的逻辑，RAM资源也更丰富。不过，对于我们的学习目标而言，无论选择哪款FPGA，资源都是绰绰有余了。考虑到芯片不到七折、开发半半价不止的价格，这块“逻辑派”绝对是物超所值了。

此外，这块板子的一大特色是同时搭载了MCU和FPGA。我十分推崇这种组合架构，MCU负责处理通信、显示等应用层业务，FPGA负责处理实时性要求高的采集和处理逻辑，这种组合避免了使用FPGA去实现应用层复杂功能的成本，还可以用MCU的外设快速实现一些常用功能，如AES加密、USB、网络通信等等，不过目前市面上这样的开发板仍十分少见。ZYNQ就是这种架构的集大成者，ARM核（PS）和FPGA（PL）集成在同一块芯片上，通过片上高速公路（AXI总线）通信。而在“逻辑派”这块板子上，MCU和FPGA住在两个独立的房子里，只能通过SPI这样的“乡间小路”来沟通，如果能换成诸如FMC的城市大道，体验还会更上一层楼。

## MCU (PS端) 开发环境

我选择用 `CMake + Arm GNU Toolchain + CLion` 这套环境来开发MCU。为什么抛弃祖传的Keil？原因有三：
1.  **版权：**Keil是付费软件，破解软件既不道德也不安全。
2.  **体验：**CLion作为专业的C/C++ IDE，代码补全、重构、调试体验和AI功能，都比Keil领先一个时代。
3.  **跨平台：** 这套工具链在Linux、Windows甚至Mac OS上体验都完全一致。

如果你对这套环境感兴趣，可以参考[我的项目仓库](https://github.com/I0Y0I/Logicpi_PS_starter)。重点关注`CMakeLists.txt`（项目构建的说明书）和`gd32f303cbt6.lds`（链接器脚本，指示代码和数据在RAM和FLASH中的位置）。

> **题外话：** GNU工具链默认的C标准库[Newlib](https://sourceware.org/newlib/)功能齐全，但有点臃肿。GD32这颗MCU的RAM资源比较紧张，所以我后续计划将项目迁移到更轻量级的[Picolibc](https://github.com/picolibc/picolibc)，这是一个专为嵌入式优化的C库。届时，我会专门写一篇文章分享这个瘦身过程。

## FPGA (PL端) 开发环境

传统上，FPGA开发流程被各大厂商严格管控着：用Intel的FPGA就得装Quartus，用AMD的就得装Vivado。高云也提供了自家的Gowin EDA。但说实话，它在Linux高分屏下的体验简直是灾难，UI缩放一塌糊涂，用起来相当糟心。

![image-20250818235421198](/home/ioyoi/.config/Typora/typora-user-images/image-20250818235421198.png)

幸运的是，我在社区里发现了一套**完全开源**的FPGA开发工作流！这套流程由一系列命令行工具组成，覆盖了从代码编写到烧录的全过程：

*   **代码编写:** Nvim + Verible（SystemVerilog的LSP）
*   **仿真验证:** Verilator + cocotb （用Python写测例简直是在节约生命）
*   **综合:** Yosys
*   **布局布线:** nextpnr（目前支持Gowin、Lattice和部分Intel FPGA）
*   **烧录:** OpenFPGALoader（很遗憾，逻辑派自带烧录器不支持这个工具，我购买了sipeed家的[RV debugger plus](https://github.com/sipeed/RV-Debugger-BL702)进行烧录）

这一整套工具链可以用Makefile丝滑地串起来，一句`make`即可实现自动化执行。更棒的是，Yosys开发者贴心地打包了[oss-cad-suite](https://github.com/YosysHQ/oss-cad-suite-build)，可以一次性安装上述所有工具。

当然，这套开源方案并非完美，目前还缺少一些官方IDE才能实现的专属功能，比如图形化的引脚分配、生成IP核、片上逻辑分析仪等。我正在逐步解决这些不足之处，也会和大家分享我在这一过程中发现或创造的新工具。

在下一篇文章中，我会详细介绍这些工具都是如何使用的。

## 结语

过去我挖过不少坑，但这次，我希望把它填成地基，铸成高楼。

慢一点没关系，重要的是一直在路上。我们的“芯”辰大海，才刚刚起航。
