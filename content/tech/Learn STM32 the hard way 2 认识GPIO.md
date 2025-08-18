---
title: "Learn STM32 the Hard Way·2 认识GPIO"
slug: "learn-stm32-2"
subtitle: ""
defcription: ""
tags:
    - "stm32"
    - "tutorial"
date: 2024-01-15T02:31:06+08:00
draft: true
author: EvernessW


toc: false
mathjax: true
mermaid: false
---

## 重新认识STM32

成功点亮第一颗LED灯后，我们可以重新来认识一下STM32，并问问自己，当学习STM32时，我们究竟要学些什么。

首先，STM32是一类Micro Controller Unit，他不仅集成了处理器、内存、闪存，还集成了诸多外设，包括基本GPIO、定时器TIM、USART串口、ADC模数转换器等等等等，这些外设通过总线与Arm核通信，接受处理器的控制。作为处理器运算时，STM32开发和传统C开发区别不大，我们需要特别学习的便是如何控制这些外设。

![image-20240111172906896](https://img.ioyoi.me/20240111172908.webp "stm32系统结构")

STM32的外设多种多样，不过别担心，这张图只是让这些外设露一下脸，之后我们会逐一学习它们，我们现在只需要注意，外设究竟连接在哪条总线上，例如我们曾用到的GPIO均连接于APB2总线，而I2C接口则连接于APB1总线，APB2总线时钟频率高于APB1总线。

## STM32最小系统

STM32最小系统是指让STM32芯片正常工作的必需电路结构，我们手中的开发板便是一个STM32最小系统，它通常包括电源、复位、时钟、启动与调试接口电路。

### 电源

STM32采用3.3V供电，一个STM32芯片中可能存在多个VCC引脚与GND引脚，在绘制电路图时每一个引脚都需要连接至电源或地，通常STM32的最大功耗在几十毫安量级，常见的供电形式有：

* 3.3V电池供电；
* USB供电（USB中电源为5V，需要额外的5V转3.3V芯片转换）；
* SWD接口供电（SWD电源本身为3.3V，无需转换）。

为了提高ADC、DAC相关模拟电路的性能，该部分电路通过VDDA与VSSA两个引脚单独供电，可以通过磁珠或额外滤波电路将该部分供电与数字电源隔离开。

### 复位

STM32的NRST引脚输入低电平时，STM32将进行硬件复位，复位电路通常由一个上拉电阻、一个大电容与一个开关组成，其包含有两套复位逻辑，其一是在上电时，由于电容需要充电，RESET处电压由低电平缓慢上升，最后到达高电平，这一过程无需干预，自动进行，称为上电复位；而当开关按下时，RESET电压被手动短路至低电平，这一过程称为硬件复位。

<img src="https://img.ioyoi.me/20240114125906.webp" alt="image-20240114125904562" style="zoom:33%;" />

### 时钟

STM32所需的外部时钟既可以是有源时钟，也可以使用晶振提供，尽管有源时钟更为稳定，但出于成本和功耗的考虑，一般采用无源晶振的方式提供外部时钟，此时需要为晶振搭配合适的启动电容与反馈电容。STM32通常需要两个外部时钟：

* HSE时钟：一个4~16MHz高速时钟，通过PLL倍频后提供系统时钟（SYSCLK）；
* LSE时钟，一个32.768kHz时钟，经过15次分频后可以得到1Hz计时脉冲，提供RTC时钟作为准确的时间基准。

<img src="https://img.ioyoi.me/20240114132725.webp" alt="image-20240114132722926" style="zoom: 50%;" />

### 启动

启动电路通过BOOT0与BOOT1两个引脚配置了STM32的启动模式：

* BOOT0为0时，STM32从Flash存储器的起始地址处启动，这种启动模式最为常见；
* BOOT0为1，BOOT1为0时，STM32从系统内存（也是Flash，但起始地址不同，类似于Bootloader）启动，通常在Flash内引导程序出现问题时，通过串口或其他接口加载新的引导程序；
* BOOT0为1，BOOT1为1时，从SRAM启动，由于SRAM断电后不再记录信息，因此该模式通常只适用于开发时的快速调试。

### 调试接口

STM32提供SWD和JTAG两种调试接口，其中SWD仅使用SWDCLK与SWDIO两个引脚，连接更为方便，而JTAG占用5个引脚，但在烧写调试外还提供了生产测试和配置的高级功能，对于简单开发而言，SWD接口已经十分够用了。

## 复位与时钟树

### 复位

STM32系统中存在三种复位：

* System reset：重置除时钟控制和备份区意外的所有寄存器，最常见的复位形式，上文中提到的复位电路出发的就是系统复位；
* Power reset：重置除备份区外所有寄存器，比System reset更为彻底，在重新上电或在退出standby低功耗模式时触发；
* Backup domain reset：仅针对与备份区的复位。

这里我们只需要简单了解系统复位触发的几个原因，其余内容将在我们使用到时再作介绍：

* NRST引脚被拉低，即按下了复位电路中的开关按钮；
* 软件触发，比如调试时的“重新运行”，就是通过调试接口向Arm核发出了系统复位指令；
* 看门狗触发；
* 低功耗模式触发，如进入了Standby或Stop模式。

### 时钟树

时钟相当于STM32的心跳，寄存器的置位复位、时序的控制均需要通过时钟触发，接下来我们以开发板上的STM32C8T6为例来认识STM32的时钟树，打开STM32CubeMX中的Clock Configuration页，结合此前的STM32系统结构图，我们可以直观的看到时钟树的各项配置以及各个模块使用的时钟。

![image-20240114143811989](https://img.ioyoi.me/20240114143813.webp "STM32时钟树")

可以注意到，除去我们在最小系统中提供的HSE（8MHz）与SLE（32.768kHz）两个外部时钟外，STM32还内嵌HSI（8MHz）与LSI（40kHz）两个内部时钟，可以通过图中的几个选择器为SYSCLK、RTC Clock选择合适的时钟源，这些配置本质上是向CLK相关寄存器种写入不同值来实现的，之后我们会更加深入的讲解如何通过库函数来配置时钟项。

从时钟树中我们注意到：

* 总线时钟分为外设时钟和定时器时钟，在总线连接的外设中，TIMx的时钟由Timer clocks提供，其余外设时钟均由peripheral clocks提供，同时ADC的时钟可以通过单独的ADC prescaler进行更为灵活的配置；
* 当使用USB功能时，USB所需时钟频率为48MHz，而输入HSE的最大频率为16MHz，必须开启PLL倍频以满足USB的时钟需求；
* 用于操作Flash存储器的FLITFCLK时钟只能由HSI控制，频率固定为8MHz；
* 独立看门狗IWDG的时钟只能由LSI提供，而窗口看门狗WWDG挂载在APB1总线上，由APB1 peripheral clocks提供时钟；
* 部分时钟的最高频率存在限制，例如APB1总线最高频率为36MHz，而APB2总线则为72MHz，在配置时需要考虑到这些限制，否则会出现时钟错误，这里推荐采用STM32CubeMX检查时钟配置的正确性。

在使用外设时，第一步始终是开启该外设的时钟，库函数中的相关函数为：

```c
RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);
RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1, ENABLE);
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
```

这三个函数分别控制了直连与AHB总线、连接于APB1总线、连接与APB2总线的外设时钟。

## 认识GPIO

{{< note info >}}

从这里开始，本系列教程开始体现出较大的差异化，这是因为作者的主要学习方式并不是进行实验，而是通过阅读[reference manual](https://www.st.com/resource/en/reference_manual/rm0008-stm32f101xx-stm32f102xx-stm32f103xx-stm32f105xx-and-stm32f107xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)了解各个外设的基本原理、对应寄存器配置，并阅读库函数代码掌握如何调用接口配置这些外设，这是题目中“the hard way”最为重要的一步，官方提供的参考手册其实就是学习STM32的最好教材。

同时，本教程也会参考其他以实验为主的教程，适当插入一些必需的辅助知识，帮助你独立的实现对应环节的实验，对于其他教程出现的实验现象，你都可以先尝试自己实现，再参考教程提供的范例代码进行学习。

{{< /note >}}

GPIO是微控制器对外交流的通道，几乎STM32的所有功能都需要通过GPIO实现，STM32上的GPIO提供4大类共计8种工作模式：

* 数字输入（浮动、上拉、下拉）
* 模拟输入
* 数字输出（开漏、推挽）
* 复用功能输出（开漏、推挽）

### pull-up and pull-down

上拉电阻和下拉电阻在数字电路中非常常见，它们用于偏置引脚的输入，以防止它们在无输入时电平随机浮动。在无输入时（Switch断开情况下），加入上拉电阻，引脚会保持高电平，加入下拉电阻，因教会保持低电平，两者都不加，引脚处于浮动状态，输入电平的高低会随环境变化，无法预测。

![上拉电阻与下拉电阻](https://img.ioyoi.me/20240111152052.webp)

那么如何选择上下拉电阻的阻值呢？主要有两个考虑因素，一是开关断开时引脚的输入电压、二是开关闭合后电阻上的漏电流；上拉电阻越大，流入引脚的电流在电阻上形成的压降越大，会导致开关断开时引脚电压降低，偏离电源高电平；而上拉电阻学校，开关闭合时由高电平流向低电平的漏电流越大，会带来不必要的功耗和发热。经验公式为，其中 $V_{high(min)}$ 为高电平的最小电压，$V_{low(max)}$为低电平的最大电压，$I_{sink}$为最大输入电流：


$$
R_{pull-up}=\frac{V_{DD}-V_{high(min)}}{I_{sink}}
$$

$$
R_{pull-down}=\frac{V_{low(max)}-V_{GND}}{I_{sink}}
$$



### push-pull and open-drain

推挽输出和开漏输出是数字电路中的两种驱动形式。推挽输出的名字十分形象，当输出高电平时，Internal Signal为低电平，PMOS开启，NMOS关断，电流由VDD“推”向输出端口；而输出低电平时，Internal Signal为高电平，PMOS关断，NMOS开启，电流被“拉”回地；无论输出高低电平，输出引脚均有电流通过，信号上升下降速度快。

![Push-pull output schematic using two transistors (PMOS connected to VDD and NMOS connected to GND)](https://img.ioyoi.me/20240111155250.webp)

而开漏输出只使用一个NMOS实现，只能输出低电平，而当Internal Signal为低电平时，NMOS关断，输出引脚处于高阻态，因此若想驱动高电平，往往需要配合外部的上拉电阻使用，在这种情况下，输出信号上升时会受到上拉电阻的限流，上升沿斜率受到影响，通常开漏输出会有更高的功耗和更长的上升时间。

![img](https://img.ioyoi.me/20240111162137.webp)

实际使用中，推挽输出多用于信号始终单向输出的场景（如SPI、UART等），而开漏输出则适用于同一信号线上连接多个输出设备的场景（如I2C等）。

### GPIO原理

GPIO的基本结构如下图所示：

![image-20240111111908393](https://img.ioyoi.me/20240111111910.webp "GPIO原理图")

在各模式下，该结构的通断情况为：

* 数字输入

  关断Output，打开施密特触发器，根据配置连接上拉电阻和下拉电阻，Input data register在每个APB2总线时钟周期锁存一次来自施密特触发器的输入；

* 模拟输入

  关断Output，关断施密特触发器，断开上拉电阻与下拉电阻，Input data register始终保持全0；

* 数字输出与复用输出

  Output control工作在开漏模式或推挽模式，施密特触发器打开，上下拉电阻断开，Input data register在每个周期采样；

当GPIO处于输出状态时，数字输入照常工作，因此在推挽输出状态下，可以通过Input data register监控检验GPIO的实际输出情况。

STM32F10x的引脚通常只可输入0~3.3V的电压，但一些引脚在连接VDD的保护二极管处做了特殊设计，使之可以使用最高5V的输入电压，这些引脚在引脚图中会带有`ft`标注。

### 复用IO

复用IO（AFIO）是STM32中的一个重要概念，在标准数字输出模式下，输出控制信号来自Output data register，当改变寄存器值时，端口输出的电平值会同步发生变化，而复用输出则会切断输出寄存器的连接，将输出控制信号连接至复用该引脚的外设上，此时GPIO模式需要根据外设的要求进行更改，如果GPIO模式的配置不符合外设的需求，该外设可能会无法正常工作，具体各个外设需要的GPIO配置可查阅参考手册中“GPIO configurations for device peripherals”一节。

每个外设都有默认的复用PIN脚映射，当同时使用多个外设导致出现引脚冲突时，我们可以通过重映射（remap）来改别外设复用的PIN脚，例如当使用I2C1外设时，默认会占用PB6和PB7脚，经过重映射后，可以转为占用PB8和PB9脚，不过，所谓重映射不能随意指定服用的PIN脚，而是只能在几个配置中进行选择，管脚映射的具体列表可以查阅参考手册中的“Alternate function I/O and debug configuration”。

作为输入引脚时，GPIO也可以被复用，不过在复用时，Input data register仍可以读取到该引脚的值。

除外设的输入输出脚外，PIN脚还可以被复用为内部中断输入或事件输出。

## GPIO库函数

现在我们来看看如何通过库函数配置GPIO吧，GPIO相关的库函数编写于"stm32f10x_gpio.h"中，其库函数大致可如下分类：

* GPIO模式配置

  ```c
  // 配置GPIO模式
  void GPIO_Init(GPIO_TypeDef* GPIOx, GPIO_InitTypeDef* GPIO_InitStruct);
  // 恢复GPIO默认配置
  void GPIO_DeInit(GPIO_TypeDef* GPIOx);
  ```

  此外，库函数中还提供了用于填写`GPIO_InitStruct`的默认配置的函数，默认配置全部端口工作在浮动输入模式：

  ```c
  GPIO_InitTypeDef GPIO_InitStructure;
  GPIO_StructInit(&GPIO_InitStructure);
  ```

* 读取GPIO输入（本质是读取**Input data register**）

  ```c
  // 读取单个GPIO
  uint8_t GPIO_ReadInputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
  // 读取整排GPIO
  uint16_t GPIO_ReadInputData(GPIO_TypeDef* GPIOx);
  ```

* 读取GPIO输出（本质是读取**Output data register**）

  ```c
  // 读取单个GPIO
  uint8_t GPIO_ReadOutputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
  // 读取整排GPIO
  uint16_t GPIO_ReadOutputData(GPIO_TypeDef* GPIOx);
  ```

* 设置GPIO输出

  ```c
  // 置位，设置为高电平
  void GPIO_SetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
  // 复位，设置为低电平
  void GPIO_ResetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
  // 改变单个GPIO电平
  void GPIO_WriteBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin, BitAction BitVal);
  // 整体写入GPIO
  void GPIO_Write(GPIO_TypeDef* GPIOx, uint16_t PortVal);
  ```

  其中，前三个对单个GPIO电平的操作是通过Bit set/reset registers实现的，而`GPIO_Write`则是通过直接写入Output data register实现的；

* 复用AFIO配置

  ```c
  // 恢复所有AFIO配置，包括引脚重映射、事件线配置和外部中断配置
  void GPIO_AFIODeInit(void);
  // 将arm核的eventout信号输出至指定引脚
  void GPIO_EventOutputConfig(uint8_t GPIO_PortSource, uint8_t GPIO_PinSource);
  // 使能eventout信号输出
  void GPIO_EventOutputCmd(FunctionalState NewState);
  // 配置管脚重映射
  void GPIO_PinRemapConfig(uint32_t GPIO_Remap, FunctionalState NewState);
  // 将指定引脚作为外部中断，Pin x会被自动映射到EXTI x
  void GPIO_EXTILineConfig(uint8_t GPIO_PortSource, uint8_t GPIO_PinSource);
  ```

* 锁定GPIO配置（锁定模式配置，而不是锁定输入输出值）

  ```c
  void GPIO_PinLockConfig(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
  ```

## GPIO的基本使用

还记得我们上一篇文章中是如何使用GPIO的么？让我们来回顾一下，当我们使用PA0做输出时，需要：

* 确保GPIOA时钟已被使能；

  ```c
  RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
  ```

* 设置PA0为推挽输出或开漏输出，并指定刷新速率；

  ```c
  GPIO_InitTypeDef GPIO_InitStructure;
  GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
  GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
  GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
  GPIO_Init(GPIOA, &GPIO_InitStructure);
  ```

* 改变PA0上的电平；

  ```c
  GPIO_ResetBits(GPIOA, GPIO_Pin_0);
  GPIO_SetBits(GPIOA, GPIO_Pin_0);
  GPIO_WriteBit(GPIOA, GPIO_Pin_0, Bit_SET);
  ```

* 最后，我们还可以检查一下PA0输出寄存器的值是否成功改变。

  ```c
  assert(GPIO_ReadOutputDataBit(GPIOA, GPIO_Pin_0) == Bit_SET);
  ```

{{< note primary >}}

**如何同时设置多个引脚？**

我们可以使用按位或`|`连接多个引脚名称填写`GPIO_Pin`参数，例如：

```c
GPIO_SetBits(GPIOA, GPIO_Pin_0 | GPIO_Pin_1);
```

这种操作在大部分函数中都是可行的，例如在`GPIO_Init`、`GPIO_SetBits`、`GPIO_ResetBits`中，你可以自由使用这种形式，而在`GPIO_ReadInputDataBit`和`GPIO_ReadOutDataBit`中，则不可以使用这种方式同时读取多个位的值，比较特殊的是`GPIO_WriteBit`函数，当你使用`|`同时改变多个引脚值时，会发现程序正常运行，但这是有隐患的，因为如果打开参数检查，会发现该函数只允许输入单个Pin脚（使用宏`IS_GET_GPIO_PIN`检查参数），检查错误会使程序直接跳入`assert_failed`的死循环中。

{{< /note >}}

{{< note primary >}}

**关注库函数效率**

这里我们第一次讨论库函数的效率问题，以我们上文中提到的同时设置PA0、PA1高电平为例，查看`GPIO_SetBits`的实现：

```c
void GPIO_SetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)
{
  /* Check the parameters */
  assert_param(IS_GPIO_ALL_PERIPH(GPIOx));
  assert_param(IS_GPIO_PIN(GPIO_Pin));
  
  GPIOx->BSRR = GPIO_Pin;
}
```

首先该函数进行了参数检查，使用`if`语句遍历所有可选参数，判断参数是否合法，随后向BSRR寄存器中写入Pin脚值，这一操作虽然保证了寄存器值的合法性，是代码鲁棒性更高，但会引入不必要的运算开支；此外，每次该语句执行时，传入参数前都会进行一次按位与运算，这也会拖慢语句的执行速度。

当我们对自己的操作有足够的把握时，直接操作寄存器会比使用库函数拥有更高的效率，例如以下的B实现会比A实现拥有更高的效率，你可以使用示波器观察PA0上的电平变化频率来观察这一现象：

```c
// A
while(1)
{
    GPIO_SetBits(GPIOA, GPIO_Pin_0 | GPIO_Pin_1);
    GPIO_ResetBits(GPIOA, GPIO_Pin_0 | GPIO_Pin_1);
}

// B
while(1)
{
    GPIOA->BSRR = 0x00000003;
    GPIOA->BRR = 0x00000003;
}
```

{{< /note >}}

而将PA0作为输入时，操作类似：

* 确保GPIOA时钟已被使能；

  ```c
  RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
  ```

* 设置PA0为输入，并选择浮动、上拉或下拉；

  ```c
  GPIO_InitTypeDef GPIO_InitStructure;
  GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
  GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
  GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
  GPIO_Init(GPIOA, &GPIO_InitStructure);
  ```

* 读取PA0输入的电平；

  ```c
  GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0);
  GPIO_ReadInputData(GPIOA);
  ```

常用的GPIO操作就是这么简单，AFIO的操作我们会在后续文章中慢慢接触，现在让我们连接LED灯和按键开关，动手做做实验吧。

## 实验 Q&A

### 如何让程序暂停

也许在操作GPIO时，你也许希望LED灯亮一会、灭一会，如何让你的程序暂停一下呢，你也许会想到c语言中的`sleep()`函数，但很遗憾，arm gcc中没有实现这一函数，我们需要自己实现它，最简单的方式是写一个大循环，例如：

```c
void sleep()
{
    int i, j, k;
    for (i = 0; i < 1000; i++)
        for (j = 0; j < 1000; j++)
            for (k = 0; k < 1000; k++)
                ;
}
```

这种方式实现了一个最基础的延时，你可以调整循环的终止条件来定性改变延时长度，但你无法准确的控制具体延时了多长时间，更好的方式是通过Cortex-M3核中的SysTick计数器实现精确计时。你无法在STM32的参考手册或库函数中找到这一模块，因为他是在Cortex-M3核中实现的，你可以在Arm的[文档](https://developer.arm.com/documentation/ddi0337/e/Cihcbadd)中查找到与Systick配置相关的四个寄存器，它们在`core_cm3.h`头文件中的名称分别是：

* `SysTick->CTRL`：控制和状态寄存器

  ![](https://img.ioyoi.me/20240115010401.svg)

  * ENABLE：使能位，0为停用，1为启用，启用时计数器会首先读取Reload值，并开始递减，直至归零；
  * TICKINT：设置计数器归零时的操作，0为不进行任何操作，仅通过COUNTFLAG指示计数完成，1为自动调用SysTick handler；
  * CLKSOURCE：设置时钟源，0为外部时钟，1为内核时钟（也就是STM32中的系统时钟）；
  * COUNTFLAG：默认为0，计时器计数归零时变为1，一经读取则再次变为0；

* `SysTick->LOAD`：重载值寄存器

  ![](https://img.ioyoi.me/20240115010402.svg)

  * RELOAD：24位数据存储计数器的起始值；

* `SysTick->VAL`：计数值寄存器

  ![](https://img.ioyoi.me/20240115010403.svg)

  * CURRENT：24位数据存储计数器的当前值，可以读写；

* `SysTick->CALIB`：校准寄存器，使用参考时钟校准计数器记录的时间

  ![](https://img.ioyoi.me/20240115010404.svg)

  * TENMS：最接近10ms的计数值；
  * SKEW：如果为1，则TENMS计数不是准确的10ms，而是最接近的值；
  * NOREF：如果为1，则没有提供参考时钟，校准无效。

例如，如果我们的STM32系统时钟频率为72MHz，使用内核时钟作为时钟源，计数器每计72次，时间便过去了1us，可以据此编写如下函数，实现精准的延时：

```c
void sleep_us(uint32_t us)
{
    // 设定初始值为延时的72倍（如更改了系统时钟，倍数也需要修改）
    SysTick->LOAD = 72 * us;
    // 计数器手动归零
    SysTick->VAL = 0x00000000;
    // 设定时钟源为内核时钟，并开始计数
    SysTick->CTRL = 0x00000005;
    // 等待计数器归零
    while (!(SysTick->CTRL & 0x00010000))
        ;
    // 关闭计数器
    SysTick->CTRL = 0x00000004;
}
```

由于该计数器为24位，因此最大计数值为$2^{24}-1$，最长定时为233016us，如果实现更长时间的延时，可以通过搭配循环实现，如：

```c
void sleep_us(uint32_t us)
{
    // 设定初始值为延时的72倍（如更改了系统时钟，倍数也需要修改）
    SysTick->LOAD = 72 * us;
    // 计数器手动归零
    SysTick->VAL = 0x00000000;
    // 设定时钟源为内核时钟，并开始计数
    SysTick->CTRL = 0x00000005;
    // 等待计数器归零
    while (!(SysTick->CTRL & 0x00010000))
        ;
    // 关闭计数器
    SysTick->CTRL = 0x00000004;
}

void sleep_ms(uint32_t ms)
{
    while (ms--) {
        sleep_us(1000);
    }
}

void sleep_s(uint32_t s)
{
    while (s--) {
        sleep_ms(1000);
    }
}
```

### 为什么我不能使用PA15/PB3/PB4

这三个引脚在AFIO Remap中默认被JTAG调试接口占用，如果不需要JTAG功能，仅使用SWD调试，可以释放这些引脚作为通用GPIO使用，但需要注意，同时关闭JTAG和SWD是不推荐的，因为这样你将无法使用调试器接口烧写程序，需要通过切换启动模式才能够更换程序。

![image-20240115014749669](https://img.ioyoi.me/20240115014751.webp)

为使用PA15、PB3、PB4，可以先将AFIO外设中的`SWJ_CFG`置为010，即仅保留PA13、PA14作为调试口，随后便可对PA15、PB3、PB4进行正常的GPIO配置了：

```c
// AFIO控制首先需要开启AFIO外设时钟
RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);
// 设置模式为JTAG-DG Disabled and SW-DP Enabled
GPIO_PinRemapConfig(GPIO_Remap_SWJ_JTAGDisable, ENABLE);
```

### 按键输入抖动怎么办

你也许曾尝试实现这样的一个程序，当按键按下时，LED灯点亮，再次按下，LED灯熄灭，如果你使用了类似下面这段的代码，你会发现按下按键的结果是不确定的，有时改变了LED的状态，而有时又不会：

```c
// 判断按键是否按下
if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == Bit_RESET) {
    if (GPIO_ReadOutputDataBit(GPIOA, GPIO_Pin_1) == Bit_RESET) {
        // 若已点亮，则熄灭
        GPIO_SetBits(GPIOA, GPIO_Pin_1);
    } else {
        // 若未点亮，则点亮
        GPIO_ResetBits(GPIOA, GPIO_Pin_1);
    }
    // 等待按键松开
    while (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == Bit_RESET)
        ;
}
```

你可能会认为这是STM32不够灵敏导致的，其实恰恰相反，尽管你只按下了一次按键，但由于金属片之间的贴合、分离需要一个过程，输入信号并不是理想的高低电平切换，而是经过一小段连续随机的抖动才会完成切换，这种现象被称为按键抖动（switch debouncing），而STM32的采样频率很高，可能会采集到这一抖动，误认为你多次按下了这个按键，因此LED的状态被切换了许多次。

![Switch_Debouncing_Circuit_Waveform.jpg](https://img.ioyoi.me/20240115022025.webp)

有许多方法可以抑制按键抖动，首先是硬件消抖，可以利用RC低通滤波器+施密特触发器的组合，滤除高频抖动，这种消抖方式会损失一定的响应时间，并同时增加额外的功耗和成本，相比之下，软件防抖是一种更加快捷省事的方法。

软件防抖的思路是在每次电平切换后，会提供一个按键抖动的时间窗，在时间窗内会停止对输入的采样，等待一段时间，抖动消失后再重新进行输入判断；

例如：

```c
// 判断按键是否按下
if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == Bit_RESET) {
    if (GPIO_ReadOutputDataBit(GPIOA, GPIO_Pin_1) == Bit_RESET) {
        // 若已点亮，则熄灭
        GPIO_SetBits(GPIOA, GPIO_Pin_1);
    } else {
        // 若未点亮，则点亮
        GPIO_ResetBits(GPIOA, GPIO_Pin_1);
    }
    // 等待按下按键产生的抖动消失
    sleep_ms(10);
    // 等待按键松开
    while (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == Bit_RESET)
        ;
    // 等待松开按键产生的抖动消失
    sleep_ms(10);
}
```

经过实验测试，留有10ms时间窗的情况下，基本不会发生因抖动产生的误操作。不过在这种实现方式下，`sleep`延时函数会阻塞函数的运行，需要等待按键松开后才可以处理其他的事务，那么，是否会有更好的方式呢？当然有！不过需要借助中断实现，让我们熟悉中断以后再来学习吧。

## 小结

在这篇文章中，我们认识了STM32的时钟、总线与外设，并学习了最基本的外设GPIO，掌握这些知识后，我们已经可以完成包括LED、蜂鸣器、按键开关、光敏传感器等模块的实验。随后，我们将从AFIO出发，一步步学习其余的STM32外设。
