---
title:      " Dockerfile就这么简单 "
date:       2020-07-20 17:43
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Docker ]

---

当我们在使用docker时，最重要的就是镜像，只要有了镜像，我们就可以随时随地的根据镜像来创建一个容器，从而做到让我们的服务可以在任何时间任何地点任何环境下运行起来。那么镜像是怎么制作的呢？总体来讲，制作镜像有两种方法：

1. 根据一个已有的镜像运行容器，然后根据这个容器来制作我们自己的镜像；
2. 使用DockerFile来制作一个镜像模板文件，使用这个文件来创建镜像；

对于第一种方法，我们在上一篇文章中最后有提及，就是利用docker commit命令，将我们的变更打包成一个新的镜像。但是这种方法需要我们每次都运行一个容器，然后在容器中做更改后再打包，很明显这种方式效率很低，而且更改不方便。所以这种方式一般不建议大家采用。我们更多的要使用DockerFile的方式来定制我们的镜像，接下来，我们就详细的介绍一下DockerFile的制作方法。

# 一、利用Dockerfile制作镜像的准备工作

在制作Dockerfile前，我们需要做一系列的准备工作。首先，我们要创建一个目录，用来存储我们的Dockerfile，我们需要打包进镜像中的所有文件也都要放在这个这个目录下，我们制作镜像的时候也要在这个目录下来完成。其次，我们要创建一个文件名为Dockerfile，这个文件必须是大写开头，文件名必须为Dockerfile。当我们编写好我们的Dockerfile文件后，我们需要用docker build命令来执行创建镜像。

# 二、Dockerfile指令

我们准备好相关的目录和文件后，我们就可以开始编写我们的Dockerfile了，Dockerfile其实就是由一些指令组合成的，在Dockerfile中一行就是一条指令，每行开头的第一个单词就是指令本身，指令可以用大写也可以用小写，但是一般我们使用大写来表示指令。

## 1. FROM指令

每一个Dockerfile的第一个非注释行都必须使用FROM指令，这个指令指明了我们制作镜像使用的基础镜像，格式如下：

```
FROM <镜像仓库名>[:tag]
FROM <镜像仓库名>@<镜像哈希值>
```

默认情况下，docker build命令会优先从本地查找我们使用到的基础镜像，如果找不到则自动去我们的镜像仓库中查找。我们在指定基础镜像的过程中可以使用镜像名，但是此时会出现一个问题，如果有人恶意更改了镜像名，用一个错误的镜像替换了我们正常的镜像，那么此时我们就会拉取到错误的镜像。为了避免这个问题的出现，我们可以使用镜像的哈希值来指定基础镜像，就是我们上面提到的使用@符号，这样一来我们使用的镜像就不会被恶意替换掉了。

FROM指令在使用镜像名时，可以省略标签名，默认会使用latest标签。

我们上面说了，每一个Dockerfile的第一个非注释行都必须使用FROM开头，但是ARG指令是唯一一个可以在FROM指令前出现的指令，这是一个例外的情况。

## 2. LABEL指令

LABEL指令用来指定一些元数据的，比如指定这个镜像文件的作者，联系方式，描述信息等等，格式如下：

```
LABEL key1=value1 key2=value2 ... keyN=valueN
```

在docker的早期版本中并没有LABEL指令，而是使用MAINTAINER指令，MAINTAINER指令后面只能跟一个字符串，用来指定作者的信息，在新版的docker中，这个指令已经被弃用，官方推荐使用LABEL指令来实现。

## 3. RUN指令

RUN指令用来在创建镜像过程中执行一些命令，RUN指令有两种格式：

```
RUN <command>      直接跟命令
RUN ["executable", "param1", "param2"]    命令和其参数作为一个列表传入
```

这两种方式有不同的效果，RUN指令后直接跟一个命令，会将此命令运行在一个shell中，在linux中默认是/bin/sh，这也就意味着我们可以在命令字符串中引用一些shell变量。但是在第二种方式中，所有的命令和参数放在了一个列表中传入，此时就无法引用shell中的变量。

除此之外，还有一点需要注意，就是在列表中一定不要用单引号来包裹参数，每个元素都要用双引号，否则会出现docker镜像运行错误的问题。原因就是docker build时会把这些列表当做json来处理，所以要符合json字符串的规则。

