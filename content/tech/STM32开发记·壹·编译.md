---
title: "STM32开发记·壹·编译"
slug: "stm32-dev-1"
subtitle: ""
defcription: ""
tags:
    - "justwrite"
    - "stm32"
    - "vscode"
date: 2022-07-25T10:01:24+08:00
draft: true
author: EvernessW


toc: false
katex: false
mermaid: false
---

{{<justwrite 5>}}

外出实习，需要使用STM32开发板，开此系列文章记录萌新摸索的过程。

## 工具准备

网上的STM32教程基本都是用KEIL5开发的，但是对于我来说，我并不太需要KEIL提供的一些高级功能，能够提供代码补全和调试查看寄存器功能就足够了，因此本文使用了VS Code+Make的工具链进行开发。需要注意的是，软件仿真是KEIL的专有功能，需要仿真时可以使用STM32提供的专用开发软件STM32CubeMX进行快速工具链转换。

以下是我使用的一些软件：

* [STM32CubeMX](https://www.st.com/zh/development-tools/stm32cubemx.html)：STM32官方提供的工程生成器，用于生成包含驱动库的工程项目；
* [GNU Arm Embedded Toolchain](https://community.chocolatey.org/packages/gcc-arm-embedded)：ARM平台跨平台编译工具包；
* [GNU make](https://community.chocolatey.org/packages/make)：Windows下的make工具；
* [OpenOCD](https://community.chocolatey.org/packages/openocd)：片上调试工具；
* VS Code
  * C/C++：C语言支持，相关snippet包也可以自行安装；
  * Arm Assembly：Arm语法支持；
  * Cortex-Debug：调试工具；

除STM32CubeMX外，其余工具均有choco包：

```bash
choco install gcc-arm-embedded make openocd
```

## 工程生成

首先打开STM32CubeMX软件，`File-New Project`新建工程，搜索并选择所使用的芯片组，点击`Start Project`开始配置。

![](https://img.ioyoi.me/202207251107812.webp)

在弹出的功能界面中分配引脚并配置时钟，随后在`Project Manager`中设置工程路径，在`Toolchain/IDE`一栏中选择`MakeFile`。如要使用Keil 5作为开发工具，则需要选择`MDK-ARM`。开发过程中，我们可以随时更改该选项重新生成编译相关代码，用户输入的功能代码不会受到影响。

![image-20220725111305431](https://img.ioyoi.me/202207251113767.webp)

`Code Generator`选项卡中可设置生成规则，通常选择仅拷贝所需库减小工程体积，并生成独立的初始化代码文件。`Advanced Setting`选项卡中可选择HAL或LL驱动库。

![image-20220725111944808](https://img.ioyoi.me/202207251119823.webp)

设置完成后，点击右上角`GENERTE CODE`生成工程文件。

## VS Code配置

使用VS Code打开目录，主要文件结构如下：

```
PROJECT
├─Core
│  ├─Inc
│  └─Src
├─Drivers
│   ├─CMSIS
│   │  ├─Device
│   │  │  └─ST
│   │  │      └─STM32F1xx
│   │  │          └─Include
│   │  └─Include
│   └─STM32F1xx_HAL_Driver
│       ├─Inc
│       │  └─Legacy
│       └─Src
└─Makefile
```

其中，用户代码保存在`Core`文件夹中，程序入口为`Core\Src\main.c`，此时在终端中键入`make`命令，已经可以在`build`目录下生成`.hex`二进制文件与`.elf`可执行文件。

不过使用VS Code打开源文件时，会发现大量include报错，这是由于C/C++插件并未索引到头文件。

![image-20220725113302729](https://img.ioyoi.me/202207251133957.webp)

为解决该问题，需要参考`Makefile`文件对C/C++插件进行配置。在`Makefile`中，我们重点关注`C_DEFS`与`C_INCLUDES`两项参数。

```makefile
# C defines
C_DEFS =  \
-DUSE_HAL_DRIVER \
-DSTM32F103xE

# C includes
C_INCLUDES =  \
-ICore/Inc \
-IDrivers/STM32F1xx_HAL_Driver/Inc \
-IDrivers/STM32F1xx_HAL_Driver/Inc/Legacy \
-IDrivers/CMSIS/Device/ST/STM32F1xx/Include \
-IDrivers/CMSIS/Include
```

在VS Code中找到`C/C++ Edit Configuration (JSON)`命令，生成`c_cpp_properties.json`文件，键入以下配置：

![image-20220725113735382](https://img.ioyoi.me/202207251137870.webp)

```json
{
    "configurations": [
        {
            "name": "arm",
            "includePath": [
                "${workspaceFolder}/Core/Inc",
                "${workspaceFolder}/Drivers/STM32F1xx_HAL_Driver/Inc",
                "${workspaceFolder}/Drivers/STM32F1xx_HAL_Driver/Inc/Legacy",
                "${workspaceFolder}/Drivers/CMSIS/Device/ST/STM32F1xx/Include",
                "${workspaceFolder}/Drivers/CMSIS/Include"
            ],
            "defines": [
                "USE_HAL_DRIVER",
                "STM32F103xE"
            ],
            "compilerPath": "C:/ProgramData/chocolatey/lib/gcc-arm-embedded/tools/gcc-arm-none-eabi-10-2020-q4-major/bin/arm-none-eabi-gcc.exe",
            "intelliSenseMode": "windows-gcc-arm",
            "browse": {
                "limitSymbolsToIncludedHeaders": true,
                "databaseFilename": "",
                "path": [
                    "${workspaceFolder}"
                ]
            }
        }
    ],
    "version": 4
}
```

其中，`name`项可以自定义配置名称，`includePath`与`defines`分别对应`Makefile`中的`C_DEFS`与`C_INCLUDES`，`compilerPath`为`arm-none-eabi-gcc.exe`所在的路径，`intelliSenseMode`需要选择`gcc-arm`或`windows-gcc-arm`。

保存配置文件后，返回源文件，发现红色波浪线已经消失，头文件可以被正常索引。

## 编译

上文已经提到，可以使用`make`命令直接进行编译，为方便调试工具进行编译调试，可在VS Code中创建对应任务，对应`tasks.json`如下：

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "type": "shell",
            "command": "make",
            "args": []
        },
        {
            "label": "clean",
            "type": "shell",
            "command": "make",
            "args": [
                "clean"
            ]
        }
    ]
}
```

