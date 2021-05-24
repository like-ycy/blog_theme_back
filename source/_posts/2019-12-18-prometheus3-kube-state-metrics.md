---
title:      " kube-state-metrics 组件安装 "
date:       2019-12-18
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Prometheus ]

---

在这里 kube-state-metrics 是作为 prometheus 的一个 exporter 来使用，主要用来提供集群中的 deployment、daemonset、cronjob 等服务的监控数据。

yaml文件地址：https://github.com/ILIKETWICE/prometheus-k8s

具体配置清单文件如下：

```bash
├── kube-state-metrics-deploy.yaml
├── kube-state-metrics-rbac.yaml
└── kube-state-metrics-svc.yaml
```

各个配置文件的作用如下：

**kube-state-metrics-rbac.yaml：**该文件主要定义了 kube-state-metrics 容器访问 k8s apiserver 所需的 ServiceAccount、ClusterRole 以及 ClusterRoleBinding；

**prometheus-deploy.yaml**：该文件是 kube-state-metrics 的主要的部署文件，里面定义了一些具体参数，可以根据实际集群的规格做相应的调整；

**prometheus-svc.yaml**：该文件定义了 kube-state-metrics 的 Service，可以根据自己需求做相应的更改，这里只需要使用默认的 ClusterIP 就可以了，因为它只提供给集群内部的 promethes 访问；

配置清单文件准备好后就可以 apply 了：

```bash
[root@master-10 kube-state-metrics]# kubectl apply -f . --dry-run
deployment.apps/kube-state-metrics configured (dry run)
serviceaccount/kube-state-metrics configured (dry run)
clusterrole.rbac.authorization.k8s.io/kube-state-metrics configured (dry run)
role.rbac.authorization.k8s.io/kube-state-metrics-resizer configured (dry run)
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics configured (dry run)
rolebinding.rbac.authorization.k8s.io/kube-state-metrics configured (dry run)
service/kube-state-metrics configured (dry run)
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

需要注意的是上述信息是部署完成各个组件的部署信息，这里只关注 kube-state-metrics-c8b4bcf8-jstlr pod 即可.