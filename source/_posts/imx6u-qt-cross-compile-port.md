---
title: I.MX6U 嵌入式 Qt 移植实战：从交叉编译到开发板运行
date: 2026-10-17 15:30:00
tags: [嵌入式, Qt, 交叉编译, I.MX6U, Linux, tslib]
categories: 嵌入式
description: 在 I.MX6U 开发板上完成 Qt5.12.9 的交叉编译、移植与自研计算器 GUI 程序的部署运行，记录全流程与踩坑经验
top_img:
cover:
---

## 前言

本文记录在正点原子 ATK-I.MX6U（ARM Cortex-A7）开发板上完成 Linux 软硬件适配的完整过程：从交叉编译 tslib 与 Qt5.12.9 源码，到移植运行库至根文件系统，再到自主开发一款带 LED 控制的计算器 GUI 程序并在开发板上运行、开机自启。文中如实记录了移植过程中遇到的 Qt 平台插件不兼容、网络 IP 异常、tslib 段错误等典型问题及解决思路，希望能给同样做嵌入式 Qt 移植的同学一些参考。

## 实验目的

- 熟悉操作系统在特定硬件平台的环境构建流程；
- 熟悉内核的配置、编译及移植；
- 熟悉文件系统、用户环境的定制化开发；
- 掌握应用软件在不同平台的迁移及开发技术。

## 开发环境

### 硬件环境

**上位机**

- 设备：台式计算机，搭载 VMware Workstation 虚拟机
- 虚拟机系统：Ubuntu 20.04 LTS 64 位
- 虚拟机配置：内存 4GB，CPU 4 核，磁盘 60GB
- 网络模式：桥接模式，与开发板处于同一局域网

**嵌入式开发板**

- 开发板型号：正点原子 ATK-I.MX6U（ARM Cortex-A7 架构，主频 792MHz）
- 显示外设：7 寸电阻 LCD 显示屏，分辨率 800×480，帧缓冲设备 `/dev/fb0`
- 网络外设：100M 以太网 FEC 网口，支持 DHCP / 静态 IP
- 输入外设：电阻触摸屏、USB Host 主口（可外接 USB 鼠标 / U 盘）
- 控制外设：用户 LED 指示灯、有源蜂鸣器，挂载 sysfs 设备驱动
- 存储介质：8GB EMMC，存放 Linux 内核、设备树、根文件系统
- 调试接口：USB 转串口（UART 调试串口，波特率 115200）

**辅助硬件**

- USB 转 TTL 串口模块：用于串口终端登录开发板
- 千兆网线：连接虚拟机与开发板网口，实现局域网通信
- USB 有线鼠标：替代故障触摸屏完成界面点击交互
- 5V/2A 电源适配器：给 I.MX6U 开发板供电

**网络拓扑**

| 节点 | IP |
| --- | --- |
| Ubuntu 虚拟机 | 192.168.1.200 |
| MX6U 开发板（静态） | 192.168.1.201 |
| 网关路由器 | 192.168.1.1 |

子网掩码统一为 `255.255.255.0`，通信方式为 TCP/IP，使用 ssh 远程终端、scp 文件传输。

### 软件环境

**交叉编译工具链**

- 版本：`gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf`
- 关键说明：后缀 `gnueabihf` 代表 ARM 硬浮点架构，与 I.MX6U 根文件系统 ABI 完全匹配。若使用软浮点工具链会出现动态库加载失败、程序段错误。
- 存放路径：`/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin`

核心编译工具：

| 工具 | 用途 |
| --- | --- |
| `arm-linux-gnueabihf-gcc` | C 语言交叉编译器 |
| `arm-linux-gnueabihf-g++` | C++ 交叉编译器 |
| `arm-linux-gnueabihf-strip` | 程序瘦身工具（可选） |

**源码软件包**

- Qt 图形库源码：`qt-everywhere-src-5.12.9`，存放路径 `/home/ll/book/qt-everywhere-src-5.12.9`
- 触摸屏驱动库：`tslib-1.21`，存放路径 `/home/ll/book/tslib-1.21`
- 中文字体文件：`wqy-zenhei.ttc`（文泉驿正黑开源中文字体）

**开发与调试工具**

