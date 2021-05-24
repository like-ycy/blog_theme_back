---
title:      " Nginx的这些妙用，你肯定有不知道的 "
excerpt: "Nginx 因为它的稳定性、丰富的模块库、灵活的配置和较低的资源消耗而闻名 。目前应该是几乎所有项目建设必备。今天通过这篇攻略让你快速通关 Nginx。"
date:       2019-12-19
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Nginx ]

---

# Nginx 简介

Nginx 是一个免费、开源、高性能、轻量级的 HTTP 和反向代理服务器，也是一个电子邮件（IMAP/POP3）代理服务器，其特点是占有内存少，并发能力强。

Nginx 由内核和一系列模块组成，内核提供 Web 服务的基本功能，如启用网络协议，创建运行环境，接收和分配客户端请求，处理模块之间的交互。

Nginx 的各种功能和操作都由模块来实现。Nginx 的模块从结构上分为：

- **核心模块：**HTTP 模块、EVENT 模块和 MAIL 模块。
- **基础模块：**HTTP Access 模块、HTTP FastCGI 模块、HTTP Proxy 模块和 HTTP Rewrite 模块。
- **第三方模块：**HTTP Upstream Request Hash 模块、Notice 模块和 HTTP Access Key 模块及用户自己开发的模块。

这样的设计使 Nginx 方便开发和扩展，也正因此才使得 Nginx 功能如此强大。

Nginx 的模块默认编译进 Nginx 中，如果需要增加或删除模块，需要重新编译 Nginx，这一点不如 Apache 的动态加载模块方便。

如果有需要动态加载模块，可以使用由淘宝网发起的 Web 服务器 Tengine，在 Nginx 的基础上增加了很多高级特性，完全兼容 Nginx，已被国内很多网站采用。

Nginx 有很多扩展版本：

- **开源版 nginx.org**

- **商业版 NGINX Plus**

- **淘宝网发起的 Web 服务器 Tengine**

- **基于 Nginx 和 Lua 的 Web 平台 OpenResty**

# Nginx 作为 Web 服务器

Web 服务器也称为 WWW（World Wide Web）服务器，主要功能是提供网上信息浏览服务，常常以 B/S（Browser/Server）方式提供服务：

- **应用层使用 HTTP 协议。**
- **HTML 文档格式。**
- **浏览器统一资源定位器(URL)。**

Nginx 可以作为静态页面的 Web 服务器，同时还支持 CGI 协议的动态语言，比如 Perl、PHP 等，但是不支持 Java。

Java 程序一般都通过与 Tomcat 配合完成。作为一名 Java 程序员，肯定要理解下 Nginx 和 Tomcat 的区别了。

Nginx、Apache 和 Tomcat：

- **Nginx：**由俄罗斯程序员 Igor Sysoev 所开发的轻量级、高并发 HTTP 服务器。

- **Apache HTTP Server Project：**一个 Apache 基金会下的 HTTP 服务项目，和 Nginx 功能类似。

- **Apache Tomcat：**是 Apache 基金会下的另外一个项目，是一个 Application Server。

  更准确的说是一个 Servlet 应用容器，与 Apache HTTP Server 和 Nginx 相比，Tomcat 能够动态的生成资源并返回到客户端。

Apache HTTP Server 和 Nginx 本身不支持生成动态页面，但它们可以通过其他模块来支持（例如通过 Shell、PHP、Python 脚本程序来动态生成内容）。

一个 HTTP Server 关心的是 HTTP 协议层面的传输和访问控制，所以在 Apache/Nginx 上你可以看到代理、负载均衡等功能。

客户端通过 HTTP Server 访问服务器上存储的资源（HTML 文件、图片文件等等）。

通过 CGI 技术，也可以将处理过的内容通过 HTTP Server 分发，但是一个 HTTP Server 始终只是把服务器上的文件如实的通过 HTTP 协议传输给客户端。

而应用服务器，则是一个应用执行的容器。它首先需要支持开发语言的运行（对于 Tomcat 来说，就是 Java），保证应用能够在应用服务器上正常运行。

其次，需要支持应用相关的规范，例如类库、安全方面的特性。对于 Tomcat 来说，就是需要提供 JSP/Sevlet 运行需要的标准类库、Interface 等。

为了方便，应用服务器往往也会集成 HTTP Server 的功能，但是不如专业的 HTTP Server 那么强大。

所以应用服务器往往是运行在 HTTP Server 的背后，执行应用，将动态的内容转化为静态的内容之后，通过 HTTP Server 分发到客户端。

## 正向代理

**正向代理：**如果把局域网外的 Internet 想象成一个巨大的资源库，则局域网中的客户端要访问 Internet，则需要通过代理服务器来访问，这种代理服务就称为正向代理。

正向代理“代理”的是客户端。比如你想去 Google 看个“动作片”，可国内不允许呀，就需要找翻墙代理，这个就是所谓的“正向代理”。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/1.png)

## 反向代理与负载均衡

反向代理正好与正向代理相反，反向代理是指以代理服务器来接收 Internet 上的连接请求，然后将请求转发到内部网络上的服务器，并将服务器上得到的结果返回给客户端。

