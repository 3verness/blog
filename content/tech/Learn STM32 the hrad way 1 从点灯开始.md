---
title: "Learn STM32 the Hrad Way·1 从点灯开始"
slug: "learn-stm32-1"
subtitle: "基于GNU GCC、OpenOCD开源工具链的STM32入门"
defcription: ""
tags:
    - "stm32"
    - "tutorial"
date: 2024-01-09T21:00:50+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

## What's the hard way?

网上已经有许多非常成熟的基于Keil的STM32的开发教程，而本系列文章反其道而行之，选择使用GNU GCC + OpenOCD的开源工具链进行STM32的入门；本系列教程计划以江协科技的[STM32开发套件](https://item.taobao.com/item.htm?id=655451342180)为基础，依次进行STM32标准库开发与HAL库开发的学习，由于作者也是刚刚接触STM32的萌新，文章可能出现诸多纰漏，欢迎读者批评指正。

{{< note primary >}}

为什么不使用Keil？

1. Keil是昂贵的商业软件，虽然大部分教程都提供了破解版使用，但这并非长久之策；
2. Keil仅支持Windows平台，无法在MacOS、Linux上进行跨平台开发；
3. Keil对与C的新标准支持较慢，如果你主要使用C++，那么GNU GCC是一个更好的选择；
4. 使用开源工具链进行开发，可以对STM32的编译、链接、烧写过程有更加深入的认识；
5. 开源工具链可以和诸多通用工具链进行联动，灵活性更高，便于后续的CI/CD。

Anyway，工具只是工具，当我们了解其原理后，可以很快的从一种工具链迁移至另一种。

{{< /note >}}

{{< note info >}}

如果你想一步步跟随本文的脚步去学习，强烈建议你购买与作者完全相同的套件，否则可能会在第一步就会遇到诸如OpenOCD无法连接、链接文件不合适等bug。

{{< /note >}}

## Let's get prepared!

让我们从配置开发环境开始吧，需要安装的工具全部可以从官方网站**免费**下载：

* [GNU Arm Embedded Toolchain](https://developer.arm.com/downloads/-/gnu-rm)

  Arm编译工具，可以通过scoop进行安装管理：

  ```bash
  scoop install gcc-arm-none-eabi
  ```

* [OpenOCD](https://openocd.org/)

  调试器，也可以使用类似的[pyOCD](https://pyocd.io/)等调试工具，但OpenOCD的资料教程最多，同样可以通过scoop安装：

  ```bash
  scoop install openocd
  ```

* [VS Code](https://code.visualstudio.com/)

  理论上你可以使用任意编辑器，比如Eclipse和Clion，它们都通过插件提供了很好的嵌入式开发支持。

* [Embedded IDE](https://em-ide.com/zh-cn/)

  VS Code中的嵌入式开发插件，功能十分强大，在插件中搜索`eide`安装即可；

* [Cortex-Debug](https://marketplace.visualstudio.com/items?itemName=marus25.cortex-debug)

  VS Code中的Cortex-M调试插件，在插件中搜索`cortex-debug`安装；

* [C/C++ Extension Pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools-extension-pack)

  VS Code中的C语言支持，进行语法检查和输入提示，在插件中搜索`c`安装。

完成软件安装后，需要自行设置Embedded-IDE与Cortex-Debug中的GCC路径和OpenOCD路径。

此外，为了完成库函数与HAL库的学习，还需要准备：

* [STM32Cube](https://www.st.com/en/development-tools/stm32cubemx.html)

  STM32官方提供的代码生成器工具，常用于HAL库开发，并可以帮助我们生成一些GCC需要使用的配置文件；

* [STM32标准库](https://www.st.com/en/embedded-software/stm32-standard-peripheral-libraries.html)

  提供库函数的头文件与实现，对与我们的套件，应当下载[STM32F10x standard peripheral library](https://www.st.com/en/embedded-software/stsw-stm32054.html)；

* [ST-Link驱动](https://www.st.com/en/development-tools/stsw-link009.html)

  调试器驱动，如果你购买了不一样的套件，可能会搭配DAQ调试器，需要自行查找驱动安装。

## 创建第一个工程

打开VS Code左侧Embedded-IDE标签页，选择New Project->Empty Project->Cortex-M Project，输入项目名称、保存位置后右下角弹出跳转到项目选项，点击Continue进入项目。

首先我们需要从下载的标准库拷贝一些必需的头文件，按照江协教程，我们可以在项目中新建一个`Start`文件夹存放这些文件，以下文件均位于`Libraries\CMSIS\CM3`中：

* `CoreSupport\core_cm3.h/c`

  支持Cortex-M3处理器核的CMSIS库文件，包含了一些与Cortex-M3处理器核相关的核心函数和宏定义；

* `DeviceSupport\ST\STM32F10x\stm32f10x.h`

  用于STM32F10x系列微控制器的头文件，包含了与STM32F10x相关的寄存器定义、位操作宏和其他硬件相关信息；

* `DeviceSupport\ST\STM32F10x\system_stm32f10x.h/c`

  用于STM32F10x系列微控制器的系统初始化配置文件，主要包含了系统时钟初始化和启动代码，用于设置系统时钟、中断向量表以及一些基本的系统配置；

* `DeviceSupport\ST\STM32F10x\startup\gcc_ride7\startup_stm32f10x_md.s`

  微控制器的启动文件，这是一个汇编文件，其主要目的是在微控制器上电时初始化系统并将控制权传递给用户的C语言程序。这个启动文件是连接到用户程序的二进制文件中的，确保在运行时系统能够正确初始化，并且用户程序可以正常执行。

  {{< note primary >}}

  可以注意到startup下存在着多个文件夹，分别对应着多种编译工具，每个文件夹下又提供过了多个`strtup_stm32f10x_xx.s`文件，具体拷贝哪一个需要跟据我们的微控制器芯片选择，可以参考`stm32f10x.h`中的说明进行选择：

  ![image-20240109171828990](https://img.ioyoi.me/20240109171831.webp)

  套件中的STM32F103C8T6为64Kbytes的STM32F103xx为控制器，应选择`STM32F10X_MD`，同时还要在预处理器加入该定义，这个操作将在下文中讲到。

  {{< /note >}}

此外还需要一个链接脚本文件，用于指示链接器在构建可执行文件时如何组织代码和数据，以及如何分配它们到微控制器的存储器空间，该文件有两种获取方式，一种是直接在网上查找其他人使用的对应控制器的[链接脚本](https://github.com/lucysrausch/stm32-turbotronik/blob/master/STM32F103C8Tx_FLASH.ld)，第二种方法是通过STM32CubeMX生成一个对应微控制器的空工程文件，Toolchain/IDE选择 Makefile，生成代码后，根目录下的`STM32F103C8Tx_FLASH.ld`文件即为所需链接脚本，可以将该脚本直接放置在项目根目录下。

按照嵌入式开发习惯创建`User`文件夹并在其内新建`main.c`文件编写主程序，此时文件树应如下图所示：

![image-20240109191256024](https://img.ioyoi.me/20240109191257.webp)

接下来，在Embedded IDE页配置项目：

* 在Project Resources添加Normal Folder，将`Start`和`User`添加至源文件；（注意，此后如果其他文件夹中编写了源代码，也需要将该文件夹添加进来，否则将不会编译该文件。）

  ![image-20240109190115714](https://img.ioyoi.me/20240109190117.webp)

* 在Chip Support Package中添加From Repo->Keil.STM32F1xx_DFP.pack，待支持包下载完成后展开该选项，在STM32F1xx_DFP右侧choose device选择STM32F103C8；

  ![image-20240109190256424](https://img.ioyoi.me/20240109190258.webp)

* Builder Configurations选择GNU Arm Embedded Toolchain(GCC)，CPU Type选择Cortex-M3，Linker Script File Path选择根目录下的链接脚本文件；

  ![image-20240109191434770](https://img.ioyoi.me/20240109191436.webp)

* Flasher Configurations选择OpenOCD，Chip Config选择stm32f1x.cfg，Interface Config选择stlink.cfg；

  ![image-20240109191657288](https://img.ioyoi.me/20240109191658.webp)

  {{< note warning >}}

  若你使用其它类型的烧写器，可能需要设置其他的Interface Config，可以通过连接烧写器与开发板后，通过如下命令检查OpenOCD配置是否正确：

  ```powershell
  openocd -f interface/stlink.cfg -f target/stm32f1x.cfg
  ```

  若输出如图所示，则可以使用上述配置，否则需要尝试其它interface选项。

  ![image-20240109191957164](https://img.ioyoi.me/20240109192001.webp)

  {{< /note >}}

* 在Project Attributes的Include Directories中添加`Start`和`User`目录，同样的，如果你之后想要include其他文件夹中的头文件，也需要将它添加至Include Directories，在Preprocessor Definitions中加入上文提到的`STM32F10X_MD`。

  ![image-20240109193004437](https://img.ioyoi.me/20240109193005.webp)

配置完成，现在我们来编写一个最简单的主程序来试试编译吧：

```c
#include "stm32f10x.h"

int main(void)
{
    while(1);
}
```

{{< note primary >}}

嵌入式开发时需要使用死循环阻止主程序结束，防止嵌入式系统进入未定义的状态。

{{< /note >}}

点击Build编译，发现Terminal中报错：

```
>> [ 25%] CC 'Start/core_cm3.c'
C:\Users\3verness\AppData\Local\Temp\ccSEfmyW.s: Assembler messages:
C:\Users\3verness\AppData\Local\Temp\ccSEfmyW.s:599: Error: registers may not be the same -- `strexb r0,r0,[r1]'
C:\Users\3verness\AppData\Local\Temp\ccSEfmyW.s:629: Error: registers may not be the same -- `strexh r0,r0,[r1]'
```

这是一个[年代久远的错误](https://github.com/stlink-org/stlink/issues/65)，[解决方法](https://gist.github.com/timbrom/1942280)为将`core_cm3.c`末尾处的该语句修改为（将`"=r"(result)`改为`"=&r"(result)`）：

```c
/**
 * @brief  STR Exclusive (8 bit)
 *
 * @param  value  value to store
 * @param  *addr  address pointer
 * @return        successful / failed
 *
 * Exclusive STR command for 8 bit values
 */
uint32_t __STREXB(uint8_t value, uint8_t *addr)
{
   uint32_t result=0;
  
  //  __ASM volatile ("strexb %0, %2, [%1]" : "=r" (result) : "r" (addr), "r" (value) );
   __ASM volatile ("strexb %0, %2, [%1]" : "=&r" (result) : "r" (addr), "r" (value) );
   return(result);
}

/**
 * @brief  STR Exclusive (16 bit)
 *
 * @param  value  value to store
 * @param  *addr  address pointer
 * @return        successful / failed
 *
 * Exclusive STR command for 16 bit values
 */
uint32_t __STREXH(uint16_t value, uint16_t *addr)
{
   uint32_t result=0;
  
  //  __ASM volatile ("strexh %0, %2, [%1]" : "=r" (result) : "r" (addr), "r" (value) );
   __ASM volatile ("strexh %0, %2, [%1]" : "=&r" (result) : "r" (addr), "r" (value) );
   return(result);
}
```

再次编译，在报了一堆warning后显示`[ DONE ] build successfully !`，恭喜你，你的第一个工程就打包完成了~我们以后会再来处理这些warning的。

## 点亮第一颗LED

学硬件的第一步永远都是点灯，无需连接其他模块，江协的STM32核心板上便留有两颗LED，D1是电源指示灯，上电常亮，D2一端连接VCC，一段接PC13，即GPIO C的13引脚，当PC13拉高，LED熄灭，PC13拉低，LED点亮，让我们就从这颗D2开始吧！

![image-20240109195015932](https://img.ioyoi.me/20240109195017.webp)

在STM32中，通过寄存器控制某一特定GPIO端口需要三个步骤：

1. 开启对应IO端口时钟使能，其中GPIOC连接在APB2总线上，通过`RCC_APB2ENR`寄存器设置，即需要将该寄存器第4位置1；

   ![image-20240109200542384](https://img.ioyoi.me/20240109200544.webp)

   ```
   Bit 4 IOPCEN: IO port C clock enable
   Set and cleared by software.
   0: IO port C clock disabled
   1: IO port C clock enabled
   ```

2. 设置对应IO端口为推挽输出，GPIOC 13Pin的工作模式由`GPIOC_CRH`寄存器设置，即需要将该寄存器[23:22]设置为00，[21:20]设置为11（50MHz工作）；

   ![image-20240109201403802](https://img.ioyoi.me/20240109201405.webp)

   ```
   CNFy[1:0]: Port x configuration bits (y= 8 .. 15)
   These bits are written by software to configure the corresponding I/O port.
   In input mode (MODE[1:0]=00):
   00: Analog mode
   01: Floating input (reset state)
   10: Input with pull-up / pull-down
   11: Reserved
   In output mode (MODE[1:0] > 00):
   00: General purpose output push-pull
   01: General purpose output Open-drain
   10: Alternate function output Push-pull
   11: Alternate function output Open-drain
   
   MODEy[1:0]: Port x mode bits (y= 8 .. 15)
   These bits are written by software to configure the corresponding I/O port.
   00: Input mode (reset state)
   01: Output mode, max speed 10 MHz.
   10: Output mode, max speed 2 MHz.
   11: Output mode, max speed 50 MHz
   ```

3. 改变指定端口输出寄存器的值，即改变`GPIOC_ODR`中的13位的值。

   ![image-20240109203101270](https://img.ioyoi.me/20240109203102.webp)

   ```
   ODRy: Port output data (y= 0 .. 15)
   These bits can be read and written by software and can be accessed in Word mode only
   ```

   {{< note info >}}

   以上是江协科技视频中给出的置位与复位方式，但个人认为这是一种不太好的方法，因为该方法同时改变了所有IO输出寄存器的值，更好的方式应当是操作`GPIOC_BSRR`寄存器，置位时，向13位写1，复位时，向29位写1。

   ![image-20240109203034103](https://img.ioyoi.me/20240109203035.webp)

   ```
   BRy: Port x Reset bit y (y= 0 .. 15)
   These bits are write-only and can be accessed in Word mode only.
   0: No action on the corresponding ODRx bit
   1: Reset the corresponding ODRx bit
   
   BSy: Port x Set bit y (y= 0 .. 15)
   These bits are write-only and can be accessed in Word mode only.
   0: No action on the corresponding ODRx bit
   1: Set the corresponding ODRx bit
   ```

   {{< /note >}}

因此，`main`函数应当这样编写：

```c
#include "stm32f10x.h"

int main(void)
{
    RCC->APB2ENR = 0x00000010;
    GPIOC->CRH = 0x00300000;
    GPIOC->ODR = 0x00000000; //开灯
    // GPIOC->ODR = 0x00002000; //关灯
    while(1);
}
```

点击编译、烧写，若OpenOCD配置成功，Terminal中会出现：

```
** Programming Started **
Info : device id = 0x20036410
Info : flash size = 64 KiB
Warn : Adding extra erase range, 0x08000338 .. 0x080003ff
** Programming Finished **
** Verify Started **
** Verified OK **
```

此时便可以通过更换`main`函数，反复刷写控制灯光亮灭了。一个简单的亮灯程序就这样完成了，虽然代码并不长，但可读性很差，相信对于初学者来说，如果没有注释，必须查阅参考手册才能知道每个语句的具体含义，这也是基于寄存器从底层开发的劣势所在，在后文中，我们将学习如何使用库函数实现相同的功能。

## 浅尝调试

最后，让我们来简单尝试一下Embedded IDE集成的调试功能，打开`.vscode`目录下的`launch.json`，删除多余项，仅保留使用OpenOCD的配置项：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "cwd": "${workspaceRoot}",
            "type": "cortex-debug",
            "request": "launch",
            "name": "openocd",
            "servertype": "openocd",
            "executable": "build\\Debug\\Template.elf",
            "runToEntryPoint": "main",
            "configFiles": [
                "interface/stlink.cfg",
                "target/stm32f1x.cfg"
            ],
            "toolchainPrefix": "arm-none-eabi",
            "svdFile": ".pack/Keil/STM32F1xx_DFP.2.3.0/SVD/STM32F103xx.svd"
        }
    ]
}
```

选择左侧的Debug标签栏，确定当前Debug名称为`openocd`，按Start Debugging开始，稍等一会后，程序暂停在主程序的开始处，此时可以单步运行，并在左侧XPERIPHERALS栏查看微控制器中各寄存器的值，此时还可以使用gdb的全部调试功能。

![image-20240109204945144](https://img.ioyoi.me/20240109204947.webp)

值得注意的是，Debug只会利用编译中生成的`elf`文件进行调试，并不会自动重新编译文件，因此每次修改源文件后，都需要手动Rebuild才可以进行调试。

## 认识标准库

上两个章节中，我们使用寄存器开发了一个简单的亮灯程序，现在我们将使用标准库重新实现该功能。

若想使用标准库，首先我们需要将标准库中`Libraries\STM32F10x_StdPeriph_Driver`中`inc`文件夹下的库头文件与`src`下的源文件添加至项目中，我们新建一个`Library`文件夹存放这些标准库文件，同样别忘了将`Library`文件夹添加至Embedded IDE的Project Resources和Include Directories。

此外，我们还需要额外的三个文件对这些头文件进行导入以及中断定义，均可以在标准库下的`Project\STM32F10x_StdPeriph_Template`内找到：

* `stm32f10x_conf.h`

  用于包含标准库头文件，并提供标准库函数参数检查函数定义；

* `stm32f10x_it.h/c`

  定义标准库中的中断函数。

由于上述文件可能需要更改，可以将它们放置在`User`目录下，此时项目文件树结构可参照[该模板工程]()。

最后留意到`stm32f10x.h`结尾处语句：

```c
#ifdef USE_STDPERIPH_DRIVER
  #include "stm32f10x_conf.h"
#endif
```

因此为导入标准库，需要在预处理器中定义`USE_STDPERIPH_DRIVER`，即在Preprocessor Definations中添加该项。

在`main.c`中输入`GPIO`会发现出现一系列自动补全的库函数，说明标准库已经成功导入。

让我们用标准库重写该功能，寄存器配置语句注释在对应的库函数语句下方，这里我建议去看看江协科技[2-2课时](https://www.bilibili.com/video/BV1th411z7sn?t=1411.4&p=4)该部分的库函数编写流程，包含了查阅注释、寻找可选参数的编程过程，值得一提的是，由于VS Code可以读取格式化注释，将鼠标移至函数名称处即可显示可选参数，比Keil要方便不少：

```c
#include "stm32f10x.h"

int main(void)
{
    // 打开外设时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
    // RCC->APB2ENR = 0x00000010;

    // 设置端口推挽输出
    GPIO_InitTypeDef GPIO_InitStructure;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOC, &GPIO_InitStructure);
    // GPIOC->CRH = 0x00300000;

    // 复位
    GPIO_ResetBits(GPIOC, GPIO_Pin_13);
    // GPIOC->ODR = 0x00000000;

    // 置位
    GPIO_SetBits(GPIOC, GPIO_Pin_13);
    // GPIOC->ODR = 0x00002000; //关灯

    while(1);
}
```

可以看到，库函数的代码比寄存器配置要繁琐的多，那么使用库函数的优势何在呢？让我们回顾一下寄存器开发的流程，当我们想实现一个设置时，首先要寻找控制该设置的寄存器，在参考手册中查找该寄存器对应的名称及各位含义，按照含义配置寄存器的零一值；而库函数开发则变为，查找设置该项的函数，查看该函数各参数的可选项，选择合适的可选项填入，其中大部分查找功能可以通过编辑器完成，同时函数名也具有更高的可读性，这就极大的提升了开发效率，以下两张GIF可以直观地反应两种方式开发的效率对比。

![1](https://img.ioyoi.me/20240110121901.webp "寄存器开发")

![1](https://img.ioyoi.me/20240110121902.webp "库函数开发")

此外，我们观察库函数的具体实现：

```c
void RCC_APB2PeriphClockCmd(uint32_t RCC_APB2Periph, FunctionalState NewState)
{
  /* Check the parameters */
  assert_param(IS_RCC_APB2_PERIPH(RCC_APB2Periph));
  assert_param(IS_FUNCTIONAL_STATE(NewState));
  if (NewState != DISABLE)
  {
    RCC->APB2ENR |= RCC_APB2Periph;
  }
  else
  {
    RCC->APB2ENR &= ~RCC_APB2Periph;
  }
}
```

发现库函数会逐一对参数进行参数检查，并通过`|=`与`&=`对寄存器单个位进行置位和复位，最大程度的避免了我们的粗心犯错，同时也不会对其他无关项造成影响，相较使用寄存器更为安全省心。

## 打包模板

由于我们以后会经常创建新工程，每次都重新复制文件过于麻烦，这里推荐将该工程打包成模板(`.ept`)，便于日后直接创建基于库函数的Embedded IDE工程。在Embedded IDE页右键该工程选择Export Eide Project Template，该操作会在项目根目录下生成一个`.ept`文件，这便是我们的模板文件，此后新建工程时，只需要选择New Project->Local Template，选择该模板文件即可生成一个除名称位置外与当前工程完全一样的新工程，无需再做任何配置！

![image-20240110123152356](https://img.ioyoi.me/20240110123154.webp)

恭喜你，你已经踏出了学习STM32的第一步，相较Keil，开源工具链的配置过程确实要繁琐的多，但你已经成功克服了大多数阻碍，现在，你便可以在一个现代化的UI界面里愉快的编写嵌入式代码了~在下一篇文章中，我们将从GPIO开始，一步步认识STM32的外设操作，一起向点灯大师进发吧！