- Qt Creator 5.12：Qt 图形界面开发 IDE，配置 ARM 交叉编译套件
- 串口终端工具：picocom，串口参数 `115200 8N1`
- 文件传输工具：scp（ssh 协议远程传输）、rzsz（串口传输备选）
- 文本编辑器：gedit、vi/vim
- 架构查看工具：file、readelf，校验 ARM/x86 程序架构

**根文件系统说明**

根文件系统基于 ARMhf 硬浮点架构，内置功能：

- 动态链接器 `/lib/ld-linux-armhf.so.3`，支持 ARM32 动态可执行程序
- 网络服务：udhcpc 自动 IP 分配、ssh-server 远程登录服务
- 设备驱动：LCD 帧缓冲 fb0、触摸屏 event 输入设备、LED / 蜂鸣器 sysfs 驱动
- 开机启动脚本：`/etc/init.d/rcS`，系统上电自动执行

## 一、ARM 交叉编译 Qt5.12.9 源码

### 1. 编译 tslib

先将压缩包上传到 Ubuntu，解压并进入源码存放目录：

```bash
tar xvf tslib-1.21.tar.bz2
```

编译 tslib 之前需要安装依赖：

```bash
sudo apt-get install autoconf
sudo apt-get install automake
sudo apt-get install libtool
```

然后执行编译命令：

```bash
./configure --host=arm-linux-gnueabihf \
    ac_cv_func_malloc_0_nonnull=yes \
    --cache-file=arm-linux-gnueabihf.cache \
    -prefix=/home/book/tslib-1.21/tslib-arm
make
make install
```

### 2. 编译 Qt 库源码

解压 Qt 源码：

```bash
tar -xvf qt-everywhere-src-5.12.9.tar.xz
```

修改 Qt 源码中的 `mkspecs` 配置（`make.conf`），注意这里的交叉编译器一定要是 Linux 自己的 ARM 交叉编译器。

进入 `qt-everywhere-src-5.12.9` 文件夹下，配置一个编辑文件 `autoconfig.sh`，完整 configure 命令如下：

```bash
./configure -prefix /home/ll/book/qt-everywhere-src-5.12.9/arm-qt \
-opensource \
-confirm-license \
-release \
-strip \
-shared \
-xplatform linux-arm-gnueabi-g++ \
-optimized-qmake \
-c++std c++11 \
--rpath=no \
-pch \
-skip qt3d \
-skip qtactiveqt \
-skip qtandroidextras \
-skip qtcanvas3d \
-skip qtconnectivity \
-skip qtdatavis3d \
-skip qtdoc \
-skip qtgamepad \
-skip qtlocation \
-skip qtmacextras \
-skip qtnetworkauth \
-skip qtpurchasing \
-skip qtremoteobjects \
-skip qtscript \
-skip qtscxml \
-skip qtsensors \
-skip qtspeech \
-skip qtsvg \
-skip qttools \
-skip qttranslations \
-skip qtwayland \
-skip qtwebengine \
-skip qtwebview \
-skip qtwinextras \
-skip qtx11extras \
-skip qtxmlpatterns \
-make libs \
-make examples \
-nomake tools -nomake tests \
-gui \
-widgets \
-dbus-runtime \
--glib=no \
--iconv=no \
--pcre=qt \
--zlib=qt \
-no-openssl \
--freetype=qt \
--harfbuzz=qt \
-no-opengl \
-linuxfb \
--xcb=no \
-tslib \
--libpng=qt \
--libjpeg=qt \
--sqlite=qt \
-plugin-sql-sqlite \
-I/home/ll/book/tslib-1.21/tslib-arm/include \
-L/home/ll/book/tslib-1.21/tslib-arm/lib \
-recheck-all
```

> **注意** 其中三处路径需要根据实际情况修改：
> 1. `/home/ll/book/qt-everywhere-src-5.12.9/arm-qt` —— Qt 库编译输出路径
> 2. `/home/ll/book/tslib-1.21/tslib-arm/include` —— tslib 头文件路径
> 3. `/home/ll/book/tslib-1.21/tslib-arm/lib` —— tslib 库文件路径

赋予脚本可执行权限，然后执行：

```bash
chmod +x autoconfigure.sh
./autoconfigure.sh
```

configure 成功后，开始编译和安装：

```bash
make -j4
make install
```

