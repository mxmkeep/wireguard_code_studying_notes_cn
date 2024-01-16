
接收来自udp socket的加密数据包，做一些校验，然后解密，再匹配allowed_ips

## wg_packet_receive()
在socket.c 文件里，初始化socket的时候，指定了UDP包接收函数wg_receive()，然后里边会调用wg_packet_receive，
所以在receive.c文件里，从此函数开始处理接收到的UDP数据包

硬解析wireguard包头后，判断数据包类型，进而做各种处理，其中最复杂的是cookie，类似syn-cookie机制

另外，对于待处理握手包，队列超2048，则丢弃

## wg_packet_consume_data
根据包头里的会话ID，从会话哈希表里找到对应的会话
	ret = wg_queue_enqueue_per_device_and_peer(&wg->decrypt_queue, &peer->rx_queue, skb,
						   wg->packet_crypt_wq, &wg->decrypt_queue.last_cpu);
这句代码，会往两个队列写数据，同时把数据包加入peer->rx_queue和wg->decrypt_queue ，其中wg->decrypt_queue会绑定多CPU并行解密处理。

而peer->rx_queue绑定的是网卡软中断，会等待数据包解密后，按顺序进行解密后逻辑处理，保证包进出的顺序一致

----------
还没写完，待续。。。
