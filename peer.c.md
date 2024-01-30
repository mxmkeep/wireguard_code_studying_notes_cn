peer在wireguard里是一个很重要的概念，表示VPN的对端，另外interface表示本地

在wireguard的配置里，有一个interface和n个peer

wireguard内核模块加载到内核的时候，是不会自动创建虚拟网卡interface和peer的

两者的创建，是由wireguard的用户态工具wg，读取配置文件，然后通过netlink发送给内核模块进行创建

## wg_peer_create
peer创建，来自netlink.c的set_peer函数调用
```c
MAX_PEERS_PER_DEVICE = 1U << 20,  //1048576
  //最多支持104万peer
	if (wg->num_peers >= MAX_PEERS_PER_DEVICE)
		return ERR_PTR(ret);
```

 未完待续...
