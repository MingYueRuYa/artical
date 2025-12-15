## Qt QtWebEngine 白屏的解决方案

最近在项目中有同事反馈，软件在开启的瞬间和长时间挂机之后，会出现白屏的现象。

先来看看白屏的常见原因和解决方案

**1、QtWebEngine** **白屏最常见的 5 大原因和解决方案：**

| 主要原因                  | 解决方式                          |
| ------------------------- | --------------------------------- |
| GPU 加速问题              | 禁用 GPU、使用 Software OpenGL    |
| QtWebEngineProcess 未启动 | 检查目录结构、权限、杀软拦截      |
| 资源路径错误              | 开启 DevTools、自查 qrc/file 路径 |
| OpenGL 不完整             | 用 Software OpenGL 或 ANGLE       |
| 沙箱限制                  | 禁用 WebEngine sandbox            |

**2、下面按类别给出详细排查和解决方案。**

**1、在开机的瞬间造成的白屏现象：**

其实是QtWebEngine还未加载完成，可以接受QtWebEngine加载完成信号，在接受完成之后在现实界面。

```cpp

connect(view, &QWebEngineView::loadFinished, this, [=](bool ok){
    if(ok) {
       emit webLoadFinished();
    } else {
       emit webLoadError();
    }
});
```

**2、GPU / 硬件加速导致的白屏（最常见）**

在下列场景中很容易出现白屏：

- 显卡驱动过旧
- 虚拟机（VMware/VirtualBox）
- Win7 + 旧 Intel 显卡
- 服务器环境 / 不支持 GPU
- QtWebEngine 被 GPU blocklist 限制

### 解决方案：禁用硬件加速（试试就好用）

```cpp
QCoreApplication::setAttribute(Qt::AA_UseSoftwareOpenGL);
QWebEngineSettings::globalSettings()->setAttribute(
        QWebEngineSettings::Accelerated2dCanvasEnabled, false);
QWebEngineSettings::globalSettings()->setAttribute(
        QWebEngineSettings::WebGLEnabled, false);
```

或启动参数：

```cpp
QCoreApplication::setAttribute(Qt::AA_ShareOpenGLContexts);
QApplication app(argc, argv);

QtWebEngine::initialize();
QWebEngineProfile::defaultProfile()->setHttpUserAgent("Chrome"); // 防止兼容模式
```

单独禁用 GPU：

```cpp
QWebEngineCommandLine::setArguments({"--disable-gpu", "--disable-software-rasterizer"});
```

**3、QWebEngineProcess 子进程无法正常启动**

```cpp
Failed to start QWebEngineProcess
Render process terminated
```

通常原因：

- 程序目录无写入权限（C:\Program Files）
- 被杀毒软件阻止
- 程序打到 UAC 限制目录
- 同目录多个版本 Qt 混用

### 解决方案：

#### （1）给程序目录写权限

```bat
icacls "%~dp0" /grant Everyone:F /T
```

#### （2）确保 `QWebEngineProcess.exe` 在运行目录：

结构必须是：

```bat
MyApp.exe
QtWebEngineProcess.exe
QtWebEngineCore.dll
resources\
    locales\
```

#### （3）程序启动时检查：

```cpp
if (!QFile::exists(qApp->applicationDirPath() + "/QtWebEngineProcess.exe")) {
    qDebug() << "Missing QWebEngineProcess";
}
```

### 4、资源路径错误（QRC / 本地文件未加载

如果你的页面是：

- qrc:/xxx
- file:///
- 本地 HTML + JS + CSS

白屏可能是静态资源没有加载成功。

### **解决方案：**

#### （1）开启调试日志查看资源是否 404

```cpp
QWebEngineSettings::defaultSettings()->setAttribute(
    QWebEngineSettings::DeveloperExtrasEnabled, true);
```

按 `F12` 看 Network。

#### （2）检查资源是否打进资源包

```cpp
<RCC>
  <qresource prefix="/">
    <file>html/index.html</file>
    <file>html/js/app.js</file>
  </qresource>
</RCC>
```

（3）使用 QUrl::fromLocalFile

```cpp
view->load(QUrl::fromLocalFile("C:/app/resources/index.html"));
```

# **5、OpenGL 环境不完整**

症状：

- 新显卡 + 旧驱动
- Windows Server 系统（默认无 GPU）
- 远程桌面（RDP）导致 GPU 被禁用
- Qt 报错 “OpenGL context failed”

###  **解决方案**：

#### （1）使用 ANGLE 或软件渲染

```cpp
QCoreApplication::setAttribute(Qt::AA_UseSoftwareOpenGL);
```

#### （2）在远程桌面必须在控制台运行

尝试：

```cpp
mstsc /admin
```

# **6、沙箱 / 安全策略导致无法运行**

症状：

- win7 权限低
- 公司电脑有 EDR/杀软
- windows defender 限制

### **解决方案：**

禁用 WebEngine 沙箱：

```cpp
QWebEngineCommandLine::setArgument("--no-sandbox");
```

或使用：

```cpp
qputenv("QTWEBENGINE_DISABLE_SANDBOX", "1");
```

## **总结：**

1. **确认 GPU 是否导致**
   强制禁用 GPU，一般 90% 可解决。
2. **查看 QtWebEngineProcess 是否正常启动**
   打开任务管理器观察是否有 `QtWebEngineProcess.exe`。
3. **打开 F12 调试工具是否能看到资源加载**
   Network 是否是 404 或 pending。
4. **看日志output.log**

```
set QTWEBENGINE_CHROMIUM_FLAGS=--enable-logging --v=1
```

   5.**尝试加载远程网页测试网络**

```
view->load(QUrl("https://www.baidu.com"));
```
