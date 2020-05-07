---
title: "使用Python绘制电路图"
slug: "circuit-diagram-with-python"
subtitle: ""
defcription: ""
tags:
    - "python"
    - "tool"
    - "tutorial"
date: 2020-04-20T21:32:44+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

## 引子

由于最近上网课，很多作业可以以电子版提交，需要绘制的电路图一下子变多了。之前一直采用手绘+[circuitlab](https://www.circuitlab.com/editor/)+multisim的绘图方式，但它们都有一些这样那样的缺点。手绘太累，multisim的排线以及符号功能有点难用，circuitlab应该是一种比较好的解决方案，但依然存在不支持Latex符号的缺陷，致使许多下标符号不能正常显示，且自定义芯片的功能比较差。

后来在逛知乎的时候发现了[SchemDraw](https://schemdraw.readthedocs.io/)，只是一个Python下的电路图库，调用matplotlib库进行绘图，相比以上几个工具，其优点主要有下：

* 较为丰富的图形库（相比circuitlab）
* 方便的自定义元器件
* 支持Latex符号
* 元器件角度旋转调整

当然首先还是要劝退一波：

* SchemDraw不支持自动布线，所有的线长都需要手动配置，因此只适合抄图而不适合直接作图。
* SchemDraw适合键盘爱好者，对鼠标依赖者不友好。如果你偏爱word而不是latex/markdown，那么SchemDraw可能并不适合你。

## 安装

pip安装

```bash
pip install SchemDraw
```

## 基础使用

通用模板：

```python
import SchemDraw
import SchemDraw.elements as e

d = SchemDraw.Drawing()

d.add(e.XXX, ...)
d.add(e.XXX, ...)

d.draw()
d.save('out.svg')
```

所有的绘制工作都需要通过`d.add()`接口完成：

```python
schemdraw.Drawing.add(elm_def, **kwargs)
```

首先必须的是元件名称`elm_def`，既可以是`elements`包中包含的元器件，也可以是自定义的元器件，文后附有官方文档给出一些[元器件](#元件速查表)。其后常用[可选参数](https://schemdraw.readthedocs.io/en/latest/usage/placement.html)如下：

|       参数        |                  说明                   |        示例        |
| :---------------: | :-------------------------------------: | :----------------: |
| `label=(string)`  | 标注，可用`top/bot/rgt/lft`前缀设置位置 | `botlabel="$U_i$"` |
|   `d=(string)`    |     方向，默认值为上一元器件的方向      |     `d='left'`     |
|  `theta=(float)`  |            方向角，0表示向右            |    `theta=-45`     |
|    `l=(float)`    |        长度，一般用`d.unit`控制         |    `l=d.unit/2`    |
| `xy=(ele.anchor)` |              元器件起始点               |    `xy=op.out`     |
| `anchor=(string)` |          选择元器件的连接锚点           |   `anchor='out'`   |
| `to=(ele.anchor)` |   设置终点，可用`tox/toy`来设置单坐标   |    `tox=op.in1`    |
| `reverse=(bool)`  |               反转元器件                |   `reverse=True`   |

在绘图过程中，若有需要调用的元器件，则在添加该元器件时需要用变量记录记录：

```python
op = d.add(e.OPAMP, anchor='in2')
```

推荐采用的绘图顺序是`push()/pop()`的连续绘图方式，`d.push()`是储存当前位置，`d.pop()`则可以回到上一储存位置，下面用一个实例来说明该绘图过程。

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200422120755.svg)

```python
d.add(e.DOT_OPEN, label='$a$')
d.add(e.LINE, d='right', l=d.unit * 2)
d.add(e.DOT)
d.push()
d.add(e.LINE, theta=-45, l=d.unit / 2)
d.add(e.DOT)
d.push()
d.add(e.LINE, theta=45, l=d.unit / 4)
d.add(e.CAP, theta=-45, label='$C_1$')
d.add(e.LINE, theta=-135, l=d.unit / 4)
d.add(e.DOT)
d.pop()
d.add(e.LINE, theta=-135, l=d.unit / 4)
d.add(e.RES, theta=-45, label='$R_1$')
d.add(e.LINE, theta=45, l=d.unit / 4)
d.add(e.LINE, theta=-45, l=d.unit / 2)
d.add(e.DOT)
d.push()
d.add(e.LINE, d='down', l=d.unit * 2)
d.add(e.DOT_OPEN, botlabel='$d$')
d.pop()
d.add(e.CAP, theta=-135, label='$C_2$')
d.add(e.RES, theta=-135, label='$R_2$')
d.add(e.DOT)
d.push()
d.add(e.LINE, d='left', l=d.unit * 2)
d.add(e.DOT_OPEN, label='$b$')
d.pop()
d.add(e.LINE, theta=135, l=d.unit / 2)
d.add(e.RES, theta=135, label='$R_3$')
d.add(e.LINE, theta=135, l=d.unit / 2)
d.add(e.DOT)
d.push()
d.add(e.LINE, d='down', l=d.unit * 2)
d.add(e.DOT_OPEN, botlabel='$c$')
d.pop()
d.add(e.LINE, theta=45, l=d.unit / 2)
d.add(e.RES, theta=45, label='$R_4$')
d.add(e.LINE, theta=45, l=d.unit / 2)
```

在这张电路图中，绘图起点选择了左上角的A点，按照顺时针的顺序进行绘制，当遇到结点时，即使用`d.push()`记录该节点，随后绘制支路，绘制完一条支路后使用`d.pop()`返回节点继续绘制其他支路，直至图形完全绘制。

## 自定义元器件

暂时还没有这方面的例子，等我用到了再来补充QAQ

## 示例

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200422120533.svg)

```python
op = d.add(e.OPAMP)
d.add(e.LINE, d='left', xy=op.in1, l=d.unit / 6)
d.add(e.LINE, d='up', l=d.unit / 3)
d.add(e.DOT)
d.push()
d.add(e.LINE, d='right')
d.add(e.LINE, d='down', toy=op.out)
d.add(e.DOT)
d.push()
d.add(e.LINE, d='left', tox=op.out)
d.pop()
d.add(e.LINE, d='right', l=d.unit / 6)
d.add(e.DOT_OPEN, label='$U_o$')
d.pop()
d.add(e.CAP, d='left', label='$C_3$')
d.add(e.LINE, d='down', toy=op.in2)
d.add(e.DOT, botlabel='$a$')
d.push()
d.add(e.RES, d='left', label='$R_1$')
d.add(e.DOT_OPEN, label='$U_i$')
d.pop()
d.add(e.RES, d='right', label='$R_2$')
d.add(e.DOT)
d.push()
d.add(e.LINE, tox=op.in2)
d.pop()
d.add(e.CAP, d='down', label='$C_4$')
d.add(e.GND)
```

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200422121242.svg)

