
wireguard采用的是Noise密码框架，握手采用1RTT，一个来回就实现了密钥生成。
这个代码文件主要是负责握手过程的实现。
握手过程是采用非对称算法ECDH
以下是涉及1RTT握手过程的4个函数
```c
wg_noise_handshake_create_initiation()   //客户端发起握手
wg_noise_handshake_consume_initiation()  //服务端接收握手请求进行处理
wg_noise_handshake_create_response()     //服务端创建握手应答做回复
wg_noise_handshake_consume_response()    //客户端收到握手应答并做处理
```

以下流程图是根据握手代码整理的，简化版本的握手流程，省略了多步骤，不保证准确，具体请看代码
![image](https://github.com/mxmkeep/wireguard_code_reading_cn/assets/20048552/4e6fa18c-5d2e-4b00-a835-3b5cf80bee8b)


下面是握手函数wg_noise_handshake_create_initiation的实现过程
```c
bool
wg_noise_handshake_create_initiation(struct message_handshake_initiation *dst,
				     struct noise_handshake *handshake)
{
	u8 timestamp[NOISE_TIMESTAMP_LEN];
	u8 key[NOISE_SYMMETRIC_KEY_LEN];
	bool ret = false;

	/* We need to wait for crng _before_ taking any locks, since
	 * curve25519_generate_secret uses get_random_bytes_wait.
	 */
	wait_for_random_bytes();

	down_read(&handshake->static_identity->lock);
	down_write(&handshake->lock);

	if (unlikely(!handshake->static_identity->has_identity))
		goto out;

	dst->header.type = cpu_to_le32(MESSAGE_HANDSHAKE_INITIATION);
  // 初始化 chaining_key ，代码写了固定值
  // 根据服务端公钥初始化 hash，采用的是blake2s算法
	handshake_init(handshake->chaining_key, handshake->hash,
		       handshake->remote_static);

	/* e */  //生成临时私钥，并计算出对应公钥
	curve25519_generate_secret(handshake->ephemeral_private);
	if (!curve25519_generate_public(dst->unencrypted_ephemeral,
					handshake->ephemeral_private))
		goto out;
  //临时公钥会放入包头，根据临时公钥，更新chaining_key=kdf(dst->unencrypted_ephemeral) 和 hash
	message_ephemeral(dst->unencrypted_ephemeral,
			  dst->unencrypted_ephemeral, handshake->chaining_key,
			  handshake->hash);

	/* es */  //根据椭圆 ECDH 非对称加密算法，计算出密钥，并使用密钥用kdf算法派生出chaining_key 和 临时加密用的密钥key
	if (!mix_dh(handshake->chaining_key, key, handshake->ephemeral_private,
		    handshake->remote_static))
		goto out;

	/* s */  //使用临时密钥key加密自己的公钥，写入包头
	message_encrypt(dst->encrypted_static,
			handshake->static_identity->static_public,
			NOISE_PUBLIC_KEY_LEN, key, handshake->hash);

	/* ss */  //precomputed_static_static 是自己的永久私钥和对方永久公钥通过DH算法计算出来的，双方一致，用这个继续更新chaining_key和key
	if (!mix_precomputed_dh(handshake->chaining_key, key,
				handshake->precomputed_static_static))
		goto out;

	/* {t} */  //用key加密时间戳
	tai64n_now(timestamp);
	message_encrypt(dst->encrypted_timestamp, timestamp,
			NOISE_TIMESTAMP_LEN, key, handshake->hash);
        //为会话生成ID并保存
	dst->sender_index = wg_index_hashtable_insert(
		handshake->entry.peer->device->index_hashtable,
		&handshake->entry);

	handshake->state = HANDSHAKE_CREATED_INITIATION;
	ret = true;

out:
	up_write(&handshake->lock);
	up_read(&handshake->static_identity->lock);
	memzero_explicit(key, NOISE_SYMMETRIC_KEY_LEN);
	return ret;
}
```
握手发起包抓包报文内容
![image](https://github.com/mxmkeep/wireguard_code_reading_cn/assets/20048552/3e46aa10-6dee-473f-8bc3-f1b5eaf444f1)

头部信息
![image](https://github.com/mxmkeep/wireguard_code_reading_cn/assets/20048552/c80b9fa2-1bda-4465-b40a-a92bb55b87c6)



