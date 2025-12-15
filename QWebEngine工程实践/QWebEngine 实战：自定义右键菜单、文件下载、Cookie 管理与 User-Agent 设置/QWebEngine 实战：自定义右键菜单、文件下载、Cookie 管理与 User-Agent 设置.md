## **QWebEngine 实战：自定义右键菜单、文件下载、Cookie 管理与 User-Agent 设置**

`QWebEngine` 基于 `Chromium` 内核，功能强大，但很多能力需要手动扩展才能满足业务需求。本文将通过四个常见场景，给出可直接使用的代码示例：

1. **自定义右键菜单（ContextMenu）**
2. **文件下载管理（Download）**
3. **Cookie 读取 / 设置 / 持久化**
4. **UserAgent 自定义**

------

# **1. 自定义右键菜单（Context Menu）**

`QWebEngineView` 默认使用 `Chromium` 的菜单，如果我们想接管右键菜单：

- 禁用默认菜单：`setContextMenuPolicy(Qt::CustomContextMenu)`
- 捕获右键事件
- 自己定义 `QAction`
- 在页面中执行 JS 或 C++ 逻辑

## **1.1 右键菜单 Demo**

**MyWebView.h**

```cpp
#pragma once
#include <QWebEngineView>

class MyWebView : public QWebEngineView {
    Q_OBJECT
public:
    explicit MyWebView(QWidget *parent = nullptr);

private slots:
    void onCustomContextMenuRequested(const QPoint &pos);
};
```

**MyWebView.cpp**

```cpp
#include "MyWebView.h"
#include <QMenu>
#include <QClipboard>
#include <QApplication>

MyWebView::MyWebView(QWidget *parent)
    : QWebEngineView(parent)
{
    setContextMenuPolicy(Qt::CustomContextMenu);
    connect(this, &QWebEngineView::customContextMenuRequested,
            this, &MyWebView::onCustomContextMenuRequested);
}

void MyWebView::onCustomContextMenuRequested(const QPoint &pos)
{
    QMenu menu;

    QAction *reloadAct = menu.addAction("刷新");
    QAction *copyUrlAct = menu.addAction("复制当前 URL");
    QAction *inspectAct = menu.addAction("打开 DevTools");

    QAction *sel = menu.exec(mapToGlobal(pos));
    if (!sel) return;

    if (sel == reloadAct) {
        reload();
    } elseif (sel == copyUrlAct) {
        QApplication::clipboard()->setText(url().toString());
    } elseif (sel == inspectAct) {
        page()->setDevToolsPage(new QWebEnginePage(page()->profile()));
    }
}
```

# **2. 文件下载管理（Download）**

`QWebEngineProfile` 有信号：

```
void downloadRequested(QWebEngineDownloadItem *download)
```

我们可以接管文件下载流程，比如：

- 指定保存路径
- 显示下载进度
- 保存完成回调

### **2.1 下载示例**

**MainWindow 构造函数中添加：**

```cpp
connect(profile, &QWebEngineProfile::downloadRequested,
        this, &MainWindow::onDownloadRequested);
```

### **2.2 下载代码示例**

```cpp
void MainWindow::onDownloadRequested(QWebEngineDownloadItem *item)
{
    QString path = QFileDialog::getSaveFileName(
        this, "保存文件", item->path(), "");

    if (path.isEmpty()) {
        item->cancel();
        return;
    }

    item->setPath(path);
    item->accept();

    connect(item, &QWebEngineDownloadItem::receivedBytesChanged, this, [item]() {
        qDebug() << "下载进度: " << item->receivedBytes() << "/" << item->totalBytes();
    });

    connect(item, &QWebEngineDownloadItem::finished, this, [item]() {
        qDebug() << "下载完成：" << item->path();
    });
}
```

# **3. 管理 Cookie（读取 / 写入 / 持久化）**

Qt 的 Cookie 管理核心类：

- **QWebEngineCookieStore**（从 profile 获取）
- 支持添加、删除、监听变化

### **3.1 获取 CookieStore**

```cpp
QWebEngineCookieStore *store = page()->profile()->cookieStore();
```

### **3.2 读取 Cookie 示例**

