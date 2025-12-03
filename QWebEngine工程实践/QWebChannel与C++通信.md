## 使用 QWebChannel 实现 JS 与 C++ 双向通信（超详细 + 踩坑总结 + Demo）

在基于 **QWebEngine** 的项目中，要让 **前端 JavaScript** 与 **后端 C++** 互相通信，是非常关键的能力。
 Qt 官方提供的方案就是 **QWebChannel**，它能让你像调用本地对象一样从 JS 访问 C++，并且支持信号/槽、异步回调等。

但实际项目中常见各种问题：

- JS 侧无法拿到对象？
- 信号不触发？
- 跨线程导致闪退？
- 对象销毁后 JS 仍然在调用？
- Page/Page再创建导致 channel 失效？

本文将带你彻底搞懂 QWebChannel 的机制，避坑，并给出可运行的 Demo。

### 一、QWebChannel 的通信原理

JS 与 C++ 的交互流程如下：
```cpp
C++ QObject ←→ WebChannel Transport (QWebEngine) ←→ JS 对象
```

实际通信依赖两部分：

### ① **C++ 侧：QObject + QWebChannel**

- QObject 必须继承自 QObject
- 想暴露给 JS 的属性/方法必须加 Q_INVOKABLE 或 Q_PROPERTY
- 信号可以直接给 JS 发送事件
- 将 QObject 注册进 QWebChannel：

```cpp
channel->registerObject("bridge", myBridgeObject);
```

### ② **JS 侧：qwebchannel.js**

网页加载后必须初始化：

```js
new QWebChannel(qt.webChannelTransport, function(channel) {
    window.bridge = channel.objects.bridge;
});
```

然后：

```js
// 调用 C++
bridge.sendMessage("hello");

// 接收 C++ 信号
bridge.messageChanged.connect(function(msg){
    console.log("C++ emit:", msg);
});
```

### 二、一个可跑的双向通信 Demo（最小可运行）

####  1. C++ 端代码

### **bridge.h**

```cpp
#pragma once
#include <QObject>

class Bridge : public QObject {
    Q_OBJECT
public:
    explicit Bridge(QObject* parent = nullptr) : QObject(parent) {}

    // JS 调用 C++
    Q_INVOKABLE void sendMessage(const QString& msg) {
        qDebug() << "JS 调用 C++：" << msg;
        emit messageChanged("C++ 收到：" + msg);
    }

signals:
    // C++ → JS
    void messageChanged(const QString& msg);
};
```

### **mainwindow.cpp**

```
#include "mainwindow.h"
#include <QWebEngineView>
#include <QWebEnginePage>
#include <QWebChannel>

MainWindow::MainWindow(QWidget* parent)
    : QMainWindow(parent)
{
    auto* view = new QWebEngineView(this);
    auto* page = new QWebEnginePage(this);
    view->setPage(page);
    setCentralWidget(view);

    // 创建 C++ 对象
    auto* bridge = new Bridge(this);

    // 创建 QWebChannel
    auto* channel = new QWebChannel(this);
    channel->registerObject("bridge", bridge);
    page->setWebChannel(channel);

    view->load(QUrl("qrc:/index.html"));

    // 模拟 3 秒后给 JS 发消息
    QTimer::singleShot(3000, [bridge]() {
        emit bridge->messageChanged("来自 C++ 的问候！");
    });
}
```

###  2. 前端：`index.html`

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8" />
    <script src="qrc:///qtwebchannel/qwebchannel.js"></script>
</head>
<body>
    <h2>QWebChannel Demo</h2>
    <input id="msg" placeholder="发送给 C++ 的消息">
    <button onclick="callCpp()">发送</button>

    <p id="log"></p>

    <script>
        new QWebChannel(qt.webChannelTransport, function (channel) {
            window.bridge = channel.objects.bridge;

            // C++ → JS
            bridge.messageChanged.connect(function (msg) {
                log("收到 C++：" + msg);
            });
        });

        function callCpp() {
            const msg = document.getElementById("msg").value;
            bridge.sendMessage(msg);
        }

        function log(msg) {
            document.getElementById("log").innerHTML += msg + "<br>";
        }
    </script>
</body>
</html>
```

这个 Demo 能实现：
 ✔ JS 调 C++
 ✔ C++ 调 JS（信号）
 ✔ 异步通信
 ✔ 页面加载自动初始化 WebChannel

## 三、QWebChannel 常见问题与避坑指南（非常重要）

### ❗ 1. **JS 侧总是拿不到对象（undefined）**

典型错误：

```
console.log(bridge); // undefined
```

### ✔ 正确做法：所有 JS 调用必须在 QWebChannel 初始化之后

```
new QWebChannel(qt.webChannelTransport, function(channel){
    window.bridge = channel.objects.bridge;
});
```

⚠ **不要**在 `window.onload` 或顶层就调用 bridge。

### 2. **跨线程访问导致崩溃（最常见）**

若你把 `Bridge` 放到子线程，会直接崩溃。

原因：
 QWebChannel 通信必须在 UI / WebEngine 所在线程使用，否则 QObject 会被跨线程访问。

### 正确方案：

- **bridge 必须在主线程**
- 如果你需要跨线程，可在 Bridge 中封装信号转发：

```
Q_INVOKABLE void updateDataFromWorkerThread(const QString& msg) {
    emit messageChanged(msg); // 仍然在主线程发
}
```

Qt 会自动通过 queued connection 切回主线程。

###  3. **页面重新加载后 channel 失效**

当你 reload()、load() 新页面后：
 之前的 JS 对象全部失效。

#### 必须在每次页面加载后重新建立 WebChannel

示例：

```cpp
connect(page, &QWebEnginePage::loadFinished, this, [=](bool ok){
    if(ok) {
        page->setWebChannel(channel);
    }
});
```

⚠ Qt 5 必做，Qt 6 已自动处理但建议仍写上。

###  4. **C++ 信号没有触发 JS 回调**

常见原因：

- Bridge 对象被销毁
- 信号签名不匹配
- JS 回调写在 WebChannel 初始化外部
- 信号参数为自定义类型但未 qRegisterMetaType

#### ✔ 排查顺序：

1. C++ 打印信号是否发出
2. JS log 是否有回调
3. 参数类型是否是 QVariant 能转的基本类型
4. Bridge 是否挂靠在 MainWindow（不要让其随 WebPage 销毁）

### 5. **复杂参数（对象/数组）导致 JS 不接收**
支持：
- QString
- int/double/bool
- QVariantList（JS Array）
- QVariantMap（JS Object）

但不支持 C++ 自定义结构体。

#### 解决：

```
QVariantMap obj;
obj["name"] = "Tom";
obj["age"] = 12;
emit messageChanged(obj);
```

JS：

```js
bridge.messageChanged.connect(function (obj) {
    console.log(obj.name);
});
```

### 五、总结

QWebChannel 是 Qt WebEngine 中最可靠、最强大的前后端通信方式，但需要注意：

✔ Bridge 必须在主线程
✔ JS 必须在初始化回调后才能使用对象
✔ 参数使用 QVariant 可序列化
✔ 页面刷新后必须重新 setWebChannel
✔ 跨线程调用需谨慎（信号转发）