安装成功会生成一个 `arm-qt` 文件夹。这个编译等待的时长比较久，耐心等待即可。

## 二、Qt 库移植

### 1. 启动串口

正常启动系统、挂载 nfs 时可以看到 `root@ATK-IMX6U:~#` 提示符。但我这里卡在了 TFTP 加载阶段（报 ARP 超时）。

更换设备树，手动输入启动命令后，仍然出现卡住、网络超时的现象。最后我更换了启动命令，使用 EMMC 本地系统，再挂载 nfs 文件，问题解决。

### 2. 保证网络通畅

强制设置静态 IP（和虚拟机同一网段）、添加默认网关并验证：

```bash
# 1. 强制设置静态 IP（和虚拟机同一网段）
ifconfig eth0 192.168.1.201 netmask 255.255.255.0 up
# 2. 添加默认网关
route add default gw 192.168.1.1
# 3. 验证网络是否 ping 通
ping 192.168.1.200 -c 4
```

ping 通后即可进行后续文件传输。

### 3. scp 上传 tslib 与 arm-qt

通过 scp 将完整的 tslib 与 arm-qt 上传至开发板，并在开发板串口校验目录完整性。

### 4. 踩坑：x86 版本与 ARM 版本不兼容

这里出现了一个比较棘手的问题：我前几次编译的 `arm-qt` 都是 x86-64 版本，不是 ARM 版本。

这会导致后面运行可执行文件时报找不到 `linuxfb` 库的错误。这个库其实是有的，但是版本不兼容会直接让程序崩溃。

这个问题本质是 **插件与 Qt 主库版本不兼容**。刚开始的解决方法是修改环境变量、尝试软连接、强制指定插件路径，都解决不了。只能查询更详细的问题，排查到 **库架构 / 编译浮点 ABI 和 i.MX6U 板子不匹配**，表现为非法指令：

> Qt 尝试加载 `libqlinuxfb.so` → 加载触发非法指令直接崩溃 → Qt 捕获加载失败，判定该平台插件不存在，于是打印：
> `Could not find the Qt platform plugin "linuxfb" in ""`

**最终解决办法：重新交叉编译 Qt。**

在 `~/qt-everywhere-src-5.12.9$` 源码目录下运行，先清理之前的编译产物：

```bash
make distclean
rm -rf config.* Makefile
export TSLIB_PATH=/home/ll/book/tslib-1.21/install
```

完整 configure 命令：

```bash
./configure \
-prefix /usr/lib/arm-qt \
-xplatform linux-arm-gnueabi-g++ \
-opensource \
-confirm-license \
-release \
-shared \
-no-opengl \
-no-eglfs \
-linuxfb \
-tslib \
-I${TSLIB_PATH}/include \
-L${TSLIB_PATH}/lib \
-nomake examples \
-nomake tests \
-nomake tools \
-skip qtdoc
```

之后重新编译、下载，并检查是否是 ARM 版本。

## 三、程序运行

### 1. 测试板子库是否可用

确认环境变量当前值，若未设置或为空则重新设置并导出：

```bash
# 确认环境变量当前值
echo "QT_QPA_PLATFORM_PLUGIN_PATH = $QT_QPA_PLATFORM_PLUGIN_PATH"
echo "QT_QPA_PLATFORM = $QT_QPA_PLATFORM"

# 如果未设置或为空，重新设置并导出
export QT_QPA_PLATFORM_PLUGIN_PATH=/usr/lib/arm-qt/plugins
export QT_QPA_PLATFORM=linuxfb:fb=/dev/fb0
export QT_QPA_FB_TSLIB=1

# 再次确认
echo "After export:"
echo "QT_QPA_PLATFORM_PLUGIN_PATH = $QT_QPA_PLATFORM_PLUGIN_PATH"
echo "QT_QPA_PLATFORM = $QT_QPA_PLATFORM"

# 运行示例程序
/usr/lib/arm-qt/examples/widgets/animation/animatedtiles/animatedtiles
```

> 这个示例显示不完整，可能是开发板的问题，尝试了几次都这样。接着测试触摸。

### 2. 创建自己的可执行文件

先创建一个简单的程序执行，确认流程跑通后，再开始创建自己的可执行文件——我做的是一个 **计算器 + LED 控制** 的界面。

#### calculatorwidget.h

