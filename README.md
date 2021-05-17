# 同济大学软件学院操作系统课程项目——进程管理：电梯调度模拟
## 1.项目简介

电梯模拟情况：模拟一栋有20层楼的大楼，有五部型号相同（开关门时间相同，运行速度相同）的电梯，序号分别为*1,2,3,4,5*号。本项目实现的是一个用户操作界面，用户可在电梯内部按下具体楼层的运行按钮，电梯开门按钮、电梯关门按钮、以及报告电梯损坏的报警按钮；同时，用户可在电梯外部每个楼层按下上行与下行按钮。对于报告损坏的电梯，可以通过电梯外部控制台的修理按钮进行修复。

## 2.开发环境与项目结构

- 开发工具:

  1.Visual Studio Code 2019

  2.PyCharm Community Edition 2020.1.3 x64 (Qt Designer,PyUIC5 as External Tools)

- 开发语言:  Python3

- 所使用的库:

  PyQt5 (使用Qt为前端框架，PyQt5为支持python3的Qt版本框架)

- 项目文件结构:

  *src*: 包含该项目的Python3源代码：

     —*utils.py*: 常量声明，电梯类声明定义，读写工具声明定义，组件命名规则设定

     —*elevator_ui.py*: Qt框架UI主窗口类声明定义，包含界面组件，按钮触发、动画管理方法

     —*dispatcher.py*: 调度器声明定义，电梯列表，按钮响应机制，内外调度方法与更新机制
  
     —*main.py*: Qt框架下主函数界面
  
  *resources*:包含该项目所有的图片(.png)，可执行文件图标(.ico)与动画文件(.gif)
  
  *style*:用于界面美化的样式文件
  
  *main.exe* 可执行文件，**运行时需要与src,resources文件夹相同目录**

## 3.功能简介

### 3.1 界面说明与功能描述:

<img src="https://gitee.com/mount-potato/markdown-img-hosting/raw/master/pic/20210512235430.png" alt="img" style="zoom: 30%;" />

项目主界面见上，其中电梯的功能组件与可视化组件有:

- **电梯图标**: 电梯的可视化，**<font color='red'>含开门关门动画，依赖资源文件中的gif动图</font>**
- **LCD显示器**: 电梯楼层的可视化，实时显示电梯所在楼层
- **控制台日志记录**: 电梯调度命令的可视化，输出用户命令与指令响应的记录日志

- **电梯内部楼层按钮**: 1-20楼电梯按钮，电梯内调度主要控件。
- **电梯内部开门按钮**: 手动打开电梯门操作所需按钮，当电梯运行时无法使用
- **电梯内部关门按钮**: 手动关闭电梯门操作所需按钮，当电梯运行时无法使用
- **电梯内部报警按钮**: 电梯故障时报警按钮。任何正常运行状态下都可使用
- **电梯外部上楼按钮**: 1-20楼上楼请求按钮，电梯外调度主要控件
- **电梯外部修理按钮**: 模拟对已损坏的电梯进行修理的按钮

用户可使用的主要功能有：

1. **电梯内部楼层请求的内调度**:用户在电梯内按下楼层按钮， **<font color='red'>所在电梯通过内调度安排楼层抵达的先后顺序</font>**
2. **电梯外部上下楼请求的外调度**:用户在电梯外按上下楼按钮，**<font color='red'>系统通过外调度方法计算出可选择的最佳电梯并调度其前往</font>**，若无可用电梯时给出提示
3. **电梯内部开关门请求**:在电梯停止时进行手动开关电梯门，**<font color='red'>开门关门状态可以立即互相打断覆盖，重复按下开门会延长电梯开门等待时间</font>**
4. **电梯内部报警请求**:用户在电梯正常运行时按下报警按钮，电梯被停用，**<font color='red'>电梯门关闭，该电梯所有内调度请求失效，所有外调度请求交由其他可用电梯进行外调度</font>**。当所有电梯都发生故障时，**<font color='red'>外部上下楼按钮将被禁用</font>**，此时系统无法进行所有正常操作
5. **电梯外部修理请求**:用户在电梯外部按下修复按钮，对应电梯恢复正常并停止在当前楼层，**<font color='red'>恢复该电梯内部楼层按钮状态，只要有一台电梯可用便恢复电梯外部上下楼按钮状态</font>**
6. **查看电梯调度情况**:通过文本框实时查看历史调度命令与调度系统的响应情况

电梯基本状态描述：

- 每1秒更新所有电梯状态一次
- 电梯每1秒上升或下降一层楼
- 电梯开门动画用时0.7秒，开门等待时间3.3秒，关门时间0.7秒
- 报警故障时，停止在当前所在楼层，电梯门关闭



## 4.调度功能实现与调度算法详述:

电梯问题中的调度问题主要包含内调度与外调度两个方面。内调度指单个电梯内所有任务的调度，在本问题中具体指当用户在电梯内点击不同的楼层按钮时，电梯根据内调度机制决定先后到达次序(**主要问题：选择电梯内调度的顺序机制**)。外调度指在电梯外部发送命令请求时系统对请求的合理分配，在本问题中具体指当用户在电梯外某楼层点击上楼或下楼按钮时如何调配电梯前往(**主要问题：如何将请求进行合理分配**)。

