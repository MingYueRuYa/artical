## 一、架构层坑点

### 🧩 1. 多进程模型复杂

`QWebEngine` 采用 `Chromium` 的多进程架构（主进程 + Renderer + GPU + Utility）。

调试困难，崩溃dump不在主进程；子进程路径固定、难以修改；多实例时子进程可能互相干扰。

建议：

- 使用 `QWebEngine::initialize()`前设置 `QCoreApplication::setApplicationName()`，隔离 `Profile`。

- 使用环境变量启用日志：

```cpp
set QTWEBENGINE_CHROMIUM_FLAGS=--enable-logging --v=1
```

子进程名称如需修改，需在编译时修改 QtWebEngine 模块（非官方支持）。

### ⚙️ 2. 不支持多实例独立运行

多个实例默认共用 `%LOCALAPPDATA%\QtWebEngine\` 目录。

导致 `cookie/session/GPU` 资源冲突。

建议：

```cpp
QWebEngineProfile *profile = new QWebEngineProfile("instance_profile", this);
```

## 二、性能层坑点

### 🚫 3. GPU 加速崩溃

某些显卡或远程桌面环境下崩溃报错：

```kFatalFailure: ES3 is blocklisted/disabled/unsupported driver```

原因：显卡驱动或虚拟化环境与 `Chromium` 不兼容。

建议：

```cpp
qputenv("QTWEBENGINE_CHROMIUM_FLAGS", "--disable-gpu");
QCoreApplication::setAttribute(Qt::AA_UseSoftwareOpenGL);
```

### 🧠 4. 内存占用高、释放不及时

每个页面独立渲染进程；

`delete QWebEngineView` 并不会立即释放内存；

多页面频繁创建/销毁导致内存暴涨。

建议：

- 复用单实例 `View`；

- 使用 `QWebEngineProfile::clearHttpCache() `定期清理；

- 避免频繁调用 `load()`或 `setHtml()`。

## 三、兼容层坑点

### 🧩 5. 高 DPI 字体与字间距异常

特别是俄文、中文等非拉丁字体，在高 DPI 下字距被放大。

建议：

```cpp
QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);
```

在 CSS 层控制字体，避免使用`“Microsoft JhengHei”`等字距不稳定字体。

### ⚠️ 6. Chromium 特性受限

不支持：WebRTC录屏、`WebUSB`、`WebSerial`、部分`ServiceWorker`；

不支持 `Chrome` 扩展。

建议：

- 用 `QtWebChannel` 替代扩展；

- 对接本地逻辑时通过 `QWebChannel` 通信。

## 四、部署层坑点

### 📦 7. 打包体积庞大

`QWebEngine` 带完整 `Chromium` 内核，打包后通常 >150MB；

缺少 .pak / .dat / QWebEngineProcess.exe 直接崩溃。

建议：

使用 windeployqt --qmldir . 自动收集；

保留所有 .pak 文件；

可尝试压缩 `QtWebEngineCore.dll`。

### 🧱 8. 沙箱路径问题

中文路径或特殊字符可能导致进程启动失败。

建议：

```cpp
set QTWEBENGINE_DISABLE_SANDBOX=1
```

或安装路径仅使用 ASCII。

实际项目我们都是沙箱属性关闭了。

## 五、API 层坑点

### 🔄 9. 加载信号触发不稳定

`loadStarted`、`loadFinished` 可能多次触发；

重定向或 iframe 加载不触发；

`urlChanged` 异步延迟。

建议：

- 使用 `loadProgress`；

- 或在网页注入 `JS`，通过 `QWebChannel` 主动回调。

## 六、其他隐蔽坑点

坑点 说明

叠加控件无效 `QWebEngineView` 为独立 GPU 图层，`QWidget` 透明控件无法覆盖PDF、视频播放崩溃 缺少媒体解码组件`Cookies` 不同步 与 `QNetworkAccessManager` 独立
代理设置不生效 需在进程启动前设置 QTWEBENGINE_CHROMIUM_FLAGS="--proxy-server=..."

## 💡 建议总结

对于多实例、高稳定性需求，可考虑 自编译 `QtWebEngine` 或 `CEF (Chromium Embedded Framework)`。

- 调试复杂问题时，使用 `--remote-debugging-port + Chrome DevTools`追踪内核日志。
- 避免中文路径、频繁销毁 `WebView`、过多同时打开页面。
- 若仅需轻量网页渲染，可替换为 `QTextBrowser` 或嵌入轻量 `WebView`组件。