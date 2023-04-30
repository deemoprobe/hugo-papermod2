---
title: "DockerbuildContext"
date: 2023-04-26T18:35:27+08:00
lastmod: 2023-04-26T18:35:27+08:00
author: ["deemoprobe"]
keywords: 
- docker
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- docker
description: "本文包含docker build上下文、构建方式和实例"
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
showbreadcrumbs: false #顶部显示文章发布路径
cover:
    image: "img/docker.png" #图片路径例如：posts/tech/123/123.png
    zoom:  # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---


## Docker架构

![20230426185353](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20230426185353.png)

Docker是一种C/S架构，`Docker Client`通过Docker提供的REST API与守护进程`Docker Daemon`（dockerd）进行通信，Docker客户端和守护进程可以在同一个系统上运行，也可以将Docker客户端连接到远程Docker守护进程。守护进程负责镜像的构建以及docker容器的运行和分发，守护进程在构建镜像过程中获取本地文件时就需要根据指定的上下文路径来查找（也就是本文的重点：context，后面有实例进行分析）。

![20230426184804](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20230426184804.png)

- `Docker Daemon` Docker守护进程dockerd，用来监听Docker API的请求和管理Docker对象：镜像、容器、网络和Volume
- `Docker Client` Docker客户端，执行docker命令与`Docker Daemon`（dockerd）进行交互
- `Docker Registry` Docker镜像仓库，Docker默认从公共仓库Docker Hub上查找镜像，当然也可按需搭建私有仓库（如Harbor），客户端使用`docker pull`或`docker run`命令时，如果本地未曾拉取相关镜像，就会从配置的Docker镜像仓库中去拉取镜像，使用`docker push`命令时，会将构建的镜像推送到对应的镜像仓库中
- `Images` Docker镜像，镜像是一个制度模板，带有Docker容器的说明
- `Containers` Docker容器，容器是镜像的可运行实例，可以使用Docker REST API或者CLI来操作容器，容器的实质是进程，容器进程拥有独立的命名空间，如rootfs、网络配置、进程空间、用户ID。容器运行在一个隔离的环境里，这种特性使得容器封装的应用比传统进程更加安全

## docker build构建方式

docker build [选项] [上下文路径|URL|-]

- 常用选项：`-t test:v1`为构建的镜像打标签

构建方式（以`-t test:v1`标签为例）：

- 1. `docker build -t test:v1 .` 此时镜像构建文件就在当前目录且名称为`Dockerfile`
- 2. `docker build -t test:v1 -f /Path/To/Dockerfile [context]` 此时Dockerfile可为任意名称，只要内容格式符合Dockerfile标准即可
- 3. `docker build -t test:v1 [Git Repository]` 此时根据git仓库写好的逻辑进行构建
- 4. `docker build -t test:v1 [URL/test.tar.gz]` 根据URL中压缩包进行构建，dockerd自动解压压缩包并以压缩包内容为上下文构建镜像
- 5. `docker build -t test:v1 - < Dockerfile` 不推荐，从标准输入构建，无法指定上下文，无法进行COPY之类操作
- 6. `docker build -t test:v1 - < [test.tar.gz]` 从标准输入的压缩包中构建镜像，dockerd自动解压压缩包并以压缩包内容为上下文构建镜像