此时代理服务器对外表现就是一个服务器，客户端对代理是无感知的。反向代理“代理”的是服务端。

再比如，你想本本分分的在“优酷”上看个“爱情片”，youku.com 会把你的请求分发到存放片片的那台机器上，这个就是所谓的“反向代理”。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/2.png)

为什么使用反向代理，原因如下：

- **保护和隐藏原始资源服务器**
- **加密和 SSL 加速**
- **通过缓存静态资源，加速 Web 请求**
- **实现负载均衡**

**负载均衡：**TODO: 留一个负载均衡详细介绍传送门。

**地址重定向：**Nginx 的 Rewrite 主要的功能就是实现 URL 重写，比如输入 360.com  跳转到了 360.cn，baidu.cn 跳转到了 baidu.com。



## 动静分离

为了加快网站的解析速度，可以把动态页面和静态页面由不同的服务器来解析，加快解析速度，降低原来单个服务器的压力。

这里指的就是让动态程序（Java、PHP）去访问应用服务器，让缓存、图片、JS、CSS 等去访问 Nginx。



## 配置文件

nginx.conf 配置文件主要分为三部分：

- **全局块**
- **Events 块**
- **HTTPS 块**

Nginx 配置语法：

- 配置文件由指令和指令块构成
- 每条指令以分号（;）结尾，指令和参数间以空格符分隔
- 指令块以大括号{}将多条指令组织在一起
- include 语句允许组合多个配置文件以提高可维护性
- 使用 # 添加注释
- 使用 $ 定义变量
- 部分指令的参数支持正则表达式

### 全局块

全局配置部分用来配置对整个 Server 都有效的参数。主要会设置一些影响 Nginx 服务器整体运行的配置指令，包括配置运行 Nginx 服务器的用户（组）、允许生成的 Worker Process 数，进程 PID 存放路径、日志存放路径和类型以及配置文件的引入等。

示例如下：

```bash
user nobody;
worker_processes  4;
error_log  /data/nginx/logs/error.log  notice; 
```



### Events 块

Events 块涉及的指令主要影响 Nginx 服务器与用户的网络连接，常用的设置包括是否开启对多 Work Process 下的网络连接进行序列化，是否允许同时接收多个网络连接，选取哪种事件驱动模型来处理连接请求，每个 Word Process 可以同时支持的最大连接数等。

```bash
events {
    #每个 work process 支持的最大连接数为 1024.
    worker_connections  1024;
}
```



### HTTP 块

这算是 Nginx 服务器配置中最频繁的部分，代理、缓存和日志定义等绝大多数功能和第三方模块的配置都在这里。 需要注意的是：HTTP 块也可以包括 HTTP 全局块、Server 块。

**①HTTP 全局块**

HTTP 全局块配置的指令包括文件引入、MIME-TYPE 定义、日志自定义、连接超时时间、单链接请求数上限等。

```bash
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
```

**②Server 块**

这块和虚拟主机有密切关系，虚拟主机从用户角度看，和一台独立的硬件主机是完全一样的，该技术的产生是为了节省互联网服务器硬件成本。

每个 HTTP 块可以包括多个 Server 块，而每个 Server 块就相当于一个虚拟主机。

而每个 Server 块也分为全局 Server 块，以及可以同时包含多个 Locaton 块。

**全局 Server 块：**也被叫做“虚拟服务器”部分，它描述的是一组根据不同server_name指令逻辑分割的资源，这些虚拟服务器响应 HTTP 请求，因此都包含在 HTTP 部分。

最常见的配置是本虚拟机主机的监听配置和本虚拟主机的名称或 IP 配置。

```bash
server {
  listen       80;
  #server_name也支持通配符，*.example.com、www.example.*、.example.com
  server_name  localhost;
  #charset koi8-r;
  #access_log  logs/host.access.log  main;
```

**Location 块：**一个 Server 块可以配置多个 Location 块。

这块的主要作用是基于 Nginx 服务器接收到的请求字符串（例如 server_name/uri-string），对虚拟主机名称 （也可以是 IP 别名）之外的字符串（例如前面的 /uri-string）进行匹配，对特定的请求进行处理。

地址定向、数据缓存和应答控制等功能，还有许多第三方模块的配置也在这里进行。

**Location 指令说明：**该指令用于匹配 URL。

语法如下：

```
location [ = | ~ | ~* | ^~] uri{}
```

- **= ：**该修饰符使用精确匹配并且终止搜索。
- **~：**该修饰符使用区分大小写的正则表达式匹配。
- **~\*：**该修饰符使用不区分大小写的正则表达式匹配。
- **^~：**用于不含正则表达式的 URI 前，要求 Nginx 服务器找到标识 URI 和请求字符串匹配度最高的 Location 后，立即使用此 Location 处理请求，而不再使用 Location 块中的正则 URI 和请求字符串做匹配。

?>Tip 注意：如果 URI 包含正则表达式，则必须要有 ~ 或者 ~* 标识。

当一个请求进入时，URI 将会被检测匹配一个最佳的 Location：

