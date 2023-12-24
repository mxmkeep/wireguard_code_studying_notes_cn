# wireguard代码中文阅读笔记
wireguard代码阅读笔记，wireguard代码详解

最近在学习wireguard，是一个很优秀的项目，所以其代码很快被并入到内核当中去。

本项目是阅读代码过程中记录下来的笔记，代码版本 https://git.zx2c4.com/wireguard-linux-compat  适合像我之前没接触内核模块代码的人并且想了解wireguard代码的人阅读。

有问题可以提issues讨论。

|代码文件|描述|
|--|--|
|allowedips.c|allowedips的数据结构实现，采用二叉前缀树保存|
|cookie.c|实现类似syn-cookie 的逻辑，握手包里的两个mac变量的计算逻辑|
|device.c|用ip link add创建虚拟网卡时会调用到的一些函数|
|main.c|modprobe wireguard命令加载内核模块时，对应的初始化逻辑|
|messages.h|握手过程的相关枚举值和数据结构|
|netlink.c|创建netlink socket供用户态程序设置和查询配置|
|noice.c|实现握手过程并创建会话，四个状态对应4个函数|
|peer.c|peer管理代码实现，哈希链保存peer ， key为public key|
|peerlookup.c|会话管理代码实现，哈希链保存会话，key为随机数|
|queueing.c|数据包流转过程用到的消息队列|
|ratelimiter.c|当握手量出现超载时，会对单IP握手速率进行限制，此为实现逻辑|
|receive.c|接收来自udp socket的加密数据包，进行解密处理逻辑，后续会从对应的虚拟网卡发送出去|
|send.c|接收来着虚拟网卡的数据包，加密处理逻辑，后续会通过udp socket发送出去|
|socket.c|udp socket实现，负责收发udp包|
|timers.c|几个定时器的实现逻辑，定时握手、定时keepalive、超时逻辑等|
|uapi/wireguard.h|netlink接口对接用户态程序用到的各种枚举值|
|crypto/zinc/blake2s|一种哈希算法，用来在kdf里派生密钥|
|crypto/zinc/chacha20|流密码加密算法，用来做对称加解密|
|crypto/zinc/curve25519|非对称加密ECDH，握手阶段使用|
|crypto/zinc/poly1305|消息一致性算法|