RUN指令执行的命令的结果会被打包到镜像当中，而且Dockerfile中后续的指令也可以使用。使用SHELL指令可以改变默认使用的shell。

## 4. CMD指令

CMD指令是用来指定基于我们的镜像创建容器时，容器中运行的命令的，和RUN不同的地方在于，RUN是在构建镜像时执行的命令，CDM是在创建容器时执行的命令。在一个Dockerfile中只可以有一个CDM指令，如果定义了多个CMD指令，那只有最后一个CMD指令会生效。CMD指令使用格式如下：

```
CMD ["executable","param1","param2"] (exec form, this is the preferred form exec格式，这是推荐的格式)
CMD ["param1","param2"] (as default parameters to ENTRYPOINT 为ENTRYPOINT参数提供参数)
CMD command param1 param2 (shell form shell格式的命令)
```

CMD指令可以直接指定一个可执行命令，就是上述的第一和第三种方式，当创建容器时会去执行这个命令，而且需要注意的是，第三种方式是默认在shell中执行的，可以引用shell变量，而第一种方式并不会启动shell，所以就无法引用shell变量。

在采用第二种方式时，此时并没有指定可执行的命令，而是只指定了参数，此时，这些参数将作为ENTRYPOINT指令的参数，关于ENTRYPOINT指令，我们稍后介绍。

## 5. ENTRYPOINT指令

ENTRYPOINT指令也是用来指定基于我们的镜像创建容器时需要执行的命令的，其使用格式如下：

```
ENTRYPOINT ["executable", "param1", "param2"] (exec form, preferred 推荐格式，使用json)
ENTRYPOINT command param1 param2 (shell form shell格式)
```

既然我们已经有了CMD指令了，那为什么我们还要弄一个ENTRYPOINT指令出来呢？这两者的区别在于，当我们使用CMD指令创建好镜像后，在使用这个镜像启动容器时，我们可以改变容器默认的命令，而自己定义启动容器时的命令，比如我们的CMD指令是启动nginx，但是我们在启动容器的时候可以指定命令来启动一个bash，此时，我们在命令行中指定的指令就替换掉了我们的CMD命令。但是我们如果使用ENTRYPOINT指令来指定执行的命令，那么在命令行中启动镜像时，在镜像名之后我们自己指定的命令将不会执行，而是作为参数传递给了ENTRYPOINT命令。而且，在命令行中指定的命令，第一个参数并没有被传递给`ENTRYPOINT`，这是因为我们的docker默认认为第一个参数是要执行的命令，而其之后的才是真正的参数，参见如下所示，我们的“echo” 字符串并没有被输出出来：

```
[root@localhost img1]# cat Dockerfile 
FROM centos:centos7
ENTRYPOINT echo "abc" $@
[root@localhost img1]# docker run --rm centos:testv3 echo aaaaa bbbbb ccccc
abc aaaaa bbbbb ccccc
[root@localhost img1]#
```

这个特性可以使我们在运行容器时禁止自定义启动命令，保证了容器运行结果与我们的预期完全一致。但是，我们并不是完全不能更改这个命令，docker为我们提供了`--entrypoint`参数来修改这个命令。但是这个参数和命令要写在镜像名之前才会生效。

```
[root@localhost img1]# cat Dockerfile 
FROM centos:centos7
ENTRYPOINT echo "abc"
[root@localhost img1]# docker run -ti --rm centos:testv1 --entrypoint /bin/bash
abc
[root@localhost img1]# docker run -ti --rm --entrypoint /bin/bash centos:testv1
[root@a0c502e6ba2f /]# exit
exit
```

在上面CMD命令的部分，我们可以给`CMD`命令不指定执行的命令而只指定参数，此时这些参数就会被传递给ENTRYPOINT指令。

此外，还需要注意一点，我们使用列表的格式来编写命令时，要注意使用双引号来包裹各个参数，而不是单引号。

Shell形式可防止使用任何CMD或`run` 命令行参数覆盖掉我们的运行命令，但具有以下缺点：ENTRYPOINT将作为`/bin/sh -c`的子命令启动，该子命令不传递信号。这意味着可执行文件将不是容器的`PID 1`，并且不会接收Unix信号，因此您的可执行文件将不会从`docker stop <container>`接收到`SIGTERM`。

