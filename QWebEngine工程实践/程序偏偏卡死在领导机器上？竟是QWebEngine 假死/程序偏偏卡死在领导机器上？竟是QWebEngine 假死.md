飞书上领导发来消息说：过来看下，程序好像卡死了。

我硬着头皮小跑过去。熟练的利用工具生成dump，生成好dump马上挠了挠头的说了一句马上分析尽快给出结果。

立马打开WinDbg分析堆栈，还是熟悉的配方。

![](.\01.png)

这个堆栈看得出来是卡死在gpu上面。



在使用 Qt QWebEngine 进行桌面端开发时，许多开发者都会遇到这样一个问题：**程序并没有崩溃，但界面完全不响应，窗口标题栏显示“未响应”，功能卡住无法操作——这就是典型的“假死”。**

# **一、什么是假死？**

在桌面应用中，**假死（Hang / Freeze / Not Responding）** 指的是：

> **程序进程仍在运行（没有退出），但主线程停止处理消息或进入阻塞，导致界面无法交互。**

其表现包括：

- 窗口标题栏显示 *Not Responding / 未响应*
- 点击界面无反应、窗口无法拖动
- UI 不更新、画面卡住
- CPU 使用率可能为 0%，也可能为 100%（主线程忙循环）
- 程序未崩溃，也未退出，只是“不动了”

与崩溃不同的是：
**崩溃会自动退出；假死则是“挂着但不能用了”。**

# **二、QWebEngine 为什么容易发生假死？**

Qt QWebEngine 基于 Chromium，多进程架构复杂，因此它的假死常见于以下问题：

------

## **① GPU 进程卡死（最常见）**

- WebGL、大量 Canvas 操作、视频解码等引发 GPU 线程冻结
- GPU 进程挂掉后，主进程会等待 IPC 响应
- 导致主线程阻塞 → 假死

**典型现象：**

- 内存、显存暴涨
- 控制台出现 GPU error
- 页面渲染不刷新

------

## **② WebEngine IPC 阻塞**

QWebEngine 主进程和 renderer/gpu 进程间大量依赖 IPC。
如果某个子进程无响应，就会导致：

- 多个 IPC 消息堆积
- 主线程等待回调
- UI 线程停止响应

------

## **③ JavaScript 执行堵塞**

例如页面中：

- 大量长时间 JS 循环
- WebAssembly 高负载计算
- 大量 DOM 操作阻塞 UI 线程

Chromium 渲染线程被卡住，UI 间接卡死。

## ** **

------

# **三、如何解决 QWebEngine 假死？**

下面给出可直接落地的两个解决方案:

------

## **方案一：使用 SendMessageTimeout 检测假死并自动重启（推荐）**

这是最稳定的一种方式：

> **通过向窗口发送 WM_NULL 消息，如果主线程超过一定时间没有响应，就认为程序假死。**

优点：

- 不依赖业务逻辑
- 不依赖 QWebEngine 内部机制
- 对所有窗口都适用
- 实现极其稳定（很多商业软件就是这么做的）

------

## **方案二：使用心跳机制检测 QWebEngine 是否卡住**

在 QWebEngine 主线程（UI 线程）中用 `QTimer` 定期更新心跳；
Watchdog 线程检测心跳是否过期。

如果超过阈值 → 认为主线程假死 → 重启浏览器子进程 or 重启程序。

------

# **四、基于 SendMessageTimeout 的假死检测方案（含完整 Demo）**

下面给出一个 **可直接用于生产环境的 Windows C++ 假死检测 Demo**。

### **关键点：**

我们用 `SendMessageTimeout` 向窗口发送 `WM_NULL`：

```cpp
SendMessageTimeout(hwnd,    
                   WM_NULL,0,0,
                   SMTO_ABORTIFHUNG | SMTO_NORMAL,1000, //毫秒
                   &result)
```

如果 1 秒内，窗口没有处理这个消息 → 认为窗口“未响应（假死）”。

------

# **完整 Demo：检测 QWebEngine 主窗口是否假死**

以下代码可作为外部监控程序，也可嵌入到启动器中：

```cpp
#include <windows.h>
#include <iostream>

bool IsWindowHung(HWND hwnd, DWORD timeoutMs)
{
    DWORD_PTR result = 0;

    LRESULT r = SendMessageTimeout(
        hwnd,
        WM_NULL,     // 空消息，只用来检测响应
        0,
        0,
        SMTO_ABORTIFHUNG | SMTO_NORMAL,
        timeoutMs,
        &result
    );

    return (r == 0); // r 为 0 表示超时或失败
}

int main()
{
    // 假设你已经拿到了 QWebEngine 主窗口句柄
    // 示例：假设窗口标题为 "MyQtBrowser"
    HWND hwnd = FindWindow(NULL, L"MyQtBrowser");
    if (!hwnd) {
        std::cout << "找不到窗口\n";
        return 0;
    }

    while (true) {
        if (IsWindowHung(hwnd, 1500)) { // 1.5s 超时
            std::cout << "[警告] 窗口疑似假死，准备重启...\n";

            // 这里可执行：
            // 1. 生成 dump
            // 2. Kill 进程
            // 3. CreateProcess 重启应用
            // 4. 上报日志

            // 示例：直接关闭（危险，生产需加保护）
            PostMessage(hwnd, WM_CLOSE, 0, 0);
            break;
        } else {
            std::cout << "[正常] 窗口正常响应\n";
        }

        Sleep(1000); // 每 1 秒检测一次
    }
    return 0;
}
```

# **如何与 Qt 配合？**

在 Qt 主窗口中：

```cpp
setWindowTitle("MyQtBrowser");
```

然后监控器就能检测到该窗口是否处于假死状态。