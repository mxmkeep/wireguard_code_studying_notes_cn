
如果出现wireguard高负载，即握手包缓存队列太大，超1024，则启用cookie校验功能，而cookie校验，会做限速，限速是针对单来源IP的。

程序假定每个握手包耗时0.05秒，单IP每秒最大20个握手包，缓冲4个(TOKEN_MAX)

实现逻辑在 wg_ratelimiter_allow 函数里

```c
//限速实现逻辑
hlist_for_each_entry_rcu(entry, bucket, hash) {
    if (entry->net == net && entry->ip == ip) {
        u64 now, tokens;
        bool ret;
        /* Quasi-inspired by nft_limit.c, but this is actually a
         * slightly different algorithm. Namely, we incorporate
         * the burst as part of the maximum tokens, rather than
         * as part of the rate.
         */
        spin_lock(&entry->lock);
        //获取纳秒当前时间(精度为)10亿分之一，
        now = ktime_get_coarse_boottime_ns();
        //获取tokens数，最大为TOKEN_MAX，2.5亿
        tokens = min_t(u64, TOKEN_MAX,
                   entry->tokens + now -
                       entry->last_time_ns);
        entry->last_time_ns = now;
        //是否限速，如果token小于5千万，则限速，一个握手包需要5千万，就是0.05秒
        ret = tokens >= PACKET_COST;
        //重新计算剩余token值：如果不需要限速，则减去5千万
        entry->tokens = ret ? tokens - PACKET_COST : tokens;
        spin_unlock(&entry->lock);
        rcu_read_unlock();
        return ret;
    }
}
```
