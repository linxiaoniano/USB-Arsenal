# 内核要求

[English](README.md) | 简体中文

`usb-arsenal check` 会询问正在运行的内核它能做什么。本页说明每项功能需要内核内建什么，以及某项
功能在哪些内核版本上需要本目录里的哪些补丁。

## 选项

| 选项 | 用于 |
|---|---|
| `CONFIG_USB_CONFIGFS` | 全部：gadget 接口（改用旧式 android gadget 构建的 3.18 内核需要重新构建） |
| 支持外设模式的 UDC 驱动 | 全部 |
| `CONFIG_USB_CONFIGFS_MASS_STORAGE` | `storage` |
| `CONFIG_USB_CONFIGFS_F_HID` | `hid` |
| `CONFIG_USB_CONFIGFS_NCM` | `tether` |
| `CONFIG_USB_CONFIGFS_RNDIS` | `tether --rndis`（5.10 之后从 GKI 中移除） |
| `CONFIG_IP_NF_NAT`, `CONFIG_IP_NF_TARGET_MASQUERADE` | `tether` 的 NAT |

模块也行：函数目录创建时，内核会加载 `usb_f_<name>`。

作为一个片段合并进 defconfig（用 `scripts/kconfig/merge_config.sh`，或直接追加到 defconfig
后面）：

```sh
# USB gadget functions for USB Arsenal
# https://github.com/ravindu644/USB-Arsenal
CONFIG_USB_CONFIGFS=y
CONFIG_USB_CONFIGFS_MASS_STORAGE=y
CONFIG_USB_CONFIGFS_F_HID=y
CONFIG_USB_CONFIGFS_NCM=y
CONFIG_USB_CONFIGFS_RNDIS=y

# NAT for tether
CONFIG_IP_NF_NAT=y
CONFIG_IP_NF_TARGET_MASQUERADE=y
```

## 补丁

每个 `<version>/` 目录存放针对该内核版本的 `git format-patch` 导出。按顺序应用一个目录：

```sh
cd /path/to/kernel
git am /path/to/USB-Arsenal/kernel-patches/<version>/*.patch
```

版本目录里的变体子目录是同一组补丁，基于大容量存储代码不同的树 rebase 过；取能干净应用的那一
组。

