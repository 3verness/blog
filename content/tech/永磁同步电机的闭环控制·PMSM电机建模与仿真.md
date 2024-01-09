---
title: "永磁同步电机的闭环控制·PMSM电机建模与仿真"
slug: "pmsm-2"
subtitle: ""
defcription: ""
tags:
    - "motor"
    - "simulink"
date: 2023-04-07T01:04:47+08:00
draft: true
author: EvernessW


toc: false
katex: false
mermaid: false
---



在[上一篇文章](../pmsm-2/)中，我们介绍了PMSM电机中的坐标变换与数学模型，这篇文章将依照推导出的数学模型，搭建PMSM电机的Simulink模型，并验证模型有效性。如果你阅读本文遇到了障碍，请先回头阅读原理推导部分。

## PMSM模型搭建

Simulink中的Simscape库中已经给出了PMSM电机模型，在Matlab命令行中输入`power_pmmotor`即可打开包含该模型的示例文件，其包含一个负载输入端口`Tm`，三相电输入端口以及包含相电流、角速度等信息的输出端口。但为了更加深入的了解PMSM的工作原理，这里我们不使用内置模型，按照此前推导的数学公式重新搭建Simulink模型。

上文中我们将电机分为三个模型，由于电气模型与磁链模型高度耦合，在搭建模型时将二者合并，最终模型框架如下：

![image-20230407014819671](https://img.ioyoi.me/20230407014821.webp "电机模型总览")

注意到我们实际输入的三项电压值与输出的三项电流值均需要经过Clark变换与Park变换转变为dp轴上分量进入模型。

{{<note warning>}}

Simulink中默认使用的dp坐标系d轴超前A/α轴90°，与上文中定义的dq坐标系不同，因此做dq变换时，需要手动将模块配置为"Aligned with phase A axis"。另外需要注意进行dq变换时，使用的相角均为电气角。

![image-20230407015313613](https://img.ioyoi.me/20230407015316.webp)

{{</note>}}

电磁模型与机械模型内部如图所示：

![image-20230407015545979](../../../../AppData/Roaming/Typora/typora-user-images/image-20230407015545979.png "电磁模型")

![image-20230407015635327](https://img.ioyoi.me/20230407015637.webp "机械模型")

## PMSM模型验证





## PMSM模型分析

在FOC控制中，$I_d=0$，因此电磁模型中$Iq$可简化为：
$$
U_q-\psi_f\omega_e=(L_qs+R)I_q
$$
而对于表贴式电机而言，$L_d=L_q$，意味着动力矩与$I_q$成正比：
$$
\frac{3}{2}p\psi_fI_q-T_m=(b_m+Js)\omega_m
$$
同时电机正常运行时，外加负载远大于自身负载（在后面仿真时进行了验证），可认为：
$$
I_q=
$$


又知：
$$
\omega_e=p\omega_m
$$
联立得
$$
U_q+\frac{p\psi_f}{Js+b_m}T_m=(R+L_qs-\frac{3}{2}\frac{p^2\psi_f^2}{(b_m+Js)})I_q
$$