```bash
# 先把基础镜像拉下来
[root@centos7 tmp]# docker pull alpine
# 示例1
[root@centos7 tmp]# cat Dockerfile
FROM alpine
COPY ./resource.tar /
RUN echo "test"
[root@centos7 tmp]# docker build -t test:v1 .
[+] Building 0.8s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 147B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [internal] load build context                                                                                                   0.2s
 => => transferring context: 10.30MB                                                                                                0.2s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => [2/3] COPY ./resource.tar /                                                                                                     0.1s
 => [3/3] RUN echo "test"                                                                                                           0.3s
 => exporting to image                                                                                                              0.2s
 => => exporting layers                                                                                                             0.2s
 => => writing image sha256:db32566b6ed4e3864f6229709631b8c7eb9b8a29d7c71de7dc7d2fb9046524f2                                        0.0s
 => => naming to docker.io/library/test:v1                                                                                          0.0s
[root@centos7 tmp]# docker images | grep v1
test         v1        db32566b6ed4   36 seconds ago   15.9MB

# 示例2
[root@centos7 tmp]# docker build -t test:v2 -f /root/tmp/Dockerfile .
[+] Building 0.1s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 147B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [internal] load build context                                                                                                   0.0s
 => => transferring context: 96B                                                                                                    0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => CACHED [2/3] COPY ./resource.tar /                                                                                              0.0s
 => CACHED [3/3] RUN echo "test"                                                                                                    0.0s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:db32566b6ed4e3864f6229709631b8c7eb9b8a29d7c71de7dc7d2fb9046524f2                                        0.0s
 => => naming to docker.io/library/test:v2                                                                                          0.0s
[root@centos7 tmp]# docker images | grep v2
test         v2        db32566b6ed4   2 minutes ago   15.9MB

# 示例3，类似下方，但似乎无法拉取，不是重点，略过
[root@centos7 tmp]# docker build -t hello-world https://github.com/docker-library/hello-world.git#master:amd64/hello-world

# 示例4
[root@centos7 tmp]# tar -zcvf test.tar.gz resource.tar Dockerfile
resource.tar
Dockerfile
# 上传到我的阿里OSS存储后，使用带压缩包的URL构建
[root@centos7 tmp]# docker build -t test:v4 https://deemoprobe.oss-cn-shanghai.aliyuncs.com/repo/test.tar.gz
[+] Building 5.4s (7/7) FINISHED
 => [internal] load remote build context                                                                                            5.0s
 => copy /context /                                                                                                                 0.2s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => CACHED [2/3] COPY ./resource.tar /                                                                                              0.0s
 => CACHED [3/3] RUN echo "test"                                                                                                    0.0s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:8f64891c181eb4171bfcc54ea3d9dd5342a51830aee3d68653e93e14d3cefdca                                        0.0s
 => => naming to docker.io/library/test:v4                                                                                          0.0s
[root@centos7 tmp]# docker images | grep v4
test         v4        8f64891c181e   32 minutes ago   15.9MB

# 示例5，由于无法指定上下文，造成COPY失败，无法构建
[root@centos7 tmp]# docker build -t test:v5 - < Dockerfile
[+] Building 0.1s (6/7)
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 158B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [internal] load build context                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => ERROR [2/3] COPY ./resource.tar /                                                                                               0.0s
------
 > [2/3] COPY ./resource.tar /:
------
Dockerfile:2
--------------------
   1 |     FROM alpine
   2 | >>> COPY ./resource.tar /
   3 |     RUN echo "test"
   4 |
--------------------
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref moby::3y6r7qr9xy0ob7osrv9tmgc5i: "/resource.tar": not found
# 删除COPY后即可构建成功
[root@centos7 tmp]# cat Dockerfile
FROM alpine
RUN echo "test"
[root@centos7 tmp]# docker build -t test:v5 - < Dockerfile
[+] Building 0.6s (6/6) FINISHED
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 136B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => CACHED [1/2] FROM docker.io/library/alpine                                                                                      0.0s
 => [2/2] RUN echo "test"                                                                                                           0.6s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:8bb31e67b5d765d567aeb1607e57225b26b447f4e5883f564307dd5e77ff9cc3                                        0.0s
 => => naming to docker.io/library/test:v5                                                                                          0.0s
[root@centos7 tmp]# docker images | grep v5
test         v5        8bb31e67b5d7   38 seconds ago   5.59MB

# 示例6
[root@centos7 tmp]# docker build -t test:v6 - < test.tar.gz
[+] Building 0.5s (7/7) FINISHED
 => [internal] load remote build context                                                                                            0.3s
 => CACHED copy /context /                                                                                                          0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => CACHED [2/3] COPY ./resource.tar /                                                                                              0.0s
 => CACHED [3/3] RUN echo "test"                                                                                                    0.0s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:d74fefe3ea1f8761835a8201235571cfad3564d4b2b51d26c3662bebe3957938                                        0.0s
 => => naming to docker.io/library/test:v6                                                                                          0.0s
[root@centos7 tmp]# docker images | grep v6
test         v6        d74fefe3ea1f   43 minutes ago   15.9MB

```

