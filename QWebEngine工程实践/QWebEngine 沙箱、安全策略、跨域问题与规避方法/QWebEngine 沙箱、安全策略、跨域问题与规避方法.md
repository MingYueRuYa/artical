## QWebEngine 沙箱、安全策略、跨域问题与规避方法

（附实际案例 + 调试技巧 + 可复现 Demo）

在基于 QWebEngine 的浏览器或嵌入式 WebView 项目中，**安全策略、跨域限制、沙箱隔离**是绕不过去的关键主题。
 了解它们如何工作、何时被触发、如何合法规避，能避免 80% 的 WebView 异常行为，例如：

- 本地文件无法访问网络资源
- iframe 加载失败、JS 被阻止
- CORS 报错
- mixed-content 被自动拦截
- JS 无法访问 QWebChannel 对象
- 页面出现 `Blocked by client` 或 `Unsafe attempt to load ...`

本文将全面梳理 QWebEngine 的沙箱、安全策略与跨域行为，并给出**可直接运行的 Qt/C++ Demo 代码**。

# 1. QWebEngine 的安全模型基础

QWebEngine 继承了 Chromium 的安全体系，主要包括：

### ✓ 1.1 沙箱模式（Sandbox）

- Chromium 的子进程（Renderer、GPU 等）默认在沙箱中运行
- 限制系统调用、网络访问、文件操作
- Qt 很少允许关闭沙箱（Qt 6 有开关，Qt 5 大多禁用）

> **典型现象**
>
> - 加载本地 HTML 后无法访问文件系统
> - JS 访问摄像头、麦克风需要权限
> - file:// 下禁止访问远程 http:// 或 https://

------

### ✓ 1.2 Content Security Policy（内容安全策略）

Chromium 自动触发以下策略：

- 禁止 HTTP 页面加载 HTTPS（mixed-content）
- 禁止跨域读取 response（CORS）
- iframe、script，img 的加载依赖来源规则

------

### ✓ 1.3 QWebEngine 专有安全策略

| 条目               | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| 本地资源访问策略   | QWebEngine 默认禁止 file:// 访问 remote 或 file://跨目录访问 |
| JS 与 C++ 通信安全 | QWebChannel 对象必须在主 frame 注册                          |
| URL Scheme 权限    | 自定义协议需要注册访问类型（Local / Secure / Standard）      |

------

# 2. 沙箱/安全策略带来的典型问题与案例

下面是实际**可复现的典型错误**。

------

## 案例 1：file:// 页面加载跨域资源失败

你有一个本地页面：

```
file:///C:/test/index.html
```

里面引用：

```
<script src="https://cdn.jsdelivr.net/npm/vue@3"></script>
```

结果 QWebEngine 控制台报错：

```
Not allowed to load local resource
Mixed Content: The page at 'file://' was loaded over file://, 
but requested https://cdn.... This request has been blocked.
```

**原因：本地文件 file:// 被视为“最高权限”，违规加载外域资源会被拒绝。**

------

## 案例 2：iframe 加载失败（Blocked by sandbox）

```
<iframe src="https://example.com"></iframe>
```

提示：

```
Unsafe attempt to load URL
```

原因：iframe 与主页面不同源，受到 **CORS + sandbox + X-Frame-Options** 限制。

------

## 案例 3：JS 无法访问 QWebChannel（object undefined）

HTML：

```
new QWebChannel(qt.webChannelTransport, (channel) => {
    window.bridge = channel.objects.bridge;
});
```

报错：

```
qt is undefined
```

原因：启用了严格 CSP 或加载方式为 `file://` 下的 iframe，导致注入脚本被阻止。

------

## 案例 4：XHR/Fetch 跨域失败（CORS）

```
fetch("https://api.xxx.com/data")
    .then(r => r.json())
```

报错：

```
CORS policy: No 'Access-Control-Allow-Origin' header present
```

同浏览器，但在 QWebEngine 特别常见，因为：

- 你可能加载的是本地 html
- 或未使用 HTTPS
- 或服务器未设置 CORS

------

# 3. 安全策略规避方案（官方合法手段）

下面给出实际工程中可用的规避（绕过）策略。

------

# 3.1 允许本地文件访问外部资源（file:// → http/https）

> Qt 5/6 默认禁用。

解决方法：
 使用 **Profile Settings** 打开以下开关：

```
QWebEngineProfile* profile = QWebEngineProfile::defaultProfile();

profile->settings()->setAttribute(
    QWebEngineSettings::LocalContentCanAccessRemoteUrls, true);
profile->settings()->setAttribute(
    QWebEngineSettings::LocalContentCanAccessFileUrls, true);
```

------

# 3.2 允许 iframe 跨域加载（规避 X-Frame）

```
profile->settings()->setAttribute(
    QWebEngineSettings::AllowRunningInsecureContent, true);
profile->settings()->setAttribute(
    QWebEngineSettings::AllowWindowActivationFromJavaScript, true);
```

如果网站本身带 `X-Frame-Options: DENY` 则无解，必须通过代理中转。

------

# 3.3 CORS 规避方式（最推荐方案）

### **方案 A：QWebEngineUrlRequestInterceptor 修改 header（推荐）**

