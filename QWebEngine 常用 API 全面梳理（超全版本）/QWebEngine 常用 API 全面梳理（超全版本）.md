## QWebEngine 常用 API 全面梳理（超全版本）

Qt WebEngine 基于 Chromium，但提供了 Qt 风格的 API。本文对 QWebEngine 的常用类与 API 进行系统梳理，帮助你快速掌握其开发全景。

### 1. QWebEngineView（视图层）

QWebEngineView 是最常用的 UI 控件，主要负责：

- 加载 URL
- 渲染 HTML
- 对接 QWebEnginePage
- 提供滚动、缩放、键鼠事件等

#### 1、常用 API

| 功能               | API                                                       |
| ------------------ | --------------------------------------------------------- |
| 加载网页           | `load(QUrl)`                                              |
| 设置页面对象       | `setPage(QWebEnginePage*)`                                |
| 获取页面对象       | `page()`                                                  |
| 加载 HTML 字符串   | `setHtml(QString html, QUrl baseUrl)`                     |
| 后退 / 前进 / 刷新 | `back()` / `forward()` / `reload()`                       |
| 放大缩小           | `setZoomFactor()`                                         |
| 触发打印           | `print()` / Qt6: `printToPdf()`                           |
| 获取标题           | `titleChanged` signal                                     |
| URL 变化           | `urlChanged` signal                                       |
| 右键菜单           | 重写 `contextMenuEvent()` 或设置 `setContextMenuPolicy()` |

### 2. QWebEnginePage（页面控制层）

页面逻辑核心，包括：

- 页面事件
- 权限控制（地理位置、摄像头、麦克风等）
- JS 执行
- 导航控制
- 下载、拦截等

#### 1、常用 API

| 功能         | API                                                          |
| ------------ | ------------------------------------------------------------ |
| 加载控制     | `acceptNavigationRequest()`                                  |
| 权限管理     | `featurePermissionRequested` / `setFeaturePermission()`      |
| JS 执行      | `runJavaScript()`                                            |
| 对话框回调   | `javaScriptAlert()` `javaScriptConfirm()` `javaScriptPrompt()` |
| 新窗口创建   | `createWindow(type)`                                         |
| 拦截链接打开 | `acceptNavigationRequest()`                                  |
| 截屏         | `grab()`（Qt6）                                              |
| 设置 Profile | `setProfile()`                                               |
| 下载事件     | `downloadRequested(QWebEngineDownloadItem*)`                 |

#### 2、常用信号

- `loadStarted`
- `loadProgress`
- `loadFinished(bool)`
- `renderProcessTerminated`
- `windowCloseRequested`
- `fileDialog()` — 处理 `<input type="file">` 上传

------

## 3. QWebEngineProfile（浏览器用户环境层）

类似于 Chrome 的用户配置文件，用于：

- cookie/storage 缓存
- 代理配置
- 持久化路径
- 全局网络设置
- 跨站策略

#### 1、常用 API

| 功能         | API                                   |
| ------------ | ------------------------------------- |
| Cache 路径   | `setCachePath()` / `cachePath()`      |
| Storage 路径 | `setPersistentStoragePath()`          |
| HTTP UA      | `setHttpUserAgent()`                  |
| CookieStore  | `cookieStore()`                       |
| 默认 Profile | `QWebEngineProfile::defaultProfile()` |
| 禁用持久化   | `setPersistentCookiesPolicy()`        |

#### 2、代理设置

```
profile->setHttpProxy(QNetworkProxy(QNetworkProxy::HttpProxy, "proxy.com", 8080));
```

------

### 4. QWebEngineSettings（全局与页面设置）

可控制 **渲染、JS、插件、安全** 等能力。

#### 2、常用开关

| 设置项           | API                               |
| ---------------- | --------------------------------- |
| 是否允许 JS      | `setAttribute(JavascriptEnabled)` |
| 是否允许本地存储 | `LocalStorageEnabled`             |
| 是否允许跨域读取 | `HyperlinkAuditingEnabled`        |
| 是否允许插件     | `PluginsEnabled`                  |
| WebGL            | `WebGLEnabled`                    |
| 加载图片         | `AutoLoadImages`                  |
| 缩放             | `setDefaultTextEncoding()`        |

#### 2、设置示例

```cpp
auto s = view->settings();
s->setAttribute(QWebEngineSettings::JavascriptEnabled, true);
s->setAttribute(QWebEngineSettings::PluginsEnabled, true);
s->setAttribute(QWebEngineSettings::LocalContentCanAccessRemoteUrls, true);
```

------

### 5. QWebEngineCookieStore（Cookies 管理）

主要用于：

- 获取 Cookies
- 添加/删除 Cookies
- 监听 Cookies 变化

#### 1、常用 API

| 功能             | API                             |
| ---------------- | ------------------------------- |
| 读取所有 cookies | `loadAllCookies()`              |
| 添加 Cookie      | `setCookie(QNetworkCookie)`     |
| 删除 Cookie      | `deleteCookie()`                |
| Cookie 变化回调  | `cookieAdded` / `cookieRemoved` |

------

### 6. QWebEngineScript / ScriptCollection

用于向页面注入 JS。

#### 1、常用 API

| 功能             | API                                                    |
| ---------------- | ------------------------------------------------------ |
| 添加脚本         | `scripts().insert(QWebEngineScript)`                   |
| 移除脚本         | `scripts().remove()`                                   |
| 执行时机         | `QWebEngineScript::DocumentReady` / `DocumentCreation` |
| 注入到每个 frame | `setRunsOnSubFrames(true)`                             |

脚本示例：

```cpp
QWebEngineScript script;
script.setName("inject-js");
script.setInjectionPoint(QWebEngineScript::DocumentReady);
script.setSourceCode("console.log('hello');");
view->page()->scripts().insert(script);
```

------

### 7. QWebEngineHistory

浏览器历史记录。

#### 1、常用 API

| 功能           | API                          |
| -------------- | ---------------------------- |
| 访问条目       | `QWebEngineHistory::items()` |
| 后退/前进      | `back()` / `forward()`       |
| 返回特定 index | `goToItem()`                 |
| 清空历史       | `clear()`                    |

------

### 8. QWebEngineDownloadItem（下载管理）

用于处理文件下载，类似于 Chrome 的下载流程。

#### 1、常用 API

| 功能         | API                            |
| ------------ | ------------------------------ |
| 设置保存路径 | `setPath()`                    |
| 接受下载     | `accept()`                     |
| 取消         | `cancel()`                     |
| 下载进度     | `receivedBytes` / `totalBytes` |

示例：

```cpp
connect(profile, &QWebEngineProfile::downloadRequested,
        [](QWebEngineDownloadItem *item){
            item->setPath("download.bin");
            item->accept();
        });
```

------

### 9. QWebEngineUrlRequestInterceptor / Handler

常用于：

- 网络拦截
- 修改 header
- 屏蔽广告
- 自定义协议

#### 1、核心 API

| 功能         | API                               |
| ------------ | --------------------------------- |
| 拦截 URL     | `interceptRequest()`              |
| 修改 header  | `setHttpHeader()`                 |
| 拦截某些协议 | 使用 `QWebEngineUrlSchemeHandler` |

------

### 10.总结

![](./01.png)
