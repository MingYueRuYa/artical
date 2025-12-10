## Qt 布局介绍：从入门到实战

在 Qt 开发中，**布局管理（Layout）** 是 UI 编程最基础、最关键的部分之一。一个优秀的布局不仅能让界面自适应各种窗口大小，还能让开发者专注业务，而不必手动控制控件的位置与尺寸。

本文将系统介绍 Qt 中常用的布局方式，并给出每种布局的实际案例（基于 QWidget）。

# 1. 为什么要使用布局？

如果你直接设置控件的 `setGeometry()` 或 `move()`，窗口一拉伸界面就乱了。
 布局管理器可以自动完成：

- 控件的大小、位置自适应
- 随父容器变化自动调整
- 保持控件间的合理间距
- 避免手动计算坐标

Qt 提供了多种布局：

- QHBoxLayout（横向）
- QVBoxLayout（纵向）
- QGridLayout（网格）
- QFormLayout（表单）
- QStackedLayout（堆叠切换）
- 嵌套布局（组合布局，实现复杂界面）

接下来逐个讲解。

------

### 2. QHBoxLayout（水平布局）

从左到右排列控件，是最常用的布局之一。

#### 案例：创建一个横向按钮条
```cpp
#include <QApplication>
#include <QWidget>
#include <QPushButton>
#include <QHBoxLayout>

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);

    QWidget w;
    w.setWindowTitle("QHBoxLayout Demo");

    auto *btn1 = new QPushButton("确定");
    auto *btn2 = new QPushButton("取消");
    auto *btn3 = new QPushButton("帮助");

    auto *layout = new QHBoxLayout();
    layout->addWidget(btn1);
    layout->addWidget(btn2);
    layout->addWidget(btn3);

    w.setLayout(layout);
    w.show();
    return a.exec();
}
```
效果

按钮从左到右依次排列，窗口拉伸时自动等比扩展。

------

### 3. QVBoxLayout（垂直布局）

控件从上到下排列。

案例：一个简单的登录界面布局

```cpp
#include <QVBoxLayout>
#include <QLineEdit>
#include <QPushButton>

QWidget *createLoginWidget()
{
    QWidget *w = new QWidget;

    auto *editUser = new QLineEdit;
    auto *editPass = new QLineEdit;
    auto *btnLogin = new QPushButton("登录");

    editUser->setPlaceholderText("用户名");
    editPass->setPlaceholderText("密码");
    editPass->setEchoMode(QLineEdit::Password);

    auto *layout = new QVBoxLayout;
    layout->addWidget(editUser);
    layout->addWidget(editPass);
    layout->addWidget(btnLogin);

    w->setLayout(layout);
    return w;
}
```

### 4. QGridLayout（网格布局）

类似表格，通过行列来安排控件。

案例：计算器按钮布局

```cpp
#include <QGridLayout>
#include <QPushButton>

QWidget *createCalcWidget()
{
    QWidget *w = new QWidget;

    auto *layout = new QGridLayout;

    QStringList keys = {"7", "8", "9",
                        "4", "5", "6",
                        "1", "2", "3",
                        "0", "+", "-"};

    int row = 0;
    int col = 0;

    for (int i = 0; i < keys.size(); ++i) {
        auto *btn = new QPushButton(keys[i]);
        layout->addWidget(btn, row, col);

        col++;
        if (col == 3) { col = 0; row++; }
    }

    w->setLayout(layout);
    return w;
}
```

特点

- 每个单元格自动对齐
- 可指定控件跨行/跨列
- 适合复杂表格和键盘布局

----------

### 5. QFormLayout（表单布局）

适合 “标签 + 输入框” 结构，如登录界面、设置面板等。

案例：用户信息表单

```cpp
#include <QFormLayout>
#include <QLineEdit>
#include <QSpinBox>

QWidget *createForm()
{
    QWidget *w = new QWidget;

    auto *layout = new QFormLayout;

    auto *editName = new QLineEdit;
    auto *editEmail = new QLineEdit;
    auto *spinAge = new QSpinBox;

    layout->addRow("姓名：", editName);
    layout->addRow("邮箱：", editEmail);
    layout->addRow("年龄：", spinAge);

    w->setLayout(layout);
    return w;
}
```

特点

- 左右对齐美观
- 自动为标签设置统一宽度
- 表单界面最佳选择

### 6. QStackedLayout（堆叠布局）

像 PPT 一样多页切换，适合登录/注册切换、分步骤界面等。

####  案例：登录 / 注册界面切换
```cpp
#include <QStackedLayout>
#include <QPushButton>

QWidget *createStack()
{
    QWidget *w = new QWidget;

    auto *pageLogin = new QWidget;
    auto *layoutLogin = new QVBoxLayout;
    layoutLogin->addWidget(new QLineEdit("用户名"));
    layoutLogin->addWidget(new QLineEdit("密码"));
    layoutLogin->addWidget(new QPushButton("登录"));
    pageLogin->setLayout(layoutLogin);

    auto *pageRegister = new QWidget;
    auto *layoutRegister = new QVBoxLayout;
    layoutRegister->addWidget(new QLineEdit("邮箱"));
    layoutRegister->addWidget(new QLineEdit("设置密码"));
    layoutRegister->addWidget(new QPushButton("注册"));
    pageRegister->setLayout(layoutRegister);

    auto *stack = new QStackedLayout;
    stack->addWidget(pageLogin);
    stack->addWidget(pageRegister);

    // 按钮切换
    auto *btnSwitch = new QPushButton("切换登录/注册");
    QObject::connect(btnSwitch, &QPushButton::clicked, [=]() {
        stack->setCurrentIndex((stack->currentIndex()+1)%2);
    });

    auto *mainLayout = new QVBoxLayout;
    mainLayout->addLayout(stack);
    mainLayout->addWidget(btnSwitch);

    w->setLayout(mainLayout);
    return w;
}
```

### 7. 嵌套布局：构建复杂界面

Qt 支持将多个布局嵌套使用，构建复杂界面。

案例：经典界面结构 = 左侧菜单 + 右侧内容区

```cpp
QWidget *createComplexUI()
{
    QWidget *w = new QWidget;

    // 左侧
    auto *btnHome = new QPushButton("主页");
    auto *btnSettings = new QPushButton("设置");

    auto *leftLayout = new QVBoxLayout;
    leftLayout->addWidget(btnHome);
    leftLayout->addWidget(btnSettings);
    leftLayout->addStretch();

    // 右侧
    auto *label = new QLabel("右侧主要内容界面");
    auto *rightLayout = new QVBoxLayout;
    rightLayout->addWidget(label);
    rightLayout->addStretch();

    // 总布局（水平）
    auto *mainLayout = new QHBoxLayout;
    mainLayout->addLayout(leftLayout, 1);
    mainLayout->addLayout(rightLayout, 4);

    w->setLayout(mainLayout);
    return w;
}
```

### 总结：Qt 布局的“九大妙用”
| 妙用                  | 价值           |
| ------------------- | ------------ |
| ① 自动适配各种分辨率         | UI 永不变形      |
| ② 控件自适应内容            | 多语言友好        |
| ③ 对齐美观         | 减少手工布局工作量    |
| ④ 伸缩策略            | 窗口放大缩小自然     |
| ⑤ 隐藏/显示控件布局重排       | 高级设置折叠必备     |
| ⑥ 复杂 UI 的基础     | VSCode、微信都能做 |
| ⑦ QSplitter 可拖动布局   | 现代应用标配       |
| ⑧ QScrollArea 可滚动页面 | 内容无限扩展       |
| ⑨ 动态布局        | 聊天框、动态表单等    |

