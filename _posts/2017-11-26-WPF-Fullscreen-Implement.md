---
layout:     post
title:      WPF的程序的全屏实现
date:       2017-11-26
summary:    WPF全屏实现，WPF fullscreen implement
categories: jekyll pixyll
---

# WPF的程序的全屏实现

这个功能看起来是如此的简单，一般常规来说只要三局xaml代码即可。

``` xml
ResizeMode="NoResize"
WindowStyle="None"
WindowState="Maximized"
```

这个在Win10下绝大多数情况下是可行的，除非在看到软件界面前狂点任务栏。

在Win7中如果是以这样的正确方式操作：双击仅且双击打开、Enter打开、右键打开，这几种情况下也是没有问题的，但如果在Win7中快速三次点击鼠标，或者双击后，在看到界面前，点击了任务栏或者其他窗口，那就一定会出现任务栏覆盖，或者其他窗口覆盖软件程序，导致不能干净的全屏。

## 可能的解决方法

### Win32设置前置窗口

这个应该是网上最常见的方法了，因为现象看起来就是窗口没有前置，所以有人尝试如下设置代码后听说可行。

``` csharp
[DllImport("user32.dll")]
public static extern bool SetForegroundWindow(IntPtr hWnd);

SetForegroundWindow(new WindowInteropHelper(this).Handle);//调用
```

### 置顶窗口

想法和上面的方法是一致的，就是将窗口带到前面，相当于一个trick，有些人说ok，但针对Win7，可以将窗口盖住其他窗口，但还是被任务栏遮挡。

``` c#
//先置顶，再取消置顶
Application.Current.MainWindow.Topmost = true;
Application.Current.MainWindow.Topmost = false;
```

### 激活窗口

想法还是和上面一致，既然窗口在激活的时候会前置，那么设置一下激活就好了。

``` c#
//在显示之后激活，但在Win7中有时候还是会被其他窗口遮挡，更不用说任务栏了
Application.Current.MainWindow.ShowActivated = true;

//选择一个时机调用该方法进行激活，在Win7中将窗口前置，但还是会被任务栏遮盖
Application.Current.MainWindow.Activate();
```

### 最后的方法：隐藏任务栏

所以从上面几种方法来看，将窗口前置已经搞定，只要解决任务栏遮盖问题就好了。但经过各种偏方测试，Win7下的任务栏实在太顽强，没有找到方法解决遮盖问题。

因此只能用一个取巧的方法，由于窗口被任务栏遮盖的时候还不是Foreground状态，所以这个时候可以将任务栏隐藏，并再次激活窗口、窗口置顶、将窗口获取焦点，这样子应该就妥了。

``` c#
Taskbar.Hide();

Application.Current.MainWindow.Activate();
Application.Current.MainWindow.Topmost = true;
Application.Current.MainWindow.Topmost = false;
Application.Current.MainWindow.Focus();

Taskbar.Show();//延迟执行
```

Taskbar的显示隐藏的具体代码参考可以看[CodeProject文章](https://www.codeproject.com/Articles/25572/Hiding-the-Taskbar-and-Startmenu-start-orb-in-Wind)

### 最后的最后，上面的都错了

其实上面通过隐藏任务栏再显示任务栏的做法，如果在Win7下，打开软件的过程中，不停点击任务栏的话，问题还是存在的，所以就需要用到下面的方法了。

非常重要的一个点，见Solution4：https://www.codeproject.com/Questions/279441/Bring-Process-to-the-front

http://www.cnblogs.com/crazypebble/archive/2012/09/12/2681857.html

片段可用代码如下：

``` c#
[DllImport("user32.dll")]
public static extern uint GetWindowThreadProcessId(IntPtr hWnd, IntPtr ProcessId);

[DllImport("user32.dll")]
public static extern IntPtr GetForegroundWindow();

[DllImport("kernel32.dll")]
public static extern uint GetCurrentThreadId();

[DllImport("user32.dll")]
public static extern bool AttachThreadInput(uint idAttach, uint idAttachTo, bool fAttach);

[DllImport("user32.dll", SetLastError = true)]
public static extern bool BringWindowToTop(IntPtr hWnd);

[DllImport("user32.dll")]
public static extern bool ShowWindow(IntPtr hWnd, uint nCmdShow);

public static void ForceWindowToForeground(IntPtr hWnd)
{
    uint foreThread = GetWindowThreadProcessId(GetForegroundWindow(), IntPtr.Zero);
    uint appThread = GetCurrentThreadId();
    const uint SW_SHOW = 5;

    if (foreThread != appThread)
    {
        AttachThreadInput(foreThread, appThread, true);//用于欺骗Windows
        BringWindowToTop(hWnd);
        ShowWindow(hWnd, SW_SHOW);
        AttachThreadInput(foreThread, appThread, false);
    }
    else
    {
        BringWindowToTop(hWnd);
        ShowWindow(hWnd, SW_SHOW);
    }
}

//使用方法如下：
var handle = new WindowInteropHelper(this).Handle;
if (Win32.GetForegroundWindow() != handle)//判断当前为非前置窗口时才执行以下代码
{
    Log.Info("Current window is not foreground window");
    Activate();
    Win32.ForceWindowToForeground(new WindowInteropHelper(this).Handle);//最关键的代码
}
```



### 额外话题

1. [如果只想全屏后保持任务栏（没有测试过）](https://www.codeproject.com/articles/107994/taskbar-with-window-maximized-and-windowstate-to-n)
2. [分辨率改变后全屏窗口显示异常（没有测试过）](https://blog.onedevjob.com/2010/10/19/fixing-full-screen-wpf-windows/)





### 