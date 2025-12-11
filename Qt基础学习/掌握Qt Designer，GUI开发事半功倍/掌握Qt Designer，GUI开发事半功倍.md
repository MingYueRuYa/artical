## 掌握Qt Designer，GUI开发事半功倍

上一篇文章中，我们说到了布局都是用代码控制的，但是一旦UI复杂上来了，用代码控制其实非常低效。

Qt Designer 是 Qt 官方提供的 **可视化界面设计工具**，也是 Qt 生态中最重要的生产力工具之一。它通过拖拽控件、可视化布局、实时预览等方式，大幅提升 UI 开发效率。本文将从基础概念到高级技巧，由浅入深，带你系统掌握 Qt Designer 的使用方法。

### 一、Qt Designer 是什么？

Qt Designer 是一个专门设计 **.ui 文件（XML 格式）** 的工具，它的核心作用是：

- 使用拖拽方式快速构建 UI
- 自动生成布局而无需写大量代码
- 提供控件的属性配置面板
- 支持信号与槽连接（可视化）
- 可以与 Qt Creator 无缝集成

最终生成的 `.ui` 文件可以通过 uic工具 自动转成 C++ 代码，使用方法就像头文件include就可以了。

### 二、Qt Designer 基础界面介绍

Qt Designer 的界面主要由以下部分构成：

#### 1. 控件盒（Widget Box）

左侧区域，包含所有可用控件，例如：

- QPushButton、QLabel、QLineEdit
- QHBoxLayout、QGridLayout
- QTableView、QTreeWidget 等

拖拽即可放入窗口内。

#### 2. 表单编辑区（Form Editor）

中心区域，用于可视化编辑 UI，所见即所得。

#### 3. 属性编辑器（Property Editor）

右侧区域，用于调整控件属性，例如：

- text、icon
- size policy、geometry
- stylesheet（QSS）
- signals/slots（Qt Creator 中）

#### 4. 对象树（Object Inspector）

层级树结构显示 UI 中的控件，方便选中、重命名、调整结构。

### 三、从零开始：创建你的第一个 UI

#### 步骤 1：创建 Form

文件 → 新建 → Qt → "Main Window" / "Dialog" / "Widget"

适用于不同场景：

- **Widget**：任意自定义窗口（最常见）
- **MainWindow**：带菜单栏、工具栏
- **Dialog**：对话框

#### 步骤 2：拖拽控件

例如拖一个 QLabel 和 QPushButton，并摆放到合适位置。

#### 步骤 3：使用布局管理

*这是 Qt Designer 中最核心的概念之一。*

选择多个控件 → 右键 → “Lay Out Horizontally/Vertically/Grid”

布局的作用是让窗口可以自适应缩放，避免控件乱跑。

#### 步骤 4：保存 ui

文件 → 保存为 `mainwindow.ui`

#### 步骤 5：在代码中使用它
```cpp
#include "ui_mainwindow.h"

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    connect(ui->pushButton, &QPushButton::clicked, this, [](){
        qDebug() << "Button clicked!";
    });
}
```

四、进阶：掌握控件属性与布局技巧
1. 常用属性详解

sizePolicy
控制控件如何在布局中伸缩，是 Qt 布局的核心属性之一。

- Fixed

- Preferred

- Expanding

- MinimumExpanding

stylesheet（样式表）
你可以直接在 Qt Designer 中写 QSS，例如：

```cpp
QPushButton {
    border-radius: 8px;
    background: #4A90E2;
    color: white;
}
```

alignment 对齐
控制控件在布局中的对齐方式，比如 Qt::AlignCenter。

### 2. 高级布局技巧

- 使用 **Spacer（伸缩条）** 填充空白区域
- 使用 **Grid Layout** 构建表格布局
- 使用 **QSplitter** 实现可拖拽分割界面
- 使用 **Promoted Widget** 嵌入自定义控件
- 使用 **Tab Widget** 构建多界面结构
- 使用 **Scroll Area** 适配长内容页面

## 五、进一步深入：Qt Designer 的高级能力

### 1. Promoted Widget（提升控件）

当你需要放入一个自定义控件时，你可以：

1. 放一个基本控件（如 QWidget）
2. 右键点击 → Promote to…
3. 填入你的类名，例如 `MyCustomWidget`

Designer 就会把它当成你的自定义控件处理。

这是大型界面开发中必学技能。

### 2. 使用 Qt Designer 实现信号与槽连接（Qt Creator）

在 Qt Creator 的 **“设计”模式**下，可以通过 UI 完成：

- 选择控件
- 右键 → Go to Slot
- 或在 “信号与槽” 编辑器连接事件

无需手写 connect 代码（初学者非常友好）

### 3. 使用资源管理器（Resource Browser）

Designer 完整支持 Qt 的 `.qrc` 资源文件：

- Image
- Icon
- SVG
- 字体 font
- qml（混合项目）

你可以直接将 qrc 中的资源绑定到控件属性上。

### 六、与 Qt Creator / C++ 的交互流程（专业用法）

专业开发中的常见工作流：

1. 使用 Qt Designer 设计 UI
2. Qt Creator 或 cmake 自动生成 ui_xxx.h
3. 在 C++ 中 **使用 ui 指针操作控件**
4. 对于复杂界面，将 UI 和逻辑分离为多个类
5. 控件之间的交互由信号和槽负责绑定

```bat
mainwindow.ui  ← 界面结构
mainwindow.cpp ← 控制逻辑
mainwindow.h   ← 事件处理、信号槽声明
```

### 七、最佳实践（行业经验总结）

#### 1. UI 与逻辑彻底分离

避免把过多逻辑写到 ui 类中，否则代码会变得难以维护。

#### 2. 控件命名规范

按钮：

```bat
btnLogin
btnSubmit
```

输入框：

```bat
editUserName
editPassword
```

避免 `pushButton_12` 这种难懂的名字。

#### 3. 尽量使用布局，不要用绝对定位

绝对位置会让 UI 在不同 DPI 上跑偏。

#### 4. 尽量写 QSS 而不是手动 setStyle

QSS 更强、可复用、跨平台一致。

#### 5. 尽量使用 qrc 管理资源

避免路径错误、跨平台不兼容。

### 八、总结

Qt Designer 是 Qt 开发中效率最高的工具之一，学习它能让你：

- 快速构建界面
- 减少重复劳动
- 改善界面适配
- 与 QSS / 自定义控件完美结合
- 将 UI 与逻辑分离，代码更清晰

从小型工具到大型软件，Qt Designer 都能胜任。
 它是 Qt 初学者必学、Qt 程序员高级进阶的必备技能。

如果你正在 Qt 学习从 0 到 1 的道路上，掌握 Qt Designer 会让你进步非常快。