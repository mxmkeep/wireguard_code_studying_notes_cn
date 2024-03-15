
wireguard的配置非常简单，没有各种超时配置和重传配置项，所以让使用者感觉wireguard是无状态的。

而这种实现，全靠程序性的自动处理和固定的各超时时间和重传次数

timers.c 文件就是定时器实现逻辑代码，总共有5个定时器，去自动为何wireguard的各种状态

5个定时器在 wg_timers_init 函数进行初始化
```c
void wg_timers_init(struct wg_peer *peer)
{
//初始化5个定时器及其对应回调函数
	timer_setup(&peer->timer_retransmit_handshake,
		    wg_expired_retransmit_handshake, 0);
	timer_setup(&peer->timer_send_keepalive, wg_expired_send_keepalive, 0);
	timer_setup(&peer->timer_new_handshake, wg_expired_new_handshake, 0);
	timer_setup(&peer->timer_zero_key_material,
		    wg_expired_zero_key_material, 0);
	timer_setup(&peer->timer_persistent_keepalive,
		    wg_expired_send_persistent_keepalive, 0);
	INIT_WORK(&peer->clear_peer_work, wg_queued_expired_zero_key_material);
```

在理解5个定时器之前，先看看messages.h里边的enum limits 枚举
```c
enum limits {
	REKEY_AFTER_MESSAGES = 1ULL << 60,
	REJECT_AFTER_MESSAGES = U64_MAX - COUNTER_WINDOW_SIZE - 1,
	REKEY_TIMEOUT = 5,   //init握手包超时时间
	REKEY_TIMEOUT_JITTER_MAX_JIFFIES = HZ / 3,
	REKEY_AFTER_TIME = 120, //客户端超2分钟没握手则重新发起握手
	REJECT_AFTER_TIME = 180,  //*3 定时清理会话
	INITIATIONS_PER_SECOND = 50,
	MAX_PEERS_PER_DEVICE = 1U << 20,
	KEEPALIVE_TIMEOUT = 10,    //10秒发一次keepalive
	MAX_TIMER_HANDSHAKES = 90 / REKEY_TIMEOUT, //主动握手最多次数 18
	MAX_QUEUED_INCOMING_HANDSHAKES = 4096, /* TODO: replace this with DQL */
	MAX_STAGED_PACKETS = 128,
	MAX_QUEUED_PACKETS = 1024 /* TODO: replace this with DQL */
};
```
## 1 timer_retransmit_handshake
5秒没应答，则重新发起握手，超过18次，即19次后则放弃握手，直到用户层有主动往对方发包(如ping或tcp链接)，则重新开始计数握手。
但 如果开启了 PersistentKeepalive ，那19次后也不会停止，会一直5秒主动尝试握手
回调函数 wg_expired_retransmit_handshake
```c
//主动发起握手包时，会调用
wg_packet_send_handshake_initiation(
void wg_timers_handshake_initiated(struct wg_peer *peer)
{       //REKEY_TIMEOUT=5 秒 加随机数
	mod_peer_timer(peer, &peer->timer_retransmit_handshake,
		       jiffies + REKEY_TIMEOUT * HZ +
		       prandom_u32_max(REKEY_TIMEOUT_JITTER_MAX_JIFFIES));
}

回调函数： wg_expired_retransmit_handshake(
//from_timer宏偏移，根据成员取完整结构体peer
struct wg_peer *peer = from_timer(peer, timer,
	timer_retransmit_handshake);
握手超过18次，则不发起
if (peer->timer_handshake_attempts > MAX_TIMER_HANDSHAKES) 

没超过则重新发起握手
wg_packet_send_queued_handshake_initiation(peer, true);

```

## 2 timer_send_keepalive
回调函数 wg_expired_send_keepalive ， keepalive发送定时器

```c
static void wg_expired_send_keepalive(struct timer_list *timer)
{
	struct wg_peer *peer = from_timer(peer, timer, timer_send_keepalive);
        //KEEPALIVE_TIMEOUT  10秒发一次keepalive
	wg_packet_send_keepalive(peer);
	if (peer->timer_need_another_keepalive) {
		peer->timer_need_another_keepalive = false;
		mod_peer_timer(peer, &peer->timer_send_keepalive,
			       jiffies + KEEPALIVE_TIMEOUT * HZ);
	}
}

//如果有到对方发的包过来，则10秒内不需要发keepalive
void wg_timers_data_received(struct wg_peer *peer)
{
	if (likely(netif_running(peer->device->dev))) {
		if (!timer_pending(&peer->timer_send_keepalive))
			mod_peer_timer(peer, &peer->timer_send_keepalive,
				       jiffies + KEEPALIVE_TIMEOUT * HZ);
		else
			peer->timer_need_another_keepalive = true;
	}
}
```

