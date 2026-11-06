# 第 10 篇：WinCC 组态与 VB 脚本讲解

> 原文发表于 2024-07-09，迁移至 GitHub 时进行了排版整理与附图本地化（修复了原网页导出时代码块折叠遮挡正文的问题）。

本篇基于前文 PLC 梯形图工程已搭建完成的前提下，介绍上位机 **WinCC RT Professional** 监控界面的组态流程与 VB 脚本编写方法。

---

## 一、新建 WinCC 界面与网络组态

1. 添加 **PC 主控站** 和 **WinCC 可视化界面**：

   ![添加 PC 系统与 WinCC 应用](images/10-wincc-hmi-vbscript/fig-01.png)

2. 添加 **以太网通信组态**：

   ![添加 IE General 以太网卡](images/10-wincc-hmi-vbscript/fig-02.png)

   > **注意**：因为使用的是普通电脑以太网卡连接 PLC，所以通信模块应选择添加 **`IE General`**。

3. 网络连接建立后的最终拓扑效果如下：

   ![PLC 与 PC 站以太网连接拓扑](images/10-wincc-hmi-vbscript/fig-03.png)

---

## 二、新建画面图形及连接变量

根据[第 1 篇赛题准备](01-competition-analysis-and-setup.md)中的界面规范，我们首先建立全部监控画面框架：

![WinCC 画面项目树目录](images/10-wincc-hmi-vbscript/fig-04.png)

赛题所需的 **7 个核心画面** 如下：

1. **六部电梯的状态画面**

   ![六部电梯状态画面](images/10-wincc-hmi-vbscript/fig-05.png)

2. **主画面（电梯监控系统入口界面）**

   ![电梯监控系统主画面](images/10-wincc-hmi-vbscript/fig-06.png)

3. **初始化楼层设定界面**（由于 HMI 与 DB 块变量双向绑定，既可以在梯形图中改，也可以直接在此界面中在线修改）

   ![初始化楼层设定界面](images/10-wincc-hmi-vbscript/fig-07.png)

4. **外呼监控界面**

   ![外呼监控界面](images/10-wincc-hmi-vbscript/fig-08.png)

5. **电梯状态总览界面**

   ![电梯状态总览界面](images/10-wincc-hmi-vbscript/fig-09.png)

6. **电梯运行监控画面**

   ![电梯运行监控画面](images/10-wincc-hmi-vbscript/fig-10.png)

7. **登录界面**

   ![用户登录界面](images/10-wincc-hmi-vbscript/fig-11.png)

---

### 工具箱三类对象的区分与使用

博途 WinCC 右侧工具箱提供了丰富的组态控件，其中 **基本对象**、**元素** 和 **控件** 三者的定位需要重点区分：

![WinCC 工具箱面板分类](images/10-wincc-hmi-vbscript/fig-12.png)

#### 2.1 基本对象

简单来说，**基本对象**是自身不主动进行复杂数据输入交互的几何图形（如方框、圆、线条、静态文本等），是构成画面布局的基本单元，但可以通过绑定 PLC 变量的布尔值或整型值在 **动画 → 外观** 窗格中动态改变颜色或闪烁状态：

![基本对象的外观动画变量绑定](images/10-wincc-hmi-vbscript/fig-13.png)

#### 2.2 元素

**元素**是 WinCC 人机交互的核心，通过输入/输出域（I/O 域）、按钮、开关实现上位机与 PLC 的数据双向传递。

- **I/O 域**：可以配置属性、动画、事件、文本等参数，通过设置 **过程值** 绑定对应的 PLC / HMI 变量：

  ![I/O 域过程值绑定（一）](images/10-wincc-hmi-vbscript/fig-14.png)

  ![I/O 域过程值绑定（二）](images/10-wincc-hmi-vbscript/fig-15.png)

- **按钮与开关**：通常在 **事件（Events）** 选项卡中配置动作，例如单击按下按钮后激活指定画面或弹出窗口：

  ![按钮单击事件配置](images/10-wincc-hmi-vbscript/fig-16.png)

#### 2.3 控件

**控件**主要用于系统级参数、报警记录或用户权限的集中展示与管理（初赛通常不作深入考察，例如“用户视图”控件可用于后台管理登录账号与密码）：

![WinCC 控件面板示例](images/10-wincc-hmi-vbscript/fig-17.png)

---

## 三、设置弹出画面与登录验证（附 VB 脚本详解）

在 Day 1 和 Day 2 的比赛中都会考察登录验证与提示弹窗，下面以**登录界面“确认”按钮**为例进行演示。