- 没有正则表达式的 Location 被作为最佳的匹配，独立于含有正则表达式的 Location 顺序。
- 在配置文件中按照查找顺序进行正则表达式匹配。在查找到第一个正则表达式匹配之后结束查找。由这个最佳的 Location 提供请求处理。

```bash
 location / {
     root   html;
    index  index.html index.htm;
    }

  #error_page  404              /404.html;

  # redirect server error pages to the static page /50x.html
  #
  error_page   500 502 503 504  /50x.html;
  location = /50x.html {
      root   html;
  }
  location / {
      #try_files指令将会按照给定的参数顺序进行匹配尝试
      try_files $uri $uri/ /index.html;
  }
```



nginx.conf 详细配置如下：

```bash
#定义Nginx运行的用户和用户组
user www www; 

#nginx进程数，通常设置成和cpu的数量相等
worker_processes 4; 

#全局错误日志定义类型，[debug | info | notice | warn | error | crit]
#error_log  /data/nginx/logs/error.log;
#error_log  /data/nginx/logs/error.log  notice;

#日志文件存放路径 access_log path [format [buffer=size | off]]
access_log /data/nginx/logs/lazyegg.com/web/access.log combinedio;

#进程pid文件
#pid        logs/nginx.pid;

#指定进程可以打开的最大描述符：数目
#工作模式与连接数上限
##这个指令是指当一个nginx进程打开的最多文件描述符数目，理论值应该是最多打开文件数（ulimit -n）与nginx进程数相除，但是nginx分配请求并不是那么均匀，所以最好与ulimit -n 的值保持一致。
#这是因为nginx调度时分配请求到进程并不是那么的均衡，所以假如填写10240，总并发量达到3-4万时就有进程可能超过10240了，这时会返回502错误。
worker_rlimit_nofile 65535;

#################################  events  ###############################
events {
    #参考事件模型，use [ kqueue | rtsig | epoll | /dev/poll | select | poll ]; epoll模型
    use epoll
    #单个进程最大连接数（最大连接数=连接数+进程数）
    worker_connections  1024;

    #keepalive 超时时间
    keepalive_timeout 60;

    #客户端请求头部的缓冲区大小。
    client_header_buffer_size 4k;

    #这个将为打开文件指定缓存，默认是没有启用的，max指定缓存数量，建议和打开文件数一致，inactive是指经过多长时间文件没被请求后删除缓存。
    open_file_cache max=65535 inactive=60s;
    #这个是指多长时间检查一次缓存的有效信息。
    open_file_cache_valid 80s;
        #open_file_cache指令中的inactive参数时间内文件的最少使用次数，如果超过这个数字，文件描述符一直是在缓存中打开的，如上例，如果有一个文件在inactive时间内一次没被使用，它将被移除。
    open_file_cache_min_uses 1;

    #语法:open_file_cache_errors on | off 默认值:open_file_cache_errors off 使用字段:http, server, location 这个指令指定是否在搜索一个文件是记录cache错误.
    open_file_cache_errors on;
}

##############################   http    ##################################

#设定http服务器，利用它的反向代理功能提供负载均衡支持
http{
    #文件扩展名与文件类型映射表
    include mime.types;

    #默认文件类型
    default_type application/octet-stream;

    #默认编码
    charset utf-8;

    #服务器名字的hash表大小
    server_names_hash_bucket_size 128;

    #客户端请求头部的缓冲区大小。
    client_header_buffer_size 32k;

    #客户请求头缓冲大小。
    large_client_header_buffers 4 64k;

    #允许客户端请求的最大单个文件字节数
    client_max_body_size 8m;

    #开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数来输出文件，对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络I/O处理速度，降低系统的负载。注意：如果图片显示不正常把这个改成off。
    sendfile on;

    #开启目录列表访问，适合下载服务器，默认关闭。
    autoindex on;

    #此选项允许或禁止使用socke的TCP_CORK的选项，此选项仅在使用sendfile的时候使用
    tcp_nopush on;

    tcp_nodelay on;

    #长连接超时时间，单位是秒
    keepalive_timeout 120;

    #FastCGI相关参数是为了改善网站的性能：减少资源占用，提高访问速度。下面参数看字面意思都能理解。
    fastcgi_connect_timeout 300;
    fastcgi_send_timeout 300;
    fastcgi_read_timeout 300;
    fastcgi_buffer_size 64k;
    fastcgi_buffers 4 64k;
    fastcgi_busy_buffers_size 128k;
    fastcgi_temp_file_write_size 128k;

    #gzip模块设置
    gzip on; #开启gzip压缩输出
    gzip_min_length 1k;    #最小压缩文件大小
    gzip_buffers 4 16k;    #压缩缓冲区
    gzip_http_version 1.0; #压缩版本（默认1.1，前端如果是squid2.5请使用1.0）
    gzip_comp_level 2;     #压缩等级
    gzip_types text/plain application/x-javascript text/css application/xml;    #压缩类型，默认就已经包含textml，所以下面就不用再写了，写上去也不会有问题，但是会有一个warn。
    gzip_vary on;

    #开启限制IP连接数的时候需要使用
    #limit_zone crawler $binary_remote_addr 10m;

        #负载均衡配置
    upstream lazyegg.net {

        #upstream的负载均衡，weight是权重，可以根据机器配置定义权重。weigth参数表示权值，权值越高被分配到的几率越大。
        server 192.168.80.121:80 weight=3;
        server 192.168.80.122:80 weight=2;
        server 192.168.80.123:80 weight=3;

        #nginx的upstream目前支持4种方式的分配
        #1、轮询（默认）
        #每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。
        #2、weight
        #指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。
        #例如：
        #upstream bakend {
        #    server 192.168.0.14 weight=10;
        #    server 192.168.0.15 weight=10;
        #}
        #2、ip_hash
        #每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。
        #例如：
        #upstream bakend {
        #    ip_hash;
        #    server 192.168.0.14:88;
        #    server 192.168.0.15:80;
        #}
        #3、fair（第三方）
        #按后端服务器的响应时间来分配请求，响应时间短的优先分配。
        #upstream backend {
        #    server server1;
        #    server server2;
        #    fair;
        #}
        #4、url_hash（第三方）
        #按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效。
        #例：在upstream中加入hash语句，server语句中不能写入weight等其他的参数，hash_method是使用的hash算法
        #upstream backend {
        #    server squid1:3128;
        #    server squid2:3128;
        #    hash $request_uri;
        #    hash_method crc32;
        #}

        #tips:
        #upstream bakend{#定义负载均衡设备的Ip及设备状态}{
        #    ip_hash;
        #    server 127.0.0.1:9090 down;
        #    server 127.0.0.1:8080 weight=2;
        #    server 127.0.0.1:6060;
        #    server 127.0.0.1:7070 backup;
        #}
        #在需要使用负载均衡的server中增加 proxy_pass http://bakend/;

        #每个设备的状态设置为:
        #1.down表示单前的server暂时不参与负载
        #2.weight为weight越大，负载的权重就越大。
        #3.max_fails：允许请求失败的次数默认为1.当超过最大次数时，返回proxy_next_upstream模块定义的错误
        #4.fail_timeout:max_fails次失败后，暂停的时间。
        #5.backup： 其它所有的非backup机器down或者忙的时候，请求backup机器。所以这台机器压力会最轻。

        #nginx支持同时设置多组的负载均衡，用来给不用的server来使用。
        #client_body_in_file_only设置为On 可以讲client post过来的数据记录到文件中用来做debug
        #client_body_temp_path设置记录文件的目录 可以设置最多3层目录
        #location对URL进行匹配.可以进行重定向或者进行新的代理 负载均衡
    }

       #虚拟主机的配置
    server {
        #监听端口
        listen 80;

        #域名可以有多个，用空格隔开
        server_name lazyegg.net;
        #默认入口文件名称
        index index.html index.htm index.php;
        root /data/www/lazyegg;

        #对******进行负载均衡
        location ~ .*.(php|php5)?$
        {
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index index.php;
            include fastcgi.conf;
        }

        #图片缓存时间设置
        location ~ .*.(gif|jpg|jpeg|png|bmp|swf)$
        {
            expires 10d;
        }

        #JS和CSS缓存时间设置
        location ~ .*.(js|css)?$
        {
            expires 1h;
        }

        #日志格式设定
        #$remote_addr与$http_x_forwarded_for用以记录客户端的ip地址；
        #$remote_user：用来记录客户端用户名称；
        #$time_local： 用来记录访问时间与时区；
        #$request： 用来记录请求的url与http协议；
        #$status： 用来记录请求状态；成功是200，
        #$body_bytes_sent ：记录发送给客户端文件主体内容大小；
        #$http_referer：用来记录从那个页面链接访问过来的；
        #$http_user_agent：记录客户浏览器的相关信息；
        #通常web服务器放在反向代理的后面，这样就不能获取到客户的IP地址了，通过$remote_add拿到的IP地址是反向代理服务器的iP地址。反向代理服务器在转发请求的http头信息中，可以增加x_forwarded_for信息，用以记录原有客户端的IP地址和原来客户端的请求的服务器地址。
        log_format access '$remote_addr - $remote_user [$time_local] "$request" '
        '$status $body_bytes_sent "$http_referer" '
        '"$http_user_agent" $http_x_forwarded_for';

        #定义本虚拟主机的访问日志
        access_log  /usr/local/nginx/logs/host.access.log  main;
        access_log  /usr/local/nginx/logs/host.access.404.log  log404;

        #对 "/connect-controller" 启用反向代理
        location /connect-controller {
            proxy_pass http://127.0.0.1:88; #请注意此处端口号不能与虚拟主机监听的端口号一样（也就是server监听的端口）
            proxy_redirect off;
            proxy_set_header X-Real-IP $remote_addr;

            #后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            #以下是一些反向代理的配置，可选。
            proxy_set_header Host $host;

            #允许客户端请求的最大单文件字节数
            client_max_body_size 10m;

            #缓冲区代理缓冲用户端请求的最大字节数，
            #如果把它设置为比较大的数值，例如256k，那么，无论使用firefox还是IE浏览器，来提交任意小于256k的图片，都很正常。如果注释该指令，使用默认的client_body_buffer_size设置，也就是操作系统页面大小的两倍，8k或者16k，问题就出现了。
            #无论使用firefox4.0还是IE8.0，提交一个比较大，200k左右的图片，都返回500 Internal Server Error错误
            client_body_buffer_size 128k;

            #表示使nginx阻止HTTP应答代码为400或者更高的应答。
            proxy_intercept_errors on;

            #后端服务器连接的超时时间_发起握手等候响应超时时间
            #nginx跟后端服务器连接超时时间(代理连接超时)
            proxy_connect_timeout 90;

            #后端服务器数据回传时间(代理发送超时)
            #后端服务器数据回传时间_就是在规定时间之内后端服务器必须传完所有的数据
            proxy_send_timeout 90;

            #连接成功后，后端服务器响应时间(代理接收超时)
            #连接成功后_等候后端服务器响应时间_其实已经进入后端的排队之中等候处理（也可以说是后端服务器处理请求的时间）
            proxy_read_timeout 90;

            #设置代理服务器（nginx）保存用户头信息的缓冲区大小
            #设置从被代理服务器读取的第一部分应答的缓冲区大小，通常情况下这部分应答中包含一个小的应答头，默认情况下这个值的大小为指令proxy_buffers中指定的一个缓冲区的大小，不过可以将其设置为更小
            proxy_buffer_size 4k;

            #proxy_buffers缓冲区，网页平均在32k以下的设置
            #设置用于读取应答（来自被代理服务器）的缓冲区数目和大小，默认情况也为分页大小，根据操作系统的不同可能是4k或者8k
            proxy_buffers 4 32k;

            #高负荷下缓冲大小（proxy_buffers*2）
            proxy_busy_buffers_size 64k;

            #设置在写入proxy_temp_path时数据的大小，预防一个工作进程在传递文件时阻塞太长
            #设定缓存文件夹大小，大于这个值，将从upstream服务器传
            proxy_temp_file_write_size 64k;
        }

        #本地动静分离反向代理配置
        #所有jsp的页面均交由tomcat或resin处理
        location ~ .(jsp|jspx|do)?$ {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:8080;
        }
    }
}
```



