# QtWebEngine 自动重启方案

在实际项目中不可避免的会遇到QWebengine崩溃和假死的问题。
在无法避免的情况下，我们一种可靠的机制能够重启。
由于QtWebEngine 使用多进程架构，渲染进程由 QWebEngineProcess.exe 负责。当渲染进程崩溃时，主程序不会崩溃，但界面会卡死或显示“渲染进程崩溃”。

同时有个关键点就是**数据必须保存在c++层面**，这样QtWebengine崩溃了界面的数据还保存，重启之后传递数据即可。

### 一、检测 QtWebEngine 渲染进程是否崩溃

Qt 自带 renderProcessTerminated 信号，可监控渲染进程异常退出。

```cpp
connect(view->page(), &QWebEnginePage::renderProcessTerminated, this, &MainWindow::onRenderProcessCrashed);
void MainWindow::onRenderProcessCrashed(
    QWebEnginePage::RenderProcessTerminationStatus status,
    int exitCode)
{
    qWarning() << "QWebEngine render process crashed:"
               << status << exitCode;
    restartWebEngine();
}
```



### 二、实现自动重启（核心逻辑）

由于 QtWebEngine 初始化比较复杂，我们不建议在主线程直接重建 QtWebEngineView。
最稳定的方式是：
删除原有 WebView延迟一小段时间（比如 200~500ms）
全新创建 QWebEngineView + QWebEnginePage恢复到用户原来的页面或状态

```cpp
void MainWindow::restartWebEngine()
{
    qInfo() << "Restarting QWebEngine...";
    if (view) {
        view->deleteLater();
        view = nullptr;
    }
    QTimer::singleShot(300, this, [this]() {
        initWebEngine();
    });
}
```

初始化函数：


```cpp

void MainWindow::initWebEngine()
{
    view = new QWebEngineView(this);
    // 必须重新创建 QWebEngineProfile，否则会继续使用崩溃的缓存
    QWebEngineProfile *profile = new QWebEngineProfile(this);
    QWebEnginePage *page = new QWebEnginePage(profile, view);
    view->setPage(page);
    connect(page, &QWebEnginePage::renderProcessTerminated,
            this, &MainWindow::onRenderProcessCrashed);
    view->load(QUrl(currentUrl));
    layout()->addWidget(view);
}
```

### 三、增强：自动捕获 GPU 崩溃并重启

Chromium GPU 进程崩溃不会触发上面的信号。
我们可以通过以下方式检测：

1. 监听 QWebEnginePage::loadFinished(false)
2. 检查 Chromium 日志中是否出现 GPU 崩溃条目
3. 自动重启 WebEngine

最简单可落地方案：

```cpp

connect(page, &QWebEnginePage::loadFinished, this,
        [this](bool ok) {
            if (!ok) {
                qWarning() << "WebEngine load failed, restart.";
                restartWebEngine();
            }
        });
```
### 四、增强：强制终止崩溃的 QWebEngineProcess

有时候 QWebEngineProcess 出现僵尸状态，不会自动退出。我们可以主动杀掉它：

```cpp

void killWebEngineProcess()
{
#ifdef Q_OS_WIN
    system("taskkill /im QtWebEngineProcess.exe /f >nul 2>nul");
#endif
}
```

在重启前执行：
```cpp
void MainWindow::restartWebEngine()
{
    killWebEngineProcess();
    ...
}
```

### 五、增强稳定性：避免使用旧缓存

如果 WebEngine 反复崩溃，通常是缓存损坏，我们可以在崩溃后自动清理：

```cpp
QDir("userdata/").removeRecursively();
```

或者强制使用临时 Profile：

```
profile->setPersistentStoragePath(QStandardPaths::writableLocation(        QStandardPaths::TempLocation) + "/qtwebengine_tmp");
```

### 六、总结

要实现 QWebEngine 的自动重启，核心要点是：

1. 捕获 renderProcessTerminated 信号
2. 删除旧视图，延迟重建新实例
3. 避免复用崩溃的 Profile
4. 必要时杀僵尸进程
5. 加载失败也触发自动恢复