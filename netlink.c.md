用来与用户态程序wg做交互的代码文件
* 接收来自wg工具(读取配置文件)的配置内容，写入内核模块缓存
* 接收来自wg工具(命令行)的peer数据获取请求，读取内核模块里peer统计数据后返回给wg展示

## netlink创建
首先main.c模块初始化时调用
```c
int __init wg_genetlink_init(void)
{     //在内核中注册neilink，genl_family里设置了genl_ops
      //这个netlink服务的名字就为wireguard，用户态往family为wireguard的netlink发数据就可以了
      //而.ops = genl_ops,  配置了各种回调函数
      //然后用户态创建netlink socket就可以发消息过来
	return genl_register_family(&genl_family);
}
```
其中genl_family为特定结构体
用户态wg工具创建名为wireguard的netlink socket，往里边发送数据，就能被wireguard内核模块监听的netlink收到
```c
static struct genl_family genl_family
#ifndef COMPAT_CANNOT_USE_GENL_NOPS
__ro_after_init = {
	.ops = genl_ops,
	.n_ops = ARRAY_SIZE(genl_ops),
#else
= {
#endif
	.name = WG_GENL_NAME,   //值为 wireguard
	.version = WG_GENL_VERSION,
	.maxattr = WGDEVICE_A_MAX,
	.module = THIS_MODULE,
#ifndef COMPAT_CANNOT_INDIVIDUAL_NETLINK_OPS_POLICY
	.policy = device_policy,
#endif
	.netnsok = true
};
```
而.ops = genl_ops,  配置了各种回调函数
```
struct genl_ops genl_ops[] = {
	{
		.cmd = WG_CMD_GET_DEVICE,
#ifndef COMPAT_CANNOT_USE_NETLINK_START
		.start = wg_get_device_start,
#endif
		.dumpit = wg_get_device_dump,  //负责处理 打印peer信息请求
		.done = wg_get_device_done,
#ifdef COMPAT_CANNOT_INDIVIDUAL_NETLINK_OPS_POLICY
		.policy = device_policy,
#endif
		.flags = GENL_UNS_ADMIN_PERM
	}, {
		.cmd = WG_CMD_SET_DEVICE,
		.doit = wg_set_device,    //负责处理 设置wireguard配置请求
#ifdef COMPAT_CANNOT_INDIVIDUAL_NETLINK_OPS_POLICY
		.policy = device_policy,
#endif
		.flags = GENL_UNS_ADMIN_PERM
	}
};

```

## wg_set_device
会收到wg工具发来的请求，请求内容格式为多层attr协议
struct nlattr *attr
具体的属性名称和类型，需配置nla_policy
```c
static const struct nla_policy peer_policy[WGPEER_A_MAX + 1] = {
	[WGPEER_A_PUBLIC_KEY]				= NLA_POLICY_EXACT_LEN(NOISE_PUBLIC_KEY_LEN),
	[WGPEER_A_PRESHARED_KEY]			= NLA_POLICY_EXACT_LEN(NOISE_SYMMETRIC_KEY_LEN),
	[WGPEER_A_FLAGS]				= { .type = NLA_U32 },
	[WGPEER_A_ENDPOINT]				= NLA_POLICY_MIN_LEN(sizeof(struct sockaddr)),
	[WGPEER_A_PERSISTENT_KEEPALIVE_INTERVAL]	= { .type = NLA_U16 },
	[WGPEER_A_LAST_HANDSHAKE_TIME]			= NLA_POLICY_EXACT_LEN(sizeof(struct __kernel_timespec)),
	[WGPEER_A_RX_BYTES]				= { .type = NLA_U64 },
	[WGPEER_A_TX_BYTES]				= { .type = NLA_U64 },
	[WGPEER_A_ALLOWEDIPS]				= { .type = NLA_NESTED },   //allowedips有多个，所以设置成NLA_NESTED，类似为一个数组对象，下一层attr
	[WGPEER_A_PROTOCOL_VERSION]			= { .type = NLA_U32 }
};
```
传入的配置解出来后，就可以设置内核peer配置了

## wg_get_device_dump
peer数据展示请求也类似，接收attr格式的请求，然后读取内存数据，封装成attr格式返回回去
其中有一个点，就是peer多的话，返回的数据量很大，单次超过内核分配的sk_buff，则可以返回skb->len长度，让内核先返回数据，然后再继续回调wg_get_device_dump，
继续读取peer数据，直到返回0，表示dump结束


