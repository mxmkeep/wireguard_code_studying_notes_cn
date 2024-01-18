
接收来自udp socket的加密数据包，做一些校验，然后解密，再匹配allowed_ips

主要函数的调用顺序为：
1. wg_packet_receive
2. wg_packet_consume_data
3. wg_packet_decrypt_worker
4. wg_packet_rx_poll
5. counter_validate
6. wg_packet_consume_data_done

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

而peer->rx_queue绑定的是网卡软中断，会等待数据包解密后，按顺序进行解密后逻辑处理，保证单个peer包进出的顺序一致


## counter_validate
在处理解密后的数据包wg_packet_rx_poll()逻辑中，会调用counter_validate()函数检验是否存在重放攻击

RFC6479 使用位图来保存counter，可以检测是否重放，这样就可以允许counter乱序
当然如果发来的counter比本地记录的counter小超过Window_size，则会被丢弃

## wg_packet_consume_data_done
会调用wg_allowedips_lookup_src函数校验来源的内网IP是否在当前peer配置的allowed_ip里边

allowed_ip 配置
* 对于发起的客户端，用来配置路由和匹配目标IP查找对应peer，就是发送到哪个peer。
* 对应接收方服务端来说，就是用来校验来源IP是否合法。

就是wireguard里提到的一个概率，叫 密钥路由： 密钥和内网IP路由绑定在一起，保证数据来源的安全性