> 注意到v6最后构建，时间并不是最新（而是43 minutes ago），因为v1/v2/v4/v6均采用同一个Dockerfile构建，系统已经有缓存，所以直接用了v1的镜像，然后改了个标签。

## context上下文

上面构建过程中注意`.`表示上下文（context）：在镜像构建过程中`COPY`、`ADD`和`RUN`均会产生镜像层，`.`表示在当前路径，构建过程中文件均在当前目录寻找，指定其他上下文，会在其他上下文寻找，比如：`../file/`，Dockerfile构建时会在上级目录的file文件夹下找文件。

```bash
[root@centos7 tmp]# tree ./
./
├── Dockerfile
├── resource.tar
├── test
│   └── Dockerfile
└── test.tar.gz

1 directory, 4 files
# ./Dockerfile内容
[root@centos7 tmp]# cat Dockerfile
FROM alpine
COPY resource.tar /
RUN echo "test"
# ./test/Dockerfile内容
[root@centos7 tmp]# cat ./test/Dockerfile
FROM alpine
COPY ../resource.tar /
RUN echo "test"

[root@centos7 tmp]# cd test
# 以test目录为上下文路径，build时无法找到resource.tar
[root@centos7 test]# docker build -t demo:v1 .
[+] Building 0.1s (6/7)
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 149B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [internal] load build context                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => ERROR [2/3] COPY ../resource.tar /                                                                                              0.0s
------
 > [2/3] COPY ../resource.tar /:
------
Dockerfile:2
--------------------
   1 |     FROM alpine
   2 | >>> COPY ../resource.tar /
   3 |     RUN echo "test"
   4 |
--------------------
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref moby::rbutk0he141r0g9cluqn9j2dr: "/resource.tar": not found

# 以上级目录为上下文路径，可见Dockerfile使用的是上下文中的，也就是上级目录
[root@centos7 test]# docker build -t demo:v1 ..
[+] Building 0.1s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 145B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [internal] load build context                                                                                                   0.0s
 => => transferring context: 96B                                                                                                    0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => CACHED [2/3] COPY resource.tar /                                                                                                0.0s
 => CACHED [3/3] RUN echo "test"                                                                                                    0.0s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:e1db4e1d4a736862c254cedaa742808b0014088825c214160e820101b6bf5e7c                                        0.0s
 => => naming to docker.io/library/demo:v1                                                                                          0.0s

# 指定test目录下的Dockerfile，上下文指定为上级目录，构建成功，而且Dockerfile正确
[root@centos7 test]# docker build -t demo:v2 -f ./Dockerfile ..
[+] Building 0.1s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 149B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [internal] load build context                                                                                                   0.0s
 => => transferring context: 96B                                                                                                    0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => CACHED [2/3] COPY ../resource.tar /                                                                                             0.0s
 => CACHED [3/3] RUN echo "test"                                                                                                    0.0s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:7a6f61cab683bae940af0ec86c4b8f1a0638dba2a407c36473d51a84f1acfe83                                        0.0s
 => => naming to docker.io/library/demo:v2                                                                                          0.0s

# 删除上级目录的Dockerfile，进一步验证了上下文路径的作用
[root@centos7 test]# rm -rf ../Dockerfile
[root@centos7 test]# pwd
/root/tmp/test
[root@centos7 test]# docker build -t demo:v3 -f /root/tmp/test/Dockerfile ..
[+] Building 0.1s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 149B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [internal] load build context                                                                                                   0.0s
 => => transferring context: 96B                                                                                                    0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => CACHED [2/3] COPY ../resource.tar /                                                                                             0.0s
 => CACHED [3/3] RUN echo "test"                                                                                                    0.0s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:7a6f61cab683bae940af0ec86c4b8f1a0638dba2a407c36473d51a84f1acfe83                                        0.0s
 => => naming to docker.io/library/demo:v3                                                                                          0.0s

# 修改Dockerfile变更资源文件路径，指定当前目录为上下文
[root@centos7 test]# mkdir demo
[root@centos7 test]# cp ../resource.tar demo/
[root@centos7 test]# vim Dockerfile
[root@centos7 test]# cat Dockerfile
FROM alpine
COPY demo/resource.tar /
RUN echo "test"
[root@centos7 test]# docker build -t demo:v4 .
[+] Building 0.4s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 151B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => [internal] load build context                                                                                                   0.2s
 => => transferring context: 10.30MB                                                                                                0.2s
 => CACHED [2/3] COPY demo/resource.tar /                                                                                           0.0s
 => CACHED [3/3] RUN echo "test"                                                                                                    0.0s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:3ac067b6d5dcfd571f43c1983627d587e372f140e097eea88825c57996a9c871                                        0.0s
 => => naming to docker.io/library/demo:v4                                                                                          0.0s

# 指定目录为上下文
[root@centos7 test]# cd ..
[root@centos7 tmp]# ls
resource.tar  test  test.tar.gz
[root@centos7 tmp]# docker build -t demo:v5 -f test/Dockerfile test
[+] Building 0.1s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 151B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => [internal] load build context                                                                                                   0.0s
 => => transferring context: 185B                                                                                                   0.0s
 => CACHED [2/3] COPY demo/resource.tar /                                                                                           0.0s
 => CACHED [3/3] RUN echo "test"                                                                                                    0.0s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:3ac067b6d5dcfd571f43c1983627d587e372f140e097eea88825c57996a9c871                                        0.0s
 => => naming to docker.io/library/demo:v5                                                                                          0.0s

# 上下文中有Dockerfile时，不需要指定Dockerfile具体路径
[root@centos7 tmp]# docker build -t demo:v6 test
[+] Building 0.1s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 151B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [internal] load build context                                                                                                   0.0s
 => => transferring context: 185B                                                                                                   0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => CACHED [2/3] COPY demo/resource.tar /                                                                                           0.0s
 => CACHED [3/3] RUN echo "test"                                                                                                    0.0s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:2180e7f9ba294fb968084810c4f0b9e77ec79a5f8bdde88d5167ebed41ab4107                                        0.0s
 => => naming to docker.io/library/demo:v6                                                                                          0.0s

# 如果构建文件名称不为Dockerfile，无论是否存在于上下文，都需要指定
[root@centos7 tmp]# mv test/Dockerfile test/Dockerfile-demo
[root@centos7 tmp]# docker build -t demo:v7 test
[+] Building 0.1s (2/2) FINISHED
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 2B                                                                                                  0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
ERROR: failed to solve: failed to read dockerfile: open /var/lib/docker/tmp/buildkit-mount3807940358/Dockerfile: no such file or directory
[root@centos7 tmp]# docker build -t demo:v7 test -f test/Dockerfile-demo
[+] Building 0.1s (8/8) FINISHED
 => [internal] load build definition from Dockerfile-demo                                                                           0.0s
 => => transferring dockerfile: 156B                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                    0.0s
 => [internal] load build context                                                                                                   0.0s
 => => transferring context: 185B                                                                                                   0.0s
 => [1/3] FROM docker.io/library/alpine                                                                                             0.0s
 => CACHED [2/3] COPY demo/resource.tar /                                                                                           0.0s
 => CACHED [3/3] RUN echo "test"                                                                                                    0.0s
 => exporting to image                                                                                                              0.0s
 => => exporting layers                                                                                                             0.0s
 => => writing image sha256:cd2c1eec8aa54818a6541d798900d95f0c51304604472a0246037ea26f938b35                                        0.0s
 => => naming to docker.io/library/demo:v7                                                                                          0.0s
```
