---
title:      " AlertManager 组件安装 "
date:       2019-12-18
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Prometheus ]

---

在 Prometheus 监控中， AlertManager 主要是用来除了告警信息和发送告警的。

yaml文件地址：https://github.com/ILIKETWICE/prometheus-k8s

具体配置清单文件如下：

```bash
├── alertmanager-cm.yaml
├── alertmanager-deploy.yaml
├── alertmanager-ingress.yaml
├── alertmanager-operated-svc.yaml
├── alertmanager-pvc.yaml
├── alertmanager-rbac.yaml
└── alertmanager-svc.yaml
```

各个配置文件的作用如下：

**alertmanager-cm.yaml：**该配置文件主要是针对 Alertmanager 做的一些基础配置以及一些告警途径，包括邮箱、企业微信以及Slack **；**

**alertmanager-deploy.yaml：** 该配置文件主要定义 alertmanager 的部署文件,真正生产环境需要考虑后端存储的可靠性；

**alertmanager-ingress.yaml**：该文件主要定义了 alertmanager 的 Ingress 规则，可选，非必须；

**alertmanager-operated-svc.yaml**：该文件主要定义**alertmanager-operated服务，服务地址会在部署文件中用到；**

**alertmanager-svc.yaml**：该文件主要用来定义 alertmanager 服务；

**alertmanager-rbac.yaml**：该文件主要定义了部署文件用到的 ServiceAccount；

配置清单文件准备好后就可以 apply 了：

```bash
[root@master-10 alertmanager]# kubectl apply -f . --dry-run
configmap/alertmanager-config configured (dry run)
deployment.apps/alertmanager configured (dry run)
ingress.extensions/prometheus-alert-ingress configured (dry run)
service/alertmanager-operated configured (dry run)
persistentvolumeclaim/alertmanager configured (dry run)
serviceaccount/alertmanager configured (dry run)
service/alertmanager configured (dry run)
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

需要注意的是上述信息是部署完成各个组件的部署信息，这里只关注 alertmanager pod 即可；

到这里基本整个 prometheus 就部署完成了，告警及其规则也配置好了，这里只是配置了微信企业告警，需要其他方式告警可以参考https://aeric.io/post/prometheus-alertmanager-config/