## 6. EXPOSE指令

EXPOSE指令是用来暴露容器的端口的，其使用格式如下：

```
EXPOSE <port> [<port>/<protocol>...]
```

这个指令可以一次性指定暴露多个端口，且可以指定端口的协议，默认情况下是使用TCP协议，我们还可以自己定义使用的协议：

```
EXPOSE 8080/udp  暴露UDP协议的8080端口
```

但是需要注意的是，在使用了EXPOSE指令后指定的端口，在运行容器时并不会自动的建立容器和宿主机的映射关系，而是当我们运行容器时指定-P选项后其才会将这些端口映射到宿主机上，且我们在定义Dockerfile时不能指定容器端口映射到宿主机上的端口，只能是随机映射一个宿主机上的端口。

## 7. ENV指令

ENV指令用于创建环境变量，这些环境变量可以在构建镜像阶段供Dockerfile之后的指令所引用，其格式如下：

```
ENV <key> <value>
ENV <key>=<value> ...
```

第一种格式用来设置单个的环境变量，ENV指令后被空格分隔的第一个字符串会被当成是环境变量的KEY，后面的所有值都会被当成是该KEY的VALUE值，第二种格式可以一次设置多个环境变量，使用等号来声明KEY和VALUE，如果VALUE部分包含空格，我们可以用引号将VALUE部分引起来，也可以用反斜杠对空格做转义处理。例如：

```
# 第一种格式，一行定义一对环境变量
ENV myName John Doe
ENV myDog Rex The Dog
ENV myCat fluffy

# 第二种方式，一行定义多对环境变量
ENV myName="John Doe" myDog=Rex\ The\ Dog \
    myCat=fluffy
```

通过ENV指令设置的环境变量将被保留在生成的镜像中，我们用此镜像创建容器后，可以用docker inspect 命令来查看，也可以在运行容器时，使用`docker run --env <key>=<value>`的方式来指定。

## 8. ADD指令

ADD指令用来向镜像中添加文件，其有两种格式：

```
ADD [--chown=<user>:<group>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

`--chown`选项可以在添加文件时改变文件的属主和属组，但是需要注意，这个特性只支持Linux类型的容器，在windows容器上不起作用。

ADD指令可以从`<src>`指定的文件、目录或者URL拷贝文件到镜像文件系统中的`<dest>`路径下，并且可以指定多个`<src>`，在有多个`<src>`时，最后一个作为目的地址，其前面的字段都会作为`<src>`字段。同时，在原地址字段中，也支持正则匹配。并且，目的地址是一个绝对路径，或者当`WORKDIR`指令指定了工作目录后，也可以是这个目录下的相对路径。而原文件必须在Dockerfile所在的目录下或其子目录下。

添加包含特殊字符（例如[和]）的文件或目录时，您需要按照Golang规则转义那些路径，以防止将它们视为匹配模式。例如，要添加名为arr [0] .txt的文件，请使用以下命令：

```
ADD arr[[]0].txt /mydir/    # copy a file named "arr[0].txt" to /mydir/
```

如果没有添加`--chown`标志，所有新添加的文件或目录属主属组默认是0。`--chown`标志允许提供属主名和属组名，如果提供了用户名或组名，则将使用容器的根文件系统`/etc/passwd`和`/etc/group`文件分别执行从名称到整数UID或GID的转换，也可以提供其对应的UID和GID，如果只提供了属主，则默认会使用和属主UID相同的GID来指定属组，如下都是正确的定义格式：

```
ADD --chown=55:mygroup files* /somedir/
ADD --chown=bin files* /somedir/
ADD --chown=1 files* /somedir/
ADD --chown=10:11 files* /somedir/
```

如果通过`--chown`标志使用用户名或者属主名来指定属主或属组，而在容器的文件系统中不存在 `/etc/passwd` 或者 `/etc/group` 文件，此时构建镜像时会在ADD操作时失败。但是使用数字来指定时，创建镜像的时候并不会去查找此UID或GID是否存在，也不会依赖容器的根文件系统。需要注意的是，如果源文件是一个URL，而这个URL需要登录认证的话，那么需要使用wget或者curl的方式来下载文件，ADD指令并不能完成登录认证。

**「ADD指令遵循如下的规则：」**

1. 如果是URL，并且不以斜杠结尾，则从URL下载文件并将其复制到;
2. 如果是URL，并且以斜杠结尾，则从URL推断文件名，并将文件下载到/。例如，ADD http://example.com/foobar /，将创建文件 /foobar。该URL必须具有具体的路径及文件名，以便在这种情况下可以找到适当的文件名（例如这样的URL：http://example.com将不起作用）;
3. 如果是目录，则将复制目录的整个内容，包括文件系统元数据。注意，此时目录本身并不会被复制，而是递归复制这个目录下的所有文件;
4. 当是本地的一个通过gzip, bzip2 or xz压缩的tar压缩包，ADD指令会自动将这个包解压。但是如果是一个URL时则不会解压。

> ❝
>
> **「注意」**：文件是否被识别为压缩格式仅根据文件的内容而不是文件的名称来确定。例如，如果一个空文件碰巧以.tar.gz结尾，则该文件将不会被识别为压缩文件，并且不会生成任何类型的解压缩错误消息，而是会将文件简单地复制到目标位置。
>
> ❞

1. 如果是任何其他类型的文件，则将其与其元数据一起单独复制。在这种情况下，如果以尾斜杠/结束，则它将被视为目录，并且的内容将写入/base();
2. 如果直接或由于使用通配符而指定了多个资源，则必须是目录，并且必须以斜杠/结尾;
3. 如果不以斜杠结尾，它将被视为常规文件，并且的内容将写入;
4. 如果不存在，它将与路径中所有缺少的目录一起创建。

## 9. COPY指令

COPY用于向镜像中复制文件，用法与ADD指令类似，但是也有一些区别，其格式如下：

```
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