## 3 timer_new_handshake
主动发消息出去，如果15秒内没回复，则重新发起握手，回调函数 wg_expired_new_handshake

```c
//15秒后，如果对方没回复消息，则重新发起握手
static void wg_expired_new_handshake(struct timer_list *timer)
{
	struct wg_peer *peer = from_timer(peer, timer, timer_new_handshake);

	pr_debug("%s: Retrying handshake with peer %llu (%pISpfsc) because we stopped hearing back after %d seconds\n",
		 peer->device->dev->name, peer->internal_id,
		 &peer->endpoint.addr, KEEPALIVE_TIMEOUT + REKEY_TIMEOUT);
	/* We clear the endpoint address src address, in case this is the cause
	 * of trouble.
	 */
	wg_socket_clear_peer_endpoint_src(peer);
	wg_packet_send_queued_handshake_initiation(peer, false);
}

//消息阶段，主动发送包出去后，会调用此定时器
wg_packet_tx_worker(
  wg_timers_data_sent(peer);
```

## 4 timer_zero_key_material
会话定时清理器
```c
握手完成，或者握手失败，都会定时(180秒*3)判断会话是否需要清理掉
static void wg_expired_zero_key_material(struct timer_list *timer)
{
	struct wg_peer *peer = from_timer(peer, timer, timer_zero_key_material);

	rcu_read_lock_bh();
	if (!READ_ONCE(peer->is_dead)) {
		wg_peer_get(peer);
		if (!queue_work(peer->device->handshake_send_wq,
				&peer->clear_peer_work))
			/* If the work was already on the queue, we want to drop
			 * the extra reference.
			 */
			wg_peer_put(peer);
	}
	rcu_read_unlock_bh();
}
```

## 5 timer_persistent_keepalive

如果启用了定时keepalive选项，则无论会话状态如何，都会定时发送keepalive
在设置的时间内，会发送keepalive，有两种情况
1. 连接成功，但双方没数据交互，那正常流程是2分钟后再握手，如果设置了这个，就算没数据交互，也会定时走keepalive
2. 连接失败，原有keepalive在19次后不再发握手包，如果设置了这个，那会定时走keepalive，重新激活5秒握手尝试，一直走下去
```c
static void wg_expired_send_persistent_keepalive(struct timer_list *timer)
{
	struct wg_peer *peer = from_timer(peer, timer,
					  timer_persistent_keepalive);

	if (likely(peer->persistent_keepalive_interval))
		wg_packet_send_keepalive(peer);
}
```

程序里还有其他定时逻辑判断
## 6 超2分钟没握手需要重新握手
这个逻辑是在数据交互阶段，客户端主动调用
```c
static void keep_key_fresh(struct wg_peer *peer)
{
	struct noise_keypair *keypair;
	bool send;

	rcu_read_lock_bh();
	keypair = rcu_dereference_bh(peer->keypairs.current_keypair);
	send = keypair && READ_ONCE(keypair->sending.is_valid) &&
	       (atomic64_read(&keypair->sending_counter) > REKEY_AFTER_MESSAGES ||
		(keypair->i_am_the_initiator &&
		 wg_birthdate_has_expired(keypair->sending.birthdate, REKEY_AFTER_TIME)));
	rcu_read_unlock_bh();

	if (unlikely(send))
		wg_packet_send_queued_handshake_initiation(peer, false);
}
```

## 7 握手时间超3分钟，拒绝应答
握手时间超3分钟，则会丢弃所有收到的包，拒绝应答
```c
static bool decrypt_packet(struct sk_buff *skb, struct noise_keypair *keypair,
			   simd_context_t *simd_context)
{
	struct scatterlist sg[MAX_SKB_FRAGS + 8];
	struct sk_buff *trailer;
	unsigned int offset;
	int num_frags;

	if (unlikely(!keypair))
		return false;

	if (unlikely(!READ_ONCE(keypair->receiving.is_valid) ||
                 //REJECT_AFTER_TIME = 180 ，  REJECT_AFTER_MESSAGES很大
		  wg_birthdate_has_expired(keypair->receiving.birthdate, REJECT_AFTER_TIME) ||
		  keypair->receiving_counter.counter >= REJECT_AFTER_MESSAGES)) {
		WRITE_ONCE(keypair->receiving.is_valid, false);
		return false;
	}
```

## 最后一个包不是本方发的，需在10秒后发一个keepalive过去
表示自己有收到最后的包，然后双方安静

## 最后一个包是本方发的，但没收到对方的keepalive，需在15秒后主动发起握手
确认对方是否掉线，主动握手跟之前流程一样，发送19次






