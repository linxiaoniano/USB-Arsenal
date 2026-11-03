<img src="docs/resources/readme/hero.png" alt="USB Arsenal：从手机启动 ISO 或共享磁盘，通过 USB 线共享网络，在主机上使用键盘和鼠标" width="100%">

[![Latest release](https://img.shields.io/github/v/release/ravindu644/USB-Arsenal?label=Release&style=for-the-badge)](https://github.com/ravindu644/USB-Arsenal/releases/latest)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg?style=for-the-badge)](./LICENSE)

# USB Arsenal

[English](README.md) | 简体中文

USB Arsenal 通过一根线，把已 root 的 Android 手机同时变成多个 USB 设备：一个存放任意
镜像的 USB 驱动器或 CD-ROM，一个共享手机网络的网卡，以及一套由手机驱动的键盘和鼠标。

App 会替你完成全部设置，停止时把 USB 端口还给 Android，ADB 和文件传输随即恢复。

## 你能得到什么

- **用手机启动电脑。** 把 ISO 或磁盘镜像放进手机，电脑就会把它识别为 USB 驱动器或
  CD/DVD。App 会根据镜像自动选出正确的那一种。
- **USB 驱动器。** 把任意镜像文件共享给电脑，只读或可写，最多同时八个。
- **共享手机网络。** 电脑通过这根线上网。也可以让一个
  [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) OpenWrt 容器充当
  路由器。
- **键盘和鼠标。** 在 App 里就能在电脑上打字、移动指针。
- **用脚本打字。** 写一个 [Quacky](https://github.com/ravindu644/Quacky) 脚本，
  手机就会以全速把它输入到电脑，一次运行完成整个安装
  （[语言](https://github.com/ravindu644/Quacky/blob/main/docs/language.md)）。
- **同时使用。** 驱动器、网络和键盘共用一根线，每一项都能单独开关，互不影响。
- **决定呈现方式。** 选择电脑看到的设备名称和 ID。
- **总能恢复。** 一旦出问题，全部停止会把端口还给 Android。
- **十一种 LCD 主题**，从默认的琥珀色到 Game Boy 绿。

同一个二进制也能运行在带 USB device 端口的 Linux 板子上，以及特权
[Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) 容器里。

## 用手机安装电脑

U 盘、键盘和网线，全都合在一根 USB 线里。把手机插到一台只有屏幕、别无他物的机器上：

1. **存储**提供安装 ISO，机器从它启动。
2. **键鼠**从手机向安装程序输入，可以手动，也可以用 [Quacky](https://github.com/ravindu644/Quacky)
   脚本跑完整个安装。
3. **共享网络**让安装程序通过手机联网，下载软件包和更新。

三者同时运行，关掉其中一个不影响其他。

## 环境要求

- 一台已 root、运行 Android 8 或更新版本的手机。
- 内核支持 USB configfs。App 在安装任何东西之前会替你检查
  （[内核要求](kernel-patches/README.zh.md)）。

## 快速开始

1. 从[最新版本](https://github.com/ravindu644/USB-Arsenal/releases/latest)安装 APK，
   授予它 root 权限，让它安装后端。
2. 把 ISO 或磁盘镜像放到 `/sdcard/USB-Arsenal/Disks`，打开**存储**并选中它。
3. 在首页打开开关，把手机插到电脑上。

App 能做的每件事，root shell 都能用 `usb-arsenal` 做到（[用法](docs/zh/usage.md)）。
Linux 板子和 KernelSU 模块见[安装指南](docs/zh/installation.md)。

## 截图

<table>
  <tr>
    <td width="33%" align="center"><img src="docs/resources/readme/01-home.png" width="100%"><br>首页</td>
    <td width="33%" align="center"><img src="docs/resources/readme/02-storage.png" width="100%"><br>存储</td>
    <td width="33%" align="center"><img src="docs/resources/readme/03-tether.png" width="100%"><br>共享网络</td>
  </tr>
  <tr>
    <td width="33%" align="center"><img src="docs/resources/readme/04-keys-mouse.png" width="100%"><br>键鼠</td>
    <td width="33%" align="center"><img src="docs/resources/readme/05-identity.png" width="100%"><br>身份</td>
    <td width="33%" align="center"><img src="docs/resources/readme/06-themes.png" width="100%"><br>主题</td>
  </tr>
</table>

## 安全

> [!WARNING]
>
> USB Arsenal 以 root 运行，并控制手机的 USB 端口。
>
> 任何能写入 `/dev/hidg*` 节点的人，都能在它所连接的电脑上打字。这些节点默认是
> `0660 root`；只在确有需要时才放宽权限。
>
> 配置档可以借用真实厂商的 USB ID。这由你自己决定。

## 文档

| 页面 | 内容 |
|------|----------------|
| [App](docs/zh/app.md) | 参数设置、各个界面、按键和 TalkBack。 |
| [安装](docs/zh/installation.md) | KernelSU 模块、手动安装、Linux 板子。 |
| [用法](docs/zh/usage.md) | 每个 `usb-arsenal` 命令和选项，附示例。 |
| [共享网络](docs/zh/tethering.md) | NAT 模式、OpenWrt 网关模式、主机一侧。 |
| [容器](docs/zh/containers.md) | 在容器里运行 USB Arsenal。 |
| [身份配置档](docs/zh/identity.md) | 厂商和产品 ID、名称、序列号、默认值。 |
| [内核要求](kernel-patches/README.md) | 按功能列出的内核选项和补丁。 |
| [故障排查](docs/zh/troubleshooting.md) | 现象、原因和解决办法。 |

开发者笔记（架构、内核调研、设计语言、路线图）在
[docs/developer](docs/developer/) 里。

## 致谢

- Linux USB gadget 子系统：真正干活的是 `f_mass_storage`、`f_hid`、`f_ncm`、`f_rndis`
  和 configfs。
- DriveDroid，它证明了手机能启动 PC。
- [libinput](https://gitlab.freedesktop.org/libinput/libinput)，App 的触摸板遵循它的
  点按、滚动和手势规则。
- [Jersey 10](https://fonts.google.com/specimen/Jersey+10)、[Silkscreen](https://fonts.google.com/specimen/Silkscreen)
  和 [VT323](https://fonts.google.com/specimen/VT323)，App 使用的字体，遵循
  [SIL Open Font License 1.1](https://scripts.sil.org/OFL)。

## 许可证

[GNU General Public License v3.0](./LICENSE)。版权所有 (C) 2026
[ravindu644](https://github.com/ravindu644)。
