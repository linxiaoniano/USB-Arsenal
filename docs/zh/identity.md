# 身份配置档

[English](../identity.md) | 简体中文

配置档是一个可编辑的 `key=value` 文件，存放用户可能想改的所有内容：主机看到的 ID、字符串和
序列号，以及每条命令的默认值。想要多少份都行，放在哪里都行；同一时间只有一份生效。代码：
`src/config.c`。格式和加载逻辑沿用 Droidspaces 的 `config.c`：
- `#` 注释，键和值会去掉首尾空白；
- 值非法或键未知时给出警告，并保留默认值；
- 写入是原子的（mkstemp、fsync、rename）。

```sh
usb-arsenal identity generate ~/pixel.conf    # every key, with its default
usb-arsenal identity set ~/pixel.conf         # make it the active profile
usb-arsenal identity show                     # values in force and their source
usb-arsenal identity unset                    # back to the built-in defaults
```

命令行选项总是优先于配置档：`tether start --address ...` 会忽略 `tether_address`。

## 键

| 键 | 默认值 | 说明 |
|---|---|---|
| `id_vendor`, `id_product` | `0x1d6b`, `0x0104` | Linux Foundation 多功能复合设备 |
| `bcd_device`, `bcd_usb` | `0x0100`, `0x0200` | |
| `manufacturer`, `product`, `configuration` | `USB Arsenal` | 可打印 ASCII，最多 126 个字符 |
| `serial` | 空 | 为空时：`ro.serialno`，或 `/etc/machine-id` |
| `max_power` | `250` | mA，0 到 500 |
| `storage_inquiry` | 空 | SCSI vendor(8) product(16) revision(4)，最多 28 个字符；为空时：内核的 "Linux File-Stor Gadget" |
| `tether_dev_mac`, `tether_host_mac` | 空 | 为空时：由设备种子推导，保持稳定 |
| `storage_mode` | `auto` | `auto`, `cdrom`, `dvd`, `disk`, `rw` |
| `tether_function` | `ncm` | `ncm`, `rndis` |
| `tether_address` | `192.168.42.129/24` | 本端；主机拿到下一个地址 |
| `tether_dns` | 空 | 为空时：`1.1.1.1,8.8.8.8` |
| `tether_upstream` | 空 | 为空时：自动检测 |

这些默认值用在[参数](usage.md#setup)留空的地方：参数才是这台手机实际运行的，配置档是它回退到
的值。
| `hid_perm`, `hid_group` | `0660`, `0` | 八进制；数字 gid |

## 当前配置档的位置

`<workspace>/Gadget/identity.conf`（Android 上是 `/data/local/USB-Arsenal/...`，
Linux 上是 `/var/lib/USB-Arsenal/...`）是下面两种之一：
- **指向你的文件的符号链接**，用它的规范路径。修改会在下一次 bind 时生效，不需要再执行 `set`。
  从容器里用时，把文件放在 Android 也能看到的路径上（[containers.md](containers.md)）。
- **不存在**。静默使用内置默认值。

如果符号链接的目标消失，每条命令都会输出
`Profile <path> is gone; using the built-in defaults`，然后继续使用默认值。

gadget 处于活动状态时执行 `identity set` 会重新插拔一次，主机立刻就能看到新的 ID。`generate`
只写你自己的文件，不需要 root。

## 需要知道的

- `lsusb` 用自带的 usb.ids 数据库给设备命名（`1d6b:0104` 显示为 "Linux Foundation
  Multifunction Composite Gadget"）。设备真正发送的字符串在 `lsusb -v` 的 `iManufacturer`、
  `iProduct` 和 `iSerial` 下面，Windows 和 macOS 显示的是这些。
- 大容量存储、HID 和 NCM 按类绑定，RNDIS 按其 Microsoft OS 描述符绑定，所以任何 VID/PID 在
  任何主机上都能用。借用真实的厂商 ID（Google `18d1`、Samsung `04e8`）也行，但它属于那个
  厂商，是否这样做由你决定。如果主机上有该厂商针对这个确切 PID 的驱动，它可能会尝试绑定。