| 补丁 | 建议用于 | 内核 |
|---|---|---|
| configfs：允许创建多个 gadget | 能跑起来：没有它，第二个 gadget 创建失败，`check` 会指出这个补丁 | [4.14](4.14/0001-usb-gadget-configfs-allow-creating-more-than-one-gad.patch), [4.19](4.19/0001-usb-gadget-configfs-allow-creating-more-than-one-gad.patch) |
| configfs：支持主线 gadget 在 Android 旁边启动 | 能跑起来：没有它，手工构建的 gadget 永远不会绑定 | [4.14](4.14/0002-usb-gadget-configfs-support-mainline-gadget-bring-up.patch), [4.19](4.19/0002-usb-gadget-configfs-support-mainline-gadget-bring-up.patch) |
| configfs：注销 gadget 自己的 android 设备 | 保持 Android 的 USB 存活：没有它，移除任何 gadget 都会杀掉 Android 自己的，直到重启（USB Arsenal 从不移除自己的 gadget，所以两种情况都避开了这个缺陷） | [4.9](4.9/0001-usb-gadget-configfs-unregister-the-gadget-s-own-andr.patch), [变体](4.9/t295/0001-usb-gadget-configfs-unregister-the-gadget-s-own-andr.patch), [5.10](5.10/0001-usb-gadget-configfs-unregister-the-gadget-s-own-andr.patch), [5.15](5.15/0001-usb-gadget-configfs-unregister-the-gadget-s-own-andr.patch), [变体](5.15/sm8550/0001-usb-gadget-configfs-unregister-the-gadget-s-own-andr.patch), [6.6](6.6/0001-usb-gadget-android_configfs_uevent-unregister-the-ga.patch)；在 4.14 和 4.19 上是第一个补丁的一部分 |
| f_ncm：修复三星针对第二个实例的改动 | `tether` 走 NCM：没有它，带三星 NCM 附加改动的内核上 `tether` 会改用 RNDIS | [4.9](4.9/0002-usb-gadget-f_ncm-fix-Samsung-s-changes-for-a-second-.patch), [变体](4.9/t295/0002-usb-gadget-f_ncm-fix-Samsung-s-changes-for-a-second-.patch), [4.14](4.14/0003-usb-gadget-f_ncm-fix-Samsung-s-changes-for-a-second-.patch), [4.19](4.19/0003-usb-gadget-f_ncm-fix-Samsung-s-changes-for-a-second-.patch) |
| f_mass_storage：面向 Mac OS X 和 Windows 的 CD-ROM 模拟 | Windows 上的 `storage --mode cdrom`：没有它，Windows 从不挂载该光盘 | 各两个补丁：[4.9](4.9/0003-usb-gadget-f_mass_storage-Make-CD-ROM-emulation-work.patch), [4.14](4.14/0004-usb-gadget-f_mass_storage-Make-CD-ROM-emulation-work.patch), [4.19](4.19/0004-usb-gadget-f_mass_storage-Make-CD-ROM-emulation-work.patch) 以及[变体](4.9/t295/0003-usb-gadget-f_mass_storage-Make-CD-ROM-emulation-work.patch)（5.10 及以上已包含） |
| f_mass_storage：为大 CD-ROM 镜像模拟 DVD-ROM | 用于超过 2.36G 的镜像的 `storage`：没有它，主机读到的是截断的光盘（`--mode dvd` 不需要它） | [4.9](4.9/0005-usb-gadget-f_mass_storage-emulate-a-DVD-ROM-for-big-.patch), [变体](4.9/t295/0005-usb-gadget-f_mass_storage-emulate-a-DVD-ROM-for-big-.patch), [4.14](4.14/0006-usb-gadget-f_mass_storage-emulate-a-DVD-ROM-for-big-.patch), [4.19](4.19/0006-usb-gadget-f_mass_storage-emulate-a-DVD-ROM-for-big-.patch), [5.10](5.10/0002-pikvm-msd-dvd-support.patch), [5.15](5.15/0002-pikvm-msd-dvd-support.patch), [变体](5.15/sm8550/0002-usb-gadget-f_mass_storage-emulate-a-DVD-ROM-for-big-.patch), [6.6](6.6/0002-pikvm-msd-dvd-support.patch), [6.12](6.12/0001-pikvm-msd-dvd-support.patch) |
| f_hid：只在 /dev/hidgN 关闭时释放 HID 状态 | `hid`：没有它，HID 停止时仍持有节点的进程可能让内核 panic（USB Arsenal 会先关闭这类进程） | 各三个补丁：[4.9](4.9/0006-usb-gadget-f_hid-fix-f_hidg-lifetime-vs-cdev.patch)（变体也一样）, [4.14](4.14/0008-usb-gadget-f_hid-fix-f_hidg-lifetime-vs-cdev.patch), [4.19](4.19/0007-usb-gadget-f_hid-fix-f_hidg-lifetime-vs-cdev.patch) 4.19.270 之前（之后的 4.19 以及 5.10 及以上已包含） |
| f_hid：在重新绑定后保留其队列和 cdev | `hid`：没有它，重新插拔时仍持有节点的进程可能让内核 panic（USB Arsenal 会先关闭这类进程） | 各两个补丁，在上面三个适用的地方排在其后：[4.9](4.9/0009-usb-gadget-f_hid-move-list-and-spinlock-inits-from-b.patch), [4.14](4.14/0011-usb-gadget-f_hid-move-list-and-spinlock-inits-from-b.patch), [4.19](4.19/0010-usb-gadget-f_hid-move-list-and-spinlock-inits-from-b.patch), [5.10](5.10/0004-usb-gadget-f_hid-move-list-and-spinlock-inits-from-b.patch), [5.15](5.15/0005-usb-gadget-f_hid-move-list-and-spinlock-inits-from-b.patch)（变体也一样）, [6.6](6.6/0003-usb-gadget-f_hid-move-list-and-spinlock-inits-from-b.patch)（6.12 已包含） |
| f_hid：可写的 bInterval configfs 属性 | `hid --interval`：没有它，键盘按内核默认的 1 ms 轮询 | [4.9](4.9/0011-usb-gadget-hid-add-a-writable-bInterval-configfs-att.patch)（变体也一样）, [4.14](4.14/0007-usb-gadget-hid-add-a-writable-bInterval-configfs-att.patch), [5.10](5.10/0003-usb-gadget-hid-add-a-writable-bInterval-configfs-att.patch), [5.15](5.15/0004-usb-gadget-hid-add-a-writable-bInterval-configfs-att.patch) |

USB Arsenal 读取补丁是否存在，从不去试探，因为试探正是会让未打补丁的内核 panic 的原因：
`check` 显示能否创建我们自己的 gadget，`tether` 在存在三星 NCM 附加改动但没有 f_ncm 修复时
选 RNDIS，`hid --interval` 在属性缺失时给出警告。
