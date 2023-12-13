
wireguard采用的是Noise密码框架，握手采用1RTT，一个来回就实现了密钥生成。
握手过程是采用非对称算法ECDH，
这个代码文件主要是负责握手过程的实现。
以下是涉及1RTT握手过程的4个函数
```c
wg_noise_handshake_create_initiation()   //客户端发起握手
wg_noise_handshake_consume_initiation()  //服务端接收握手请求进行处理
wg_noise_handshake_create_response()     //服务端创建握手应答做回复
wg_noise_handshake_consume_response()    //客户端收到握手应答并做处理
```
以下流程图是根据握手代码整理的，简化版本的握手流程，省略了多步骤，不保证准确，具体请看代码
![1702482229203](https://github.com/mxmkeep/wireguard_code_reading_cn/assets/20048552/f8a4aaa3-488c-49d2-a197-65a392516342)