## Nginx 配置：负载均衡

随着互联网信息的爆炸性增长，负载均衡（Load Balance）已经不再是一个很陌生的话题。

顾名思义，负载均衡即是将负载分摊到不同的服务单元，既保证服务的可用性，又保证响应足够快，给用户很好的体验。

快速增长的访问量和数据流量催生了各式各样的负载均衡产品，很多专业的负载均衡硬件提供了很好的功能，但却价格不菲。

这使得负载均衡软件大受欢迎，Nginx 就是其中的一个，在 Linux 下有 Nginx、LVS、Haproxy 等等服务可以提供负载均衡服务。

Nginx 的负载均衡是 Proxy 模块和 Upstream 模块搭配实现的。Upstream模块将会启用一个新的配置区段，在该区段定义了一组上游服务器。

**实现效果：配置负载均衡。**

①同时启动两个 Tomcat（为了方便验证效果，修改 Tomcat 端口号的同时，顺便将 Tomcat 默认欢迎页面 apache-tomcat-9.0.29/webapps/ROOR 目录下的 index.jsp 修改下，看下 8081 欢迎页的“蛋蛋”没）：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/3.png)

②修改 nginx.conf：

```bash
http {
    upstream myserver {
        server localhost:8080;
        server localhost:8081;
    }
    server {
        listen 80;
        location / {
            proxy_pass http://myserver;
        }
    }
}
```



