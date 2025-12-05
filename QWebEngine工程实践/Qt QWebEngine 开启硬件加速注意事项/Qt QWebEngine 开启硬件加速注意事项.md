在实际项目中，`QWebEngine`相当大的一部分崩溃来自开启了硬件加速导致的。

**1、**软件渲染**和**硬件加速**的区别:**

| 项目             | 未开启硬件加速                 | 开启硬件加速                            |
| ---------------- | ------------------------------ | --------------------------------------- |
| **页面渲染性能** | 使用 CPU 软件渲染，速度较慢    | 使用 GPU 加速合成，页面滚动与动画更流畅 |
| **CPU 占用**     | 高，尤其是播放视频或复杂网页时 | 低，大部分绘制交由显卡完成              |
| **内存占用**     | 相对较低                       | 稍高（显存占用增加）                    |
| **能耗**         | CPU 功耗高                     | GPU 功耗高但效率更好                    |
| **兼容性**       | 稳定，几乎所有环境可运行       | 在部分旧显卡/虚拟机中可能不兼容         |
| **视频解码**     | 软件解码，容易卡顿             | GPU 硬件解码流畅播放 1080p/4K           |
| **UI 叠加窗口**  | 正常可叠加（QWidget/Dialogs）  | 部分平台下窗口叠加会闪烁或不显示        |
| **渲染延迟**     | 略高，滚动略卡顿               | 响应快、渲染延迟低                      |

简单说如果你的网页没有各种炫酷动画，只是静态的页面，其实利用软件渲染的模式，完全足够。

**2、开启硬件加速后的常见问题与解决方案:**

| 问题现象                                                     | 原因分析                                           | 解决方案                                                     |
| ------------------------------------------------------------ | -------------------------------------------------- | ------------------------------------------------------------ |
| **程序在部分机器启动崩溃（如 Intel 核显老显卡）**            | GPU 驱动兼容性差，QtWebEngine 调用 GPUContext 失败 | 禁用 GPU 黑名单：`--ignore-gpu-blacklist` 或退回 CPU 渲染    |
| **在 QWebEngineView 上叠加 QWidget/QDialog 时闪烁或无法显示** | GPU 合成层导致 Qt 窗口与 Chromium 窗口层冲突       | 使用独立窗口显示 QWebEngineView，或关闭硬件加速              |
| **远程桌面环境下网页空白、不渲染**                           | RDP 虚拟显卡不支持 GPU 加速                        | 启动时检测环境变量：`if (qEnvironmentVariableIsSet("SESSIONNAME")) disableGPU()` |
| **播放视频时画面绿屏或撕裂**                                 | 部分解码器与显卡驱动不兼容                         | 添加参数：`--disable-gpu-compositing` 或更新显卡驱动         |
| **程序退出卡顿或崩溃在 d3d11.dll**                           | Direct3D 资源释放异常                              | 在退出前调用 `QWebEngineView::close()` 并延时退出主线程      |
| **Linux 环境无法启用 GPU**                                   | 需要 EGL + VAAPI 支持                              | 启动参数加 `--use-gl=egl --enable-features=VaapiVideoDecoder` 并安装 VAAPI 驱动 |

在实际项目中，我们还会检测在同一台机器上如果`QWebEngine`多次崩溃，我们就会采用软件渲染模式，这是种不得已的方案。

**3、开启硬件加速:**

在 `main()` 函数中设置

```cpp

#include <QApplication>
#include <QWebEngineView>
#include <QWebEngineSettings>
#include <QtWebEngineCore/QWebEngineProfile>

int main(int argc, char *argv[])
{
    // 启用GPU加速参数
    QApplication::setAttribute(Qt::AA_UseOpenGLES);
    QApplication::setAttribute(Qt::AA_ShareOpenGLContexts);

    QApplication app(argc, argv);

    // 强制启用GPU加速
    QCoreApplication::setApplicationName("WebEngine GPU Test");
    QCoreApplication::setApplicationVersion("1.0");

    QWebEngineView view;
    view.page()->profile()->setHttpCacheType(QWebEngineProfile::DiskHttpCache);
    view.page()->profile()->setPersistentCookiesPolicy(QWebEngineProfile::AllowPersistentCookies);

    view.setUrl(QUrl("https://www.youtube.com")); // 用视频站测试是否GPU工作
    view.resize(1280, 720);
    view.show();

    return app.exec();
}
```

