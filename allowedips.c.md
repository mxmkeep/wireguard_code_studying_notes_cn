allowedips数据结构
```c
struct allowedips {
	struct allowedips_node __rcu *root4;
	struct allowedips_node __rcu *root6;
	u64 seq;
} __aligned(4);
```
allowedips是一颗二叉前缀树，用来保存所有peer的allowedips配置，是全局的，挂在device全局变量下。
所以，如果peer间的allowedips有重复或范围覆盖，则会匹配最小范围的peer或最后配置的peer

allowedips_node结构体的具体解释
```c
struct allowedips_node {
	struct wg_peer __rcu *peer;
	struct allowedips_node __rcu *bit[2];  //两个子节点，所以是二分查找树
	u8 cidr,   //子网掩码 1.1.0.0/25  中的25
        bit_at_a,   // bit_at_a = cidr / 8U , 表示固定IP的最后字节，例如掩码为25，bit_at_a=25/8=3 ，IP的前3字节是固定的，后面的才是覆盖地址范围
        bit_at_b,  //bit_at_b = 7U - (cidr % 8U) ， a的前3字节固定，那第4字节从哪开始表示覆盖地址，7-25%8=6 ， 表示第4字节的右边6位是掩码
        bitlen;  //v4 为 32
	u8 bits[16] __aligned(__alignof(u64)); //保存v4 或 v6 IP

	/* Keep rarely used members at bottom to be beyond cache line. */
	unsigned long parent_bit_packed;  //父节点位置
	union {
		struct list_head peer_list;  //在peer链表里的位置
		struct rcu_head rcu; //（Read-Copy-Update），数据安全更新
	};
};
```
一颗allowedips前缀树具体的样子，可以通过执行wireguard里的单元测试代码 wg_allowedips_selftest() 函数 生成并绘制出来
IPv4测试代码的信息在系统日志里显示如下
```
digraph trie {
        "0.0.0.0/0"[style=bold, color="#357f1e"];
        "0.0.0.0/0" -> "0.0.0.0/1";
        "0.0.0.0/0" -> "192.0.0.0/8";
        "0.0.0.0/1"[style=dotted, color="#000000"];
        "0.0.0.0/1" -> "10.0.0.0/15";
        "0.0.0.0/1" -> "64.15.112.0/20";
        "10.0.0.0/15"[style=dotted, color="#000000"];
        "10.0.0.0/15" -> "10.0.0.0/24";
        "10.0.0.0/15" -> "10.1.0.0/27";
        "10.0.0.0/24"[style=dotted, color="#000000"];
        "10.0.0.0/24" -> "10.0.0.0/25";
        "10.0.0.0/24" -> "10.0.0.128/25";
        "10.0.0.0/25"[style=bold, color="#9b1223"];
        "10.0.0.128/25"[style=bold, color="#86954c"];
        "10.1.0.0/27"[style=dotted, color="#000000"];
        "10.1.0.0/27" -> "10.1.0.0/28";
        "10.1.0.0/27" -> "10.1.0.16/29";
        "10.1.0.0/28"[style=dotted, color="#000000"];
        "10.1.0.0/28" -> "10.1.0.0/29";
        "10.1.0.0/28" -> "10.1.0.8/29";
        "10.1.0.0/29"[style=dotted, color="#000000"];
        "10.1.0.0/29" -> "10.1.0.0/30";
        "10.1.0.0/29" -> "10.1.0.4/30";
        "10.1.0.0/30"[style=bold, color="#9b1223"];
        "10.1.0.4/30"[style=bold, color="#86954c"];
        "10.1.0.8/29"[style=bold, color="#114313"];
        "10.1.0.16/29"[style=bold, color="#7cc29d"];
        "64.15.112.0/20"[style=bold, color="#3692b0"];
        "64.15.112.0/20" -> "64.15.123.128/25";
        "64.15.123.128/25"[style=bold, color="#c40ca4"];
        "192.0.0.0/8"[style=dotted, color="#000000"];
        "192.0.0.0/8" -> "192.95.5.64/27";
        "192.0.0.0/8" -> "192.168.0.0/16";
        "192.95.5.64/27"[style=bold, color="#114313"];
        "192.168.0.0/16"[style=bold, color="#114313"];
        "192.168.0.0/16" -> "192.168.4.0/24";
        "192.168.4.0/24"[style=bold, color="#9b1223"];
        "192.168.4.0/24" -> "192.168.4.4/32";
        "192.168.4.4/32"[style=bold, color="#86954c"];
 }
```
用graphviz工具绘制
![image](https://github.com/mxmkeep/wireguard_code_studying_notes_cn/assets/20048552/8ed6c190-03d6-4d45-89c2-e7c26c86fc45)

从图片上，可以很直观的看出前缀二叉树的模样，其中，虚线框的节点，表示算法自动添加的中间节点，没有保存peer信息，查找时会被过滤掉


prefix_matches函数是前缀匹配的具体操作
```c
static u8 common_bits(const struct allowedips_node *node, const u8 *key,
		      u8 bits)
{
	if (bits == 32)
               //做异或操作，然后使用fls函数（find last set，找到最后一个置位）找到最后一位不同
               //用32减去，表示相同位数长度
		return 32U - fls(*(const u32 *)node->bits ^ *(const u32 *)key);
	else if (bits == 128)
		return 128U - fls128(
			*(const u64 *)&node->bits[0] ^ *(const u64 *)&key[0],
			*(const u64 *)&node->bits[8] ^ *(const u64 *)&key[8]);
	return 0;
}

//IP前缀匹配操作
static bool prefix_matches(const struct allowedips_node *node, const u8 *key,
			   u8 bits)
{
	/* This could be much faster if it actually just compared the common
	 * bits properly, by precomputing a mask bswap(~0 << (32 - cidr)), and
	 * the rest, but it turns out that common_bits is already super fast on
	 * modern processors, even taking into account the unfortunate bswap.
	 * So, we just inline it like this instead.
	 */
        //相同位数长度大于节点的cidr，则表示节点node地址覆盖了key，返回true
        //cidr数字越小，覆盖范围越大，所以比cidr大，说明被cidr覆盖了
	return common_bits(node, key, bits) >= node->cidr;
}

```

