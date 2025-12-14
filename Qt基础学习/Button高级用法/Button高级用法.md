## QPushButton 高级使用指南

### 1、目录

1. 自定义状态按钮（状态机 + 属性驱动）
2. Checkable / ButtonGroup 的高级用法
3. QStyle + styleOption 控制按钮绘制
4. 动画按钮（悬浮 / 按下 / 波纹）

### 一、状态驱动的 QPushButton（替代 if/else）

#### 典型场景

- 登录中 / 成功 / 失败
- 上传中 / 暂停 / 完成
- 禁用态原因提示

#### 核心思想

**不用 setText / setEnabled 乱改，而是用动态属性 + QSS**

#### Demo 1：状态按钮（state = normal / loading / success）

**StateButton.h**

```cpp
#pragma once
#include <QPushButton>

class StateButton : public QPushButton
{
    Q_OBJECT
public:
    enum State {
        Normal,
        Loading,
        Success
    };
    Q_ENUM(State)

    explicit StateButton(QWidget *parent = nullptr);

    void setState(State state);

private:
    State m_state;
};
```

**StateButton.cpp**

```cpp
#include "StateButton.h"

StateButton::StateButton(QWidget *parent)
    : QPushButton(parent)
{
    setFixedHeight(40);
    setState(Normal);
}

void StateButton::setState(StateButton::State state)
{
    m_state = state;
    setProperty("state", state);

    switch (state) {
    case Normal:
        setText("提交");
        setEnabled(true);
        break;
    case Loading:
        setText("处理中...");
        setEnabled(false);
        break;
    case Success:
        setText("完成");
        setEnabled(true);
        break;
    }

    style()->unpolish(this);
    style()->polish(this);
}
```

**QSS**

```css
StateButton[state="0"] {
    background-color: #409EFF;
    color: white;
}
StateButton[state="1"] {
    background-color: #909399;
    color: white;
}
StateButton[state="2"] {
    background-color: #67C23A;
    color: white;
}
```

**要点**

- Q_PROPERTY + QSS = 状态机
- 非常适合“按钮即状态”的业务组件

### 二、Checkable + QButtonGroup 的工程化用法

#### 典型误区

- 每个按钮自己写 clicked
-  手动维护互斥逻辑

**Demo 2：模式切换按钮组（类似工具栏）**

```cpp
auto *group = new QButtonGroup(this);
group->setExclusive(true);

QStringList modes = {"选择", "绘制", "擦除"};
QHBoxLayout *layout = new QHBoxLayout;

for (int i = 0; i < modes.size(); ++i) {
    QPushButton *btn = new QPushButton(modes[i]);
    btn->setCheckable(true);
    btn->setProperty("mode", i);
    group->addButton(btn, i);
    layout->addWidget(btn);
}

connect(group, &QButtonGroup::idClicked, this, [](int id) {
    qDebug() << "当前模式：" << id;
});
```

**QSS**

```css
QPushButton {
    border: 1px solid #dcdfe6;
    padding: 6px 12px;
}
QPushButton:checked {
    background-color: #409EFF;
    color: white;
}
```

 **要点**

- `QButtonGroup` = 状态管理器
- 按钮只是 View，不是逻辑中心

### 三、使用 QStyle 精准控制绘制（替代 QSS）

#### 什么时候不用 QSS？

- 想精确控制图标 / 文本布局
- 做跨平台一致的按钮
- 性能敏感（大量按钮）

**Demo 3：重绘按钮（文本 + 图标布局）**

```cpp
class StyleButton : public QPushButton
{
public:
    using QPushButton::QPushButton;

protected:
    void paintEvent(QPaintEvent *event) override {
        QStyleOptionButton opt;
        initStyleOption(&opt);

        QPainter p(this);
        style()->drawControl(QStyle::CE_PushButtonBevel, &opt, &p, this);

        QRect iconRect(10, 10, 20, 20);
        QRect textRect(40, 0, width() - 40, height());

        icon().paint(&p, iconRect);
        p.drawText(textRect, Qt::AlignVCenter, text());
    }
};
```

**要点**

- `initStyleOption` 非常关键
- 能兼容 hover / press / disable 状态

### 四、动画按钮（专业 UI 的灵魂）

**Demo 4：Hover 缩放 + 阴影动画**

```cpp
QPropertyAnimation *anim = new QPropertyAnimation(button, "geometry");
anim->setDuration(200);

button->installEventFilter(this);

bool eventFilter(QObject *obj, QEvent *event) override {
    if (obj == button) {
        QRect r = button->geometry();
        if (event->type() == QEvent::Enter) {
            anim->setStartValue(r);
            anim->setEndValue(r.adjusted(-3, -3, 3, 3));
            anim->start();
        } else if (event->type() == QEvent::Leave) {
            anim->setStartValue(button->geometry());
            anim->setEndValue(r.adjusted(3, 3, -3, -3));
            anim->start();
        }
    }
    return QObject::eventFilter(obj, event);
}
```

 **要点**

- 动画不要写在 QPushButton 内部
- 用行为层包裹，利于复用

### 总结：高级者如何“用好” QPushButton

| 层级 | 关注点              |
| ---- | ------------------- |
| 初级 | clicked / setText   |
| 中级 | QSS / checkable     |
| 高级 | 状态驱动 / QStyle   |
| 架构 | 组件封装 / 行为复用 |