COPY指令也可以复制多个文件，也支持通配符匹配，用法基本类似ADD指令，但是COPY指令只能接受一个本地文件或目录，不能COPY远程的URL。而且COPY的文件也必须放在Dockerfile同级目录或其同级目录之下的目录中。

**「COPY指令遵循如下规则：」**

1. 如果是目录，则将复制目录的整个内容，包括文件系统元数据。且目录本身不被复制，仅其内容被复制；
2. 如果是任何其他类型的文件，则将其与其元数据一起单独复制。在这种情况下，如果以尾斜杠/结束，则它将被视为目录，并且的内容将写入/base()；
3. 如果直接或由于使用通配符而指定了多个资源，则必须是目录，并且必须以斜杠/结尾；
4. 如果不以斜杠结尾，它将被视为常规文件，并且的内容将写入；
5. 如果不存在，它将与路径中所有缺少的目录一起创建；

## 10. VOLUME指令

VOLUME指令用于挂载宿主机上的目录到容器中，其格式如下：

```
VOLUME ["/data"]
```

VOLUME指令创建具有指定名称的挂载点，并将其标记为保存来自本地主机或其他容器的外部安装的卷。该值可以是JSON数组，`VOLUME ["/var/log/"]` 或具有多个参数的纯字符串，例如`VOLUME /var/log` 或 `VOLUME /var/log/var/db`。在指定挂载点后，docker创建容器时，会把挂载点下已经存在的文件移动到卷中。

关于Dockerfile中的卷，请记住以下几点。

1. 基于Windows的容器上的卷：使用基于Windows的容器时，容器内的卷的目的地必须是以下之一：

   a、不存在的或空目录

   b、C盘以外的磁盘分区

2. 从Dockerfile内更改卷：如果在声明了卷后有任何构建步骤更改了卷内的数据，则这些更改将被丢弃;

3. JSON格式：列表被解析为JSON数组。您必须用双引号（"）而不是单引号（'）括起单词;

4. 主机目录在容器运行时声明：主机目录（挂载点）从本质上说是依赖于主机的。这是为了保留镜像的可移植性，因为不能保证给定的主机目录在所有主机上都可用。因此，您无法从Dockerfile中挂载主机目录。VOLUME指令不支持指定host-dir参数。创建或运行容器时，必须指定挂载点。

## 11. USER指令

