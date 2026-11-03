# Android App

[English](../app.md) | 简体中文

App 是同一个 `usb-arsenal` 二进制的前端。它负责安装后端，里面的每个开关和按键都通过 root
执行一条 `usb-arsenal` 命令。手机的端口从不被 App 自己占用。它做的每件事，root shell
也能做（[usage.md](usage.md)）。

## 环境要求

- Android 8.0 或更新版本，已 root。推荐 KernelSU。
- 内核能通过 `usb-arsenal check` 的检查（[内核要求](../../kernel-patches/README.md)）。
  App 在安装任何东西之前会替你运行这项检查。

## 安装

从 [release](https://github.com/ravindu644/USB-Arsenal/releases) 取
`usb-arsenal-v<version>-<hash>.apk` 安装。同一版本的每次构建都会更新前一个。

首次启动会走一遍设置流程：

1. **Root。** 根管理器询问时授予 App root 权限。如果拒绝了，就在管理器里授予，然后
   重新打开 App。
2. **控制器。** 仅在手机有多个 USB device 控制器时出现。选线插在的那个；开心的 Pin
   标出电脑正在使用的那个。之后可以在设置里更改。
3. **检查。** 随附的二进制会探测内核。`missing` 或 `caveat` 行会说明代价；无法运行
   USB Arsenal 的手机会停在这里，并给出原因。
4. **后端。** 把二进制安装到 `/data/local/USB-Arsenal/bin/usb-arsenal`，并安装开机
   模块。如果此前模块没有 SELinux 规则，会提示重启一次。

## 使用

**首页**显示当前状态。每个功能都有一行，点开进入对应界面，还有一个开关，按该界面的
设置启动或停止它。底部的状态栏显示端口在谁手里：Android，或者 gadget 以及哪个控制器。
**Stop all** 会停止一切，把端口还给 Android，ADB 和 MTP 随之恢复。

**存储**从 `/sdcard/USB-Arsenal/Disks` 提供镜像：以 `.iso`、`.img`、`.bin`、
`.raw`、`.ima` 或 `.dd` 结尾的文件。其他文件不会列出，**Pick image** 也会拒绝它们。
在条带上选一个 LUN（最多八个）、给它选一个镜像，以及主机看到它的方式：

| 模式 | 主机看到 |
|---|---|
| `auto` | 镜像有分区表就是磁盘，否则是 CD-ROM；超过 CD-ROM 上限则是 `dvd` |
| `cd` | 一个 CD-ROM |
| `dvd` | 一个 2048 字节扇区的只读磁盘，用于任意大小的普通 ISO（Windows ISO） |
| `disk` | 一个只读磁盘 |
| `rw` | 一个可写磁盘 |

[usage.md](usage.md#storage) 有详细说明。

**Create** 在那里新建镜像（稀疏文件瞬间完成，也可以完整写出）。**Pick image** 打开系统
选择器；给不出真实文件路径的选择器会被拒绝，这种情况下请把镜像复制到该文件夹。

**共享网络**把手机的网络共享给主机：NCM，老版本 Windows 用 RNDIS，使用你设置的地址和
DNS，上游从手机的上行链路里选（每个有默认路由的接口），或者用 `auto`。安装了
[Droidspaces](https://github.com/ravindu644/Droidspaces-OSS)
后，网关模式会把链路交给它正在运行的某个容器
（[tethering.md](tethering.md)）。

**HID** 设置报告间隔，并打开**键鼠**：一块带笔记本手势的触摸板（点按、双指和三指点按、
点按拖动、双指滚动、捏合），一条滚轮条和 L M R 按键，以及一个按键页，包含 F1 到 F12、
编辑键、修饰键和方向键，还有一个输入框，主机收到的就是框里的文字。速度和自然滚动在
触摸板下方。全屏按钮会把手机在横屏下变成一整块宽触摸板。手柄的左摇杆移动指针；A 和 X
点击。

**日志**显示后端的日志：从 App 或 shell 执行的每条命令及其输出，也包括共享网络进程的
输出，按时间排序，跨启动保留。Clear 会清空它。

**设置**：动效、触感和主题（十一种双色 LCD 外观，默认 Amber），身份配置档
（[identity.md](identity.md)；配置档存放在 `/sdcard/USB-Arsenal/Profiles`），
多控制器手机上的控制器选择，为 root shell 把 `usb-arsenal` 放进 PATH，Requirements
（内核更新后再做一次内核检查）以及 Reinstall。

**每个界面设置的东西都属于后端。** 各项选择、共享网络选项、HID 间隔和控制器都存放在
`/data/local/USB-Arsenal/Gadget/setup.conf`，也就是 `usb-arsenal setup` 编辑的同一个
文件（[usage.md](usage.md#setup)），所以在 App 里做的设置，从 root shell 直接执行
`usb-arsenal storage start` 就能用上；在 shell 里做的设置也会显示在 App 里；清除 App
数据或重装都不会丢失任何东西。每个界面会先画出它上次知道的状态，等后端回应后再纠正；
三次读取都没有回应后，首页的状态栏会显示 `no answer from usb-arsenal`，并继续显示
最后一次的结果。

## 不用触摸屏

每个界面都能用键盘、方向键或手柄操作：Tab 或方向键移动，Enter 或 A 按下，Back、
Escape 或 B 返回，L1 和 R1 在聚焦的分段之间切换。TalkBack 会读出每个控件的名称和
状态。App 遵循系统的字体大小和"移除动画"设置，并自带动效和触感开关。

## 测试环境

[README](../../README.zh.md#tested-on) 中列出的 Galaxy A16 5G（内核 5.15）和 Galaxy S10
（内核 4.14），主机为一台 Fedora PC。