### 4.1 电梯任务的核心数据结构以及调度算法实现

在每部电梯中，设置了两个队列，分别是第一任务队列$$by\_wask$$与第二任务队列$$oppo\_task$$。其中第一任务队列中任务的优先级高于第二任务队列，在第一任务队列中所有任务执行完毕并出队后，第二任务队列的内容才会搬运到第一任务队列中开始执行。调度算法规定，任务入队的情况如下：

对外调度命令：

- 当**外调度命令与电梯运行状态相反**，或**电梯上行时调度楼层低于电梯楼层**，以及**电梯下行时调度楼层高于电梯楼层**时，放入第二队列
- 否则，放入第一队列

对内调度命令：

- 当**电梯上行时调度楼层低于电梯楼层**，以及**电梯下行时调度楼层高于电梯楼层**时，放入第二队列
- 否则，放入第一队列

在这个过程中，伴随着对第一任务队列，第二任务队列的排序。

这一机制实际上借鉴了LOOK调度算法，让电梯在最底层和最顶层之间连续往返运行，在运行过程中响应处在于电梯运行方向相同的各楼层上的请求，并在电梯所移动的方向上不再有请求时立即改变运行方向，最大程度地保证电梯运行距离最短化。具体到电梯实例中也就是：

- 电梯静止时，所有任务可以放入第一队列，优先进行
- 电梯上升或下降时，对调度方向与电梯运行方向一致的任务优先进行响应，不一致的放入第二队列次要响应

每个队列的元素以二元组形式存储，即$$(target\_level,order)$$，其中$$target\_level$$表示调度命令所在的楼层，$$order$$是对应任务的命令，包含：$$UP$$:上升的外调度命令；$$DOWN:$$下降的外调度命令；$$INNER$$:内调度命令。$$order$$的作用主要有两个：1.对外调度按下的按钮，可以根据$$target\_level$$与$$order$$定位到对应按钮，电梯到达时可以恢复该按钮的长亮状态；2.电梯故障时对电梯已有任务进行外调度内调度区分，从而实现对外调度任务的重新分配与内调度任务的删除。

### 4.2 外调度的最优分配机制

在外调度选择最佳电梯调度时，首先挑选出可以运行的电梯，也就是调度方向与运行方向一致的电梯。然后，对所有可以运行的电梯，计算运行时间最短的电梯为最优电梯，时间的计算标准是:上楼时间（以简单的目标楼层与电梯楼层之差的绝对值计算）与当前等待时间（电梯若处于静止状态时开门等待的时间）之和。这样的调度方式可以充分利用每一部电梯，防止出现单部电梯持续被调度，出现资源利用不充分的情况。

### 4.3 其它功能的实现介绍

#### 4.3.1 电梯内部警报的触发与修理

电梯内部触发警报机制后，电梯内调度相关按钮禁用，开关门按钮失效，此时电梯内调度任务全部失效，任务队列中所有的外调度任务（order为UP或DOWN）分配给其他可用电梯进行外调度。外调度按钮只要有一台电梯可用就处于正常运行状态，而当全部电梯都失效后，外调度按钮也全部失效。修理电梯时，电梯恢复停止状态。修理一部电梯时，恢复内调度按钮，初始化电梯状态，电梯可继续正常运行。

#### 4.3.2 电梯内部开门关门按钮

电梯内部的开门关门按钮在电梯静止时才能使用，开门关门按钮可以互相进行状态覆盖，也就是说：按下开门按钮，不管电梯是否静止或者正在关闭仓门，都会立即打开，反之同理。重复按下开门按钮，电梯会持续恢复开门等待时间，也就是延长了电梯等待的时间间隔。按下关门按钮，电梯状态时间将调至关门前1秒，此时电梯会马上关门。

#### 4.3.3 电梯计时器更新机制

每隔一秒钟，遍历所有的电梯，对每一台电梯的调度情况进行更新。更新的内容主要有：电梯的状态，电梯楼层显示器的数字，电梯状态持续时间等。在更新电梯情况时，需要取出电梯第一任务队列$$by\_task$$中的首个任务，根据该任务的情况，进行楼层的加减，楼层数字显示器的变更，以及电梯上升、下降、静止状态的变更。	



## 5.运行截图

<img src="https://gitee.com/mount-potato/markdown-img-hosting/raw/master/pic/20210513103947.png" alt="img" style="zoom: 28%;" /> <img src="https://gitee.com/mount-potato/markdown-img-hosting/raw/master/pic/20210513104135.png" alt="img" style="zoom: 28%;" /> <img src="https://gitee.com/mount-potato/markdown-img-hosting/raw/master/pic/20210513104136.png" alt="img" style="zoom: 28%;" /> 

（主界面已于功能简介中用于说明展示，此处不再重复放出）

左：电梯内调度机制展示

中：电梯外调度机制展示

右：电梯报警与已有任务分配机制展示