USER指令设置运行镜像时要使用的用户名（或UID）以及可选的用户组（或GID），以及Dockerfile中的所有RUN，CMD和ENTRYPOINT指令。其格式如下：

```
USER <user>[:<group>] or
USER <UID>[:<GID>]
```

## 12、WORKDIR指令

WORKDIR指令为Dockerfile中跟在其后的所有RUN，CMD，ENTRYPOINT，COPY和ADD指令设置工作目录。如果WORKDIR不存在，即使以后的Dockerfile指令中未使用，它也将被创建。其格式如下：

```
WORKDIR /path/to/workdir
```

WORKDIR指令可在Dockerfile中多次使用。如果提供了相对路径，则它将相对于上一个WORKDIR指令的路径。例如：

```
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
```

该Dockerfile中最后一个pwd命令的输出为`/a/b/c`。

WORKDIR指令可以解析以前使用ENV设置的环境变量。你只能使用在Dockerfile中显式设置的环境变量。例如：

```
ENV DIRPATH /path
WORKDIR $DIRPATH/$DIRNAME
RUN pwd
```

pwd命令的运行结果就是`/path/$DIRNAME`。

## 13. ARG指令

ARG指令定义了一个变量，用户可以在创建镜像时使用--build-arg=参数将其传递给构建器。如果用户指定了未在Dockerfile中定义的ARG变量，则构建会输出警告。其格式如下：

```
ARG <name>[=<default value>]
```

在Dockerfile中可以包含一个或多个变量。

> ❝
>
> **「注意:」** 不建议使用创建镜像时使用变量来传递诸如github密钥，用户凭据等机密。创建镜像时变量值对于使用docker history命令的镜像的任何用户都是可见的。
>
> ❞

在定义ARG变量时，可以给变量赋初值，如果在创建镜像时没有传入变量值，那么就会使用这个初始值：

```
FROM busybox
ARG user1=someuser
ARG buildno=1
...
```

ARG变量也遵从先定义后使用的惯例，而且，Dockerfile中后定义的同名变量会覆盖之前的变量的值。

可以使用ARG或ENV指令来指定RUN指令可用的变量。使用ENV指令定义的环境变量始终会覆盖同名的ARG指令。我们来看一个例子：

```
FROM ubuntu
ARG CONT_IMG_VER
ENV CONT_IMG_VER v1.0.0
RUN echo $CONT_IMG_VER
```

我们创建镜像时使用如下命令，给`CONT_IMG_VER`传入不同的变量值：

```
$ docker build --build-arg CONT_IMG_VER=v2.0.1 .
```

在这种情况下，RUN指令使用v1.0.0而不是用户传递的ARG设置：v2.0.1，就是因为ENV指令定义的环境变量覆盖了同名的ARG变量。

Docker具有一组预定义的ARG变量，您可以在Dockerfile中使用它们而无需相应的ARG指令:

```
HTTP_PROXY
http_proxy
HTTPS_PROXY
https_proxy
FTP_PROXY
ftp_proxy
NO_PROXY
no_proxy
```

默认情况下，这些预定义变量从Docker历史记录的输出中删除。删除它们可以降低意外泄漏`HTTP_PROXY`变量中的敏感身份验证信息的风险。如果需要在docker历史记录中输出这些默认变量值，则需要我们在Dockerfile中显示的使用ARG指令指定这个变量。

## 14. ONBUILD指令

当我们的镜像被作为基础镜像执行构建时，此时ONBUILD指令就会生效。其格式如下：

```
ONBUILD [INSTRUCTION]
```

运作方式如下：当它遇到`ONBUILD`指令时，构建器将触发器添加到正在构建的镜像的元数据中，该指令不会影响当前版本。构建结束时，所有触发器的列表都存储在镜像清单中的OnBuild键下。可以使用`docker inspect`命令查看它们。稍后，可以使用FROM指令将该镜像用作新构建的基础镜像，作为处理FROM指令的一部分，下游构建器将查找ONBUILD触发器，并以与注册时相同的顺序执行它们。如果任何触发器失败，那么FROM指令将中止，从而导致构建失败。如果所有触发器都成功，则FROM指令完成，并且构建照常继续。执行完触发器后，将从最终镜像中清除触发器。换句话说，它们不是`孙子代`版本所继承的。