```cpp
#ifndef CALCULATORWIDGET_H
#define CALCULATORWIDGET_H

#include <QWidget>
#include <QLineEdit>
#include <QPushButton>
#include <QGridLayout>
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QFile>
#include <QDebug>

class CalculatorWidget : public QWidget
{
    Q_OBJECT
public:
    explicit CalculatorWidget(QWidget *parent = nullptr);

private slots:
    void digitClicked();
    void operatorClicked();
    void equalsClicked();
    void clearClicked();
    void toggleLED();

private:
    QLineEdit *display;
    QString currentInput;
    QString previousOperator;
    double firstOperand;
    bool newInput;

    const QString LED_PATH = "/sys/class/leds/sys-led/brightness";
    bool ledState;
    QPushButton *ledBtn;

    QPushButton* createButton(const QString &text);
    void updateDisplay();
    void calculate(double secondOperand);
};

#endif // CALCULATORWIDGET_H
```

#### main.cpp

```cpp
#include <QApplication>
#include "calculatorwidget.h"

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    CalculatorWidget w;
    w.show();
    return a.exec();
}
```

#### calculatorwidget.cpp

```cpp
#include "calculatorwidget.h"
#include <QFontDatabase>
#include <QFont>
#include <QHBoxLayout>

CalculatorWidget::CalculatorWidget(QWidget *parent)
    : QWidget(parent), currentInput(""), previousOperator(""), firstOperand(0), newInput(true), ledState(false)
{
    setWindowTitle("计算器 + LED控制");
    setFixedSize(800, 480);

    // ---------- 加载中文字体 ----------
    int fontId = QFontDatabase::addApplicationFont("/usr/share/fonts/ttf/SourceHanSansCN-Regular.otf");
    QFont baseFont;
    if (fontId != -1) {
        QString family = QFontDatabase::applicationFontFamilies(fontId).at(0);
        baseFont.setFamily(family);
    } else {
        baseFont.setFamily("sans-serif");
    }
    baseFont.setPointSize(12);
    this->setFont(baseFont);
    // ---------------------------------

    // ---------- 整体深色背景 ----------
    setStyleSheet(
        "QWidget { background-color: #1e1e1e; }"
        "QLineEdit {"
        "    background-color: black;"
        "    color: #00ff00;"
        "    font-size: 28px;"
        "    padding: 8px;"
        "    border: 2px solid #444;"
        "    border-radius: 8px;"
        "    font-weight: bold;"
        "}"
    );
    // ---------------------------------

    // 主布局：水平方向，左半边黑色占位，右半边显示内容
    QHBoxLayout *mainLayout = new QHBoxLayout(this);
    mainLayout->setContentsMargins(0, 0, 0, 0);
    mainLayout->setSpacing(0);

    // 左半边黑色占位
    QWidget *leftSpacer = new QWidget(this);
    leftSpacer->setStyleSheet("background-color: black;");
    leftSpacer->setFixedWidth(500);
    mainLayout->addWidget(leftSpacer);

    // 右半边：宽度固定 300px
    QWidget *rightWidget = new QWidget(this);
    rightWidget->setFixedWidth(300);
    QVBoxLayout *rightLayout = new QVBoxLayout(rightWidget);
    rightLayout->setContentsMargins(10, 10, 10, 10);
    rightLayout->setSpacing(10);

    // 显示框
    display = new QLineEdit("0");
    display->setReadOnly(true);
    display->setAlignment(Qt::AlignRight);
    rightLayout->addWidget(display);

    // 按钮网格
    QGridLayout *grid = new QGridLayout;
    grid->setSpacing(8);

    // 创建按钮
    QPushButton *btn0 = createButton("0");
    QPushButton *btn1 = createButton("1");
    QPushButton *btn2 = createButton("2");
    QPushButton *btn3 = createButton("3");
    QPushButton *btn4 = createButton("4");
    QPushButton *btn5 = createButton("5");
    QPushButton *btn6 = createButton("6");
    QPushButton *btn7 = createButton("7");
    QPushButton *btn8 = createButton("8");
    QPushButton *btn9 = createButton("9");
    QPushButton *btnAdd = createButton("+");
    QPushButton *btnSub = createButton("-");
    QPushButton *btnMul = createButton("*");
    QPushButton *btnDiv = createButton("/");
    QPushButton *btnEq = createButton("=");
    QPushButton *btnClr = createButton("C");

    // 应用颜色样式
    QString digitStyle = "background: #333; color: white;";
    btn0->setStyleSheet(digitStyle);
    btn1->setStyleSheet(digitStyle);
    btn2->setStyleSheet(digitStyle);
    btn3->setStyleSheet(digitStyle);
    btn4->setStyleSheet(digitStyle);
    btn5->setStyleSheet(digitStyle);
    btn6->setStyleSheet(digitStyle);
    btn7->setStyleSheet(digitStyle);
    btn8->setStyleSheet(digitStyle);
    btn9->setStyleSheet(digitStyle);

    btnAdd->setStyleSheet("background: #ff9800; color: white;");
    btnSub->setStyleSheet("background: #ff9800; color: white;");
    btnMul->setStyleSheet("background: #ff9800; color: white;");
    btnDiv->setStyleSheet("background: #ff9800; color: white;");
    btnEq->setStyleSheet("background: #4CAF50; color: white;");
    btnClr->setStyleSheet("background: #f44336; color: white;");

    // LED 按钮
    ledBtn = new QPushButton("LED OFF");
    ledBtn->setStyleSheet(
        "QPushButton {"
        "    background: qlineargradient(x1:0, y1:0, x2:1, y2:0,"
        "        stop:0 #2196F3, stop:1 #1976D2);"
        "    color: white;"
        "    font-size: 16px;"
        "    padding: 10px;"
        "    border-radius: 8px;"
        "    font-weight: bold;"
        "}"
        "QPushButton:pressed { background: #0D47A1; }"
    );
    connect(ledBtn, &QPushButton::clicked, this, &CalculatorWidget::toggleLED);

    // 网格布局添加按钮（4行4列）
    grid->addWidget(btn7, 0, 0);
    grid->addWidget(btn8, 0, 1);
    grid->addWidget(btn9, 0, 2);
    grid->addWidget(btnDiv, 0, 3);
    grid->addWidget(btn4, 1, 0);
    grid->addWidget(btn5, 1, 1);
    grid->addWidget(btn6, 1, 2);
    grid->addWidget(btnMul, 1, 3);
    grid->addWidget(btn1, 2, 0);
    grid->addWidget(btn2, 2, 1);
    grid->addWidget(btn3, 2, 2);
    grid->addWidget(btnSub, 2, 3);
    grid->addWidget(btn0, 3, 0);
    grid->addWidget(btnClr, 3, 1);
    grid->addWidget(btnEq, 3, 2);
    grid->addWidget(btnAdd, 3, 3);

    rightLayout->addLayout(grid);
    rightLayout->addWidget(ledBtn);

    mainLayout->addWidget(rightWidget);

    // 信号连接
    QList<QPushButton*> digits = {btn0, btn1, btn2, btn3, btn4, btn5, btn6, btn7, btn8, btn9};
    for (auto btn : digits) connect(btn, &QPushButton::clicked, this, &CalculatorWidget::digitClicked);
    connect(btnAdd, &QPushButton::clicked, this, &CalculatorWidget::operatorClicked);
    connect(btnSub, &QPushButton::clicked, this, &CalculatorWidget::operatorClicked);
    connect(btnMul, &QPushButton::clicked, this, &CalculatorWidget::operatorClicked);
    connect(btnDiv, &QPushButton::clicked, this, &CalculatorWidget::operatorClicked);
    connect(btnEq, &QPushButton::clicked, this, &CalculatorWidget::equalsClicked);
    connect(btnClr, &QPushButton::clicked, this, &CalculatorWidget::clearClicked);
}

QPushButton* CalculatorWidget::createButton(const QString &text)
{
    QPushButton *btn = new QPushButton(text);
    btn->setFixedSize(60, 50);
    btn->setFont(QFont("", 18));
    btn->setStyleSheet(
        "QPushButton {"
        "    border-radius: 6px;"
        "    font-weight: bold;"
        "    border: none;"
        "}"
        "QPushButton:pressed { opacity: 0.7; }"
    );
    return btn;
}

// ---------- 槽函数实现 ----------
void CalculatorWidget::digitClicked()
{
    QPushButton *btn = qobject_cast<QPushButton*>(sender());
    if (!btn) return;
    if (newInput) {
        currentInput = "";
        newInput = false;
    }
    currentInput += btn->text();
    updateDisplay();
}

void CalculatorWidget::operatorClicked()
{
    QPushButton *btn = qobject_cast<QPushButton*>(sender());
    if (!btn) return;
    if (!newInput) {
        calculate(currentInput.toDouble());
    }
    previousOperator = btn->text();
    firstOperand = display->text().toDouble();
    newInput = true;
}

void CalculatorWidget::equalsClicked()
{
    calculate(currentInput.toDouble());
    previousOperator.clear();
    newInput = true;
}

void CalculatorWidget::clearClicked()
{
    currentInput.clear();
    firstOperand = 0;
    previousOperator.clear();
    newInput = true;
    display->setText("0");
}

void CalculatorWidget::calculate(double secondOperand)
{
    double result = 0;
    if (previousOperator == "+") result = firstOperand + secondOperand;
    else if (previousOperator == "-") result = firstOperand - secondOperand;
    else if (previousOperator == "*") result = firstOperand * secondOperand;
    else if (previousOperator == "/") {
        if (secondOperand == 0) {
            display->setText("Error");
            return;
        }
        result = firstOperand / secondOperand;
    } else {
        display->setText(QString::number(secondOperand));
        return;
    }
    display->setText(QString::number(result));
}

void CalculatorWidget::updateDisplay()
{
    display->setText(currentInput.isEmpty() ? "0" : currentInput);
}

void CalculatorWidget::toggleLED()
{
    QFile file(LED_PATH);
    if (!file.open(QIODevice::WriteOnly)) {
        qDebug() << "Failed to open LED device";
        return;
    }
    ledState = !ledState;
    file.write(ledState ? "1" : "0");
    file.close();
    ledBtn->setText(ledState ? "LED ON" : "LED OFF");
}
```

