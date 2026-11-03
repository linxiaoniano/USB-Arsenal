# Installation

[English](../installation.md) | 简体中文

USB Arsenal 是一个静态二进制文件。在 Android 上由 App 或 KernelSU 模块安装，
模块在开机时加载它的 SELinux 规则。在 Linux 上你只需复制这个二进制文件。

## Android：KernelSU 模块

1. 构建 zip，或从 release 中获取：
   ```sh
   make module          # output/usb-arsenal-v<version>.zip, every architecture inside
   ```
2. 在 KernelSU 中刷入并重启。（模块遵循标准 KernelSU/Magisk 布局，但尚未在测试设备
   上刷入过；App 安装的是同样的文件，该安装方式已经测试过。）
3. 在 root shell 中：
   ```sh
   usb-arsenal check
   ```

模块做的事：
- **按设备的 ABI 安装二进制文件**，位置是 `/data/local/USB-Arsenal/bin/usb-arsenal`，
  并把 `/system/bin/usb-arsenal` 链接到它，因此重启后在任意 root shell 中都能直接
  使用 `usb-arsenal`。（从 App 安装时，该链接是一个设置开关。）
- **在开机时加载 `sepolicy.rule`，** 它让内核能通过 loop 设备读取 dvd LUN 的镜像。
- **卸载时归还端口。**

后台不运行任何东西：每条命令执行完就退出（只有 `tether` 会保留它的网络进程）。
容器自己运行这个二进制文件；见 [containers.md](containers.md)。

## Android：手动安装

```sh
make aarch64                        # or armhf / x86_64 / x86
adb push output/usb-arsenal /data/local/tmp/
adb shell su -c "/data/local/tmp/usb-arsenal check"
```

从 Android 接管 USB 端口会中断 USB 上的 ADB，直到 `usb-arsenal stop`。请在手机上的
root 终端、容器内通过 SSH，或用 `nohup` 通过 ADB 运行命令，这样命令在 ADB 断开后
仍能执行完。

## Linux 开发板

开发板的 USB 端口必须能作为设备工作。在树莓派上，是 Pi 4 或 5 的 USB-C 端口，或
Zero 的数据 micro-USB 端口：

```
# /boot/firmware/config.txt
dtoverlay=dwc2
```

USB Arsenal 在缺失时自行挂载 configfs 并加载 `libcomposite`。然后：

```sh
make native          # or make aarch64 / armhf to cross-compile
sudo ./output/usb-arsenal check
```

在 Linux 上，工作目录是 `/var/lib/USB-Arsenal`。如果已有别的 gadget 占用了该端口，
USB Arsenal 会指出它并拒绝去动它。

## 构建

构建是静态的，链接 musl。`make` 会在 `$MUSL_CROSS`、
`~/toolchains/<triplet>-cross/bin`、`PATH` 和 `/opt/cross/bin` 中查找
`<triplet>-gcc`。

| 目标 | 输出 |
|---|---|
| `make native` | 本机架构 |
| `make aarch64`, `armhf`, `x86_64`, `x86` | 交叉构建到 `output/usb-arsenal` |
| `make module` | 包含全部四个架构的 KernelSU/Magisk zip |

## 卸载

在 KernelSU 中移除模块：它的卸载脚本会运行 `usb-arsenal stop` 并删除二进制文件。
手动安装的，运行 `usb-arsenal stop`，然后删除二进制文件和
`/data/local/USB-Arsenal`（或 `/var/lib/USB-Arsenal`）。