```python
d.add(e.DOT_OPEN, label='$U_i$')
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.DOT)
d.push()
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.CAP, label='$C$')
d.add(e.DOT)
d.push()
d.add(e.CAP, label='$C$')
d.add(e.DOT)
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.DOT_OPEN, label='$U_o$')
d.pop()
d.add(e.LINE, d='down', l=d.unit / 2)
d.add(e.RES, d='down', botlabel='$R/2$')
d.add(e.LINE, d='left', l=d.unit / 8)
d.add(e.GND)
d.pop()
d.add(e.LINE, d='down', l=d.unit / 2)
d.add(e.RES, d='right', label='R')
d.add(e.DOT)
d.push()
d.add(e.RES, label='R')
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.LINE, d='up', l=d.unit / 2)
d.pop()
d.add(e.CAP, d='down', label='$2C$')
d.add(e.LINE, d='right', l=d.unit / 8)
```

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200422121404.svg)

```python
d.add(e.DOT_OPEN, label='$U_i$')
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.DOT)
d.push()
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.CAP, label='$C$')
d.add(e.DOT)
d.push()
d.add(e.CAP, label='$C$')
d.add(e.DOT)
d.add(e.LINE, d='right', l=d.unit / 4)
op1 = d.add(e.OPAMP, anchor='in2')
d.pop()
d.add(e.LINE, d='down', l=d.unit / 2)
d.add(e.RES, d='down', botlabel='$R/2$')
d.pop()
d.add(e.LINE, d='down', l=d.unit / 2)
d.add(e.RES, d='right', label='R')
d.add(e.DOT)
d.push()
d.add(e.RES, label='R')
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.LINE, d='up', l=d.unit / 2)
d.pop()
d.add(e.CAP, d='down', label='$2C$')
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.DOT)
d.add(e.LINE, d='right', tox=op1.in1)
op2 = d.add(e.OPAMP, anchor='out', reverse=True)
d.add(e.LINE, d='left', xy=op1.in1, l=d.unit / 4)
d.add(e.LINE, d='up', l=d.unit / 2)
d.add(e.LINE, d='right', l=d.unit * 1.6)
d.add(e.LINE, d='down', toy=op1.out)
d.add(e.DOT)
d.add(e.RES, d='down', label='$R_1$')
d.add(e.LINE, d='down', toy=op2.in2)
d.add(e.DOT)
d.add(e.RES, d='down', label='$R_2$')
d.add(e.GND)

d.add(e.LINE, d='right', xy=op1.out, l=d.unit * 0.6)
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.DOT_OPEN, label='$U_o$')
d.add(e.LINE, d='right', xy=op2.in2, l=d.unit * 0.6)
d.add(e.LINE, d='right', xy=op2.in1, l=d.unit * 0.2)
d.add(e.LINE, d='up', l=d.unit / 2)
d.add(e.LINE, d='left', l=d.unit * 1.2)
d.add(e.LINE, d='down', toy=op2.out)
d.add(e.DOT)
```

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200422121500.svg)

