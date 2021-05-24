---
title:      " 阿里云搭建shadowsocks-vpn "
date:       2020-01-02
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ 阿里云 ]


---


**脚本可能已不可用，未测试** 2020-5-29


阿里云服务器购买国外节点，建议香港、日本、新加坡

系统选择linux，脚本支持centos、debian、ubuntu

1、下载脚本文件，并执行

```bash
$ wget --no-check-certificate -O shadowsocks-all.sh
https://github.com/ILIKETWICE/shadowsocks-install/blob/master/shadowsocks-all.sh
chmod +x shadowsocks-all.sh
$ ./shadowsocks-all.sh 2>&1 | tee shadowsocks-all.log
```

2、选择脚本（Python、R、Go、libev），任选一个：(这里我选择的是Go)

```bahs
Which Shadowsocks server you'd select:
1.Shadowsocks-Python
2.ShadowsocksR
3.Shadowsocks-Go
4.Shadowsocks-libev
Please enter a number (default 1):3
```

3、我选择的是`Shadowsocks-Go`，输入3......然后，输入密码和端口，笔者直接回车用默认：

```bash
You choose = Shadowsocks-Go
 
Please enter password for Shadowsocks-Go
(default password: teddysun.com):  # 输入你的密码
 
password = teddysun.com
 
Please enter a port for Shadowsocks-Go [1-65535]
(default port: 8989):   # 输入端口
 
port = 8989
 
 
Press any key to start...or Press Ctrl+C to cancel  # 回车
```

4、安装成功后，命令行出现：

```bash
Congratulations, Shadowsocks-Go server install completed!
Your Server IP        :  xx.xx.xx.xx
Your Server Port      :  8989
Your Password         :  teddysun.com
Your Encryption Method:  aes-256-cfb
 
Welcome to visit: https://teddysun.com/486.html
Enjoy it!
```

5、注意阿里云ECS主机要打开“安全组”，添加入口规则