---
title: 用 QML 写一个计算器：纯前端实现与状态机设计
date: 2026-10-16 20:00:00
tags: [Qt, QML, 前端]
categories: Qt
description: 用 QML 单文件实现一个完整桌面计算器，涵盖属性绑定、Repeater 委托、内联组件、自定义信号、状态机设计与错误态处理
top_img:
cover:
---

## 前言

之前用 Qt Widgets 写过一个计算器，C++ 里管状态、界面里管渲染。这次换 QML 来写——全部逻辑塞进一个 `Main.qml`，C++ 侧只负责加载引擎。写完发现 QML 的声明式语法 + 属性绑定天然适合做这种"状态驱动界面"的应用，比 Widgets 简洁不少。

本文记录整个项目的实现思路和踩过的坑，涉及的知识点：`property` 绑定机制、`readonly` 修饰符、`Repeater` / `model` / `delegate` 三件套、内联 `component`、自定义 `signal`、`Layout.*` 附加属性、`required property` 等。

## 一、项目结构

```
CalculatorQml/
├── CMakeLists.txt       # CMake 构建 + QML 模块注册
├── main.cpp             # C++ 入口，加载 QML 引擎
├── Main.qml             # 计算器界面 + 全部逻辑（核心）
├── calculator.h/.cpp    # 空壳类（预留扩展）
├── app.ico              # 应用图标
└── appCalculator.rc     # Windows 资源文件
```

整个计算器是**纯 QML 单文件实现**，没有一行 C++ 计算逻辑。`calculator.h/.cpp` 只是空壳，留着以后如果需要把计算搬到 C++ 时用。

## 二、QML 模块注册与加载

### CMake 侧

```cmake
qt_add_qml_module(appCalculatorQml
    URI CalculatorQml
    QML_FILES
        Main.qml
)
```

`qt_add_qml_module` 把 `Main.qml` 注册为一个 QML 模块，URI 是 `CalculatorQml`。这个 URI 就是后续 C++ 加载时的"命名空间"。

### C++ 侧

```cpp
QQmlApplicationEngine engine;
engine.loadFromModule("CalculatorQml", "Main");
```

`loadFromModule` 的第一个参数要和 CMake 里的 `URI` **完全一致**，第二个参数是 QML 文件名（不含扩展名）。

> **坑 1**：最初 URI 写的 `Calculators`（带 s），`loadFromModule` 写的 `CalculatorQml`，运行直接报 `No module named "Calculators" found`。两边名字必须逐字符对上。

## 三、界面搭建

### 窗口与布局

```qml
ApplicationWindow {
    id: window
    width: 340; height: 440
    title: "计算器"
    color: "#ffffff"

    ColumnLayout {
        anchors.fill: parent
        // 算式行 → 结果行 → 弹性留白 → 按钮网格
    }
}
```

整体是一个 `ColumnLayout` 从上到下排四块：算式行、结果行、弹性留白、按钮网格。`Layout.fillWidth` / `Layout.fillHeight` 是**布局附加属性**——只有放在 Layout 容器里的直接子项才能用，普通 `anchors` 布局里写不生效。

### Repeater 批量生成按键

24 个按键不可能手写 24 个 `Rectangle`，用 `Repeater` 配合一个数组就行：

```qml
readonly property var keys: [
    "%" , "CE" , "C" , "⌫" ,
    "1/x" , "x²" , "√x" , "÷" ,
    "7" , "8" , "9" , "×" ,
    "4" , "5" , "6" , "−" ,
    "1" , "2" , "3" , "+" ,
    "+/−" , "0" , "." , "="
]

GridLayout {
    columns: 4
    Repeater {
        model: window.keys
        delegate: CalcButton {
            required property string modelData
            label: modelData
            kind: window.kindOf(modelData)
            onClicked: window.dispatch(modelData)
        }
    }
}
```

三个关键概念：

| 概念 | 作用 |
|------|------|
| `model` | 数据清单，这里是一个字符串数组 |
| `delegate` | 模板，Repeater 为每个数据项复制一份 |
| `modelData` | 当 model 是数组时，用 `modelData` 取当前项 |

`required property string modelData` 是 QML 6 引入的语法——声明"这个属性必须由创建者（Repeater）赋值"，比直接用 `model.modelData` 更安全，类型不匹配会在加载时报错。

## 四、状态设计

计算器的核心是状态管理。我用了 7 个属性：

```qml
property string current: "0"       // 当前输入/显示的字符串
property real   accumulator: 0     // 已确认的左操作数
property string pendingOp: ""      // 待执行的运算符
property bool   isNewEntry: true   // 下次按数字是否重新开始输入
property bool   isError: false     // 是否处于错误状态
property string errorMsg: ""       // 错误文案
property string expr: ""           // 算式行文本
```

