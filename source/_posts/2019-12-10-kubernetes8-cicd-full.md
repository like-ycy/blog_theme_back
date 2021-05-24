---
title:      "【Kubernetes系列】第8篇 CI/CD 之全流程实践"
date:       2019-12-10
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Kubernetes ]

---

# 前言

1）本实践中已经的示例代码及jenkins-agent镜像已经推送归档至github，-->传送门（https://github.com/Kubernetes-Best-Pratice/）

2）注意本实践中均为内网数据，你测试时一定要改为自己的环境的有效数据。

3) 由于本实践涉及组件较多，若有操作不明确的话，你可以后台留言，我们一起完善。

4) 具体操作时若有不清楚，或是错误可以留言，大家一起解决。


# 1、准备基础数据

**step1：**配置Gitlab

- 创建项目
- 上传示例代码

*注: 本次示例使用的gitlab项目地址：*

*http://gitlab.hanker.com/colynn/hanker-hello.git*

**step2：**配置Harbor

创建项目, 用于存储构建的镜像

*注: 本次示例使用的harbor地址为* 

*10.0.0.185:5000/hanker/hanker-hello:v1*

**step3：**Jenkins验证信息

- 添加 gitlab 帐号信息

 <u>操作指引：【Credentials】-> 【System】-> 【Global credentials】-> 【Add Credentials】</u>

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-10/1-3-a.png)

- harbor 信息

 <u>操作指引：【Credentials】-> 【System】-> 【Global credentials】-> 【Add Credentials】</u>

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-10/1-3-b.png)

- k8s namespace验证信息

 在你的k8s master节点上执行如下操作：

  1) 创建serviceaccount

```bash
$ kubectl -n devops create serviceaccount jenkins-robot
```

命令输出：

```bash
serviceaccount/jenkins-robot created
```

  2) 角色绑定

```bash
$ kubectl -n devops create rolebinding jenkins-robot-binding --clusterrole=cluster-admin --serviceaccount=devops:jenkins-robot
```

命令输出：

```bash
rolebinding.rbac.authorization.k8s.io/jenkins-robot-binding created
```

3) 获取 ServiceAccount

```
$ kubectl -n devops get serviceaccount jenkins-robot -o go-template --template='{{range .secrets}}{{.name}}{{"\n"}}{{end}}'jenkins-robot-token-n8w6b
```

 4) 基于base64解码 ServiceToken

```
$ kubectl -n devops get secrets jenkins-robot-token-n8w6b -o go-template --template '{{index .data "token"}}' | base64 --decode
```
命令输出：
```
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZXZvcHMiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlY3JldC5uYW1lIjoiamVua2lucy1yb2JvdC10b2tlbi1uOHc2YiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJqZW5raW5zLXJvYm90Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiOTcyZTY0OGYtMTYxZC00NmM5LWI0ZjgtYjFkNTdlOWY4NTBjIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRldm9wczpqZW5raW5zLXJvYm90In0.ArQvcaEqCaeU1ZcJ6nOC5rLaTZr_vLDrpLCt87asltMUWj2gSli_mXUTrl09hBnBDXI3A1D4rJXHKLHjIAA4nN8qRIRGbpqSNzDwmqJr-jmmmWWZFrZ3n3Al9-13KJnNOK8pcWr70rt3Rsigt4B6CIQ0-ZLK8BZhvROJSifeOfJ6xe2KBqdXBv1ccZZZfEhPLgGbaR5yWm5jLvOMr2MQiPDrZoHOEkcMt-C0xipytOp4sJCJ4bQhb-UoMu1owYydxbd6O7xO71fvqP_bMDpZXC601nA2ggK7h-vi6CJffHv5MM59q8X_DWe1NnZS6KXiMmkXqAmBn10Yu20PNj-kjg
```

 5) 添加 Secret text验证信息

<u>操作指引：【首页】->【Credentials】-> 【System】-> 【Global credentials】-> 【Add Credentials】-> 选择【Secret text】类型</u>

然后将上一步 解码的结果 更新至 Secret, Pipeline 中

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-10/1-5.png)

# 2、如何创建Jenkins pipline

**step1：**创建jenkins pipeline item

<u>操作指引：【首页】->【New Item】</u>

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-10/2-1.png)

**step2:** pipeline script 步骤说明

*注: pipeline主要包含三个阶段（检出代码、制作镜像、部署服务），下面跟大家解释下，如何编写pipeline， 借助Pipeline Syntax生成的只是部分代码，你可以根据语言规范将其完善。*

阶段1: 检出代码

<u>操作指引：【首页】->【hanker-hello-demo】-> 【Pipeline Syntax】</u>

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-10/2-2-a.png)

*注: 本实践中选取的 git: Git 类型，当然你也可以选择 checkout: Check out from version control*

获取到该步骤的脚本

