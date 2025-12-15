# Qt WebEngine Demo

一个基于 Qt 6 + WebEngineWidgets 的示例应用，展示以下能力：

- Web 与 C++ 之间的双向通信（继承抽象基类 `WebBridge` 即可完成交互）
- 一键清理缓存 / Cookie / 历史记录（`QWebEngineProfile`）
- 动态调整窗口透明度与不透明度
- 启用多项 WebEngine 常用特性（JavaScript、WebGL、Clipboard、LocalContent 等）
- 通过 Chromium Flags 强制开启 GPU 合成与 GPU 光栅化
- 可通过可执行目录下的 `config.json` 指定 WebEngine 远程调试端口
- 页面加载完成前发送给网页的消息会自动缓存，待页面通知 C++ 已就绪后批量发送
- `WebEngineSignals` 工具类可一次性绑定 QWebEngineView/Page 的常用信号，方便在其它类中继承复用
- 独立消息面板负责 Web ↔ C++ 消息收发与日志记录

> 如需 Qt 5，请自行将 `find_package(Qt6 ...)` 改成 `Qt5` 并将链接库替换成 `Qt5::` 前缀。

## 目录结构

```
.
├── CMakeLists.txt
├── resources.qrc
├── src
│   ├── browserwindow.cpp/.h      # UI 逻辑
│   ├── messageconsole.cpp/.h     # Web 消息收/发面板
│   ├── webenginepane.cpp/.h      # 封装 QWebEngineView / Profile
│   ├── main.cpp                  # 程序入口
│   └── webbridge.cpp/.h          # WebBridge 基类 + BasicBridge 默认实现
└── web
    └── index.html            # Demo 页面，引用 qwebchannel.js
```

## 使用 Visual Studio 2022

1. 安装 **Visual Studio 2022**（含 Desktop development with C++ 工作负载）。
2. 安装 **Qt VS Tools** 扩展，并在 VS 中注册你的 Qt for MSVC 套件（例如 `6.6.3_msvc2019_64`）。
3. 双击 `WebEngineDemo.sln` 打开解决方案。
4. 若 Qt VS Tools 中注册的名称不同，编辑 `WebEngineDemo.vcxproj` 里的 `<QtInstall>` 值或在 VS 的“Qt 项目设置”里选择正确的版本。
5. 直接 `F5`/`Ctrl+F5` 运行，或从配置管理器选择 Debug / Release。

项目使用 `QtMsBuild` 自动处理 MOC/RCC，若提示找不到 `QtMsBuild`，请确认扩展已安装并在 `Qt VS Tools -> Manage Qt Versions` 中点“Update QtMsBuild”。

### 硬件加速/光栅化

程序入口会在启动前设置：

- `Qt::AA_UseDesktopOpenGL` 与共享 OpenGL 上下文，确保使用硬件 OpenGL；
- 自定义 `QSurfaceFormat`（Core Profile 4.5、三缓冲、VSync）；
- `QTWEBENGINE_CHROMIUM_FLAGS` 包含 `--ignore-gpu-blocklist --enable-gpu-rasterization --enable-zero-copy` 等参数，强制使用 GPU 合成、GPU 光栅化及硬件视频解码。

如需进一步调试，可通过设置 `QT_LOGGING_RULES="qt.webengine.*=true"` 或修改上述 flags。

## 使用 CMake（可选）

仍然保留跨平台 CMake 构建，可用于命令行或 CLion/Qt Creator：

1. 准备 Qt 6 + CMake ≥ 3.21，并设置 `Qt6_DIR` 或 `CMAKE_PREFIX_PATH`。
2. 生成工程：
   ```powershell
   cmake -S . -B build -DCMAKE_PREFIX_PATH="C:/Qt/6.6.3/msvc2019_64"
   ```
3. 编译与运行：
   ```powershell
   cmake --build build --config Release
   build/Release/WebEngineDemo.exe
   ```

运行后即可在工具栏中体验：

- 主页按钮加载内置 `index.html`
- “清空缓存” 清理当前 profile 的缓存/Cookie
- “透明模式 + 滑块” 控制窗口透明度
- “消息面板” 聚合 Web ↔ C++ 消息；在底部面板输入消息直接发送到网页，网页返回的信息也会记录在同一面板
- 网页输入框可把文本送回 C++，必要时还会弹出 MessageBox 提示
- Debug 构建默认设置 `QTWEBENGINE_REMOTE_DEBUGGING=9223`（若 `config.json` 未指定端口），可用 Chrome DevTools 连接 `http://127.0.0.1:<端口>`
- 若页面尚未完成加载，C++ 发送的消息会缓存在队列中；网页在 `QWebChannel` 建立后会主动调用 `bridge.notifyPageReady()` 告知 C++ 已就绪，此时缓冲的消息会按顺序发送到 JS。
- 想统一监听 QWebEngine 事件时，可继承 `WebEngineSignals` 并调用 `bind(QWebEngineView*)`，即可收到加载、权限、下载、协议注册等信号的集中转发。

## 配置文件（config.json）

程序启动时会在可执行文件所在目录查找 `config.json`，当前支持以下字段：

- `remoteDebugPort`：整数端口，若存在且有效，将自动设置 `QTWEBENGINE_REMOTE_DEBUGGING`，无论 Debug 还是 Release。

示例：

```json
{
    "remoteDebugPort": 9333
}
```


## 继承一个类即可完成通信

`WebBridge` 是一个抽象基类，负责声明 Web/C++ 通信的统一接口，并自动暴露到 `QWebChannel` 中：

- JS 侧调用 `bridge.sendToCpp(payload)`，会触发 `WebBridge::sendToCpp`，再回调 `virtual void onMessageFromWeb(const QString &payload)`；
- C++ 侧调用 `dispatchToWeb(payload)`（或在子类中调用 `sendToWeb` 的封装）即可把字符串广播回 JS，对应信号 `messageFromCpp`；
- 默认实现 `BasicBridge` 只是一个空壳，示例通过 `WebEnginePane(new BasicBridge, this)` 安装它；
- 若需要自定义协议，只需继承 `WebBridge` 并覆写 `onMessageFromWeb()` / `onMessageFromCpp()`，再把实例交给 `WebEnginePane` 即可，无需重复配置 `QWebChannel`。

示例：

```cpp
class CustomBridge : public WebBridge {
    Q_OBJECT
protected:
    void onMessageFromWeb(const QString &payload) override {
        if (payload == "ping") {
            dispatchToWeb("pong");
        }
    }
};

auto *pane = new WebEnginePane(new CustomBridge, parent);
```