### 属性绑定：界面自动刷新

QML 的 `property` 不是普通变量，它有**绑定**机制。界面上写：

```qml
Text {
    text: window.isError ? window.errorMsg : window.current
}
```

这不是一次性赋值，而是一个**响应式绑定**——`isError` 或 `current` 一变，`text` 自动更新，不需要手动调 `setText()`。这比 Widgets 的 `updateDisplay()` 手动刷新省心太多。

### 状态机思路

三个标志位组合出不同阶段：

| `pendingOp` | `isNewEntry` | `isError` | 阶段 |
|:---:|:---:|:---:|------|
| `""` | `true` | `false` | 初始 / 刚算完，等新输入 |
| `""` | `false` | `false` | 正在输入第一个数 |
| `"+"` 等 | `true` | `false` | 刚按了运算符，等右操作数 |
| `"+"` 等 | `false` | `false` | 正在输入右操作数 |
| - | - | `true` | 出错了，大部分按键禁用 |

每个按键函数进来第一件事就是判断当前处于哪个阶段，然后决定行为。

## 五、按键分发与运算逻辑

### 总入口 dispatch

所有按键点击都走 `dispatch(btn)` 一个入口，再分发到具体函数：

```qml
function dispatch(btn) {
    if (isDigitKey(btn)) { inputDigit(btn); return }
    switch (btn) {
        case ".":  inputDecimal();  break
        case "=":  equals();        break
        case "+":  setOperator("+"); break
        case "−":  setOperator("−"); break
        // ... 其余 case
    }
}
```

### 先存后算模型

运算符不是立即计算，而是"存下左操作数和运算符，等右操作数齐了再算"：

```qml
function setOperator(op) {
    if (pendingOp !== "" && !isNewEntry) {
        compute()          // 连续运算：先算前一步
        if (isError) return
    } else {
        accumulator = parseFloat(current)  // 首次：存左操作数
    }
    pendingOp = op
    isNewEntry = true    // 下次数字键开始新输入
}
```

`equals` 和 `compute` 分离：`equals` 负责拼算式文本和收尾，`compute` 只做纯数学运算。这样连续按运算符（`3 + 5 ×`）和按等号（`3 + 5 =`）都能复用 `compute`。

### 一元运算

`x²`、`√x`、`1/x`、`%` 都是对当前值直接变换，不涉及 `accumulator`：

```qml
function applySquare() {
    if (isError) return
    var value = parseFloat(current)
    current = format(value * value)
    isNewEntry = true
}
```

## 六、格式化与错误处理

### format：去尾零

JavaScript 的浮点数有精度问题（`0.1 + 0.2 = 0.30000000000000004`），用 `toPrecision(15)` 保留 15 位有效数字，再去掉尾部多余的零：

```qml
function format(value: real): string {
    var str = value.toPrecision(15)
    if (str.indexOf(".") !== -1 && str.indexOf("e") === -1)
        str = str.replace(/0+$/, "").replace(/\.$/, "")
    return str
}
```

`indexOf("e") === -1` 是为了跳过科学计数法（`1e+20`），否则正则 `/0+$/` 会误删。

> **坑 2**：最初写成 `str.indexOf("e" === -1)`，括号位置错了——`"e" === -1` 先算出来是 `false`，变成 `indexOf(false)`，逻辑完全乱套。`indexOf` 的括号要包住 `"e"`，`=== -1` 在外面。

### 错误态处理

```qml
function showError(msg) {
    errorMsg = msg
    isError = true
    accumulator = 0
    pendingOp = ""
    isNewEntry = true
}
```

进入错误态后，通过 `isOnDisabled` 判定哪些按键要禁用：

```qml
function isOnDisabled(btn) {
    return ["%", "1/x", "x²", "√x", "÷", "×", "−", "+", "+/−", "."].indexOf(btn) !== -1
}
```

Repeater 委托里绑定到 `enabled`：

```qml
delegate: CalcButton {
    enabled: !(window.isError && window.isOnDisabled(modelData))
    // ...
}
```

错误态下只有 `CE`、`C`、`⌫`、`=` 和数字键可用，按数字键会自动清空错误并开始新输入。

除零做了细分：`0 ÷ 0` 显示"结果未定义"，`非零 ÷ 0` 显示"除数不为零"。浮点比较用 `fuzzyZero`（阈值 `1e-12`）避免精度误差。

## 七、CalcButton 内联组件

24 个按钮样式统一，抽成一个内联 `component`：

