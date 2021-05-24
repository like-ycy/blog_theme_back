---
title:      " Docker安装sonarqube "
date:       2020-01-06
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Gitlab ]


---

**Sonarqube**，是一种自动代码审查工具，可检测代码中的错误，漏洞和代码异常。它可以与您现有的工作流程集成，以实现跨项目分支和请求请求的连续代码检查。

1、拉取镜像

```bash
docker pull soanrqube 
docker pull postgres
```

2、启动postgres

```bash
docker run -d --name postgresql --restart=always \
-p 5432:5432 \
-e POSTGRES_USER=sonarqube \
-e POSTGRES_PASSWORD=sonarqube \
-e POSTGRES_DB=sonarqube \
postgres
```

POSTGRES_DB：如果未指定此参数，那么第一次启动容器时创建的默认数据库将使用POSTGRES_USER的值

3、启动soanrqube

```bash
docker run -d --name sonarqube --restart=always \
-p 9000:9000 \
-v /opt/sonarqube/conf:/opt/sonarqube/conf \
-v /opt/sonarqube/data:/opt/sonarqube/data \
-v /opt/sonarqube/logs:/opt/sonarqube/logs \
-v /opt/sonarqube/extensions:/opt/sonarqube/extensions \
-e sonar.jdbc.username=sonarqube \
-e sonar.jdbc.password=sonarqube \
-e sonar.jdbc.url=jdbc:postgresql://192.168.1.6:5432/sonarqube \
sonarqube
```

4、浏览器打开192.168.1.6:9000 账号:admin 密码:admin

5、安装中文插件，configuration--market--搜索Chinese Pack

6、安装soanr-scanner（安装在gitlab-runner服务器上，可以搭配gitlab-ci）

[官网下载](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner/)

解压放入/usr/local/目录下

7、项目文件根目录中创建sonar-project.properties 文件

```bash
#项目的key
sonar.projectKey=admin
#sonarqube的主机地址
sonar.host.url=https://192.168.1.6:9000
#项目的名字（这个名字在sonar界面显示的,此处根据项目名称修改）
sonar.projectName=sonar-test
#项目的版本（这个名字在sonar界面显示的,根据项目版本修改）
sonar.projectVersion=1.0
#需要分析源码的目录，多个目录用英文逗号隔开（根据需要修改）
sonar.sources=./
#编码格式
#sonar.sourceEncoding=UTF-8
#项目所用语言
sonar.language=java
#登录账号
sonar.login=admin
#登录密码
sonar.password=admin
#包含与源文件对应的已编译字节码文件 (如果没编译可以不写,默认创建target/sonar目录)
sonar.java.binaries=target/sonar
```

8、cd 到项目文件中，执行/usr/local/sonar-scanner/bin/sonar-scanner

9、再次进入web界面，可以看到分析结果