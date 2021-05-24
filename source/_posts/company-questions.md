---
title:      " 面试的公司及其问题 "
date:       2021-05-24
banner_img: "https://gitee.com/like-ycy/images/raw/master/blog/header.jpg"
tags: [ 面试 ]
---

# 面试的公司及其问题

**之前的问题（公司没记录）**

- k8s service的调度算法
- StatefulSet 可以挂在多个pvc吗？
- prometheus 数据类型
- docker 内部运行free -m看见的内存是宿主机还是容器的，为什么
- 查看容器内进程
- 使用nodeport 这台node没有对应的pod 怎么访问到的
- ingress 转发过程
- pod启动不了，排查思路
- redis集群分片原理
- mysql增量备份原理
- git push -f会发生什么？如何结合git rebase合并分支
- k8s所能识别的最小单元
- 不属于ansible的模块 scp ping copy k8s
- jenkins用户密码存在哪里，数据库还是文本文件
- k8s secret的类型
- mysql优化方法 硬件资源、增加slave、增大binlog保存天数、优化配置文件
- redis适合的场景 session共享（单点登录）、页面缓存、队列、发布/订阅
- nginx支不支持2层负载
- nginx默认监听localhost
- flannel三种模式  udp、vxlan、host-gw
- 查看主机上连接不同mysql的总数
- 负载均衡的实现方式
- docker kill和stop的区别
- 限制某个ip:port组合最多只能允许来自同一个ip的n个并发连接
- 某台机器升级内核无法启动，排查
- nginx日志格式化 log-format，时间、主机、url
- nginx低磁盘IO高并发的优化
- nginx https禁用不安全的密码算法
- 通过nginx重写将url：http://www.abc.com/qi/api 重写为 https://www.abc.com/old_api
- docker build如何传入参数
- cdn是什么，什么情况下使用cdn
- DANT和SANT的区别
- sed grep awk
- pv pvc是什么，区别
- kafka zookeeper
- ansible
- 算法对一组数字做取反。不让用python自带的for循环和语法糖
- mysql存储引擎的数据结构用的啥
- docker 容器间直接数据共享



## 乘法云（龙域中心B座9层）

**笔试部分**

![](https://gitee.com/like-ycy/images/raw/master/interview/chengfayun/%E4%B9%98%E6%B3%95%E4%BA%91%E7%AC%94%E8%AF%951.jpg)

![](https://gitee.com/like-ycy/images/raw/master/interview/chengfayun/%E4%B9%98%E6%B3%95%E4%BA%91%E7%AC%94%E8%AF%952.jpg)



**对话部分**（会问笔试中的题目）

- 阿里云月消费多少
- k8s集群多大，用的什么版本
- 平时shell脚本写的多吗，写过什么脚本
- shell 参数$$ $# $@
- 手写shell脚本，遍历1-100
- 手写python代码，遍历一个元组中的数据，打印出来（面试官主要看的是代码格式是否正确）
- 浏览器访问百度，客户端和服务端之间https证书的流程
- https建立TLS连接之后，会使用什么加密算法对数据进行加密
- docker容器内执行uname -r查看到的内核信息是容器操作系统的还是宿主机操作系统的
- docker容器内执行cat /etc/os-release看到的Linux版本信息是容器操作系统的还是宿主机器的
- 服务器安装docker有没有什么注意事项
- 如何查看mysql的慢查询。优化慢查询的思路是什么？
- 系统内核参数的优化，（说出部分参数即可）
- TCP三次握手，四次挥手
- 系统盘/  30G (已经用满)，数据盘/data 200G（用了30%），找到占用最大的那个文件
- tcpdump 抓取DNS的数据包，什么命令
- 你们公司的发版流程
- gitlab-runner 执行器 shell，docker，k8s的区别
- gitlab的升级，有没有升级过，怎么升级的
- prometheus监控都监控哪些指标
- prometheus获取数据的模式， pull，push两种的区别，第三方exporter的默认方式（默认为pull）
- 生产上有没有遇到什么问题
- k8s node节点显示no reday，然后执行kubectl命令node会有响应吗？
- cdn的数据请求流程，怎么找到的离客户端最近的那个节点？
- k8s  service的四种类型，有什么区别
- k8s 用的什么网络组件，我的回答flannel和calico，然后会问会有什么区别
- 假设基于git仓库开发了一个项目，开修改了10个文件，这时候紧急插入一个需求，请问可以使用什么git命令临时保存修改，去开发新的需求