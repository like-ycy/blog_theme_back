---
title:      " Linux内核调优 "
date:       2020-07-20 17:08 
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Linux ]

---

以nginx 10k并发连接为优化目标，附简单介绍，不一一解释。

### tcp容量规划

```
net.ipv4.tcp_mem  = 262144  524288 786432
net.core.wmem_max = 16777216
net.core.wmem_default = 131072
net.core.rmem_max = 16777216
net.core.rmem_default = 131072
net.ipv4.tcp_wmem = 4096    131072  16777216
net.ipv4.tcp_rmem = 4096    131072  16777216
```

**net.ipv4.tcp_mem ** 单位是内存页，一般是4k，三个值分别代表tcp内存使用的水平，低、中、高， 低表示无内存压力，中级表示内存压力状态，高表示内存吃紧，最高峰时系统将会拒绝分配内存。262144 代表1G内存，即（262144x4/1024/1024），其他类推。

下面的参数单位都是字节 net.core.wmem_max 和net.core.wmem_default 会覆盖net.ipv4.tcp_wmem 的第二第三个值， 同理，net.core.rmem_max 和 net.core.rmem_default 会覆盖net.ipv4.tcp_rmem 的第二第三个值。稍微提高tcp读写缓冲区的容量，可以增加tcp传输效率，比如上文默认值131072=128k，现有一个1M的文件传输，只需8次传输即可，比较适合图片类传输。但也不是越大越好，比如一个文字页面只有15k，使用128k的内存显然有些浪费。上文tcp压力状态下的容量为2G，对应tcp读写缓冲区128k，可应对的连接数为16384 (2048x1024/128)，可满足10k要求。

### tcp连接行为管理

```
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_max_tw_buckets = 8192
net.ipv4.tcp_retries1 = 3
net.ipv4.tcp_retries2 = 5
net.ipv4.tcp_keepalive_time = 1800
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_orphans = 262144
```

上面主要是tcp连接行为的伴随的参数，主要是tcp重用，增加队列，减少等待重试频率等等来提升效率。

### 内存管理

```
vm.swappiness = 5
vm.dirty_ratio = 40
vm.min_free_kbytes = 524288
vm.vfs_cache_pressure = 100
```

- vm.swappiness = 5 表示物理内存剩余5%时，才考虑使用swap，默认60，这显然非常不合理
- •vm.dirty_ratio = 40 表示拿出物理内存的40%用于写缓存，而不立即将数据写入硬盘。由于硬盘是众所周知的瓶颈，扩大它可提升写的效率，40%是个比较合适的比例。
- vm.min_free_kbytes = 524288 这个用于控制剩余内存的大小，524288=512M，可根据需要调整。如果某些任务临时需要大量内存，可临时将它调大然后调小，回收页面缓存。它比vm.drop_caches 要温和得多，后者更粗暴。
- vm.vfs_cache_pressure = 100 ，如果要尽快将脏数据刷进硬盘，提高它，比如150 。

### 内核其他行为

```
net.core.somaxconn = 8192
net.core.netdev_max_backlog = 8192
net.ipv4.ip_local_port_range = 15000 65000
net.netfilter.nf_conntrack_max = 131072
net.nf_conntrack_max = 131072
net.ipv6.conf.all.disable_ipv6 =1
net.netfilter.nf_conntrack_tcp_timeout_established =3600
net.core.rps_sock_flow_entries = 32768
```

net.core.somaxconn 表示socket的最大连接数，默认128，对于php-fpm使用unix socket情况下，需要调大。

net.netfilter.nf_conntrack_tcp_timeout_established = 3600 默认2天时间，多数情况下，调小这个参数是有益的，如果是tcp长连接，这个参数可能不太合适。

net.core.rps_sock_flow_entries 这个参数启用RPS，自动将网卡中断均匀分配到多个CPU，改进网卡性能和系统负载。

RPS还需要脚本配合

```
for fileRfc in $(ls /sys/class/net/eth*/queues/rx-*/rps_flow_cnt);do echo 2048 > $fileRfc;done
```