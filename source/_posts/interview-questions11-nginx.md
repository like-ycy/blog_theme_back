---
title:      " 运维面试题系列--Nginx、LVS "
date:       2021-03-03
banner_img: "https://gitee.com/like-ycy/images/raw/master/blog/header.jpg"
tags: [ 面试 ]

---

# Nginx LVS Apache



## 理论部分

### 1、LVS的模式：

- **NAT模式**

  - 工作过程
    - （1）：用户将请求发送至调度器上，此时LVS根据算法选择一个后端的真实服务器，将数据请求包转发给真实服务器，并在转发之前LVS会修改数据包中的目标地址以及目标端口，此时修改为后端服务器ip地址
    - （2）：后端服务器将响应的数据包返回给LVS调度器，调度器在响应数据包后会将源地址和源端口修改为vip及调度器相应端口，修改完成后，由调度器响应数据包发送给终端
  - 优点
    - 集群中的物理服务器可以使用任何支持TCP/IP操作系统，只有负载均衡器需要一个合法的IP地址。
  - 缺点
    - 扩展性有限。当服务器节点增长过多时，负载均衡器将成为整个系统的瓶颈，因为所有的请求包和应答包的流向都经过负载均衡器。当服务器节点过多时，大量的数据包都交汇在负载均衡器那，速度就会变慢
  - 注意：在NAT模式中，Real Server的网关必须指向LVS，否则报文无法送达客户端

- **TUN模式**

  - 工作过程
    - （1）：用户将请求发送至调度器上，此时LVS根据算法选择一个后端的真实服务器，将数据请求包转发给后端服务器，并在转发之前LVS会修改数据包中的目标地址以及目标端口，此时修改为后端的服务器ip地址
    - （2）：后端服务器收到请求报文后，会首先拆开第一层封装,然后发现里面还有一层IP首部的目标地址是自己lo接口上的VIP，所以会处理次请求报文，并将响应报文通过lo接口送给eth0网卡直接发送给客户端。
  - 优点
    - 负载均衡器只负责将请求包分发给后端节点服务器，而RS将应答包直接发给用户。所以，减少了负载均衡器的大量数据流动，负载均衡器不再是系统的瓶颈，就能处理很巨大的请求量。
  - 缺点
    - 隧道模式的RS节点需要合法IP，这种方式需要所有的服务器支持”IP Tunneling”(IP Encapsulation)协议，服务器可能只局限在部分Linux系统上。

- **DR模式**

  - 工作过程
    - （1）：用户将请求发送至调度器上，此时LVS根据算法选择一个后端的真实服务器，将数据请求包转发给后端服务器，并在转发之前LVS会修改数据包中的源MAC地址改为自己DIP的MAC地址，目标MAC改为了RIP的MAC地址，并将此包发送给RS
    - （2）：RS发现请求报文中的目的MAC是自己，就会将次报文接收下来，处理完请求报文后，将响应报文通过lo接口送给eth0网卡直接发送给客户端。
  - 优点
    - 和TUN（隧道模式）一样，负载均衡器也只是分发请求，应答包通过单独的路由方法返回给客户端。与VS-TUN相比，VS-DR这种实现方式不需要隧道结构，因此可以使用大多数操作系统做为物理服务器。
  - 缺点
    - 所有 RS 节点和调度器 LB 只能在一个局域网里面。
  - 注意：需要设置lo接口的VIP不能响应本地网络内的arp请求。

- **FULLNAT模式**

  - 无论是 DR 还是 NAT 模式，不可避免的都有一个问题：LVS 和 RS 必须在同一个 VLAN 下，否则 LVS 无法作为 RS 的网关。

    ​	这引发的两个问题是：

    ​	1、同一个 VLAN 的限制导致运维不方便，跨 VLAN 的 RS 无法接入。

    ​	2、LVS 的水平扩展受到制约。当 RS 水平扩容时，总有一天其上的单点 LVS 会成为瓶颈。

  - Full-NAT 由此而生，解决的是 LVS 和 RS 跨 VLAN 的问题，而跨 VLAN 问题解决后，LVS 和 RS 不再存在 VLAN 上的从属关系，可以做到多个 LVS 对应多个 RS，解决水平扩容的问题。

    Full-NAT 相比 NAT 的主要改进是，在 SNAT/DNAT 的基础上，加上另一种转换：

    - 在包从 LVS 转到 RS 的过程中，源地址从客户端 IP 被替换成了 LVS 的内网 IP。

    - 内网 IP 之间可以通过多个交换机跨 VLAN 通信。

    - 当 RS 处理完接受到的包，返回时，会将这个包返回给 LVS 的内网 IP，这一步也不受限于 VLAN。

    - LVS 收到包后，在 NAT 模式修改源地址的基础上，再把 RS 发来的包中的目标地址从 LVS 内网 IP 改为客户端的 IP。

    Full-NAT 主要的思想是把网关和其下机器的通信，改为了普通的网络通信，从而解决了跨 VLAN 的问题。采用这种方式，LVS 和 RS 的部署在 VLAN 上将不再有任何限制，大大提高了运维部署的便利性。
  
  - 总结：
    - FULL NAT 模式也不需要 LBIP 和 realserver ip 在同一个网段； full nat 跟 nat 相比的优点是：保证 RS 回包一定能够回到 LVS；因为源地址就是 LVS--> 不确定
    - full nat 因为要更新 sorce ip 所以性能正常比 nat 模式下降 10%