> ❝
>
> **「注意」**：在ONBUILD指令中再指定ONBUILD指令是不允许的，ONBUILD指令可能不会触发FROM或者MAINTAINER指令
>
> ❞

## 15. STOPSIGNAL指令

`STOPSIGNAL`指令用来设置系统发送给容器的退出信号，该信号可以是内核syscall表中对应的无符号数字，例如9，也可以是SIGNAME格式的信号名称，例如SIGKILL。

```
STOPSIGNAL signal
```

## 16. HEALTHCHECK指令

HEALTHCHECK指令是用来做容器健康检查的，这个指令是在Docker 1.12版本被加入的，在早期版本中并不支持，这个指令可以让我们自定义容器健康状态检查的脚本或者命令。其格式如下：

```
HEALTHCHECK [OPTIONS] CMD command (check container health by running a command inside the container)
HEALTHCHECK NONE (disable any healthcheck inherited from the base image)
```

我们为什么需要这样一个指令呢？是因为我们的容器是根据启动命令是否运行来判断容器是否健康的，这就导致一个问题，有时我们的应用程序确实在运行，进程并没有退出，但是此时由于bug或其他原因导致程序已经无法正常对外提供服务，那么此时我们就需要用一个命令或者脚本来检测我们的服务，这就是这个指令存在的意义。

**「HEALTHCHECK指令的OPTIONS字段可以有如下几个选项：」**

1. `--interval=DURATION (default: 30s)` 健康检测的命令将在容器启动后的DURATION秒后开始第一次运行，然后每隔DURATION秒运行一次，DURATION默认值是30秒;
2. `--timeout=DURATION (default: 30s)` 健康检测的命令的超时时间，默认30秒;
3. `--start-period=DURATION (default: 0s)` 此选项设置了当容器启动后的DURATION秒后的健康检测如果失败，不计入重试次数，这是为了给容器一个初始化的时间。但是如果这段时间中一旦健康检测为正常，则之后即使在初始化时间内，健康检测如果失败，此时会计入重试次数，默认是0秒;
4. `--retries=N (default: 3)` 健康检测的重试次数，重试N次后容器被判断为异常，则退出进程。

> ❝
>
> **「注意」**：在一个Dockerfile中只能有一个HEALTHCHECK指令，如果指定了多个指令，则只有最后一个指令生效。
>
> ❞

CMD关键字之后的命令可以是shell命令（例如`HEALTHCHECK CMD /bin/check-running`）或exec数组（与其他Dockerfile命令一样;有关详细信息，请参见ENTRYPOINT）。

**「命令的退出状态指示容器的健康状态。可能的值为：」**

```
0：success-容器健康且可以使用
1：unhealthy-容器无法正常工作
2：reserved-请勿使用此退出码
```

为了调试方便，健康检测的输出会被记录到健康状态内，我们可以通过docker inspect命令去查询，但是当前最多只能存储输出的前4096个字节，所以，健康检测的命令要尽可能简洁。

## 17. SHELL指令

SHELL指令允许覆盖用于命令的shell形式的默认shell。在Linux上，默认shell程序是`["/bin/sh","-c"]`，在Windows上，默认shell程序是`["cmd","/S","/C"]`。SHELL指令必须使用JSON形式编写。格式如下：

```
SHELL ["executable", "parameters"]
```

SHELL指令可以有多个，每个SHELL指令都会覆盖之前的设置，并且影响其之后的指令。SHELL指令也是在Docker 1.12版本中加入的，所以在更早期的版本中是不支持的。

## 18. 注意

**「很重要:」**

在我们编写Dockerfile时，每一行指令就会生成一个镜像的层，所以，我们应该尽量将相同的操作都写在同一行中，而且我们依然可以使用`\`来换行，这还是会被当成一层来处理。这样做的好处是可以减小我们的镜像文件的大小，加快容器创建的速度。

# 三、构建镜像

当我们写好了Dockerfile之后，我们就可以使用docker build命令来构建镜像了。命令如下：

```
docker build [OPTIONS] PATH | URL | -
```

在构建镜像时，我们可以添加各种参数来定制镜像，还可以直接为镜像打好标签。docker build命令支持的参数详见docker build命令官方文档，在此不再赘述。