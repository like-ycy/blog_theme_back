---
title:      "【Kubernetes系列】第3篇 Kubernetes集群安装部署"
date:       2019-12-05
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Kubernetes ]

---

本文介绍了如何通过Kubespray来进行部署高可用k8s集群，k8s版本为1.12.5。

# 1、部署手册

代码仓库：https://github.com/kubernetes-sigs/kubespray

参考文档：https://kubespray.io/#/

# 2、k8s master机器配置

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-05/2.jpg)

# 3、k8s 集群安装步骤

**step1: 设置主机间的免密登录**

由于kubespray是依赖于ansible，ansible通过ssh协议进行主机之间的访问，所以部署之前需要设置主机之间免密登录，步骤如下：

```shell
ssh-keygen -t rsa
scp ~/.ssh/id_rsa.pub root@IP:/root/.ssh
ssh root@IP
cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
```

**step2: 下载kubespray**

注意：不要通过使用github仓库master分支的代码，我这里使用的是tag v2.8.3进行部署

```shell
wget https://github.com/kubernetes-sigs/kubespray/archive/v2.8.3.tar.gz
tar -xvf v2.8.3
cd kubespray-v2.8.3
```

**step3: 配置调整**

## 3.1 更换镜像

Kubernetes安装大部分都是使用的国外的镜像，由于防火墙原因没有办法获取到这些镜像，所以需要自己创建镜像仓库并将这些镜像获取到上传到镜像仓库中。

**3.1.1 新建镜像仓库**

镜像仓库我们选用的组件是Harbor，安装步骤参考：

https://github.com/goharbor/harbor/blob/master/docs/installation_guide.md

**3.1.2 整理k8s集群部署中需要使用的镜像**

在文件roles/download/defaults/main.yml文件中，可以看到使用的全量镜像列表，注意某些镜像由于功能未使用的原因所以暂时没有用到，我们主要用到有如下镜像：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-05/3-1-2.jpg)


**3.1.3 下载所需镜像并上传至私有镜像仓库**

使用的镜像列表如下，在这里我申请了一台国外的阿里云主机，在该台主机下载所需镜像然后上传至私有镜像仓库

例如操作某个镜像时，需要执行如下命令：

```shell
docker pull gcr.io/google_containers/kubernetes-dashboard-amd64:v1.10.0
docker tag gcr.io/google_containers/kubernetes-dashboard-amd64:v1.10.0 106.14.219.69:5000/google_containers/kubernetes-dashboard-amd64:v1.10.0
docker push 106.14.219.69:5000/google_containers/kubernetes-dashboard-amd64:v1.10.0
```

**3.1.4 更改镜像地址并修改Docker配置**

在inventory/testcluster/group_vars/k8s-cluster/k8s-cluster.yml文件中添加如下配置：

```shell
# kubernetes image repo define
kube_image_repo: "10.0.0.183:5000/google_containers"
## modified by: robbin
# comment: 将使⽤的组件的镜像仓库修改为私有镜像仓库地址
etcd_image_repo: "10.0.0.183:5000/coreos/etcd"
coredns_image_repo: "10.0.0.183:5000/coredns"
calicoctl_image_repo: "10.0.0.183:5000/calico/ctl"
calico_node_image_repo: "10.0.0.183:5000/calico/node"
calico_cni_image_repo: "10.0.0.183:5000/calico/cni"
calico_policy_image_repo: "10.0.0.183:5000/calico/kube-controllers"
hyperkube_image_repo: "{{ kube_image_repo }}/hyperkube-{{ image_arch }}"
pod_infra_image_repo: "{{ kube_image_repo }}/pause-{{ image_arch }}"
dnsautoscaler_image_repo: "{{ kube_image_repo }}/cluster-proportional-autoscaler-{{ image_arch }}"
dashboard_image_repo: "{{ kube_image_repo }}/kubernetes-dashboard-{{ image_arch }}"
```

由于我们的私有镜像仓库未配置https证书，需要在 inventory/testcluster/group_vars/all/docker.yml文件中添加如下配置：

```shell
docker_insecure_registries:
- 10.0.0.183:5000
```

## 3.2 Docker安装源更改以及执行文件预处理

