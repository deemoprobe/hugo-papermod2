---
title: "DockerfileGrammer"
date: 2023-04-26T14:52:37+08:00
lastmod: 2023-04-26T14:52:37+08:00
author: ["deemoprobe"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- docker
description: "Dockerfile语法，构建以及实例分析"
weight:
slug: ""
draft: false # 是否为草稿
comments: false # 本页面是否显示评论
reward: false # 打赏
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false #顶部显示路径
cover:
    image: "img/docker.png" #图片路径例如：posts/tech/123/123.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

## 字段解析

![dockerfs](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/dockerfs.png)

### 简介

```bash
# 主要字段
FROM image[基础镜像,该文件创建新镜像所依赖的镜像]
MAINTAINER user<email>[作者姓名和邮箱]
RUN command[镜像构建时运行的命令]
ADD [文件拷贝进镜像并解压]
COPY [文件拷贝进镜像]
CMD [容器启动时要运行的命令或参数]
ENTRYPOINT [容器启动时要运行的命令]
EXPOSE port[声明端口]
WORKDIR work_directory[进入容器默认进入的目录]
ENV set_env[创建环境变量]
VOLUME [容器数据卷,用于数据保存和持久化]
ONBUILD [当前Dockerfile构建时不会调用，当子镜像依赖本镜像（FROM）构建时触发ONBUILD后的命令]
USER [指定构建镜像和运行容器的用户用户组]
ARG [构建镜像时设定的变量]
LABEL [为镜像添加元数据]
```

- FROM 基础镜像

```bash
# 如果不指定版本，默认使用latest
FROM image
FROM image:tag
FROM image@digest

# 示例
FROM nginx:1.18.0
```

- MAINTAINER 作者

```bash
MAINTAINER user
MAINTAINER email
MAINTAINER user<email>

# 示例
MAINTAINER deemo<deemo@gmail.com>
```

- RUN 构建镜像时执行的命令

```bash
# Dockerfile里的指令每执行一次会在镜像文件系统中新建一层，为了避免多层文件造成镜像过大，多条命令写在一个RUN后面
# RUN指令创建的中间镜像会被缓存，并会在下次构建中使用。如果不想使用这些缓存镜像，可以在构建时指定--no-cache参数，如：docker build --no-cache
RUN command
RUN ["<executable>","<param1>","<param2>",...]

# 示例
RUN yum install -y curl
RUN ["./test.php","dev","offline"]  #等价于 RUN ./test.php dev offline
```

- ADD 本地文件拷贝进镜像，tar类型的会自动解压

```bash
ADD <src> <dest>

# 示例
ADD file /dir/ #添加file到/dir/目录
ADD file dir/ #添加file到{WORKDIR}/dir目录
ADD fi* /dir #通配符，添加所有以fi开头的文件到/dir/目录
```

- COPY 本地文件拷贝进镜像，但不会解压

```bash
COPY <src> <dest>
```

- CMD 容器启动时（docker run时）要运行的命令或参数

```bash
# 可以设置多个CMD,但最后一个生效,前面的不生效,也可以被docker run启动容器时后面加的命令替换
CMD ["<executable>","<param1>","<param2>",...] #执行可执行文件
CMD ["<param1>","<param2>",...] #已设置ENTRYPOINT，则调用ENTRYPOINT后添加CMD参数
CMD command param1 param2 ... #执行shell内部命令

# 示例
CMD ["/usr/bin/ls","-al"]
CMD echo "hello"
```

- ENTRYPOINT 容器启动时要运行的命令

```bash
# 类似于CMD指令，但其不会被docker run的命令行参数指定的指令所覆盖
# 存在多个ENTRYPOINT时，仅最后一个生效
ENTRYPOINT ["<executeable>","<param1>","<param2>",...]

# 示例
ENTRYPOINT ["nginx", "-c"] #定参
CMD ["/etc/nginx/nginx.conf"] #变参
```

- EXPOSE 声明容器端口

```bash
# EXPOSE仅是声明端口。要使其可访问，需要在docker run运行容器时通过-p来指定端口映射，或通过-P参数来映射EXPOSE端口
EXPOSE <port> [<port>...]

# 示例
EXPOSE 80
EXPOSE 80 443
```

- WORKDIR 工作目录

```bash
WORKDIR path
```

- ENV 设置环境变量

```bash
ENV <key> <value>
ENV <key>=<value> ...

# 示例
ENV dir=/webapp
ENV name deemoprobe
```

- VOLUME 容器数据卷,用于数据保存和持久化

```bash
VOLUME ["/path/to/dir"]

# 示例
VOLUME ["/data"]
```

- USER 指定构建镜像和运行容器的用户用户组

```bash
USER user
USER user:group
USER uid
USER uid:gid
USER user:gid
USER uid:group

# 示例
USER www
USER 1080:tomcat
```

- ARG 构建镜像时设定的变量

```bash
# 与 ENV 作用一致。不过作用域不一样。ARG 设置的环境变量仅对 Dockerfile 内有效，也就是说只有 docker build 的过程中有效，构建好的镜像内不存在此环境变量。
ARG <name>[=<default value>]

# 示例
ARG user=www
```

- ONBUILD 当构建一个被继承的Dockerfile时运行命令

```bash
# 子镜像构建时触发命令并执行。就是Dockerfile里用ONBUILD指定的命令，在本次构建镜像（假设镜像名为test）的过程中不会执行。当有新的Dockerfile使用了该镜像（FROM test），这时执行新镜像的Dockerfile构建时候，会执行test镜像中Dockerfile里的ONBUILD指定的命令。
ONBUILD [INSTRUCTION]

# 示例
ONBUILD RUN yum install wget
ONBUILD ADD . /data
```

- LABEL 为镜像添加元数据

```bash
LABEL <key>=<value> <key>=<value> <key>=<value> ...

# 示例
LABEL version="1.0" des="webapp"
```

## 实例1-简单尝试

```bash
[root@demo ~]# docker build --help

Usage:  docker build [OPTIONS] PATH | URL | -

Build an image from a Dockerfile

Options:
      --add-host list           Add a custom host-to-IP mapping (host:ip)
      --build-arg list          Set build-time variables
      --cache-from strings      Images to consider as cache sources
      --cgroup-parent string    Optional parent cgroup for the container
      --compress                Compress the build context using gzip
      --cpu-period int          Limit the CPU CFS (Completely Fair Scheduler) period
      --cpu-quota int           Limit the CPU CFS (Completely Fair Scheduler) quota
  -c, --cpu-shares int          CPU shares (relative weight)
      --cpuset-cpus string      CPUs in which to allow execution (0-3, 0,1)
      --cpuset-mems string      MEMs in which to allow execution (0-3, 0,1)
      --disable-content-trust   Skip image verification (default true)
  -f, --file string             Name of the Dockerfile (Default is 'PATH/Dockerfile')
      --force-rm                Always remove intermediate containers
      --iidfile string          Write the image ID to the file
      --isolation string        Container isolation technology
      --label list              Set metadata for an image
  -m, --memory bytes            Memory limit
      --memory-swap bytes       Swap limit equal to memory plus swap: '-1' to enable unlimited swap
      --network string          Set the networking mode for the RUN instructions during build (default "default")
      --no-cache                Do not use cache when building the image
      --pull                    Always attempt to pull a newer version of the image
  -q, --quiet                   Suppress the build output and print image ID on success
      --rm                      Remove intermediate containers after a successful build (default true)
      --security-opt strings    Security options
      --shm-size bytes          Size of /dev/shm
  -t, --tag list                Name and optionally a tag in the 'name:tag' format
      --target string           Set the target build stage to build.
      --ulimit ulimit           Ulimit options (default [])
# 创建Dockerfile
[root@demo ~]# cat Dockerfile
FROM centos:centos7.9.2009 # 该镜像已经提前拉取
MAINTAINER deemo<deemo@gmail.com>
RUN curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo && sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo && yum install vim -y
CMD ["/bin/echo","hello"]
# 对比父镜像和新镜像，有sed和curl没有vim
[root@demo ~]# docker run -it --rm centos:centos7.9.2009 whereis sed
sed: /usr/bin/sed
[root@demo ~]# docker run -it --rm centos:centos7.9.2009 whereis curl
curl: /usr/bin/curl
[root@demo ~]# docker run -it --rm centos:centos7.9.2009 whereis vim
# 构建镜像
[root@demo ~]# docker build -t centos7:v2 .
[+] Building 76.1s (6/6) FINISHED
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 408B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/centos:centos7.9.2009                                                            0.0s
 => CACHED [1/2] FROM docker.io/library/centos:centos7.9.2009                                                                       0.0s
 => [2/2] RUN curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo && sed -i -e /mirrors.clou   72.2s
 => exporting to image                                                                                                              3.8s
 => => exporting layers                                                                                                             3.8s
 => => writing image sha256:30f266dbdecb307db7ee31297d333ae19a88f4add15bfd52f17bc6acfeea13f8                                        0.0s
 => => naming to docker.io/library/centos7:v2 
# 查看结果
[root@demo ~]# docker images | grep v2
centos7                     v2               30f266dbdecb   3 minutes ago    463MB
# 新镜像安装了vim，查看
[root@demo ~]# docker run -it --rm centos7:v2 whereis vim
vim: /usr/bin/vim /usr/share/vim
# 基于新镜像运行容器输出了hello
[root@demo ~]# docker run -it centos7:v2
hello
```

## 实例2-构建Tomcat

```bash
# 准备Dockerfile
[root@demo ~]# vim Dockerfile 
# 基础镜像centos:centos7.9.2009
FROM centos:centos7.9.2009
# 作者签名
MAINTAINER deemoprobe<deemoprobe@gmail.com>
# 拷贝宿主机当前目录下文件
COPY tomcat.txt /usr/local/tomcat8.txt
# 添加Tomcat安装包并解压至/usr/local
ADD apache-tomcat-8.5.53.tar.gz /usr/local
# 添加jdk安装包并解压至/usr/local
ADD jdk-8u271-linux-x64.tar.gz /usr/local
# 安装vim
RUN curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo && sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo && yum install vim -y
# 设置环境变量
ENV MYPATH /usr/local
# 指定工作目录，使用ENV设定的环境变量
WORKDIR $MYPATH
# 配置JDK环境
ENV JAVA_HOME /usr/local/jdk1.8.0_271
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/apache-tomcat-8.5.53
ENV CATALINA_BASE /usr/local/apache-tomcat-8.5.53
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
# 声明端口8080
EXPOSE 8080
# 启动
# ENTRYPOINT [ "/usr/local/apache-tomcat-8.5.53/bin/startup.sh" ]
# CMD [ "/usr/local/apache-tomcat-8.5.53/bin/catalina.sh", "run" ]
CMD /usr/local/apache-tomcat-8.5.53/bin/startup.sh && tail -F /usr/local/apache-tomcat-8.5.53/bin/logs/catalina.out

# 准备必要的文件到当前目录
[root@demo ~]# echo "tomcat" >> tomcat.txt
# 上传Tomcat和jdk安装包
[root@demo ~]# ls
apache-tomcat-8.5.53.tar.gz  Dockerfile  jdk-8u271-linux-x64.tar.gz  tomcat.txt

# 构建镜像
[root@demo ~]# docker build -t tomcat8:v1 .
[+] Building 50.1s (9/10)
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 1.20kB                                                                                              0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/centos:centos7.9.2009                                                            0.0s
 => CACHED [1/6] FROM docker.io/library/centos:centos7.9.2009                                                                       0.0s
 => [internal] load build context                                                                                                   4.0s
 => => transferring context: 153.48MB                                                                                               4.0s
 => [2/6] COPY tomcat.txt /usr/local/tomcat8.txt                                                                                    0.3s
 => [3/6] ADD apache-tomcat-8.5.53.tar.gz /usr/local                                                                                1.2s
 => [4/6] ADD jdk-8u271-linux-x64.tar.gz /usr/local                                                                                 7.6s
 => [5/6] RUN curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo && sed -i -e /mirrors.clou   48.0s
 => [6/6] WORKDIR /usr/local                                                                                                        0.0s
 => exporting to image                                                                                                              4.9s
 => => exporting layers                                                                                                             4.9s
 => => writing image sha256:dbccbeed821449bb611ff35b3071547035d925775a8b9d566e888ef6f895c3cf                                        0.0s
 => => naming to docker.io/library/tomcat8:v1                                                                                       0.0s
# 查看结果
[root@demo ~]# docker images | grep tomcat
tomcat8                     v1               dbccbeed8214   About a minute ago   833MB

# 运行一个容器
# 如果有读写权限问题可以加上--privileged=true
[root@demo ~]# docker run -d -p 1080:8080 --name myweb -v /root/web:/usr/local/apache-tomcat-8.5.53/webapps/web -v /root/tomcatlog:/usr/local/apache-tomcat-8.5.53/logs --privileged=true tomcat8:v1
d5f63d3513ba696e54fd7353e52d15a7c2582101d1c71067028c6251f2d82bef
[root@demo ~]# docker ps | grep tomcat
d5f63d3513ba   tomcat8:v1                "/bin/sh -c '/usr/lo…"   17 seconds ago   Up 15 seconds   0.0.0.0:1080->8080/tcp, :::1080->8080/tcp   myweb
# 访问Tomcat首页，直接 curl localhost:1080 可以看到返回Tomcat首页的HTML源码
[root@demo ~]# curl -I localhost:1080
HTTP/1.1 200 
Content-Type: text/html;charset=UTF-8
Transfer-Encoding: chunked
Date: Mon, 10 Jan 2022 10:38:12 GMT
# 查看WORKDIR
[root@demo ~]# docker exec d5f63d3513ba pwd
/usr/local
# 查看JDK版本
[root@demo ~]# docker exec d5f63d3513ba java -version
java version "1.8.0_271"
Java(TM) SE Runtime Environment (build 1.8.0_271-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.271-b09, mixed mode)

# 创建web项目，发布服务
[root@demo ~]# ls -l
total 149860
-rw-r--r--. 1 root root  10300600 Jan 10  2022 apache-tomcat-8.5.53.tar.gz
-rw-r--r--. 1 root root      1073 Jan 10 18:04 Dockerfile
-rw-r--r--. 1 root root 143142634 Jan 10  2022 jdk-8u271-linux-x64.tar.gz
drwxr-xr-x. 2 root root       197 Jan 10 18:36 tomcatlog
-rw-r--r--. 1 root root         7 Jan 10 18:06 tomcat.txt
drwxr-xr-x. 2 root root         6 Jan 10 18:36 web
[root@demo ~]# cd web
[root@demo web]# vim web.jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>Insert title here</title>
  </head>
  <body>
    -----------welcome------------
    <%="I am in docker tomcat8"%>
    <br>
    <br>
    <% System.out.println("=============docker tomcat8");%>
  </body>
</html>
[root@demo web]# mkdir WEB-INF
[root@demo web]# cd WEB-INF/
[root@demo WEB-INF]# vim web.xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns="http://java.sun.com/xml/ns/javaee"
  xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
  id="WebApp_ID" version="2.5">

  <display-name>test-tomcat8</display-name>
</web-app>
# 查看项目结构
[root@demo ~]# yum install tree -y
[root@demo ~]# tree
.
├── apache-tomcat-8.5.53.tar.gz
├── Dockerfile
├── jdk-8u271-linux-x64.tar.gz
├── tomcatlog
│   ├── catalina.2022-01-10.log
│   ├── catalina.out
│   ├── host-manager.2022-01-10.log
│   ├── localhost.2022-01-10.log
│   ├── localhost_access_log.2022-01-10.txt
│   └── manager.2022-01-10.log
├── tomcat.txt
└── web
    ├── WEB-INF
    │   └── web.xml
    └── web.jsp
3 directories, 12 files
# 查看容器内数据卷同步结果
[root@centos7 WEB-INF]# docker ps | grep tomcat
d5f63d3513ba   tomcat8:v1                "/bin/sh -c '/usr/lo…"   5 minutes ago   Up 4 minutes   0.0.0.0:1080->8080/tcp, :::1080->8080/tcp   myweb
[root@demo ~]# docker exec d5f63d3513ba ls -l /usr/local/apache-tomcat-8.5.53/webapps/web
total 4
drwxr-xr-x. 2 root root  21 Jan 10 10:49 WEB-INF
-rw-r--r--. 1 root root 500 Jan 10 10:48 web.jsp
[root@demo ~]# docker exec d5f63d3513ba ls -l /usr/local/apache-tomcat-8.5.53/logs
total 24
-rw-r-----. 1 root root 7173 Jan 10 10:49 catalina.2022-01-10.log
-rw-r-----. 1 root root 7173 Jan 10 10:49 catalina.out
-rw-r-----. 1 root root    0 Jan 10 10:36 host-manager.2022-01-10.log
-rw-r-----. 1 root root  459 Jan 10 10:36 localhost.2022-01-10.log
-rw-r-----. 1 root root  281 Jan 10 10:40 localhost_access_log.2022-01-10.txt
-rw-r-----. 1 root root    0 Jan 10 10:36 manager.2022-01-10.log
# 重启一下容器
[root@demo ~]# docker restart d5f63d3513ba

# 访问结果
[root@demo ~]# curl localhost:1080/web/web.jsp

<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>Insert title here</title>
  </head>
  <body>
    -----------welcome------------
    I am in docker tomcat8
    <br>
    <br>
    
  </body>
</html>

# 查看日志，可以看到访问记录，其他日志文件可以看到Tomcat启动记录等
[root@demo ~]# cd tomcatlog/
[root@demo tomcatlog]# cat localhost_access_log.2022-01-10.txt
172.17.0.1 - - [10/Jan/2022:10:38:12 +0000] "HEAD / HTTP/1.1" 200 -
172.17.0.1 - - [10/Jan/2022:10:38:21 +0000] "GET / HTTP/1.1" 200 11215
172.17.0.1 - - [10/Jan/2022:10:39:49 +0000] "GET / HTTP/1.1" 200 11215
172.17.0.1 - - [10/Jan/2022:10:39:56 +0000] "GET / HTTP/1.1" 200 11215
172.17.0.1 - - [10/Jan/2022:10:58:16 +0000] "GET /web/web.jsp HTTP/1.1" 200 352
```

> 参考文档：[Docker官方镜像库示例](https://github.com/docker-library)
