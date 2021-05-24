---
title:      " 用户访问网站慢，排查思路 "
date:       2020/07/01 17:01
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ 面试 ]


---

当出现网站慢的时候我们脑子中要映出几点原因：

1.程序代码执行方面

2.大量数据库操作

3.域名DNS解析问题

4.服务器环境

5.网络的带宽

6.用许多javascript特效

7.访问的东西大

8.系统资源不足

9.防火墙的过多使用

10.网络中某个端口形成了瓶颈导致网速变慢


\1.打开访问慢的网站观察下情况，通过火狐的fixfox 插件 或者 IE的元素查看工具，你网站里面加载的信息会一览无遗的展现出来，并且那些元素加载耗时多少秒等等情况，如何解决能，把远程耗时久的js下载到本地，或者直接删除。

\2. 我看了下页面中有多处连接数据库操作的地方，并且有远程的数据库操作，并且还有多余的数据库连接代码，话不多说，改之.

   解决完了发现的确是快点了，但是还是不理想，于是我把页面执行数据库代码放到了数据库中执行没有耗慢的情况。

\3. 关于域名DNS的情况只是其中一种情况，不要急着找域名商的问题，你可以写个没有数据操作的页面放在同台服务器域名下，看看是不是访问同样慢，如果是才有可能，你还要让你周围的人也看看，最好别是你同公司的人。

\4. 我来看看服务器的情况吧，是不是CPU使用率过高造成的呢。

   a. top  发现cpu使用也不高啊，30% 左右，但是发现一个问题，sleeping 的进程数比较多。擦，最好别是僵尸进程，现在这样的东西不多了。

   b. 查看了下timewait的量: 发现有mysqld 和 httpd 的，大部分来自于 httpd  ； 命令 netstat -ae|grep TIME_WAIT

      如何来解决timewait的量问题呢？

TIME_WAIT解决办法：

vi /etc/sysctl.conf

编辑文件，加入以下内容：
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 30  保持连接的时间
net.ipv4.tcp_max_tw_buckets = 100 这个是设置服务器同时保持的time_wait的数目 

然后执行 /sbin/sysctl -p 让参数生效。

如果还不够满意可以 再设置下Ulimit参数
cat >>/etc/security/limits.conf<<EOF
\* soft nofile 655350
\* hard nofile 655350
EOF
然后ulimit -SHn 了 让生效。