```cpp
store->loadAllCookies();
connect(store, &QWebEngineCookieStore::cookieAdded,
        this, [](const QNetworkCookie &cookie){
    qDebug() << "Cookie Added:" << cookie.name() << cookie.value();
});
```

### **3.3 设置 Cookie 示例**

```cpp
QNetworkCookie cookie;
cookie.setName("token");
cookie.setValue("123456789");
cookie.setDomain("example.com");
cookie.setPath("/");
cookie.setExpirationDate(QDateTime::currentDateTime().addDays(7));

page()->profile()->cookieStore()->setCookie(cookie);
```

### **3.4 删除 Cookie 示例**

```cpp
store->deleteCookie(cookie);
```

## **3.5 Cookie 持久化**

Qt 默认在 profile 中持久化 `Cookie`，确保你使用的是**持久 profile**：

```cpp
QWebEngineProfile *profile = new QWebEngineProfile("MyProfile", this);
profile->setPersistentCookiesPolicy(QWebEngineProfile::AllowPersistentCookies);
profile->setPersistentStoragePath("data/profile");
```

# **4. 自定义 User-Agent**

QWebEngineProfile 可设置 UA：

## **4.1 设置 UA**

```cpp
page()->profile()->setHttpUserAgent(
    "MyBrowser/1.0 (QtWebEngine Based)"
);
```

### **4.2 在加载前动态修改 UA（可按域名区分 UA）**

```cpp
connect(page(), &QWebEnginePage::urlChanged, this, [this](const QUrl &url){
    if (url.host().contains("mobile")) {
        page()->profile()->setHttpUserAgent(
            "Mozilla/5.0 Mobile Safari/537.36"
        );
    } else {
        page()->profile()->setHttpUserAgent(
            "Mozilla/5.0 Desktop Safari/537.36"
        );
    }
});
```

# **5. 完整 Demo（可直接运行）**

下面是一个最小项目包含：

- 自定义右键菜单
- 下载
- Cookie 监听
- UA 设置

### ***\*main.cpp\****

```cpp
#include <QApplication>
#include "MainWindow.h"

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    MainWindow w;
    w.show();
    return a.exec();
}
```

### **MainWindow.h**

```cpp
#pragma once
#include <QMainWindow>
#include <QWebEngineView>
#include <QWebEngineProfile>

class MainWindow :public QMainWindow {
    Q_OBJECT
public:
    MainWindow();

private slots:
    void onDownloadRequested(QWebEngineDownloadItem *item);

private:
    QWebEngineView *view;
    QWebEngineProfile *profile;
};
```

### **MainWindow.cpp**

```cpp
#include "MainWindow.h"
#include "MyWebView.h"
#include <QVBoxLayout>
#include <QFileDialog>

MainWindow::MainWindow()
{
    profile = new QWebEngineProfile("MyProfile", this);
    profile->setPersistentStoragePath("data/");
    profile->setPersistentCookiesPolicy(QWebEngineProfile::AllowPersistentCookies);
    profile->setHttpUserAgent("MyQtBrowser/1.0");

    view = new MyWebView();
    QWebEnginePage *page = new QWebEnginePage(profile, view);
    view->setPage(page);

    connect(profile, &QWebEngineProfile::downloadRequested,
            this, &MainWindow::onDownloadRequested);

    setCentralWidget(view);

    view->load(QUrl("https://www.qt.io"));
}

void MainWindow::onDownloadRequested(QWebEngineDownloadItem *item)
{
    QString file = QFileDialog::getSaveFileName(this, "保存文件", item->path());
    if (file.isEmpty()) {
        item->cancel();
        return;
    }

    item->setPath(file);
    item->accept();
}
```

# **总结**

本文展示了 QWebEngine 浏览器开发中最常用的四大功能：

| 功能              | 核心类                 | 重点                            |
| :---------------- | :--------------------- | :------------------------------ |
| 自定义右键菜单    | QWebEngineView         | 捕获 customContextMenuRequested |
| 文件下载管理      | QWebEngineDownloadItem | 接受下载、显示进度              |
| Cookie 管理       | QWebEngineCookieStore  | 监听、设置、持久化              |
| User-Agent 自定义 | QWebEngineProfile      | 动态 UA / 全局 UA               |

这些能力在构建桌面浏览器、内嵌网页容器、H5 AppShell 中都是必需的。