```qml
component CalcButton: Rectangle {
    id: btn
    property string label: ""
    property int kind: 0          // 0=数字 1=功能 2=等号
    signal clicked()              // 自定义信号

    readonly property color cBase: kind === 2 ? "#00aaff" : kind === 1 ? "#e3eefc" : "#ffffff"
    readonly property color cHover: kind === 2 ? "#0090e0" : kind === 1 ? "#d5e6fb" : "#f4f6f9"
    readonly property color cPressed: kind === 2 ? "#0077bb" : kind === 1 ? "#c6ddfa" : "#e9edf2"

    color: mouse.pressed ? cPressed : (mouse.containsMouse ? cHover : cBase)

    Text {
        anchors.centerIn: parent
        text: btn.label
        color: btn.kind === 2 ? "white" : "black"
    }

    MouseArea {
        id: mouse
        anchors.fill: parent
        hoverEnabled: true
        onClicked: btn.clicked()
    }
}
```

几个知识点：

- **`component` 内联定义**：在文件内声明可复用组件，不用单独建 `.qml` 文件
- **`signal clicked()`**：自定义信号，外部用 `onClicked:` 接收，实现组件与外层解耦
- **`readonly property`**：颜色表用 `readonly` 防止外部误改，只在组件内部根据 `kind` 计算
- **状态优先级**：`pressed > hover > base`，用三元表达式一行搞定

## 八、踩坑记录

| # | 现象 | 根因 | 修法 |
|---|------|------|------|
| 1 | `No module named "Calculators"` | CMake 的 URI 和 `loadFromModule` 的名字不一致 | 两边改成同名 |
| 2 | `6889.=` 结尾带小数点 | `equals()` 无运算符分支直接显示原始 `current` | 先 `format(parseFloat(current))` |
| 3 | 按 `−` 没反应 | 按键表里是数学减号 `−`(U+2212)，case 里是 ASCII `-`(U+002D)，`===` 匹配不上 | case 和 `compute()` 里统一用 `−` |
| 4 | 改完 case 后 `3−5=` 得 `0` | 只改了 `dispatch`，`compute()` 里判断还是 ASCII `-` | 所有关联比较处同步修改 |
| 5 | `1e+20` 显示成 `1e+2` | `format` 里 `indexOf("e" === -1)` 括号错位 | 改成 `indexOf("e") === -1` |
| 6 | `200+10%=` 得 `200.1` | `%` 只做了 `÷100` | 改为 Windows 语义：取左操作数的百分比 |

### 减号 Unicode 问题（坑 3-4）

这是最隐蔽的一个。按键表里 `"−"` 是 Unicode 数学减号（U+2212），长得跟 ASCII `-`（U+002D）几乎一样，但 `===` 严格比较下它们不相等。

```javascript
"−" === "-"  // false ！
```

`dispatch` 的 `case "−"` 和 `compute()` 里的 `if (pendingOp === "−")` 必须都用同一个字符。改了一处忘了另一处，就会出现"按键有反应但算出来不对"的灵异现象。

**教训**：一个符号被多处引用时，改一处必须全局搜索同步改所有关联处。

## 九、QML vs Widgets 对比

同一个计算器我用两种方式都写过，对比一下：

| 维度 | Qt Widgets (C++) | QML (声明式) |
|------|-----------------|--------------|
| 界面定义 | 代码手写或 .ui 文件 | QML 声明式，所见即所得 |
| 状态刷新 | 手动调 `updateDisplay()` | 属性绑定自动刷新 |
| 按键生成 | `for` 循环 + `QSignalMapper` | `Repeater` + 数组，三行搞定 |
| 样式 | QSS 或代码设置 | 属性绑定，三元表达式内联 |
| 信号连接 | `connect()` 显式连接 | `onClicked:` 声明式绑定 |
| 代码量 | ~300 行 C++ | ~400 行 QML（含样式） |

QML 的优势在于**声明式**——你描述"界面长什么样、跟什么状态绑定"，引擎自己管刷新。Widgets 是**命令式**——你得告诉它"现在把这个标签的文字改成 X"。对于状态驱动的界面，声明式省心得多。

## 总结

这个项目让我对 QML 的几个核心概念有了实操层面的理解：

1. **属性绑定是灵魂**——只要把界面属性绑到状态上，后续只管改状态，界面自动跟
2. **Repeater + 数组** 是批量生成 UI 的利器，配合 `required property` 类型安全
3. **内联 component + signal** 实现组件封装，不用拆文件
4. **状态机思路** 是计算器逻辑的核心，几个 bool 标志组合出所有阶段
5. **Unicode 相似字符** 是声明式语言的隐形坑，`===` 不做类型转换也不做 Unicode 归一化

完整代码已放在 [GitHub](https://github.com/MeloBrush/my-blog)，一个 `Main.qml` 400 行搞定全部界面和逻辑。