### 2、LVS的调度算法：

轮询，加权轮询，最小连接数，加权最小连接数，基于局部的最少连接，带复制的基于局部性的最少连接，目标地址散列调度，最短的期望的延迟，最少队列调度

### 3、三大主流软件负载均衡器对比(LVS VS Nginx VS Haproxy）

LVS：
  1、抗负载能力强。抗负载能力强、性能高，能达到F5硬件的60%；对内存和cpu资源消耗比较低
  2、工作在网络4层，通过vrrp协议转发（仅作分发之用），具体的流量由linux内核处理，因此没有流量的产生。
  2、稳定性、可靠性好，自身有完美的热备方案；（如：LVS+Keepalived）
  3、应用范围比较广，可以对所有应用做负载均衡；
  4、不支持正则处理，不能做动静分离。
  5、支持负载均衡算法：rr（轮循）、wrr（带权轮循）、lc（最小连接）、wlc（权重最小连接）
  6、配置 复杂，对网络依赖比较大，稳定性很高。

Ngnix：
  1、工作在网络的7层之上，可以针对http应用做一些分流的策略，比如针对域名、目录结构；
  2、Nginx对网络的依赖比较小，理论上能ping通就就能进行负载功能；
  3、Nginx安装和配置比较简单，测试起来比较方便；
  4、也可以承担高的负载压力且稳定，一般能支撑超过1万次的并发；
  5、对后端服务器的健康检查，只支持通过端口来检测，不支持通过url来检测。
  6、Nginx对请求的异步处理可以帮助节点服务器减轻负载；
  7、Nginx仅能支持http、https和Email协议，这样就在适用范围较小。
  8、不支持Session的直接保持，但能通过ip_hash来解决。、对Big request header的支持不是很好，
  9、支持负载均衡算法：Round-robin（轮循）、Weight-round-robin（带权轮循）、Ip-hash（Ip哈希）
  10、Nginx还能做Web服务器即Cache功能。

HAProxy的特点是：
  1、支持两种代理模式：TCP（四层）和HTTP（七层），支持虚拟主机；
  2、能够补充Nginx的一些缺点比如Session的保持，Cookie的引导等工作
  3、支持url检测后端的服务器出问题的检测会有很好的帮助。
  4、更多的负载均衡策略比如：动态加权轮循(Dynamic Round Robin)，加权源地址哈希(Weighted Source Hash)，加权URL哈希和加权参数哈希(Weighted Parameter Hash)已经实现
  5、单纯从效率上来讲HAProxy更会比Nginx有更出色的负载均衡速度。
  6、HAProxy可以对Mysql进行负载均衡，对后端的DB节点进行检测和负载均衡。
  9、支持负载均衡算法：Round-robin（轮循）、Weight-round-robin（带权轮循）、source（原地址保持）、RI（请求URL）、rdp-cookie（根据cookie）
  10、不能做Web服务器即Cache。

### 4、rewrite语法：

指令语法：rewrite 正则表达式 被替换内容[flag];

简单的小例子：

```
rewrite ^/(.*) http://www.baidu.com/ permanent; # 匹配成功后跳转到百度，执行永久301跳转
```

**常用正则表达式：**

| 字符      | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| \         | 将后面接着的字符标记为一个特殊字符或者一个原义字符或一个向后引用 |
| ^         | 匹配输入字符串的起始位置                                     |
| $         | 匹配输入字符串的结束位置                                     |
| *         | 匹配前面的字符零次或者多次                                   |
| +         | 匹配前面字符串一次或者多次                                   |
| ?         | 匹配前面字符串的零次或者一次                                 |
| .         | 匹配除“\n”之外的所有单个字符                                 |
| (pattern) | 匹配括号内的pattern                                          |

 **rewrite 最后一项flag参数：**

| 标记符号  | 说明                                               |
| --------- | -------------------------------------------------- |
| last      | 本条规则匹配完成后继续向下匹配新的location URI规则 |
| break     | 本条规则匹配完成后终止，不在匹配任何规则           |
| redirect  | 返回302临时重定向                                  |
| permanent | 返回301永久重定向                                  |

### 5、location的规则

```
语法规则： location [=|~|~*|^~] /uri/ { … }
= 		 开头表示精确匹配
^~  	 开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。nginx不对url做编码，因此请求为/static/20%/aa，可以被规则 ^~ /static/ /aa匹配到（注意是空格）。以xx开头
~ 		 开头表示区分大小写的正则匹配                  以xx结尾
~*  	 开头表示不区分大小写的正则匹配                以xx结尾
!~和!~*	分别为区分大小写不匹配及不区分大小写不匹配 的正则
/ 		 通用匹配，任何请求都会匹配到。
```

