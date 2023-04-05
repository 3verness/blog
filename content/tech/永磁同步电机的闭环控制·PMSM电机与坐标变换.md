---
title: "永磁同步电机的闭环控制·PMSM电机与坐标变换"
slug: "pmsm-1"
subtitle: ""
defcription: ""
tags:
    - "motor"
date: 2023-04-05T22:45:14+08:00
draft: false
author: EvernessW


toc: false
mathjax: true
mermaid: true
---

## 永磁同步电机原理

永磁同步电机（**P**ermanent **M**agnet **S**ynchronous **M**otor）是工业应用中一种常见的电机，其结构由永磁体转子与线圈定子组成，线圈通入交变电流时产生交变磁场，永磁体转子产生定向磁场，两磁场相互作用，引导永磁体转子指向旋转磁场方向，实现转子转动。

永磁同步电机的运行依赖磁场的同步，稳定运行时，转子与定子磁场同步转动，如果定子速度慢于磁场旋转速度，磁场力引导转子加速，反之则引导转子减速，并最终实现速度的同步。

## 永磁同步电机中的坐标系

永磁同步电机控制中通常会使用三种坐标系，分别为ABC坐标系、αβ坐标系和dq坐标系。[^1]

![Illustration-of-the-salient-pole-permanent-magnet-synchronous-motor-PMSM-model](https://img.ioyoi.me/20230405230357.webp "PMSM电机中的三个坐标系")

* ABC坐标系由三项定子线圈产生的磁场方向确定，坐标轴两两相隔120°，为固定坐标系；
* αβ坐标系是ABC坐标系对应的直角坐标系，无实际物理意义，其α轴与A轴重合，β轴超前α轴90°，同样为固定坐标系；
* dq坐标系由永磁体转子磁场方向确定，d轴平行于永磁体磁场而q轴垂直于永磁体磁场，该坐标系随转子旋转而转动，其中d轴与α轴的夹角即为电机的转动角。

三种坐标系通过依照平行四边形法则的向量合成与分解实现两两转换，其转换关系如下图所示：

<div class="mermaid" align="center">
graph LR;
A[ABC]--Clark-->B[αβ]
B--anti-Clark-->A
B--Park-->C[dq]
C--anti-Park-->B
</div>

* Clark变换
  $$
  \left[\begin{array}{c}f_\alpha\\\\f_\beta\end{array}\right]=\frac{2}{3}\left[\begin{array}{ccc}1&-\frac{1}{2}&-\frac{1}{2}\\\\0&\frac{\sqrt 3}{2}&-\frac{\sqrt3}{2}\end{array}\right]\left[\begin{array}{c}f_A\\\\f_B\\\\f_C\end{array}\right]
  $$
  {{<note primary>}}

  **如何理解系数2/3？**

  ABC三轴上的电压/电流均为交流量，以平衡三相电举例，若直接合成：
  $$
  \begin{aligned}
  f_\alpha&=U\cos( \omega t)-\frac{1}{2}U\cos (\omega t-2\pi/3)-\frac12U\cos(\omega t+2\pi/3)\\\\
  &=U\cos (\omega t)(1-\cos2\pi/3)\\\\
  &=\frac{3}{2}U\cos (\omega t)\\\\
  f_\beta&=\frac{\sqrt3}{2}U\cos (\omega t-2\pi/3)-\frac{\sqrt3}{2}U\cos (\omega t+2\pi/3)\\\\
  &=\frac{\sqrt3}{2}U\sin(\omega t)(-2\sin2\pi/3)\\\\
  &=\frac{3}{2}U\sin(\omega t)
  \end{aligned}
  $$
  注意到合成后两轴上的电压/电流无论是幅值还是功率都与原三项交流量不同，而在实际应用中我们希望控制变换前后轴上幅值或功率保持不变，因此若控制幅值不变，需乘系数2/3，而控制功率不变时，则需乘系数$\sqrt6/3$。

  {{</note>}}

* 反Clark变换
  $$
  \left[\begin{array}{c}f_A\\\\f_B\\\\f_C\end{array}\right]=\left[\begin{array}{cc}1&0\\\\-\frac{1}{2}&\frac{\sqrt3}{2}\\\\-\frac{1}{2}&\frac{\sqrt3}{2}\end{array}\right]\left[\begin{array}{c}f_\alpha\\\\f_\beta\end{array}\right]
  $$
  与Clark变换相同，这里给出的时等幅值变换的公式，若进行等功率变换，需乘系数$\sqrt6/3$。

* Park变换

  与电机电气角$\theta_e$有关：
  $$
  \left[\begin{array}{c}f_d\\\\f_q\end{array}\right]=\left[\begin{array}{cc}\cos \theta_e&\sin \theta_e\\\\-\sin \theta_e&\cos\theta_e\end{array}\right]\left[\begin{array}{c}f_\alpha\\\\f_\beta\end{array}\right]
  $$
  {{<note primary>}}

  **电气角与机械角**

  注意这里的电气角$\theta_e$与机械角$\theta_m$不同，其之间的转换关系与电机级对数$p$有关：
  $$
  \theta_e=p\theta_m
  $$
  ![img](https://img.ioyoi.me/20230405234330.webp "PMSM电机电气角与机械角的关系（图中p定义为电机级数）")

  这是由于永磁体转子中往往不止包含一对磁极，因此对于一个包含$p$对磁极的转子来说，当其旋转$2\pi/p$时，其磁场便完成了一个周期的旋转，因此电气角和机械角存在$p$倍的对应关系。

  {{</note>}}

* 反Park变换
  $$
  \left[\begin{array}{c}f_\alpha\\\\f_\beta\end{array}\right]=\left[\begin{array}{cc}\cos \theta_e&-\sin \theta_e\\\\\sin \theta_e&\cos\theta_e\end{array}\right]
  $$

## 永磁同步电机的数学模型

依照永磁同步电机的原理，可以将PMSM电机的运行过程分解为以下三个模型：电器模型、磁链模型和机械模型。

其中$u_d,u_q$为dq轴上定子电压分量，$i_d,i_q$为电流分量，$R$为定子电阻，$\psi_d,\psi_q$为定子磁链的分量，$\omega_e$为电气角速度，$L_d,L_q$为轴上电感分量，$\psi_f$为永磁体磁链，$\omega_m$为机械角速度，$J$为转动惯量，$T_m$为负载力矩，$b_m$为摩擦系数。

* 电气模型：同步电机两轴电压由电阻压降与反电动势组成，而反电动势依照来源可划分为轴上磁通变化感生电动势和磁场夹角变化感生电动势，具体可写作：
  $$
  \begin{aligned}
  u_d&=Ri_d+\frac{d}{dt}\psi_d-\omega_e\psi_q\\\\
  u_q&=Ri_q+\frac{d}{dt}\psi_q+\omega_e\psi_d
  \end{aligned}
  $$

* 磁链模型：如原理部分所示，磁场包括永磁体磁场和线圈感生磁场，其中永磁体磁场仅加载在平行于永磁体的d轴：
  $$
  \begin{aligned}
  \psi_d&=L_di_d+\psi_f\\\\
  \psi_q&=L_qi_q
  \end{aligned}
  $$
  将磁链模型带入电气模型中得到：
  $$
  \begin{aligned}
  u_d&=Ri_d+L_d\frac{di_d}{dt}-\omega_e L_qi_q\\\\
  u_q&=Ri_q+L_q\frac{di_q}{dt}+\omega_eL_di_d+\omega_e\psi_f
  \end{aligned}
  $$

* 机械模型：PMSM电机的动力矩由洛伦兹力产生，其公式为：
  $$
  T_e=\frac{3}{2}pi_q\left[i_d\left(L_d-L_q\right)+\psi_f\right]
  $$
  特别地，对于表贴式电机而言，其交直轴的磁阻差异很小，因此交直轴电感差异也很小，可认为$L_d=L_q$，因此动力矩公式简化为：
  $$
  T_e=\frac{3}{2}pi_q\psi_f
  $$
  {{<note primary>}}

  **动力矩的推导公式**

  由右手定则判断，在dq坐标系下，d轴电流受到的力矩为逆时针（正），q轴电流受到的力矩为顺时针（负），因此在固定坐标系下，依照作用力与反作用力，可知d轴电流对转子提供阻力矩，而q轴电流提供动力矩，则有：
  $$
  \begin{aligned}
  T_e&=\frac{3}{2}p(\psi_di_q-\psi_qi_d)\\\\&=\frac{3}{2}p(L_di_di_q+\psi_fi_q-L_qi_qi_d)\\\\&=\frac{3}{2}pi_q\left[i_d\left(L_d-L_q\right)+\psi_f\right]
  \end{aligned}
  $$
  {{</note>}}

  除洛伦兹力外，转子还可能会受到负载产生的阻力矩和摩擦带来的阻力矩，最终牛顿第二定律得：
  $$
  T_e-b_m\omega_m-T_m=J\frac{d\omega_m}{dt}
  $$

结合以上三个模型，即可完整描述出永磁同步电机的工作原理。

[^1]: https://www.researchgate.net/publication/327634712_Full-Speed_Range_Encoderless_Control_for_Salient-Pole_PMSM_with_a_Novel_Full-Order_SMO
[^2]: https://www.sciencedirect.com/topics/engineering/magnetomotive-force