③重启 Nginx，验证效果（默认轮询的方式，每次打开新窗口，8080 和 8081 会交替出现，同一个窗口的话需要关闭浏览器缓存)。

Nginx 分配策略：

- 轮询（默认） 每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器 Down 掉，能自动剔除。
- Weight 代表权重，默认为 1，权重越高被分配的客户端越多，指定轮询几率，Weight 和访问比率成正比，用于后端服务器性能不均的情况。

例如：

```bash
 upstream server_pool{ 
    server 192.168.5.21 weight=10; 
    server 192.168.5.22 weight=10; } 
```

ip_hash 每个请求按访问 IP 的 Hash 结果分配，这样每个访客固定访问一个后端服务器，可以解决 Session 的问题。 

例如：

```bash
upstream server_pool{
    ip_hash; server 192.168.5.21:80; 
    server 192.168.5.22:80; 
} 
```

Fair（第三方） 按后端服务器的响应时间来分配请求，响应时间短的优先分配。

```bash
upstream server_pool{ 
    server 192.168.5.21:80;
    server 192.168.5.22:80; fair;
 } 
```



## Nginx 配置：动静分离

Nginx 动静分离简单来说就是把动态跟静态请求分开，不能理解成只是单纯的把动态页面和静态页面物理分离。

严格意义上说应该是动态请求跟静态请求分开，可以理解成使用 Nginx 处理静态页面，Tomcat 处理动态页面。

动静分离从目前实现角度来讲大致分为两种：

