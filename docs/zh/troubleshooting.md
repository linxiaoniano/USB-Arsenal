# 故障排查

[English](../troubleshooting.md) | 简体中文

测试中遇到的问题，以及由已知内核限制导致的问题。

**一执行命令 ADB 就断开。**
预期行为。从 Android 手里接管 USB 端口会替换掉它的 ADB 和 MTP 设备。
`usb-arsenal stop` 会把它们带回来。仍想通过 ADB 执行命令，就把命令包进
`nohup sh -c "..." &`，让它们在 ADB 断开之后跑完，或者用手机上的 root 终端，或 SSH 进容器。

**主机能看到磁盘，但每次读取都失败（"unrecovered read error"）。**
内核读不到镜像的 SELinux 标签。容器文件系统里、`/tmp` 或 `/data/local/tmp` 里的文件会遇到
这种情况。当前版本会自动给镜像重新打标签，并打印 "Relabelled ..."。如果重新打标签本身失败，
就把镜像移到 `/sdcard` 或 `/data`。

**Windows ISO 能启动，但安装进行到一半失败。**
CD-ROM 模式在大约 2.36 GB 处停止，内核会静默切掉剩下的部分；USB Arsenal 会就此给出警告。
改用 `--mode dvd`（auto 模式对大的普通 ISO 会自动选它）；见
[usage.md](usage.md#storage)。

**WinPE 能启动，但它的程序或 `sources` 文件夹不见了。**
这个 ISO 是用 `--mode disk` 提供的。它的 512 字节扇区把 UDF 锚点放在 Windows 不看的位置，
所以 Windows 从不挂载该卷。改用 `--mode dvd`。

**`dvd` 驱动器不出现在 PC 的启动菜单里。**
较旧的固件无法从使用 2048 字节扇区且没有 MBR 的 USB 磁盘启动（一台 2012 年的笔记本在 UEFI
和 legacy 模式下都忽略了它）。在打了 DVD 补丁的内核上用 `--mode cdrom`
（见 [kernel-patches/README.zh.md](../../kernel-patches/README.zh.md)），或者换台新一些的 PC。

**用过 USB Arsenal 之后，Android 不显示 USB 通知，USB 共享网络变灰。**
2026-10-19（傍晚）之前的版本会在 `stop` 时移除自己的 gadget，Android 的 configfs 随后会误删
`android0`（见 developer/architecture.md）。ADB 仍然可用。把手机重启一次；当前版本会保留自己
的 gadget，处于未绑定且为空的状态，从不触发这个问题。`ls /sys/class/android_usb` 应该能看到
`android0`。

**"SELinux does not let the kernel read /dev/block/…"。**
直接提供的块设备（一个分区、一个 `dm-*` 设备）由内核按设备节点的标签读取，多数 Android 策略
不允许内核这么做，而且没有哪个标签在所有手机上都能用，所以 USB Arsenal 没法像对镜像文件那样
给它重新打标签。它会在主机看到一块坏盘之前停下。仍要提供的话，手动把 SELinux 设为 permissive
（`setenforce 0`）再启动一次。是否提供分区（无论是否挂载）由你决定：手机以读写方式挂载的文件
系统会在主机下面发生变化（它可能拒绝挂载，用 `--mode rw` 时更糟）。

**`dvd` 模式拒绝运行："SELinux does not let the kernel read loop devices"。**
安装模块（它会在启动时加载规则），或者加载消息里打印的规则。

**换镜像导致设备在主机上重新出现。**
在磁盘和 CD-ROM 之间切换时会出现，或者主机锁住了驱动器时也会（Linux 对它打开着的驱动器会
这样）。这是设计上的重新插拔；其他功能会随之恢复。

**共享网络已启用，但主机没有地址。**
主机的网络管理器可能没有接管这块网卡，尤其是加入存储或 HID 之后它的名字变了的时候。Linux 上
执行：`nmcli device connect <adapter>`。见 [tethering.md](tethering.md#主机侧)。

**共享网络时电脑失去了自己的网络。**
主机把所有流量都走了手机，因为有线链路压过了 Wi-Fi。手机一侧随后发生变化时（OpenWrt 重启、
执行 `stop`），电脑就卡住。给这条链路设一个 "never default route" 配置档；见
[tethering.md](tethering.md#主机侧)。

**`check` 在三星手机上把 NCM 标为不安全。**
内核带有三星的 f_ncm 缺陷：第二个 NCM 实例会让它 panic。在那里 USB Arsenal 改用 RNDIS 共享
网络。`kernel-patches/4.14/` 里的补丁 `0003` 修好了内核，NCM 会自己回来。见
[kernel-patches/README.zh.md](../../kernel-patches/README.zh.md)。

**"This is stock Samsung configfs, which cannot create a second gadget"。**
原厂三星 4.x 内核：它的 configfs 无法在 Android 的 gadget 旁边再建一个（`mkdir` 以 ENOMEM
失败，什么都不会改变）。USB Arsenal 无法在它上面运行；`kernel-patches/4.14/` 里的补丁
`0001` 和 `0002` 修好了内核。见 [kernel-patches/README.zh.md](../../kernel-patches/README.zh.md)。

**"More than one USB device controller"。**
用 `--udc NAME` 选一个；`check` 会列出来。每个 GKI 内核都内建的 `dummy_udc.*` 从不计入，
所以不会导致这个。

**"No peripheral-mode controller in /sys/class/udc now"，或某个控制器 "not in /sys/class/udc now"。**
双角色端口在作为 USB 主机期间会离开 `/sys/class/udc`（dwc3 在切换时移除它的 gadget 侧），
例如接过 OTG 设备之后。把它拔掉；端口重新作为设备时控制器就回来了，最迟在插上电脑时。

**命令被杀掉，或者手机中途断电。**
执行 `usb-arsenal stop`。它会读取工作目录里的交还记录，把端口恢复完。重启之前留下的记录会被
忽略，因为重启已经恢复过了。

**配置档的修改没有在主机上体现。**
`identity set` 只在 gadget 处于活动状态时重新插拔；否则修改在下次启动时生效。直接编辑当前配置
档同样从下次启动算起，因为当前配置档是指向你文件的链接。`identity show` 会标出已经消失的链接
文件（`(missing)`）：这时主机会拿到内置默认值。
