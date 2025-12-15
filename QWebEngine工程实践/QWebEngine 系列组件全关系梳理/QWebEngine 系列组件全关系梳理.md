# QWebEngine 系列组件全关系梳理

在 Qt WebEngine 模块中，`QWebEngineView`、`QWebEnginePage`、`QWebEngineProfile` 和 `QWebEngineSettings` 是核心类。理解它们之间的关系对于构建复杂的浏览器或嵌入式 Web 应用至关重要。

------

## **1. QWebEngineView**

- **作用**：视图组件，提供 Web 内容的显示窗口，类似于传统浏览器的窗口。
- **特点**：
  - 继承自 `QWidget`，可以直接嵌入 Qt 界面。
  - 本身并不处理 Web 内容，只负责显示和用户交互。
- **核心关联**：
  - 默认持有一个 `QWebEnginePage` 对象，可以通过 `setPage()` 或 `page()` 获取。
  - 可以通过 `settings()` 获取 `QWebEngineSettings`，实际上是获取当前 `QWebEnginePage` 的设置。

```
QWebEngineView *view = new QWebEngineView(parent);
view->load(QUrl("https://www.qt.io"));
QWebEnginePage *page = view->page(); // 获取关联的Page
```

## **2. QWebEnginePage**

- **作用**：页面逻辑管理者，处理 Web 内容加载、渲染和事件。

- **特点**：

  - 独立于视图，多个 `QWebEngineView` 可以共用同一个 `QWebEnginePage`。
  - 管理 JavaScript、导航控制、网络请求、历史记录等。

- **核心关联**：

  ```
  QWebEngineProfile *profile = QWebEngineProfile::defaultProfile();
  QWebEnginePage *page = new QWebEnginePage(profile, parent);
  view->setPage(page);
  ```

  - 绑定一个 `QWebEngineProfile`，决定该页面的缓存、cookie、存储等。
  - 持有 `QWebEngineSettings`，可自定义页面行为（JavaScript 是否启用、图片加载等）。

## **3. QWebEngineProfile**

- **作用**：浏览器配置与资源管理者，负责共享缓存、Cookie、存储和代理设置。
- **特点**：
  - 可以创建多个 `QWebEngineProfile` 实例，实现多用户隔离的浏览环境。
  - 支持持久化（写入磁盘）或临时会话（内存中）。
- **核心关联**：
  - 一个 `QWebEnginePage` 必须绑定到一个 Profile。
  - 所有使用同一 Profile 的 Page 将共享 Cookie、缓存和存储。

```
QWebEngineProfile *profile = new QWebEngineProfile("User1", parent);
profile->setPersistentCookiesPolicy(QWebEngineProfile::ForcePersistentCookies);
QWebEnginePage *page = new QWebEnginePage(profile, parent);
```

## **4. QWebEngineSettings**

- **作用**：页面行为配置管理者。
- **特点**：
  - 控制 JavaScript、CSS、插件、图片加载、字体大小等。
  - 通过 `QWebEnginePage::settings()` 获取。
  - 对某个 Page 有效，不同 Page 可以有不同设置。

```
QWebEngineSettings *settings = page->settings();
settings->setAttribute(QWebEngineSettings::JavascriptEnabled, true);
settings->setAttribute(QWebEngineSettings::LocalStorageEnabled, true);
```

## **5. 四者关系图示**

```
QWebEngineView
     │
     ▼
QWebEnginePage ──> QWebEngineProfile
     │
     ▼
QWebEngineSettings
```

**解释**：

1. `QWebEngineView` 显示页面，依赖 `QWebEnginePage`。
2. `QWebEnginePage` 控制页面加载与交互，绑定 `QWebEngineProfile`。
3. `QWebEngineProfile` 管理全局资源和会话信息。
4. `QWebEngineSettings` 配置属于 `QWebEnginePage` 的行为。

## **6. 实际使用场景**

| 需求           | 推荐方案                                                    |
| :------------- | :---------------------------------------------------------- |
| 多窗口浏览器   | 每个窗口一个 `QWebEngineView`，共用同一 `QWebEngineProfile` |
| 隔离用户会话   | 为每个用户创建独立的 `QWebEngineProfile`                    |
| 自定义页面行为 | 修改 `QWebEnginePage::settings()`                           |
| 静态页面显示   | 使用默认 `QWebEngineProfile` 即可                           |

## **7. 总结**

- **QWebEngineView**：显示窗口。
- **QWebEnginePage**：页面逻辑和事件管理。
- **QWebEngineProfile**：全局或会话级资源和设置。
- **QWebEngineSettings**：页面行为控制。

通过合理组合这四者，可以实现复杂的多用户浏览器环境、定制化网页渲染以及资源隔离管理。