- 纯粹把静态文件独立成单独的域名，放在独立的服务器上，也是目前主流推崇的方案； 
- 动态跟静态文件混合在一起发布，通过 Nginx 来分开。 

通过 Location 指定不同的后缀名实现不同的请求转发。通过 Expires 参数设置，可以使浏览器缓存过期时间，减少与服务器之前的请求和流量。

**具体 Expires 定义：**是给一个资源设定一个过期时间，也就是说无需去服务端验证，直接通过浏览器自身确认是否过期即可， 所以不会产生额外的流量。

此种方法非常适合不经常变动的资源（如果经常更新的文件， 不建议使用 Expires 来缓存）。

我这里设置 3d，表示在这 3 天之内访问这个 URL，发送一个请求，比对服务器该文件最后更新时间没有变化，则不会从服务器抓取，返回状态码 304，如果有修改，则直接从服务器重新下载，返回状态码 200。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/4.png)

①服务器找个目录存放自己的静态文件：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/12.png)

②修改 nginx.conf：

```bash
    server {
        listen       80;
        server_name  localhost;
        location /static/ {
            root   /usr/data/www;
        }
        location /image/ {
             root /usr/data/;
             autoindex on;
        }
```
③./nginx -s reload，验证效果：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/5.png)

添加监听端口、访问名字重点是添加 Location，最后检查 Nginx 配置是否正确即可，然后测试动静分离是否成功，只需要删除后端 Tomcat 服务器上的某个静态文件，查看是否能访问，如果可以访问说明静态资源 Nginx 直接返回了，不走后端 Tomcat 服务器。

## Nginx 的 Rewrite

Rewrite 是 Nginx 服务器提供的一个重要的功能，它可以实现 URL 重写和重定向功能。

场景如下：

- URL 访问跳转，支持开发设计。页面跳转、兼容性支持（新旧版本更迭）、展示效果（网址精简）等
- SEO 优化（Nginx 伪静态的支持）
- 后台维护、流量转发等
- 安全（动态界面进行伪装）

该指令是通过正则表达式的使用来改变 URI。可以同时存在一个或多个指令。需要按照顺序依次对 URL 进行匹配和处理。

该指令可以在 Server 块或 Location 块中配置，其基本语法结构如下：

```bash
rewrite regex replacement [flag];
```

①采用反向代理 Demo2 中的例子，修改 nginx.conf（只多加了一行 Rewrite）：

```bash
server {
        listen       80;
        server_name  localhost;

        location /java/ {
            proxy_pass http://127.0.0.1:8080;
            rewrite ^/java /egg/ redirect;
        }

        location /egg/ {
            proxy_pass http://127.0.0.1:8081;
        }
}
```

②./nginx -s reload，验证效果（输入 ip/java/ 被重定向到了 egg）：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/6.png)

Rewrite 指令可以在 Server 块或 Location 块中配置，其基本语法结构如下：

```
rewrite regex replacement [flag];
```

- **rewrite 的含义：**该指令是实现 URL 重写的指令。
- **regex 的含义：**用于匹配 URI 的正则表达式。
- **replacement：**将 regex 正则匹配到的内容替换成 replacement。
- **flag：**flag 标记。

flag 有如下值：

- **last：**本条规则匹配完成后，继续向下匹配新的 Location URI 规则。(不常用)
- **break：**本条规则匹配完成即终止，不再匹配后面的任何规则(不常用)。
- **redirect：**返回 302 临时重定向，浏览器地址会显示跳转新的 URL 地址。
- **permanent：**返回 301 永久重定向。浏览器地址会显示跳转新的 URL 地址。

```
rewrite ^/(.*) http://www.360.cn/$1 permanent;
```

## Nginx 高可用

如果将 Web 服务器集群当做一个城池，那么负载均衡服务器就相当于城门。如果“城门”关闭了，与外界的通道就断了。

如果只有一台 Nginx 负载服务器，当故障宕机的时候，就会导致整个网站无法访问。

所以我们需要两台以上 Nginx 来实现故障转移和高可用：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/7.png)

**那么如何配置高可用?**

**①双机热备方案**

这种方案是国内企业中最为普遍的一种高可用方案，双机热备其实就是指一台服务器在提供服务，另一台为某服务的备用状态，当一台服务器不可用另外一台就会顶替上去。

Keepalived 是什么？Keepalived 软件起初是专为 LVS 负载均衡软件设计的，用来管理并监控 LVS 集群系统中各个服务节点的状态。

