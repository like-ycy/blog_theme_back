---
title:      " Prometheus 部署架构选择 "
date:       2019-12-18
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Prometheus ]

---

本次 Prometheus 实践采用的是 Prometheus Sever + Kube-state-metrics + Grafana 的架构，每一个组件的作用如下:

**Promethues：**提供强大的数据采集、数据存储、数据展示、告警等，天生完美支持 kubernetes，CNCF 基金会的第二个成员，第一个是 Kubernetes。而且 Prometheus 里面很多思想都来源于Google 内部的监控系统 Borgmon，可以说是 Google 的干儿子。

**kube-state-metrics：**在这里作为 prometheus 的一个 exporter 来使用，提供 deployment、daemonset、cronjob 等服务的监控数据，由 kubernestes 官方提供，与 prometheus 紧密结合。因为prometheus 不能获取集群中 Deployment, Job, CronJob 的监控信息。 更多关于 kube-state-metrics 的信息可以参考：https://github.com/kubernetes/kube-state-metrics 

**Grafana**: 开源 dashboard，后端支持多种数据库，如：Influxdb、Prometheus…，插件也比较多，功能强大。非常适合用于做展示。

基本架构图可以参考下图所示：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-18/191218.png)

需要说明的是本次实践是单集群部署，Prometheus 的联邦机制暂且不考虑。本次的单集群规格信息如下：

```bash
[root@master-10 ~]# kubectl get nodes
NAME                    STATUS   ROLES    AGE   VERSION
master-10.200.100.216   Ready    master   7d    v1.15.3
node-10.200.100.214     Ready    <none>   7d    v1.15.3
node-10.200.100.215     Ready    <none>   7d    v1.15.3
```