如果你使用 systemctl start wg-quick@wg0启动wireguard服务

systemd程序会调用 wg-quick ， 是wireguard用户态工具里边的一个shell脚本，会调用如下命令:
```shell
[#] ip link add wg0 type wireguard    #创建一个类型为wireguard的虚拟网卡
[#] wg setconf wg0 /dev/fd/63         #使用用户态程序wg读取配置，刷到wireguard内核模块去
[#] ip -4 address add 192.168.220.11/24 dev wg0  #添加VPN的本地内网IP
[#] ip link set mtu 1420 up dev wg0    #设置mtu，避免添加wireguard头部后，数据包过大被分片。并且把wg0网卡启动起来
```
其中 ip link add wg0 type wireguard 和 ip link set mtu 1420 up dev wg0 就会涉及到 device.c 里边的代码处理

## wg_device_init
内核模块初始化的时候，main.c 会调用此函数。
```c
wg_device_init 函数里调用的
ret = rtnl_link_register(&link_ops);
这句代码，会在内核注册一个名为wireguard(link_ops里指定)的设备类型，然后用户态就可以创建类型为wireguard的虚拟网卡

```

## device里各函数
```c
static struct rtnl_link_ops link_ops __read_mostly = {
	.kind			= KBUILD_MODNAME,   //这个名字就是 wireguard
	.priv_size		= sizeof(struct wg_device),
	.setup			= wg_setup,
	.newlink		= wg_newlink,
};

static const struct net_device_ops netdev_ops = {
	.ndo_open		= wg_open, //这个会初始化 wg_socket_init 里边设置UDP socket回调函数 wg_receive
	.ndo_stop		= wg_stop,
	.ndo_start_xmit		= wg_xmit, //这个在虚拟网卡收到数据包后会调用
	.ndo_get_stats64	= ip_tunnel_get_stats64
};

```
rtnl_link_ops 和 net_device_ops 两结构体涉及5个函数： wg_setup、wg_newlink、wg_open、wg_stop、wg_xmit

1. wg_setup: 创建新的WireGuard设备时被调用。当你执行ip link add dev wg0 type wireguard这个命令时，内核会调用wg_setup函数来初始化新的WireGuard设备。这个函数只会在设备创建时被调用一次。
2. wg_newlink: 创建新的WireGuard设备时被调用，它在wg_setup函数之后被调用。wg_newlink函数用于处理设备创建过程中的一些额外工作，例如设置设备的私有数据或者注册设备的事件处理函数。
3. wg_open: 设备被启用时被调用。当你执行ip link set dev wg0 up这个命令时，内核会调用wg_open函数来启动WireGuard设备。这个函数通常用于执行设备启动时需要的一些初始化工作。
4. wg_stop: 设备被停用时被调用。当你执行ip link set dev wg0 down这个命令时，内核会调用wg_stop函数来停止WireGuard设备。这个函数通常用于执行设备停止时需要的一些清理工作。
5. wg_xmit: 数据包发给wg0时被调用。当内核需要将一个数据包发送到WireGuard虚拟网卡设备时，它会调用wg_xmit函数来处理这个数据包。这个函数通常用于执行数据包的加密和发送。

wg_setup -> wg_newlink -> wg_open -> wg_xmit -> wg_stop

wg_xmit每次有数据包发给wg0网卡时都会调用它