拦截请求 → 注入 CORS header 使其合法。

```
class CorsInterceptor : public QWebEngineUrlRequestInterceptor {
public:
    void interceptRequest(QWebEngineUrlRequestInfo& info) override {
        info.setHttpHeader("Access-Control-Allow-Origin", "*");
    }
};
```

注册：

```
profile->setRequestInterceptor(new CorsInterceptor);
```

------

### **方案 B：本地代理服务器（最稳定）**

使用 QLocalProxy/Nginx/Node 作为资源代理：

```
QWebEngine  →   localhost:9000   →   remote server
```

在代理补齐CORS Header：

```
Access-Control-Allow-Origin: *
```

------

### **方案 C：DevTools 禁用 CORS（调试用，不推荐生产）**

在调试模式：

> DevTools → Network → Disable CORS

但 QWebEngine 无法永久修改。

------

# 3.4 QWebChannel 无法注入的规避

最常见的原因：

- CSP 阻止注入
- iframe 加载本地脚本被禁止
- 在非 mainFrame 注入脚本

**解法：**

```
QWebEngineScript script;
script.setInjectionPoint(QWebEngineScript::DocumentCreation);
script.setRunsOnSubFrames(true);

QFile f(":/qwebchannel.js");
script.setSourceCode(f.readAll());

profile->scripts()->insert(script);
```

确保：

```
page->setWebChannel(channel);
```

------

# 3.5 自定义协议绕过安全策略（Qt 6 支持）

适用：
 你需要加载内部资源，但又不想触发 file:// 限制。

注册 scheme：

```
QWebEngineUrlScheme scheme("app");
scheme.setFlags(QWebEngineUrlScheme::SecureScheme |
                QWebEngineUrlScheme::LocalScheme);
QWebEngineUrlScheme::registerScheme(scheme);
```

即可加载：

```
app://index.html
```

完全绕过 file:// 安全限制。

------

# 4. 实战 Demo：绕过 file:// 与 CORS 限制（可运行）

这是一个可直接运行的 Qt 示例：

- 本地 file:// 页面可以访问远端 JS/CSS
- 支持 fetch/XHR 不报 CORS 错
- 支持 QWebChannel 双向通信

------

## 4.1 C++ 代码：页面加载 + 跨域设置 + 跨域拦截

```
#include <QApplication>
#include <QWebEngineView>
#include <QWebEngineProfile>
#include <QWebEngineUrlRequestInterceptor>

class CorsInterceptor : public QWebEngineUrlRequestInterceptor {
public:
    void interceptRequest(QWebEngineUrlRequestInfo& info) override {
        info.setHttpHeader("Access-Control-Allow-Origin", "*");
        info.setHttpHeader("Access-Control-Allow-Methods", "GET, POST, OPTIONS");
        info.setHttpHeader("Access-Control-Allow-Headers", "*");
    }
};

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);

    auto profile = QWebEngineProfile::defaultProfile();
    profile->settings()->setAttribute(QWebEngineSettings::LocalContentCanAccessRemoteUrls, true);
    profile->settings()->setAttribute(QWebEngineSettings::LocalContentCanAccessFileUrls, true);
    profile->setRequestInterceptor(new CorsInterceptor);

    QWebEngineView view;
    view.load(QUrl("file:///C:/test/index.html"));
    view.resize(1200, 900);
    view.show();

    return app.exec();
}
```

------

## 4.2 index.html（可复现跨域 & QWebChannel）

```
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>CORS Demo</title>
    <script src="https://cdn.jsdelivr.net/npm/vue@3"></script>
    <script src="qwebchannel.js"></script>
</head>
<body>
<h2>CORS + QWebChannel 测试页面</h2>

<button onclick="testFetch()">测试跨域 Fetch</button>

<script>
function testFetch() {
    fetch("https://httpbin.org/get")
        .then(r => r.json())
        .then(console.log)
        .catch(console.error);
}
</script>

</body>
</html>
```

运行后：

- Vue CDN 正常加载
- fetch 正常返回数据
- 未触发 CORS 错误

------

# 5. 调试思路：如何判断安全策略触发？

### 5.1 通过 DevTools

```
page->setInspectedPage(page);  // Qt6
```

查看：

- Console 报错
- Network → blocked
- Security → insecure content

------

### 5.2 启动日志

启动参数：

```
QCoreApplication::setApplicationName("browser");
QWebEngineSettings::defaultSettings()->setAttribute(
    QWebEngineSettings::ErrorPageEnabled, true);
```

Chromium 日志：

```
--enable-logging --v=99 --log-file=web.log
```

------

# 6. 总结：常见问题 → 对应解法速查表

| 问题                        | 解决方法                                    |
| --------------------------- | ------------------------------------------- |
| file:// 无法加载 HTTPS 资源 | LocalContentCanAccessRemoteUrls             |
| JS 无法和 C++ 通信          | 设置 QWebChannel + 插入 script              |
| iframe 加载失败             | AllowRunningInsecureContent + 允许跨域      |
| CORS 报错                   | 拦截器注入 Header / 本地代理                |
| script/img/video 加载失败   | Content Security Policy（建议使用代理中转） |
| 自定义协议被禁止            | 注册 scheme（Qt 6）                         |