后来又加入了可以实现高可用的 VRRP (Virtual Router Redundancy Protocol ，虚拟路由器冗余协议）功能。

因此，Keepalived 除了能够管理 LVS 软件外，还可以作为其他服务（例如：Nginx、Haproxy、MySQL 等）的高可用解决方案软件。

**②故障转移机制**

Keepalived 高可用服务之间的故障切换转移，是通过 VRRP 来实现的。

在 Keepalived服务正常工作时，主 Master 节点会不断地向备节点发送（多播的方式）心跳消息，用以告诉备 Backup 节点自己还活着。

当主 Master 节点发生故障时，就无法发送心跳消息，备节点也就因此无法继续检测到来自主  Master 节点的心跳了，于是调用自身的接管程序，接管主 Master 节点的 IP 资源及服务。

而当主 Master节点恢复时，备 Backup 节点又会释放主节点故障时自身接管的 IP 资源及服务，恢复到原来的备用角色。

实现方法如下：

①准备两台安装 Nginx 和 Keepaliver(yum install keepalived -y)的服务器

②修改两台服务器上的 /etc/keepalived/keepalived.conf

```bash
#主机
#检测脚本
vrrp_script chk_http_port {
    script "/usr/local/src/check_nginx.sh" #心跳执行的脚本，检测nginx是否启动
    interval 2                          #（检测脚本执行的间隔，单位是秒）
    weight 2                            #权重
}
#vrrp 实例定义部分
vrrp_instance VI_1 {
    state MASTER            # 指定keepalived的角色，MASTER为主，BACKUP为备
    interface ens33         # 当前进行vrrp通讯的网络接口卡(当前centos的网卡) 用ifconfig查看你具体的网卡
    virtual_router_id 66    # 虚拟路由编号，主从要一直
    priority 100            # 优先级，数值越大，获取处理请求的优先级越高
    advert_int 1            # 检查间隔，默认为1s(vrrp组播周期秒数)
    #授权访问
    authentication {
        auth_type PASS #设置验证类型和密码，MASTER和BACKUP必须使用相同的密码才能正常通信
        auth_pass 1111
    }
    track_script {
        chk_http_port            #（调用检测脚本）
    }
    virtual_ipaddress {
        192.168.16.150            # 定义虚拟ip(VIP)，可多设，每行一个
    }
}
```

```bash
# 备机
#检测脚本
vrrp_script chk_http_port {
    script "/usr/local/src/check_nginx.sh" #心跳执行的脚本，检测nginx是否启动
    interval 2                          #（检测脚本执行的间隔）
    weight 2                            #权重
}
#vrrp 实例定义部分
vrrp_instance VI_1 {
    state BACKUP                        # 指定keepalived的角色，MASTER为主，BACKUP为备
    interface ens33                      # 当前进行vrrp通讯的网络接口卡(当前centos的网卡) 用ifconfig查看你具体的网卡
    virtual_router_id 66                # 虚拟路由编号，主从要一直
    priority 99                         # 优先级，数值越大，获取处理请求的优先级越高
    advert_int 1                        # 检查间隔，默认为1s(vrrp组播周期秒数)
    #授权访问
    authentication {
        auth_type PASS #设置验证类型和密码，MASTER和BACKUP必须使用相同的密码才能正常通信
        auth_pass 1111
    }
    track_script {
        chk_http_port                   #（调用检测脚本）
    }
    virtual_ipaddress {
        192.168.16.150                   # 定义虚拟ip(VIP)，可多设，每行一个
    }
}
```

③新建检测脚本(chmod 775 check_nginx.sh)：

```bash
#!/bin/bash
#检测nginx是否启动了
A=`ps -C nginx --no-header |wc -l`        
if [ $A -eq 0 ];then    #如果nginx没有启动就启动nginx                        
      systemctl start nginx                #重启nginx
      if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then    #nginx重启失败，则停掉keepalived服务，进行VIP转移
              killall keepalived                    
      fi
fi
```
④启动 Nginx 和 Keepalived（systemctl start keepalived.service）

⑤模拟 Nginx 故障（关闭主服务器 Nginx），验证，仍可以通过配置的虚拟 IP 访问，OK。

# Nginx 原理与优化参数配置

Nginx 默认采用多进程工作方式，Nginx 启动后，会运行一个 Master 进程和多个 Worker 进程。

其中 Master 充当整个进程组与用户的交互接口，同时对进程进行监护，管理 Worker 进程来实现重启服务、平滑升级、更换日志文件、配置文件实时生效等功能。

Worker 用来处理基本的网络事件，Worker 之间是平等的，他们共同竞争来处理来自客户端的请求。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/8.png)

**master-workers 的机制的好处：**

- 可以使用 nginx-s reload 热部署。
- 每个 Worker 是独立的进程，不需要加锁，省掉了锁带来的开销。采用独立的进程，可以让互相之间不会影响，一个进程退出后，其他进程还在工作，服务不会中断，Master 进程则很快启动新的 Worker 进程。

**需要设置多少个 Worker？**Nginx 同 Redis 类似都采用了 IO 多路复用机制，每个 Worker 都是一个独立的进程，但每个进程里只有一个主线程，通过异步非阻塞的方式来处理请求，即使是成千上万个请求也不在话下。

每个 Worker 的线程可以把一个 CPU 的性能发挥到极致。所以 Worker 数和服务器的 CPU 数相等是最为适宜的。设少了会浪费 CPU，设多了会造成 CPU 频繁切换上下文带来的损耗。

```bash
#设置 worker 数量。
 worker_processes 4 
#work 绑定 cpu(4 work 绑定 4cpu)。 
 worker_cpu_affinity 0001 0010 0100 1000 
#work 绑定 cpu (4 work 绑定 8cpu 中的 4 个) 。 
 worker_cpu_affinity 0000001 00000010 00000100 00001000
```



