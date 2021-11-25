---
type: docs
title: "本地调试"
linkTitle: "本地调试"
weight: 7
---

## Intellij IDEA Debug

如下是在IDEA中启动Debug模式，进入断点后的界面，我这里是Windows，可能和Mac的图标等会有些不一样。就简单说下图中标注的8个地方：

- ① 以Debug模式启动服务，左边的一个按钮则是以Run模式启动。在开发中，我一般会直接启动Debug模式，方便随时调试代码。
- ② 断点：在左边行号栏单击左键，或者快捷键Ctrl+F8 打上/取消断点，断点行的颜色可自己去设置。
- ③ Debug窗口：访问请求到达第一个断点后，会自动激活Debug窗口。如果没有自动激活，可以去设置里设置，如图1.2。
- ④ 调试按钮：一共有8个按钮，调试的主要功能就对应着这几个按钮，鼠标悬停在按钮上可以查看对应的快捷键。在菜单栏Run里可以找到同样的对应的功能，如图1.4。
- ⑤ 服务按钮：可以在这里关闭/启动服务，设置断点等。
- ⑥ 方法调用栈：这里显示了该线程调试所经过的所有方法，勾选右上角的[Show All Frames]按钮，就不会显示其它类库的方法了，否则这里会有一大堆的方法。
- ⑦ Variables：在变量区可以查看当前断点之前的当前方法内的变量。
- ⑧ Watches：查看变量，可以将Variables区中的变量拖到Watches中查看

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20211124230835.png" style="display:block;width:80%;" alt="NAME" align=center /> </div>

在设置里勾选Show debug window on breakpoint，则请求进入到断点后自动激活Debug窗口

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20211124230857.png" style="display:block;width:80%;" alt="NAME" align=center /> </div>

## 基本应用

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20211124232202.png" style="display:block;width:50%;" alt="NAME" align=center /> </div>

`Show Execution Point` (Alt + F10)：如果你的光标在其它行或其它页面，点击这个按钮可跳转到当前代码执行的行。

`Step Over` (F8)：步过，一行一行地往下走，如果这一行上有方法不会进入方法。

`Step Into` (F7)：步入，如果当前行有方法，可以进入方法内部，一般用于进入自定义方法内，不会进入官方类库的方法，如第25行的put方法。

`Force Step Into` (Alt + Shift + F7)：强制步入，能进入任何方法，查看底层源码的时候可以用这个进入官方类库的方法。

`Step Out` (Shift + F8)：步出，从步入的方法内退出到方法调用处，此时方法已执行完毕，只是还没有完成赋值。

`Drop Frame` (默认无)：回退断点，后面章节详细说明。

`Run to Cursor` (Alt + F9)：运行到光标处，你可以将光标定位到你需要查看的那一行，然后使用这个功能，代码会运行至光标行，而不需要打断点。

`Evaluate Expression` (Alt + F8)：计算表达式，后面章节详细说明。