匹配顺序
首先精确匹配 =-》其次以xx开头匹配^~-》然后是按文件中顺序的正则匹配-》最后是交给 / 通用匹配。
当有匹配成功时候，停止匹配，按当前匹配规则处理请求。

### 6、nginx master-worker机制的好处

```
可以使用 nginx-s reload 热部署。

每个Worker 是独立的进程，不需要加锁，省掉了锁带来的开销。采用独立的进程，可以让互相之间不会影响，一个进程退出后，其他进程还在工作，服务不会中断，Master进程则很快启动新的Worker进程。
```

### 7、alias与root的区别

```
root    实际访问文件路径会拼接URL中的路径
alias   实际访问文件路径不会拼接URL中的路径，会把location后面配置的路径丢弃掉，把当前匹配到的目录指向到指定的目录
```

> 注意：
> 1. 使用alias时，目录名后面一定要加"/"。
> 2. alias在使用正则匹配时，必须捕捉要匹配的内容并在指定的内容处使用。
> 3. alias只能位于location块中。（root可以不放在location中）

### 8、nginx日志切割脚本

```
# /bin/bash
 
# 日志保存位置
base_path='/opt/nginx/logs'
# 获取当前年信息和月信息
log_path=$(date -d yesterday +"%Y%m")
# 获取昨天的日信息
day=$(date -d yesterday +"%d")
# 按年月创建文件夹
mkdir -p $base_path/$log_path
# 备份昨天的日志到当月的文件夹
mv $base_path/access.log $base_path/$log_path/access_$day.log
# 输出备份日志文件名
# echo $base_path/$log_path/access_$day.log
# 通过Nginx信号量控制重读日志
kill -USR1 `cat /opt/nginx/logs/nginx.pid`
#删除7天前的日志
cd $base_path/$log_path/
find . -mtime +7 -name "*20[1-9][3-9]*" | xargs rm -f
```

### 9、nginx优化

```
1. worker_processes 8;
nginx 进程数，建议按照cpu 数目来指定，一般为它的倍数 (如,2个四核的cpu计为8)。
2. worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;
为每个进程分配cpu，上例中将8 个进程分配到8 个cpu，当然可以写多个，或者将一
个进程分配到多个cpu。
3. worker_rlimit_nofile 65535;
这个指令是指当一个nginx 进程打开的最多文件描述符数目，理论值应该是最多打开文
件数（ulimit -n）与nginx 进程数相除，但是nginx 分配请求并不是那么均匀，所以最好与ulimit -n 的值保持一致。
4. use epoll;
使用epoll 的I/O 模型
5. worker_connections 65535;
每个进程允许的最多连接数， 理论上每台nginx 服务器的最大连接数为worker_processes*worker_connections。
6. keepalive_timeout 60;
keepalive 超时时间。
7. client_header_buffer_size 4k;
客户端请求头部的缓冲区大小，这个可以根据你的系统分页大小来设置，一般一个请求头的大小不会超过1k，不过由于一般系统分页都要大于1k，所以这里设置为分页大小。
8. open_file_cache max=65535 inactive=60s;
这个将为打开文件指定缓存，默认是没有启用的，max 指定缓存数量，建议和打开文件数一致，inactive 是指经过多长时间文件没被请求后删除缓存。
9. open_file_cache_valid 80s;
这个是指多长时间检查一次缓存的有效信息。
10. open_file_cache_min_uses 1;
open_file_cache 指令中的inactive 参数时间内文件的最少使用次数，如果超过这个数字，文件描述符一直是在缓存中打开的，如上例，如果有一个文件在inactive 时间内一次没被使用，它将被移除。
11. fastcgi_connect_timeout 300;
指定连接到后端FastCGI 的超时时间。
12. fastcgi_send_timeout 300;
向FastCGI 传送请求的超时时间，这个值是指已经完成两次握手后向FastCGI 传送请求的超时时间。
13. fastcgi_read_timeout 300;
接收FastCGI 应答的超时时间，这个值是指已经完成两次握手后接收FastCGI 应答的超时时间。
14. fastcgi_buffer_size 4k;
指定读取FastCGI 应答第一部分需要用多大的缓冲区，一般第一部分应答不会超过1k，由于页面大小为4k，所以这里设置为4k。
15. fastcgi_buffers 8 4k;
指定本地需要用多少和多大的缓冲区来缓冲FastCGI 的应答。
16. fastcgi_busy_buffers_size 8k;
这个指令我也不知道是做什么用，只知道默认值是fastcgi_buffers 的两倍。
17. fastcgi_temp_file_write_size 8k;
在写入fastcgi_temp_path 时将用多大的数据块，默认值是fastcgi_buffers 的两倍。
18. fastcgi_cache TEST
开启FastCGI 缓存并且为其制定一个名称。个人感觉开启缓存非常有用，可以有效降低CPU 负载，并且防止502 错误。
```

### 10、nginx reload的流程

**详情请跳转至**：[nginx -s reload的流程](https://like-ycy.github.io/2019/12/26/2019-12-26-nginx-reload/)