```bash
git credentialsId: 'gitlab-project-auth', url: 'http://gitlab.hanker.com/colynn/hanker-hello.git'
```
阶段2：构建建镜像操作指引,类似于阶段1

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-10/2-2-b.png)

完善获取该步骤脚本

```json
script {
    withDockerRegistry(credentialsId: 'harbor-auth', url: 'http://10.0.0.185:5000') {
        def customImage =  docker.build("10.0.0.185:5000/devops/hanker-hello:v1")
        customImage.push()
    }
}
```

*注: 支持本阶段需要jenkins-agent镜像里包含docker命令。*

 阶段3. 部署服务

参考: jenkins kubernetes cli plugin

https://github.com/jenkinsci/kubernetes-cli-plugin/blob/master/README.md

*注: 支持本阶段需要jenkins-agent镜像里包含kubectl命令。*

**step3: **设置 pipeline

*注:General/ Build Triggers/ Advanced Project Options 这三块你可以根据自己需要设置，将各阶段的脚本合并，更新至 Pipline -> Script内。*

合并后的pipeline脚本内容如下：

```json
pipeline {
    agent any
    stages {
        stage('checkout') {
            steps {
                git credentialsId: 'gitlab-project-auth', url: 'http://gitlab.hanker.com/colynn/hanker-hello.git'    
            }
        }

        stage('docker-publish') {
            steps{
                script {
                    withDockerRegistry(credentialsId: 'harbor-auth', url: 'http://10.0.0.185:5000') {
                        def customImage =  docker.build("10.0.0.185:5000/devops/hanker-hello:v1")
                        customImage.push()
                    }
                }
            }
        }

        stage('application-deploy') {
            steps {
                withKubeConfig([credentialsId: '5a5517f3-3d38-459d-bafc-12b55beeb588', serverUrl: 'https://10.0.0.182:6443']) {
                    sh '/usr/bin/kubectl apply -f k8s-setup.yml'
                }
            }
        }
    }
}
```

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-10/2-3.png)

# 3、触发构建

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-10/3-1.png)

# 4、结果确认

1) 确认 jenkina-agent 启动状态；

```bash
$ kubectl -n devops get pods |grep jnlp
jnlp-sh8zl                                 1/1     Running   0          14s
// 查看jenkins-agent pod日志
$ kubectl -n devops logs -f [jenkins-agent-pod-name]
```

*注: 如果长时间没有启动jenkins-agent, 可以确认下集群内是否有足够的资源。*

2) 确认pipeline 执行状态；

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-10/4-2.png)

3) 确认harbor镜像仓库里是否已经有新推送的镜像

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-10/4-3.png)

*注: harbor里的项目是需要你先创建好的，不然推送时会报错。*

4) 确认部署的服务状态

k8s master节点上执行如下操作:

```bash
$ kubectl -n devops get pod,deployment,svc,ingress |grep hanker-hello

pod/hanker-hello-5b7586f86d-5j7kk              1/1     Running   0          173m


deployment.extensions/hanker-hello              1/1     1            1           3h8m
service/hanker-hello-svc          ClusterIP   10.233.22.19    <none>        8080/TCP             3h8m
ingress.extensions/hanker-hello-ingress              hanker-hello-demo.dev.hanker.net                   80      3h8m
```

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-10/4-4.png)

# 5、附录

**1）自定义jenkins-agent镜像**

```bash
## 基于 https://github.com/Kubernetes-Best-Pratice/jenkins-jnlp-agent.git
$ git checkout  https://github.com/Kubernetes-Best-Pratice/jenkins-jnlp-agent.git
$ cd jenkins-jnlp-agent
$ docker build .
$ docker tag tag-name custom-private-repository-addr
```

*注: 你也可以基于基础镜像创建自定义的镜像*

**2）可以做的更完善**

1. 配置webhook, 自动触发jenkins job;
2. 当前我们实践时构建的镜像版本使用的是固定的, 你是否可以将其替换为依赖pipeline环境变量或是传参的形式，将其变是更有意义；
3. 上一篇文章（[点击这里](http://mp.weixin.qq.com/s?__biz=MzU5MTkyNzQ0MQ==&mid=2247483797&idx=1&sn=770557060a5bc4507c2ba9a8d71b1ddb&chksm=fe26c06bc951497d6596a2c9a717bcca04ad130ab8f14097ccd0cc9df9d39bb44adbc4d18768&scene=21#wechat_redirect)进入传送门）中在设置【配置Kubernetes Pod Template】时，我们提到可以挂载主机或是网络共享存储，你是否可以通过这个将你的构建快起来；
4. 我们的示例代码使用的go, 直接是镜像内打包，如何更好的就好的其他语言的构建，你可以参考Using Docker with Pipeline；
5. 你想过如何下载构建过程中的产物吗，等等

**3）参考链接**

1. https://github.com/jenkinsci/kubernetes-cli-plugin/blob/master/README.md

2. 下载kubectl:

   https://docs.docker.com/ee/ucp/user-access/kubectl/