实现画面切换/弹出共有两种方法：
- **方法一（系统函数配置）**：在 WinCC 画面管理中新建弹出画面，随后在按钮上依次配置：`事件 → 单击 → 添加函数 → 画面 → 显示弹出画面 / 激活画面 → 选择目标画面名称`。
- **方法二（VB 脚本实现条件判断与跳转）**：当运行系统后输入账号密码并单击“确认”按钮时，调用自定义 VB 脚本函数 `denglu` 判断登录成功、未输入账号密码或密码错误。

按钮绑定 `denglu` 脚本及变量表配置如图：

![确认按钮调用 VB 脚本 denglu](images/10-wincc-hmi-vbscript/fig-18.png)

### `denglu` 函数完整 VBScript 代码

```vbscript
Sub denglu()
    ' 提示：
    ' 1. 使用 <CTRL+SPACE> 或 <CTRL+I> 快捷键打开含所有对象和函数的列表
    ' 2. 使用 HMI Runtime 对象写入代码。示例：HmiRuntime.Screens("Screen_1")
    ' 3. 使用 <CTRL+J> 快捷键创建对象引用。
    ' 从此位置起写入代码：

    Dim user, pwd, A
    user = SmartTags("用户名")
    pwd  = SmartTags("密码")

    If user = "admin" And pwd = "admin" Then
        SmartTags("提示信息") = "登录成功"
        If SmartTags("提示信息") = "登录成功" Then
            A = 2
            ActivateScreenByNumber A, 0
        End If
    ElseIf user = "" And pwd = "" Then
        SmartTags("提示信息") = "错误代码 E0001：用户未登录"
        If SmartTags("提示信息") = "错误代码 E0001：用户未登录" Then
            A = 1
            ActivateScreenByNumber A, 0
        End If
    Else
        SmartTags("提示信息") = "错误代码 E0002：用户名或密码错误"
        If SmartTags("提示信息") = "错误代码 E0002：用户名或密码错误" Then
            A = 1
            ActivateScreenByNumber A, 0
        End If
    End If
End Sub
```

### 脚本关键语法解析

1. **条件跳转与画面编号变量 `A`**：
   - 通过 `If ... ElseIf ... Else` 判断账号密码：验证成功则将 `A` 设为 `2` 跳转至监控主页面（编号 2），为空或密码错误则写入对应错误提示信息并保持/返回页面 `1`。
   - `ActivateScreenByNumber A, 0` 是 WinCC VB 脚本内置系统函数，根据画面编号 `A` 切换当前显示页面。
   - 查看或修改页面编号 `A` 的方法：在项目树中右键点击目标画面 → 选择“属性”，即可在常规信息中看到该画面的默认编号（也可手动修改）：

     ![查看与修改画面编号](images/10-wincc-hmi-vbscript/fig-19.png)

2. **`Dim` 变量声明**：
   - `Dim` 类似于 C 语言中的 `int` 或 `double` 声明，用于声明一个或多个脚本局部变量并分配存储空间；
   - 由于 VBScript 是弱类型脚本语言，无需显式指定变量的数据类型。
   - **注意：编写脚本时务必确保引号、括号、逗号均为英文半角符号！**

---

### 常见问题：弹窗/文本显示不全的分辨率调整方法

在 Day 1 比赛中要求弹出提示显示 `错误代码 E0001：用户未登录` 等较长文本时，可能会遇到如图 3.1 所示**文字被截断显示不全**的问题（当时在赛场上一度非常头疼，赛后排查才发现是运行系统窗口分辨率过小所致）：

![图3.1 分辨率过小导致提示文字显示不全](images/10-wincc-hmi-vbscript/fig-20.png)
*图 3.1 文字显示不全现象*

可以按照下图步骤进入 **运行系统设置 → 画面 → 分辨率** 增大屏幕显示区域：

![图3.2 进入运行系统设置调整屏幕显示区域](images/10-wincc-hmi-vbscript/fig-21.png)
*图 3.2 增加屏幕显示区域*

![图3.3 调整画面布局尺寸](images/10-wincc-hmi-vbscript/fig-22.png)
*图 3.3 调整画面尺寸*

调整完成后的正常显示效果如下：

![调整分辨率后的完整显示效果](images/10-wincc-hmi-vbscript/fig-23.png)

WinCC 全部画面与脚本编辑完成后，点击顶部工具栏的 **启动仿真** 按钮即可联调测试：

![点击启动仿真按钮](images/10-wincc-hmi-vbscript/fig-24.png)

至此，WinCC 上位机框架搭建与 VB 脚本应用全部介绍完毕。

> 如有问题或建议，欢迎在本仓库提交 [Issue](../../issues) 交流。

---

[← 上一篇：群控调度算法](09-dispatch-algorithm.md) | [返回项目主页](../README.md) | [下一篇（附录）：群控电梯调度算法综述 →](appendix-dispatch-algorithms.md)