#### calculator.pro

```qmake
QT += widgets
SOURCES = main.cpp calculatorwidget.cpp
HEADERS = calculatorwidget.h
TARGET = calculator
CONFIG += release
```

### 3. 编译、上传与运行

交叉编译之后，用 `file` 命令检查生成的可执行文件是否是 ARM 架构，确认后通过 scp 上传到开发板：

```bash
scp calculator root@192.168.1.201:/home/root/
```

在开发板 `/home/root` 目录下查看并运行可执行文件。

> 开发板上的界面显示和前面检验 arm-qt 时一样，左边看不见，所以我直接让左半边显示黑色占位，右半边显示计算器所有组件——这就是代码里 `leftSpacer` 的由来。

### 4. 开机自启

创建启动脚本：

```bash
vi calc_start.sh
```

赋予执行权限：

```bash
chmod 755 calc_start.sh
```

运行测试：

```bash
./calc_start.sh
```

确认无误后，打开开机脚本 `rcS`：

```bash
vi /etc/init.d/rcS
```

在文件末尾添加一行：

```bash
/root/calc_start.sh
```

最后重启开发板即可开机自动运行：

```bash
reboot
```

## 实验总结

本次大作业完成了 I.MX6U 嵌入式整套适配开发：完成 tslib、Qt5.12.9 交叉移植，自主编写计算器 Qt 程序，可实现数字计算、LED 开关控制。

实验中遇到网络 IP 异常、Qt 缺少中文字体、tslib 触摸报错程序段错误、Qt 平台插件不兼容等问题，通过静态 IP 配置、部署字体、环境变量屏蔽触摸、重新匹配 ARM 编译工具链全部解决。

掌握的实操技能：交叉编译、嵌入式 Qt 开发、sysfs 硬件控制、Linux 环境变量、串口 / ssh 文件传输、开机脚本编写等。

## 问题汇总

| 编号 | 故障现象 |
| --- | --- |
| 1 | scp 传输文件提示文件不存在 / Is a directory |
| 2 | Qt 界面仅显示按钮空白边框，无中文文字 |
| 3 | tslib 加载 input 模块失败、ts_config 报错、程序段错误 |
| 4 | 开发板 DHCP 获取 169.254 无效 IP，ssh/scp 连接失败 |
| 5 | Qt 启动提示找不到 linuxfb 平台插件，程序 Aborted |

这些问题在前文实验步骤中均有提及与解决，此处不再赘述。