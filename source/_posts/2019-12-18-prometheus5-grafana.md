---
title:      " Grafana 可视化组件安装 "
date:       2019-12-18
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Prometheus ]

---

Grafana 是一个开源的 dashboard，支持用 prometheus 作为数据源，部署起来也比较简单，这里我用的是 ConfigMap 做配置，所以看起来比较复杂。

yaml文件地址：https://github.com/ILIKETWICE/prometheus-k8s

具体配置清单文件如下：

```bash
├── grafana-cm.yaml
├── grafana-dashboard.yaml
├── grafana-deploy.yaml
├── grafana-ingress.yaml
├── grafana-sa.yaml
├── grafana-secret.yaml
└── grafana-svc.yaml
```

文件有点多，其实这样做也是在 Kubernetes 集群部署 Grafana 的比较规范的方式，所有的配置都是通过 ConfigMap 做配置，上述每个文件的具体用途如下：

**grafana-cm.yaml：**该配置文件主要是 grafana 的一些配置文件，包括数据源的配置以及数据保存的位置等等**；**

**grafana-dashboard.yaml：** 该配置文件主要是 grafana 导入的 dashboard 模板，其实这些模板可以在部署完成后自己导入的；

**grafana-deploy.yaml**：该文件主要定义了 Grafana 的部署信息，可以根据自己实际情况做修改，生产环境建议做持久化存储；

**grafana-ingress.yaml**：该文件主要是定义 grafana 服务的 Ingress 的规则，通过 Ingress 暴露服务；

**grafana-sa.yaml**：该文件主要定义了 grafana 在集群中的 ServiceAccount；

**grafana-secret.yaml**：该文件主要定义了登录 Grafana 的默认账号密码，通过挂载到部署文件生效；

**grafana-svc.yaml**：该文件主要定义了 Grafana 的服务；

配置清单文件准备好后就可以 apply 了：

```bash
[root@master-10 grafana]# kubectl apply -f . --dry-run
configmap/grafana-ini configured (dry run)
configmap/grafana-datasources configured (dry run)
configmap/grafana-dashboardproviders configured (dry run)
configmap/dashboards configured (dry run)
deployment.apps/prometheus-grafana configured (dry run)
ingress.extensions/prometheus-grafana-ingress configured (dry run)
serviceaccount/grafana configured (dry run)
secret/grafana configured (dry run)
service/grafana configured (dry run)
```

部署完成后就可以看到对应的 pod 运行起来了：

```bash
[root@master-10 prometheus-server]# kubectl get pods -n prometheus-huang
NAME                                 READY   STATUS    RESTARTS   AGE
alertmanager-59577dc4bf-qcnvw        1/1     Running   0          19h
kube-state-metrics-c8b4bcf8-jstlr    2/2     Running   0          43h
node-exporter-l4228                  1/1     Running   0          42h
node-exporter-qhqf2                  1/1     Running   0          42h
node-exporter-vjtdn                  1/1     Running   0          41h
prometheus-grafana-6d8676fb6-w8cvl   1/1     Running   0          24h
prometheus-server-784595847-kqznb    1/1     Running   0          19h
```

需要注意的是上述信息是部署完成各个组件的部署信息，这里只关注 grafana pod 即可；

部署完成后直接访问就可以，因为之前 ConfigMap 已经做过配置，所以不需要单独配置数据源什么的，默认账号密码如下：

| 账号  | 密码  |
| :---: | :---- |
| admin | admin |

| Kubernetes 版本 | v 1.9 | v 1.10 | v 1.11 | v 1.12 | v 1.13 | v 1.14 | v 1.15 |
| :-------------: | :---: | :----: | :----: | :----: | :----: | :----: | :----: |
|   版本兼容性    |   x   |   ？   |   ？   |   ？   |   ？   |   ？   |   ✓    |