```python
d.add(e.DOT_OPEN, label='$U_i$')
d.add(e.RES, label='$R_4$')
d.add(e.DOT)
d.push()
d.add(e.LINE, l=d.unit / 4)
op1 = d.add(e.OPAMP, anchor='in1')
d.pop()
d.add(e.LINE, d='up', l=d.unit * 0.8)
d.add(e.DOT)
d.push()
d.add(e.LINE, d='right', l=d.unit / 10)
d.add(e.RES, d='right', label='$R_2$')
d.add(e.DOT, label='b')
d.push()
d.add(e.LINE, d='down', toy=op1.out)
d.add(e.DOT)
d.push()
d.add(e.LINE, d='left', tox=op1.out)
d.pop()
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.DOT)
d.add(e.LINE, d='up', l=d.unit / 6)
d.add(e.CAP, d='right', label='$C$')
d.add(e.LINE, d='down', l=d.unit / 6)
d.add(e.DOT)
d.push()
d.add(e.LINE, d='down', l=d.unit / 6)
d.add(e.RES, d='left', botlabel='$R$')
d.add(e.LINE, d='up', l=d.unit / 6)
d.pop()
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.DOT, label='$d$')
d.push()
d.add(e.RES, d='down', label='$R$')
d.add(e.CAP, d='down', label='$C$')
d.add(e.GND)
d.pop()
d.add(e.LINE, d='right', l=d.unit / 4)
op2 = d.add(e.OPAMP, anchor='in2')
d.pop()
d.add(e.LINE, d='right', l=d.unit / 4)
R = d.add(e.RES, d='right', label='$R_1$')
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.DOT, label='$c$')
d.push()
d.add(e.LINE, d='down', toy=op2.in1)
d.add(e.LINE, d='right', tox=op2.in1)
d.pop()
d.add(e.LINE, d='right', l=d.unit / 10)
d.add(e.RES, d='right', label='$2R_1$')
d.add(e.DOT, rgtlabel='$a$')
d.add(e.LINE, d='down', toy=op2.out)
d.push()
d.add(e.LINE, d='left', tox=op2.out)
d.pop()
d.add(e.LINE, d='right', l=d.unit / 4)
d.add(e.DOT_OPEN, label='$U_o$')
d.pop()
d.add(e.LINE, d='up', l=d.unit / 2)
d.add(e.LINE, d='right', tox=R.start)
d.add(e.RES, label='$R_3$')
d.add(e.LINE, l=d.unit * 1.35)
d.add(e.LINE, d='down', l=d.unit / 2)
d.add(e.LINE, d='left', xy=op1.in2, l=d.unit / 4)
d.add(e.GND)
```

[更多……](https://schemdraw.readthedocs.io/en/latest/gallery/gallery.html)

## 元件速查表

![](https://schemdraw.readthedocs.io/en/latest/_images/electrical_7_0.svg "单接口")

![](https://schemdraw.readthedocs.io/en/latest/_images/electrical_1_0.svg "双接口")

![](https://schemdraw.readthedocs.io/en/latest/_images/electrical_2_0.svg "电源和电动机")

![](https://schemdraw.readthedocs.io/en/latest/_images/electrical_3_0.svg "开关")

![](https://schemdraw.readthedocs.io/en/latest/_images/electrical_6_0.svg "点线")

更多详见[官方文档](https://schemdraw.readthedocs.io/en/latest/elements/electrical.html)。

