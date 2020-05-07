---
title: "Simulink生成C++代码"
slug: "simulink-to-c"
subtitle: ""
defcription: ""
tags:
    - "tutorial"
    - "matlab"
date: 2020-05-07T11:44:52+08:00
draft: false
author: EvernessW


toc: false
katex: false
mermaid: false
---

Simulink提供了很强大的控制器仿真功能，但无奈自己学艺不精，对Matlab的知识掌握属实少得可怜，因此需要借助其他的语言对仿真数据进行进一步的处理及可视化，这里谈谈Simulink模型的C++项目生成及使用方法。

## 生成

首先需要在Matlab中安装Simulate Coder。

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200507114808.png)

安装完成后可以在Simulink-APPS中找到Simulink Coder入口，打开后，在Settings-Model Settings中对生成项目进行一定的配置。

首先为了更容易的得到时间，在solver中选用了定步长的龙格-库塔法进行仿真，并设置合适的步长。

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200507225409.png)

为了方便，这里选择直接生成完整的通用C++解决方案，同时勾选仅生成代码而不进行编译，这样可以省去配置编译器带来的麻烦。

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200507165913.png)

如果在编译中遇到了`rt_StartDataLoggingWithStartTime`等其他[宏缺失的问题](https://www.mathworks.com/matlabcentral/answers/313434-where-is-defined-the-function-rt_startdataloggingwithstarttime)，需要在Code Generation-Interface-Advanced parameters中禁用MAT格式日志。

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200507170729.png)

设置完成后点击Generate Code即可生成项目，生成成功后会弹出报告，项目路径为Matlab当前的工作路径。



## 使用

这里使用一简单伺服电机模型作为示例：

![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200507225133.png)

可以看到该模型有两输入信号`In1, In2`与一输出信号`Out1`。

在Clion中使用New CMake Project from Source打开生成的project_grt_rtw目录，使其自动生成一CMake文件，编辑CMakeLists.txt，在`include_directories`中添加以下几条路径，以编译Matlab中的源文件。（需将`C:\\Program Files\\MATLAB\\R2019b`替换成本机的Matlab安装目录）

```cmake
include_directories(.
        "C:\\Program Files\\MATLAB\\R2019b\\extern\\include"
        "C:\\Program Files\\MATLAB\\R2019b\\simulink\\include"
        "C:\\Program Files\\MATLAB\\R2019b\\rtw\\c\\src")
```

新建主函数入口`main.cpp`，并将`main.cpp`加入CMakeLists.txt中的`add_executable`列表中。内容如下：

```c++
#include <iostream>
#include <iomanip>
using namespace std;

#include "test1.h"

const double STEP_SIZE = 1e-5;

int main() {
    test1ModelClass model = test1ModelClass();
    model.initialize();

    cin >> model.test1_U.In1;

    for (int i = 0; i < 1e5; i++) {
        model.test1_U.In2 = 1 + rand() / double(RAND_MAX);
        model.step();
        cout << fixed << setprecision(5) << i * STEP_SIZE << " " << model.test1_Y.Out1 << endl;
    }

    model.terminate();
}
```

对主要接口进行说明：

* `model.initialize()`：初始化模型
* `model.step()`：步进一步，相当于经过一个采样周期
* `model.terminate()`：终止模型
* `model.test1_U`：模型输入
* `model.test1_Y`：模型输出

完整项目可在[git仓库](https://git.everness.me/Everness/simulink_test)中查看。



