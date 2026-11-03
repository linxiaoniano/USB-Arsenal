# 容器

[English](../containers.md) | 简体中文

USB Arsenal 没有守护进程。每个 `usb-arsenal` 副本都以 root 身份、从它运行的位置直接驱动内核。
特权容器（例如能看到 configfs 和 sysfs 的 Droidspaces）可以直接运行这个二进制。在 a166p 上，
在 OpenWrt 容器内，`hid start` 提供了 `/dev/hidg1` 和 `/dev/hidg2`。随后从容器里执行 `stop`，
把端口交还给 Android 自带的共享网络，`android0` 保持完好。

## 在容器中运行

1. 把二进制拷进去。在 Android 侧它的位置是 `/data/local/USB-Arsenal/bin/usb-arsenal`。
2. 用 bind mount 把 Android 的工作目录挂到容器的工作目录上，然后重启容器：
   ```
   -B /data/local/USB-Arsenal:/var/lib/USB-Arsenal
   ```
   也可以在 `container.config` 里用 `bind_mounts=`。这样两侧共用同一份交还记录、同一把锁和
   同一份参数：来自 App 的命令和来自容器的命令会互相排队，哪一侧执行 `stop` 都会把端口交还。
3. 照常以 root 身份运行。

不做 bind mount 时，容器会在自己的 `/var/lib/USB-Arsenal` 里保存自己的交还记录，并在接管端口时
说明这一点。这时不要同时从 Android 侧（包括 App）运行 USB Arsenal。先停止 gadget，再停止容器。

## 共享网络

内核在 Android 自己的网络命名空间里创建共享网络接口（`usb0`），而不是在容器的命名空间里。
处在自己命名空间中的容器看不到它，因此从那里执行 `tether start` 会失败。要在容器里共享网络，
就让容器以 Droidspaces 的 **host networking mode** 运行。这样容器共用 Android 的网络栈，代价是
失去那层隔离。

网关模式（`tether start --gateway CONTAINER`，即 App 里的 Gateway 开关）则是反过来的：它在
Android 侧运行，把 `usb0` 交给某个容器。容器里不需要任何配置。

## 路径

镜像和配置档按名字打开，在运行该命令的那个副本的挂载命名空间里解析。容器私有的路径
（`/root/disk.img`）从容器里用没问题，但一旦存进参数，之后从 App 启动就会失败，并会报出该文件
名。`identity set` 也一样，它建立的链接指向那个文件。请把镜像和配置档放在两侧都能看到的路径
上，例如把 `/sdcard/USB-Arsenal` bind mount 进去。

## 工作目录

USB Arsenal 保存的所有东西都在一个目录里：

| | Android | Linux，以及任何容器内 |
|---|---|---|
| 工作目录 | `/data/local/USB-Arsenal` | `/var/lib/USB-Arsenal` |

| 路径 | 内容 |
|---|---|
| `Logs/` | `usb-arsenal.log`，每次运行和共享网络进程各写一行带时间戳的记录 |
| `Pids/`, `Net/` | 共享网络进程的 pid 和状态 |
| `Gadget/` | 交还记录、锁、参数、当前配置档、HID 权限 |
| `bin/` | 二进制 |
