# Tethering

[English](../tethering.md) | 简体中文

`usb-arsenal tether start` 通过同一根线缆为主机提供一个网络适配器，与其他功能共用这根线。
有两种方式提供。

## NAT 模式（默认）

```sh
usb-arsenal tether start
usb-arsenal tether start --address 10.42.0.1/24 --dns 9.9.9.9 --upstream wlan0
```

- **手机侧是 `192.168.42.129/24`**（`--address`）。内置 DHCP 服务器把下一个地址分配给
  主机。
- **主机的流量经 NAT 从手机当前使用的连接出去。** 在 Android 上，是 netd 标记为默认的
  网络；在 Linux 上，是主路由表的默认路由。`--upstream` 指定单个接口。
- **手机在 Wi-Fi 和移动数据之间切换时**，Android 一换网，路由就跟着走。如果 Android 的
  网络服务重启并清空了防火墙，规则会在两秒内恢复。
- **分配给主机的 DNS** 是 `1.1.1.1` 和 `8.8.8.8`，除非 `--dns` 另有指定。
- **IPv6 也支持，前提是内核能对它做 NAT**（`CONFIG_IP6_NF_NAT`；`check` 会显示
  "IPv6 NAT" 一行）。主机通过 SLAAC 获得 `fd75:7361::/64` 中的地址和一条默认路由，
  经 NAT 从手机的 IPv6 连接出去，与 Droidspaces 为容器所做的相同。公网站点主机仍优先
  用 IPv4，因为该前缀是 ULA。在纯 IPv6 运营商网络上，承载流量的是 IPv6。没有 nat 表时，
  主机只拿到 IPv4。
- **能用 NCM 的地方就用 NCM：** Windows 11、macOS 和 Linux 都有它的类驱动。RNDIS
  （`--rndis`）用于 Windows 7 和 8，`--ncm` 可以覆盖选择 RNDIS 的配置档。两者都使用
  固定 MAC 地址，所以 Windows 不会每次都新增一个适配器。
- **在没有 f_ncm 修复的三星 4.x 内核上，** USB Arsenal 自动使用 RNDIS；见
  [kernel-patches/README.md](../../kernel-patches/README.md)。

网络侧作为一个小型后台进程运行。`tether stop` 或 `stop` 结束它并移除它的防火墙规则。
共享网络期间添加存储或 HID 不会断开链路：在每次重新插拔都会重建接口的内核上，它会自行
重新配置好。

`--no-net` 只添加网络功能，不配置地址、DHCP 或 NAT，供你自己设置。它会忘记已保存的
网关；而存在网关时，`tether_net=0` 会被忽略。

## 网关模式：让 OpenWrt 当路由器

```sh
usb-arsenal tether start --gateway OpenWrt
usb-arsenal tether start --gateway OpenWrt --gateway-iface vaplan0
```

USB 链路接入一个 Droidspaces 容器（通常是 OpenWrt 容器），由它为主机做 DHCP、DNS 和
防火墙。USB Arsenal 只负责插上线缆：
- **`usb0` 留在 Android 侧**，位于一个没有地址的网桥 `ua-usb` 上。
- **一条虚拟线缆从该网桥接入容器。** 它的另一端在容器内显示为 `usb0`
  （`--gateway-iface`）。
- **容器重启时，线缆会自动插到新容器上。** 主机保留租约，容器启动完成后即恢复。
- **gadget 重新插入时，`usb0` 回到网桥上。**

`--gateway` 按名称查找 Droidspaces 容器。其他运行时用
`--gateway-pid PID` 或 `--gateway-netns /path/to/netns`。

### OpenWrt 侧（一次）

```sh
uci set network.usb=interface
uci set network.usb.device='usb0'
uci set network.usb.proto='static'
uci set network.usb.ipaddr='192.168.43.1'
uci set network.usb.netmask='255.255.255.0'
uci set network.usb.ip6assign='64'
uci set dhcp.usb=dhcp
uci set dhcp.usb.interface='usb'
uci set dhcp.usb.start='100'
uci set dhcp.usb.limit='150'
uci set dhcp.usb.leasetime='12h'
uci set dhcp.usb.ra='server'          # IPv6: odhcpd announces the /64 from ip6assign
uci set dhcp.usb.dhcpv6='server'
uci set dhcp.usb.ra_default='1'       # default router even with only a ULA prefix
uci add_list firewall.@zone[0].network='usb'   # the lan zone
uci commit
/etc/init.d/network reload; /etc/init.d/dnsmasq reload; /etc/init.d/firewall reload
/etc/init.d/odhcpd reload                       # absent on images without IPv6
```

这些是 VirtualAP 托管模式写入的设置。Android 侧在 `usb0`、网桥和 veth 上关闭了 IPv6，
所以它不会从 OpenWrt 的通告中获取地址或路由。

Android 内核通常缺少 nf_tables，因此 fw4 可能加载不了。如果主机拿到了地址但上不了网，
手动添加 NAT：
`iptables-legacy -t nat -A POSTROUTING -o eth0 -j MASQUERADE`，IPv6 用
`ip6tables-legacy -t nat -A POSTROUTING -o eth0 -j MASQUERADE`。

**已经设置好 VirtualAP 的托管模式？** 它的 `vaplan` 接口带 DHCP 和防火墙区域，所以
`--gateway-iface vaplan0` 无需改动 OpenWrt 即可使用，前提是 VirtualAP 自身没有在运行。

## 主机侧

- **适配器的名称取决于 gadget 里还有什么。** 在 Linux 上，单独时是
  `enp0s20f0u9`，与存储和 HID 并存时是 `enp0s20f0u9i3`。NetworkManager
  有时会让刚改名的适配器保持断开；`nmcli device connect <name>` 可以解决，之后名称
  保持不变。
- **主机可能开始把所有流量都经手机路由，** 因为有线链路常常优先于 Wi-Fi。如果这台电脑
  应保留自己的连接（比如你正在用的工作站），给该链路一个从不接管默认路由的配置档：
  ```sh
  nmcli connection add type ethernet con-name "USB Arsenal (no default route)" \\
    match.interface-name "enp0s20f0u*" ipv4.never-default yes ipv6.never-default yes
  ```
