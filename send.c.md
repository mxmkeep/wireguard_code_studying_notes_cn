send.c 接收来自虚拟网卡的数据包，进行鉴权和加密处理，后续通过udp socket发送出去。握手成功后，数据包传输过程中函数的调用顺序如下：
1. wg_xmit (device.c)
2. wg_packet_encrypt_worker
3. wg_packet_tx_worker
4. wg_packet_create_data_done


### wg_xmit
- 虚拟网卡收到数据包后，调用由函数 wg_xmit 进行处理
- 根据数据包的目标IP，去匹配对应的peer，wg_allowedips_lookup_dst，如果没匹配到，则丢弃
- 同时添加到 wg->encrypt_queue 和 peer->tx_queue ，多核CPU并行加密，peer维度顺序发送
### wg_packet_encrypt_worker
- 从wg->encrypt_queue取出数据进行加密
- 给工作队列peer->transmit_packet_work派任务，让CPU回调wg_packet_tx_worker

### wg_packet_tx_worker

将数据包从队列取出，然后调用wg_packet_create_data_done 函数。
使用 wg_socket_send_skb_to_peer  send4 往endpoint发包


### 工作队列workqueue_struct
在Linux内核中，工作队列是一种允许将一些工作推迟到稍后执行的机制。工作队列常常用于响应中断或处理设备驱动程序的异步事件
两个概念：工作队列和工作项；工作项绑定回调函数；工作项添加到工作队列里，稍后CPU会调用回调函数进行处理；工作队列可以添加多种工作项。
代码编写流程
1. struct workqueue_struct *handshake_send_wq;：这里定义了一个指向工作队列的指针。一个工作队列由许多要执行的工作项（work_struct）组成。
2. wg->handshake_send_wq = alloc_workqueue("wg-kex-%s",  WQ_UNBOUND | WQ_FREEZABLE, 0, dev->name);  创建工作队列
3. struct work_struct clear_peer_work;：每个要执行的工作都需要一个work_struct结构体来描述。
4. INIT_WORK(&peer->clear_peer_work, wg_queued_expired_zero_key_material)绑定工作项回调函数
5. queue_work(peer->device->handshake_send_wq, &peer->clear_peer_work);：这行代码将clear_peer_work工作项添加到handshake_send_wq工作队列中。这意味着wg_queued_expired_zero_key_material函数将在稍后由一个内核线程执行。
6. struct wg_peer *peer = container_of(work, struct wg_peer, clear_peer_work);回调函数使用这行代码从工作项获取到包含它的wg_peer结构体的指针。container_of是一个常用的宏，它根据成员的地址来获取包含这个成员的结构体的地址。
7. flush_work(&peer->clear_peer_work);：这行代码等待clear_peer_work工作项完成。如果工作项当前正在执行，那么flush_work会阻塞直到工作项执行完成。如果工作项还在队列中等待执行，那么flush_work会取消它。
8. flush_workqueue(peer->device->handshake_send_wq); 阻塞当前的执行线程，直到指定的工作队列上的所有工作项都执行完成
9. destroy_workqueue(wg->handshake_send_wq); 销毁工作队列