**连接数 worker_connection：**这个值是表示每个 Worker 进程所能建立连接的最大值。


所以，一个 Nginx 能建立的最大连接数，应该是 worker_connections*worker_processes。


当然，这里说的是最大连接数，对于 HTTP 请 求 本 地 资 源 来 说 ， 能 够 支 持 的 最 大 并 发 数 量 是 worker_connections*worker_processes，如果是支持 http1.1 的浏览器每次访问要占两个连接。

所以普通的静态访问最大并发数是：worker_connections*worker_processes /2。

而如果是 HTTP 作为反向代理来说，最大并发数量应该是 worker_connections*worker_processes/4。

因为作为反向代理服务器，每个并发会建立与客户端的连接和与后端服务的连接，会占用两个连接。

**Nginx 请求处理流程如下图：**

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/9.png)

#  Nginx 模块开发

由于 Nginx 的模块化特性，所以可以支持模块配置，也可以自定义模块，Nginx 的模块开发，程序员目前还不需要太深入。

Nginx 模块分类如下图：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/10.png)

**Nginx配置选项，解压 Nginx 后的配置操作示例：**

```
./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-pcre  --with-http_ssl_module
```

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-19/11.png)

# Nginx 面试题

①Nginx 功能，你们项目中用到的 Nginx？

- **反向代理服务器**
- **实现负载均衡**
- **做静态资源服务器**
- **作为 HTTP Server**

②Nginx 常用命令有哪些？

```bash
启动nginx    ./sbin/nginx
停止nginx    ./sbin/nginx -s stop   ./sbin/nginx -s quit
重载配置      ./sbin/nginx -s reload(平滑重启) service nginx reload
重载指定配置文件    ./sbin/nginx -c  /usr/local/nginx/conf/nginx.conf
查看nginx版本  ./sbin/nginx -v
检查配置文件是否正确  ./sbin/nginx -t
显示帮助信息  ./sbin/nginx  -h
```

③Nginx 常用配置？

```bash
worker_processes 4;   #工作进程数
work_connections 65535; #每个进程的并发能力
error_log  /data/nginx/logs/error.log;  #错误日志
```

④Nginx 是如何实现高并发的？

Nginx 采用的是多进程（单线程）&多路 IO 复用模型，异步，非阻塞。

一个主进程 Master，多个工作进程 Worker，每个工作进程可以处理多个请求 ，Master 进程主要负责收集、分发请求。

每当一个请求过来时，Master 就拉起一个 Worker 进程负责处理这个请求。同时 Master 进程也负责监控 Woker 的状态，保证高可靠性。

在 Nginx 中的 Work 进程中，为了应对高并发场景，采取了 Reactor 模型（也就是 I/O 多路复用，NIO）。

**I/O 多路复用模型：**在 I/O 多路复用模型中，最重要的系统调用函数就是 Select（其他的还有 epoll 等）。

该方法能够同时监控多个文件描述符的可读可写情况（每一个网络连接其实都对应一个文件描述符），当其中的某些文件描述符可读或者可写时，Select 方法就会返回可读以及可写的文件描述符个数。

Nginx Work 进程使用 I/O 多路复用模块同时监听多个 FD（文件描述符），当 Accept、Read、Write 和 Close 事件产生时，操作系统就会回调 FD 绑定的事件处理器。

这时候 Work 进程再去处理相应事件，而不是阻塞在某个请求连接上等待。

这样就可以实现一个进程同时处理多个连接。每一个 Worker 进程通过 I/O 多路复用处理多个连接请求。

为了减少进程切换（需要系统调用）的性能损耗，一般设置 Worker 进程数量和 CPU 数量一致。

⑤Nginx 和 Apache 的区别？

轻量级，同样起 Web 服务，比 Apache 占用更少的内存及资源抗并发，Nginx 处理请求是异步非阻塞的，而 Apache 则是阻塞型的。

在高并发下 Nginx 能保持低资源低消耗高性能高度模块化的设计，编写模块相对简单，最核心的区别在于 Apache 是同步多进程模型，一个连接对应一个进程；Nginx是异步的，多个连接（万级别）可以对应一个进程。

⑥Nginx 的 Upstream 支持的负载均衡方式？

- **轮询（默认）**
- **weight：**指定权重
- **ip_hash：**每个请求按访问 ip 的 hash 结果分配，这样每个访客固定访问一个后端服务器
- **第三方：**fair、url_hash

⑦Nginx 常见的优化配置有哪些?

- **调整 worker_processes：**指 Nginx 要生成的 Worker 数量，最佳实践是每个 CPU 运行 1 个工作进程。
- **最大化 worker_connections。**
- **启用 Gzip 压缩：**压缩文件大小，减少了客户端 HTTP 的传输带宽，因此提高了页面加载速度。
- **为静态文件启用缓存。**
- **禁用 access_logs：**访问日志记录，它记录每个 Nginx 请求，因此消耗了大量 CPU 资源，从而降低了 Nginx 性能。