**3.2.1 Docker安装源更改**

由于默认从Docker官方源安装docker，速度非常慢，这里我们更换为国内阿里源，在inventory/testcluster/group_vars/k8s-cluster/k8s-cluster.yml文件中添加如下配置：

```shell
# CentOS/RedHat docker-ce repodocker_rh_repo_base_url: 'https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/stable'
docker_rh_repo_gpgkey: 'https://mirrors.aliyun.com/docker-ce/linux/centos/gpg'
dockerproject_rh_repo_base_url: 'https://mirrors.aliyun.com/docker-engine/yum/repo/main/centos/7'
dockerproject_rh_repo_gpgkey: 'https://mirrors.aliyun.com/docker-engine/yum/gpg'
```

**3.2.2 可执行文件预处理**

另外由于需要从google以及github下载一些可执行文件，由于防火墙原因无法直接在服务器上下载，我们可以预先将这些执行文件下载好，然后上传到指定的服务器路径中

可执行文件下载地址可以在roles/download/defaults/main.yml文件中查找到，下载路径如下：

```shell
kubeadm_download_url: "https://storage.googleapis.com/kubernetes-release/release/v1.12.5/bin/linux/amd64/kubeadm"
hyperkube_download_url: "https://storage.googleapis.com/kubernetes-release/release/v1.12.5/bin/linux/amd64/hyperkube"
cni_download_url: "https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz"
```

接下来修改文件权限，并上传到每台服务器的/tmp/releases目录下

```shell
chmod 755 cni-plugins-amd64-v0.6.0.tgz hyperkube kubeadm
scp cni-plugins-amd64-v0.6.0.tgz hyperkube kubeadm root@node1:/tmp/releases
```

## 3.3 组件列表

**k8s所需要的组件**

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-05/3-3-1.png)

**可选插件列表**

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-05/3-3-2.png)

## 3.4 DNS方案

k8s的服务发现依赖于DNS，涉及到两种类型的网络：主机网络和容器网络，所以Kubespray提供了两种配置来进行管理 

**3.4.1 dns_mode**

dns_mode 主要用于集群内的域名解析，有如下几种类型，我们的技术选型是coredns，注意：选择某种dns_mode，可能需要下载安装多个容器镜像，其镜像版本也可能不同

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-05/3-4-1.png)

**3.4.2 resolvconf_mode**

resolvconf_mode主要用来解决当容器部署为host网络模式的时候，如何使用k8s的dns，这里我们使用的是docker_dns

```shell
resolvconf_mode: docker_dns
```

## 3.5 网络插件选择

**3.5.1 kube-proxy**

kube-proxy可以选择ipvs或者iptables，在这里我们选择的是ipvs模式，关于这两者的区别可以参考 华为云在 K8S 大规模场景下的 Service 性能优化实践(https://zhuanlan.zhihu.com/p/37230013)

**3.5.2 网络插件列表**

网络插件列表如下，我们的技术选型是calico，注意：选择某种网络插件，可能需要一个或多个容器镜像，其镜像版本也可能不同

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-05/3-5-2.png)

## 3.6 高可用方案

**step4: 按照如下步骤进行安装部署**

```shell
# Install dependencies from ``requirements.txt``
sudo pip install -r requirements.txt
# Copy `inventory/sample` as `inventory/mycluster`
cp -rfp inventory/sample inventory/mycluster
# Update Ansible inventory file with inventory builder
declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
CONFIG_FILE=inventory/mycluster/hosts.ini python3 contrib/inventory_builder/inventory.py ${IPS[@]}
# Review and change parameters under `inventory/mycluster/group_vars`
cat inventory/mycluster/group_vars/all/all.yml
cat inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
# Deploy Kubespray with Ansible Playbook - run the playbook as root
# The option `-b` is required, as for example writing SSL keys in /etc/,
# installing packages and interacting with various systemd daemons.
# Without -b the playbook will fail to run!
ansible-playbook -i inventory/mycluster/hosts.ini --become --become-user=root cluster.yml
```

部署完成，可以登录到k8s-master所在的主机，执行如下命令，可以看到各个组件正常

```shell
kubectl cluster-info
kubectl get node
kubectl get pods --all-namespaces
```