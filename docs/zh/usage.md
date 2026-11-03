# Usage

[English](../usage.md) | 简体中文

```
usb-arsenal [--udc NAME] [-v] <command> [<action>] [options] [args]
```

每个命令都会自我说明：`usb-arsenal <command> --help`、`usb-arsenal help
<command>` 以及不带参数的 `usb-arsenal <command>` 都会打印同一页说明和示例。
命令的选项可以放在 action 之前或之后。

| 全局选项 | 含义 |
|---|---|
| `--udc NAME` | 要使用的 USB 设备控制器。仅当 `/sys/class/udc` 列出多个时才需要。`dummy_udc.*`（内建于每个 GKI 内核）只有在它指名时才使用。 |
| `-v`, `--verbose` | 调试输出 |
| `-h`, `--help`, `-V`, `--version` | 帮助、版本 |

只有一个 gadget。每个命令添加或移除它的一部分，所以 storage、网络和 HID 可以同时
运行。添加或移除一部分会让 gadget 短暂重新插入，主机看到它重新出现，其余部分完好。
移除最后一部分会把 USB 端口交还给 Android。每个选项的默认值来自当前生效的
[identity 配置档](identity.md)；命令行给出的选项优先。

**每个功能启动时用的参数会被保存。** 带参数的 `start` 会把它们写入 setup
（`<workspace>/Gadget/setup.conf`），不带参数的 `start` 运行那里保存的内容，`apply`
让正在运行的内容与之一致。Android App 编辑同一个文件，所以任何一侧做的 setup 另一侧
都能用，清除 App 的数据不会丢失任何东西。见 [setup](#setup)。

## check, status, stop

```sh
usb-arsenal check     # what this kernel can do; MUST HAVE lines block every command
usb-arsenal status    # what is live: storage images, HID nodes, the tether link
usb-arsenal stop      # everything off, the port back to its owner
```

`check --json` 把同样的探测结果打印成一个 JSON 对象供程序读取：每个键
（`root`、`configfs`、`udc`、`gadget`、`mass_storage`、`cdrom`、`hid`、`hid_interval`、`ncm`、
`rndis`、`nat`、`nat6`）为 `1`（正常）、`0`（缺失）或 `-1`（可用但有保留），外加 `version`、
`udc_name` 和 `requirements_met`。`hid_interval` 表示 `hid --interval` 是否生效
（[kernel-patches/README.md](../../kernel-patches/README.md) 里的 f_hid 补丁）；每当 `hid` 为 `0` 时它也是 `0`。
退出码与文本报告相同。

`status --json` 把正在运行的 gadget 打印成一个 JSON 对象，每个键始终存在：

| 键 | 值 |
|---|---|
| `version`, `udc`, `udc_state` | 二进制的版本、控制器及其状态（`configured`、`not attached`、...） |
| `bound` | 我们的 gadget 持有控制器时为 `1`，控制器归属其所有者时为 `0` |
| `speed` | 主机协商的速度，即 UDC 的 `current_speed`（`full-speed`、`high-speed`、`super-speed`、...）；未绑定时为空。文本形式会把它加到 `Controller:` 行 |
| `storage` | 每个 LUN 一个对象：`lun`、`file`、`mode`（`cdrom`、`dvd`、`disk`、`rw`）、`access`（`ro`、`rw`）；已弹出的 LUN 为空字符串 |
| `hid` | 每个节点一个对象：`function`（`keyboard`、`mouse`）、`node`（`/dev/hidgN`，出现之前为空）、`interval`（它运行时使用的轮询间隔指数：`0` 表示内核默认值 1 ms；内核没有对应控制项时为 `-1`） |
| `tether` | `null`，或 `function`（`ncm`、`rndis`）、`iface`、`running`（网络侧已启用）、`address`、`ipv6`、`upstream`、`gateway`（网关模式下为 `1`）、`gateway_container`、`gateway_iface`、`gateway_waiting`（容器尚未运行）、`tx_bytes` 和 `rx_bytes`（手机这一端已发送和已接收的字节数，无法读取时为 `-1`） |
| `boot_id` | 本次启动的 id，读取者不会把重启之前的状态当成当前状态 |
| `setup`, `running` | [setup](#setup) 的每个键，分别为设置值和每个功能最后一次成功启动时使用的值；功能关闭时 `running` 为空 |
| `pending` | `storage`、`tether`、`hid`：功能已挂载且其 setup 与启动它时不同（空值按配置档的值比较）时为 `1`，也就是说 `<function> apply` 会做出改变 |

`stop` 在任何时候、任何次数下都是安全的。它也是恢复路径：如果命令中途被杀掉，或者
手机断电，`stop` 会把事情做完。

## setup

```sh
usb-arsenal setup show [--json]            # every key: as set up, and as the last start used
usb-arsenal setup set KEY=VALUE ...        # change the setup; a bad value changes nothing
usb-arsenal setup reset [storage|tether|hid]   # back to the defaults
usb-arsenal setup revert [storage|tether|hid]  # back to what runs
```

| 键 | 值 |
|---|---|
| `udc` | 当 `/sys/class/udc` 列出多个时每个命令使用的控制器；代替 `--udc` |
| `storage_lun0` 到 `storage_lun7` | 每个 drive 的 `MODE:PATH`，模式为 `auto`、`cdrom`、`dvd`、`disk` 或 `rw`；为空：没有 drive |
| `tether_function` | `ncm` 或 `rndis`；为空：使用配置档的值 |
| `tether_gateway` | Droidspaces 容器的名称；为空：不使用网关 |
| `tether_gateway_iface` | 容器内该链路的名称；为空：`usb0` |
| `tether_net` | `--no-net` 时为 `0`，否则为 `1`；有网关时忽略 |
| `tether_address`, `tether_dns`, `tether_upstream` | 同对应选项；为空：使用配置档的值 |
| `hid_interval` | 1 到 255；`0`：内核默认值 |

`setup.conf` 是 `key=value` 行加 `#` 注释，与身份配置档的格式相同。后端在它旁边维护
`running.conf`，内容与每个已使用功能最后一次成功启动时相同（功能停止时以及重启之后
清空）。带参数的 start 先把它们合并进 setup；`storage eject` 会在两个文件里都清空该
LUN。`status --json` 会报告这两个文件和 `pending`。

## storage

```sh
usb-arsenal storage start [--mode MODE] [--lun N] IMAGE
usb-arsenal storage start [--lun N] [--mode MODE] IMAGE [--lun N] [--mode MODE] IMAGE ...
usb-arsenal storage start                  # every drive the setup holds, in one replug
usb-arsenal storage apply                  # the drives to the setup, LUN by LUN
usb-arsenal storage eject [--lun N]
usb-arsenal storage stop
```

`apply` 依据内核的实际状态决定：setup 中已移除的 LUN 就地弹出，setup 自启动以来发生变化
（或主机已弹出其介质）的 LUN 会被加载，已经在以其模式提供 setup 中文件的 LUN 保持不动，
而 setup 为空时 storage 停止。同类型多个在线 LUN 上的镜像会一次性就地替换。

| 模式 | 主机看到 |
|---|---|
| `auto`（默认） | 如果镜像有 MBR 或 GPT（混合 Linux ISO、USB 镜像）则是磁盘；否则是 CD-ROM，镜像超过 CD-ROM 上限时为 `dvd` |
| `cdrom` | 只读 CD-ROM：纯 ISO9660/UDF 镜像这样启动 |
| `dvd` | 2048 字节扇区的只读磁盘：任意大小的纯 ISO，例如 Windows ISO |
| `disk` | 只读磁盘 |
| `rw` | 可写磁盘：主机可以写入的 U 盘 |

- **`--lun N`（0-7）** 再添加一个 drive。每个 LUN 在主机上都是一个独立的 drive。
- **一次 `start` 挂载多个 drive：** 最多八个 IMAGE，各自跟在自己的 `--lun` 和
  `--mode` 之后，主机在一次重新插入中就能看到全部，而不是每个 drive 插一次。没有
  `--lun` 的 IMAGE 使用前一个之后的 LUN（第一个使用 0）：

  ```sh
  usb-arsenal storage start Win11.iso --mode rw pendrive.img   # LUN 0 and LUN 1
  ```
- **一个文件，两个 drive：** 仅当两者都是只读时才允许。主机各自缓存每个 drive，
  所以可写的那一个会破坏另一个显示的内容；`start` 会拒绝。
- **对在线 LUN 执行 `start`** 会就地替换镜像，就像换一张光盘。
  - 在磁盘、`dvd` 和 CD-ROM 之间切换会重新插入 gadget，因为主机只在枚举或看到
    新光盘时读取设备类型和扇区大小。
  - 如果主机锁定了该 drive，`start` 也会重新插入，从而释放锁定。
- **`eject`** 就地清空一个 LUN；主机会看到介质消失，而 drive 仍然存在。
  在 6.6 之前的内核上，主机的锁定会导致失败，所以 `eject` 像 `start` 一样重新插入来释放它。
- **CD-ROM 模式在约 2.36 GB 处停止。** 内核会截断更大的镜像，主机读到的是被截断的光盘；
  USB Arsenal 会警告（除非内核带有 `kernel-patches/` 中的 DVD 补丁）。`dvd` 模式没有
  限制，auto 模式对大的纯 ISO 就会选择它：
  - 镜像位于一个 2048 字节块的只读 loop 设备之后。loop 在弹出、替换或停止时自行分离。
  - UEFI 固件通过 El Torito 启动它，Windows 能找到它的 UDF 卷，因为扇区大小正是 UDF
    期望的。已用 4.9 GB 的 WinPE 11 ISO 验证。
  - 它需要 4.14 或更新的内核，在 Android 上还需要一条允许内核读取 loop 设备的 SELinux
    规则。模块在启动时加载该规则；使用裸二进制时，运行
    `ksud sepolicy patch "allow kernel loop_device blk_file { open read getattr ioctl }"`
    （Magisk：`magiskpolicy --live "..."`）。没有它，`storage` 会拒绝，而不是提供一个
    主机无法读取的磁盘。
  - 这取决于固件。一台 2021 年的 MSI 笔记本的 UEFI 能启动它；一台 2012 年的 Samsung
    NP300E 在 UEFI 或 legacy 模式下甚至不列出它。BIOS 从 MBR 启动 USB 磁盘，而纯 ISO
    没有 MBR，较旧的固件也可能不接受磁盘上的 2048 字节扇区。对于较旧的 PC，在内核带有
    `kernel-patches/4.14/0004` 到 `0006`（或 `6.12/0001`）时使用 `--mode cdrom`，这两台
    笔记本都能启动。
- **在 Android 上，内核可能无法读取的镜像会被重新打标签。** 容器内或 `/data/local/tmp`
  中的文件带有内核 mass-storage 线程无法读取的 SELinux 标签，所以 USB Arsenal 会把它们
  重新打成 `/sdcard` 文件使用的标签，并给出提示。

名称以 `-` 开头的镜像要放在 `--` 之后：`storage start -- -old.iso`。setup 中的
`storage_lunN` 路径是绝对路径，永远不会被当作选项读取。

如果内核拒绝新介质，替换会把旧的放回去。对锁定 drive 的主机（6.6 之前，6.6 有
`forced_eject`），所有 LUN 一起通过一次重新插入来释放，而一个命令中的所有弹出共享
5 s 等待，所以 `start`、`apply` 和 `eject` 都能远在客户端 30 s 超时之内完成。

制作一个空白的 4 GB U 盘：`truncate -s 4G pendrive.img`，然后
`usb-arsenal storage start --mode rw pendrive.img`，再从主机上格式化它。

## hid

```sh
usb-arsenal hid start [--perm MODE] [--group GROUP] [--interval N]
usb-arsenal hid apply                      # restart with the setup's interval if it changed
usb-arsenal hid stop
usb-arsenal hid write NODE SIZE            # stdin's reports onto NODE, never blocking
```

`--interval` 会保存为 setup 的 `hid_interval`；不带参数的 `start` 使用它。在 HID 正在运行时，
请求另一个间隔的 `start` 会失败，因为间隔只随重启改变：`hid apply` 在一次重新插入中
完成这件事，并保持其他功能仍然绑定。

`start` 会添加一个 boot-protocol 键盘和一个带滚轮及水平滚轮的 5 键鼠标，然后打印
哪个 `/dev/hidgN` 对应哪个。编号取决于内核还有什么；Android 自己的 HID 功能可能已经
占用了 `hidg0`。

| 设备 | 报告 |
|---|---|
| keyboard | 8 字节：修饰键、0，然后最多六个 HID 键码 |
| mouse | 5 字节：按键、X、Y、滚轮、平移（有符号） |

```sh
printf '\0\0\x04\0\0\0\0\0' > /dev/hidg0   # press 'a'
printf '\0\0\0\0\0\0\0\0' > /dev/hidg0     # release
printf '\0\x0a\0\0\0' > /dev/hidg1         # mouse 10 px right
```

节点默认是 `0660 root`，或配置档的 `hid_perm` 和 `hid_group`。`--perm`
和 `--group`（名称，在 Android 上也可以是数字）在 HID 运行期间放宽该权限：之后不带参数的
`start` 或 `apply` 会保留它们，`hid stop` 和 `stop` 会丢弃它们，每次 gadget 重新插入时
权限都会重新应用。任何能写入节点的人都能在主机上打字。
不使用 `--interval` 时不会向 `interval` 写入任何内容，崩溃遗留的功能会被重新创建，
因此适用内核自己的默认值。写入 `0` 不行：mainline 的 `interval`
（6.x）会把它当成真实的 bInterval 0，主机会丢弃它。

`hid write NODE SIZE` 是程序在节点上保持打开使用的写入器：它从 stdin 按 `SIZE` 字节的报告读取
（1 到 64，短读会拼接），并把每个报告写入 `NODE`，`NODE` 必须是字符设备，以非阻塞方式打开。
f_hid 每次写入发送一个报告。当主机没有读取时（休眠，或比报告到来的速度慢），只保留最新的
未发送报告，一旦节点再次可写就发出去：主机最终总是处于最新状态，不会收到重放的积压。
被更新的报告在主机取走之前替换掉的报告永远不会发送，所以一次按键需要自己的轮询间隔
才能有对应的释放。stdin 结束且最后一个报告发出后它以 `0` 退出（最多等待一秒），
当主机没有取走、节点无法打开，或写入失败且错误不是 `EAGAIN`（主机离开后是 `ESHUTDOWN`）时，
它以 `1` 退出并给出消息。它不取锁。

每次重新插入和每次 `stop` 都会杀掉任何保持我们节点打开的东西，包括 `hid write`，
因为跨越 unbind 的打开节点会让许多内核 panic（[kernel-patches/README.md](../../kernel-patches/README.md)
中的 f_hid 补丁）。保持节点打开的程序需要在 gadget 恢复后重新打开它。

## tether

```sh
usb-arsenal tether start [--rndis | --ncm] [--address CIDR] [--dns IP[,IP]] [--upstream IFACE]
usb-arsenal tether start --gateway CONTAINER [--gateway-iface IFACE]
usb-arsenal tether start --no-net
usb-arsenal tether start                   # with the setup's options
usb-arsenal tether apply                   # restart with the setup if it changed, in one replug
usb-arsenal tether stop
```

除 `--gateway-pid`、`--gateway-netns` 和 `--foreground` 之外的每个选项都会保存到 setup 中；
不带参数的 `start` 运行它。

主机获得一个网络适配器、一个通过 DHCP 分配的地址，以及一条经手机连接出去的路由。
默认手机是 `192.168.42.129/24`，主机在 `.130`，在内核支持 IPv6 NAT 时还有来自
`fd75:7361::/64` 的 SLAAC IPv6。`--gateway` 改为把链路交给一个 OpenWrt 容器。
[tethering.md](tethering.md) 涵盖两种模式和主机侧。

## identity

```sh
usb-arsenal identity generate ~/my.conf   # every setting, with its default
usb-arsenal identity set ~/my.conf        # make it active
usb-arsenal identity show [--json]
usb-arsenal identity show [--json] FILE...  # what each file gives, read as `set` reads it
usb-arsenal identity unset
```

`identity show` 会指出文件已消失的配置档：`Profile: /path/x.conf (missing)`，然后是
生效的内置值。`identity show --json` 打印一个对象：

```json
{"profile":"/sdcard/x.conf","missing":"","keys":{"id_vendor":"0x1d6b",...}}
```

`profile` 是当前生效的配置档（使用内置默认值时为空），`missing` 是悬空链接指向的路径，
`keys` 保存每个键生效的值，派生值为 `""`。带 FILE 时打印一个数组，每个 FILE 一个对象：
`{"path":"<realpath>","errors":N,"keys":{...}}`。每个 FILE 都经过 `identity set` 使用的
加载器，所以 `0104` 按 C 的读法是八进制，` key = v` 这种空格写法可用，缺失或错误的键
会回退到默认值；`errors` 统计错误行数（`-1`：文件无法读取，退出码为 `1`）。值的打印
方式统一：ID 为 `0x%04x`（`0x0104`），`max_power` 和 `hid_group` 为十进制，`hid_perm`
为带前导 `0` 的八进制（`0660`），字符串原样输出。

见 [identity.md](identity.md)。

## 退出码

| 代码 | 含义 |
|---|---|
| 0 | 完成 |
| 1 | 失败：缺少内核功能、写入被拒绝、端口忙 |
| 2 | 用法错误：未知的命令、action 或选项，或值不合法 |
