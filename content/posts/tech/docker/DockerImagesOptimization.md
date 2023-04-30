---
title: "DockerImagesOptimization"
date: 2023-04-26T18:43:22+08:00
lastmod: 2023-04-26T18:43:22+08:00
author: ["deemoprobe"]
keywords: 
- docker
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- docker
description: "Docker镜像体积优化"
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

## 优化原理

docker镜像层有以下特点（和镜像大小息息相关）：

- `RUN`、`COPY` 和 `ADD` 指令会在已有镜像层的基础上创建一个新的镜像层，执行指令产生的所有文件系统变更会在指令结束后作为一个镜像层整体提交
- 镜像层具有`copy-on-write`的特性，如果去更新其他镜像层中已存在的文件，会先将其复制到新的镜像层中再修改，造成双倍的文件空间占用
- 如果去删除其他镜像层的一个文件，只会在当前镜像层生成一个该文件的删除标记，并不会减少整个镜像的实际体积

```bash
[root@centos7 tmp]# cat Dockerfile
FROM alpine
COPY resource.tar /
RUN touch /resource.tar
RUN rm -f /resource.tar
[root@centos7 tmp]# ls -lh resource.tar
-rw-r--r--. 1 root root 9.9M Apr 26 15:46 resource.tar
[root@centos7 tmp]# docker build -t demo:v1 .
[+] Building 20.3s (9/9) FINISHED
 => [internal] load .dockerignore                                                                                                   0.0s
 => => transferring context: 2B                                                                                                     0.0s
 => [internal] load build definition from Dockerfile                                                                                0.0s
 => => transferring dockerfile: 177B                                                                                                0.0s
 => [internal] load metadata for docker.io/library/alpine:latest                                                                   17.5s
 => [internal] load build context                                                                                                   0.2s
 => => transferring context: 10.30MB                                                                                                0.2s
 => [1/4] FROM docker.io/library/alpine@sha256:21a3deaa0d32a8057914f36584b5288d2e5ecc984380bc0118285c70fa8c9300                     1.6s
 => => resolve docker.io/library/alpine@sha256:21a3deaa0d32a8057914f36584b5288d2e5ecc984380bc0118285c70fa8c9300                     0.0s
 => => sha256:21a3deaa0d32a8057914f36584b5288d2e5ecc984380bc0118285c70fa8c9300 1.64kB / 1.64kB                                      0.0s
 => => sha256:e7d88de73db3d3fd9b2d63aa7f447a10fd0220b7cbf39803c803f2af9ba256b3 528B / 528B                                          0.0s
 => => sha256:c059bfaa849c4d8e4aecaeb3a10c2d9b3d85f5165c66ad3a4d937758128c4d18 1.47kB / 1.47kB                                      0.0s
 => => sha256:59bf1c3509f33515622619af21ed55bbe26d24913cedbca106468a5fb37a50c3 2.82MB / 2.82MB                                      1.2s
 => => extracting sha256:59bf1c3509f33515622619af21ed55bbe26d24913cedbca106468a5fb37a50c3                                           0.3s
 => [2/4] COPY resource.tar /                                                                                                       0.1s
 => [3/4] RUN touch /resource.tar                                                                                                   0.4s
 => [4/4] RUN rm -f /resource.tar                                                                                                   0.5s
 => exporting to image                                                                                                              0.2s
 => => exporting layers                                                                                                             0.2s
 => => writing image sha256:1113b0fb22dabb56b6926960eafd2a6aaf183d1bd2373f1be10df67c73714f37                                        0.0s
 => => naming to docker.io/library/demo:v1                                                                                          0.0s
[root@centos7 tmp]# docker history demo:v1
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
1113b0fb22da   13 seconds ago   RUN /bin/sh -c rm -f /resource.tar # buildkit   0B        buildkit.dockerfile.v0
<missing>      13 seconds ago   RUN /bin/sh -c touch /resource.tar # buildkit   10.3MB    buildkit.dockerfile.v0
<missing>      14 seconds ago   COPY resource.tar / # buildkit                  10.3MB    buildkit.dockerfile.v0
<missing>      17 months ago    /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>      17 months ago    /bin/sh -c #(nop) ADD file:9233f6f2237d79659…   5.59MB
[root@centos7 tmp]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
demo         v1        1113b0fb22da   24 seconds ago   26.2MB
```

上面例子可以分析出：alpine基础镜像大小+5.59MB，resource.tar COPY解压后+10.3MB，touch命令执行时复制了一份+10.3MB，rm虽然显示文件大小为0，但实际上前几层的镜像依然存在。最终镜像大小26.2MB。

## 镜像分析

- `docker history IMAGE`：查看IMAGE构建过程，展示镜像层信息
- `dive IMAGE`：第三方分析工具，分析镜像层结构，每层镜像所包含的文件以及体积，比history更加详细

[dive安装参考官方文档](https://github.com/wagoodman/dive#installation)

```bash
[root@centos7 go]# dive demo:v1
┃ ● Layers ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │ Current Layer Contents ├──────────────────────────────────────────
Cmp   Size  Command                                                  Permission     UID:GID       Size  Filetree
    7.0 MB  FROM 580fdc606fb93e5                                     drwxr-xr-x         0:0     841 kB  ├── bin
     10 MB  COPY resource.tar / # buildkit                           -rwxrwxrwx         0:0        0 B  │   ├── arch → /bin/busybox
     10 MB  RUN /bin/sh -c touch /resource.tar # buildkit            -rwxrwxrwx         0:0        0 B  │   ├── ash → /bin/busybox
       0 B  RUN /bin/sh -c rm -f /resource.tar # buildkit            -rwxrwxrwx         0:0        0 B  │   ├── base64 → /bin/busybox
                                                                     -rwxrwxrwx         0:0        0 B  │   ├── bbconfig → /bin/busybox
│ Layer Details ├─────────────────────────────────────────────────── -rwxr-xr-x         0:0     841 kB  │   ├── busybox
                                                                     -rwxrwxrwx         0:0        0 B  │   ├── cat → /bin/busybox
Tags:   (unavailable)                                                -rwxrwxrwx         0:0        0 B  │   ├── chattr → /bin/busybox
Id:     580fdc606fb93e5fc291e8df154398e0585362a58b8e5b23bfd9d47ca369 -rwxrwxrwx         0:0        0 B  │   ├── chgrp → /bin/busybox
8a79                                                                 -rwxrwxrwx         0:0        0 B  │   ├── chmod → /bin/busybox
Digest: sha256:f1417ff83b319fbdae6dd9cd6d8c9c88002dcd75ecf6ec201c8c6 -rwxrwxrwx         0:0        0 B  │   ├── chown → /bin/busybox
894681cf2b5                                                          -rwxrwxrwx         0:0        0 B  │   ├── cp → /bin/busybox
Command:                                                             -rwxrwxrwx         0:0        0 B  │   ├── date → /bin/busybox
#(nop) ADD file:9a4f77dfaba7fd2aa78186e4ef0e7486ad55101cefc1fabbc1b3 -rwxrwxrwx         0:0        0 B  │   ├── dd → /bin/busybox
85601bb38920 in /                                                    -rwxrwxrwx         0:0        0 B  │   ├── df → /bin/busybox
                                                                     -rwxrwxrwx         0:0        0 B  │   ├── dmesg → /bin/busybox
│ Image Details ├─────────────────────────────────────────────────── -rwxrwxrwx         0:0        0 B  │   ├── dnsdomainname → /bin/busy
                                                                     -rwxrwxrwx         0:0        0 B  │   ├── dumpkmap → /bin/busybox
                                                                     -rwxrwxrwx         0:0        0 B  │   ├── echo → /bin/busybox
Total Image size: 28 MB                                              -rwxrwxrwx         0:0        0 B  │   ├── ed → /bin/busybox
Potential wasted space: 21 MB                                        -rwxrwxrwx         0:0        0 B  │   ├── egrep → /bin/busybox
Image efficiency score: 25 %                                         -rwxrwxrwx         0:0        0 B  │   ├── false → /bin/busybox
                                                                     -rwxrwxrwx         0:0        0 B  │   ├── fatattr → /bin/busybox
Count   Total Space  Path                                            -rwxrwxrwx         0:0        0 B  │   ├── fdflush → /bin/busybox
    3         21 MB  /resource.tar                                   -rwxrwxrwx         0:0        0 B  │   ├── fgrep → /bin/busybox
    2           0 B  /etc                                            -rwxrwxrwx         0:0        0 B  │   ├── fsync → /bin/busybox
                                                                     ...
```

## 镜像优化

镜像（体积）优化最主要的目的是方便镜像更快速拉取和分发，同时节省硬盘空间，提升部署效率，优化方式可以从下面几个方式入手：

- 1. 优化基础镜像，在不影响业务维护的情况下，尽可能使用更小的基础镜像，如：alpine或image:alpine
- 2. 优化Dockerfile中指令，如多条RUN指令`&&`串联写在同一行，使之仅生成一层镜像，防止分层过多
- 3. ~~使用`--squash`参数压缩镜像层，官方提示已弃用~~（本人目前版本是23.0.4）
- 4. 多阶段构建，更多信息可以查看[官网介绍](https://docs.docker.com/build/building/multi-stage/)
- 5. 根据dive分析结果优化
  - 对于基础镜像（最底层）大于500MB的，建议选择更小的基础镜像，如基于alpine的镜像
  - 对于分层大于10层的镜像，获取各层的命令汇总展示，建议调整指令，是否已使用串连的形式
  - 避免产生无用的缓存，如pip应使用`--no-cache-dir`禁用缓存，`yum makecache`也会产生缓存
  - 避免无用文件（.md .pdf .txt .doc等一些文档文件不应该出现在镜像中）
  - 避免安装多余的软件

```bash
# 先拉取镜像到本地，加快后面的构建速度
docker pull centos:latest
docker pull alpine:latest
docker pull golang:latest
docker pull ubuntu:latest
docker pull fedora:28
```

### 优化基础镜像

```bash
[root@centos7 base]# pwd
/root/dockerfile/base
[root@centos7 base]# cat Dockerfile
FROM centos
CMD ["echo", "hi"]
[root@centos7 base]# docker build -t base:v1 .
# 修改基础镜像
[root@centos7 base]# cat Dockerfile
FROM alpine
CMD ["echo", "hi"]
[root@centos7 base]# docker build -t base:v2 .
[root@centos7 base]# docker images | grep base
base         v2        025106dd85eb   4 weeks ago     7.05MB
base         v1        108460ab34b0   19 months ago   231MB
```

### 串联命令

```bash
[root@centos7 multirun]# pwd
/root/dockerfile/multirun
[root@centos7 multirun]# cat Dockerfile
FROM fedora:28
RUN dnf install -y nginx
RUN dnf clean all
RUN rm -rf /var/cache/yum
[root@centos7 multirun]# docker build -t multirun:v1 .
# 分析镜像
[root@centos7 multirun]# docker images| grep mul
multirun     v1        2666d7aa527a   2 minutes ago    488MB
[root@centos7 multirun]# docker history multirun:v1
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
2666d7aa527a   17 seconds ago   RUN /bin/sh -c rm -rf /var/cache/yum # build…   0B        buildkit.dockerfile.v0
<missing>      18 seconds ago   RUN /bin/sh -c dnf clean all # buildkit         1.79MB    buildkit.dockerfile.v0
<missing>      19 seconds ago   RUN /bin/sh -c dnf install -y nginx # buildk…   226MB     buildkit.dockerfile.v0
<missing>      4 years ago      /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>      4 years ago      /bin/sh -c #(nop) ADD file:42ddab590052fda98…   261MB
<missing>      4 years ago      /bin/sh -c #(nop)  ENV DISTTAG=f28container …   0B
<missing>      4 years ago      /bin/sh -c #(nop)  LABEL maintainer=Clement …   0B
[root@centos7 multirun]# dive multirun:v1
┃ ● Layers ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │ Current Layer Contents ├──────────────────────────────────────────
Cmp   Size  Command                                                  Permission     UID:GID       Size  Filetree
    260 MB  FROM 9bfbc900bde2f55                                     -rwxrwxrwx         0:0        0 B  ├── bin → usr/bin
    226 MB  RUN /bin/sh -c dnf install -y nginx # buildkit           dr-xr-xr-x         0:0        0 B  ├── boot
    1.8 MB  RUN /bin/sh -c dnf clean all # buildkit                  drwxr-xr-x         0:0        0 B  ├── dev
       0 B  RUN /bin/sh -c rm -rf /var/cache/yum # buildkit          drwxr-xr-x         0:0     1.8 MB  ├── etc
                                                                     -rw-------         0:0        0 B  │   ├── .pwd.lock
│ Layer Details ├─────────────────────────────────────────────────── -rw-r--r--         0:0     4.5 kB  │   ├── DIR_COLORS
                                                                     -rw-r--r--         0:0     5.2 kB  │   ├── DIR_COLORS.256color
Tags:   (unavailable)                                                -rw-r--r--         0:0     4.6 kB  │   ├── DIR_COLORS.lightbgcolor
Id:     9bfbc900bde2f55fafbeeed2be52e80c9b06244dee226dc54a354f9d09ea -rw-r--r--         0:0       94 B  │   ├── GREP_COLORS
1910                                                                 drwxr-xr-x         0:0      203 B  │   ├── X11
Digest: sha256:17beab58d693a5e55591675c3dcc5b575a44e476d125408102ce7 drwxr-xr-x         0:0        0 B  │   │   ├── applnk
5348011f807                                                          drwxr-xr-x         0:0        0 B  │   │   ├── fontpath.d
Command:                                                             drwxr-xr-x         0:0      203 B  │   │   ├── xinit
#(nop) ADD file:42ddab590052fda98865a9ecd9dc28c8d3ba7922a6d21b1f182a drwxr-xr-x         0:0      203 B  │   │   │   └── xinitrc.d
0a3769419677 in /                                                    -rwxr-xr-x         0:0      203 B  │   │   │       └── 50-systemd-us
                                                                     drwxr-xr-x         0:0        0 B  │   │   └── xorg.conf.d
│ Image Details ├─────────────────────────────────────────────────── -rw-r--r--         0:0       16 B  │   ├── adjtime
                                                                     -rw-r--r--         0:0     1.5 kB  │   ├── aliases
                                                                     drwxr-xr-x         0:0        0 B  │   ├── alternatives
Total Image size: 488 MB                                             -rwxrwxrwx         0:0        0 B  │   │   ├── cifs-idmap-plugin → /
Potential wasted space: 234 MB                                       -rwxrwxrwx         0:0        0 B  │   │   └── libnssckbi.so.x86_64
Image efficiency score: 54 %                                         drwxr-xr-x         0:0        0 B  │   ├── bash_completion.d
                                                                     -rw-r--r--         0:0     3.0 kB  │   ├── bashrc
Count   Total Space  Path                                            drwxr-xr-x         0:0        0 B  │   ├── binfmt.d
    2         48 MB  /var/cache/dnf/fedora-filenames.solvx           drwxr-xr-x         0:0        0 B  │   ├── chkconfig.d
    2         47 MB  /var/cache/dnf/fedora-f21308f6293b3270/repodata drwxr-xr-x         0:0        0 B  │   ├── cifs-utils
/a53009478e29c551710df570a5dc726ce5160300c7c1ecde852895ab8a6fcf72-fi -rwxrwxrwx         0:0        0 B  │   │   └── idmap-plugin → /etc/a
lelists.xml.gz                                                       drwxr-xr-x         0:0      614 B  │   ├── crypto-policies
    2         24 MB  /var/cache/dnf/updates-filenames.solvx          drwxr-xr-x         0:0        0 B  │   │   ├── back-ends
    2         23 MB  /var/cache/dnf/updates-8bd9ef368505a5fd/repodat -rwxrwxrwx         0:0        0 B  │   │   │   ├── bind.config → /us
a/15dc5106b1b18f200a2f520b561e59fd8a75d921acfd7edf34e8d88db3b9220a-f -rwxrwxrwx         0:0        0 B  │   │   │   ├── gnutls.config → /
ilelists.xml.gz                                                      -rwxrwxrwx         0:0        0 B  │   │   │   ├── java.config → /us
    2         21 MB  /var/cache/dnf/fedora.solv                      -rwxrwxrwx         0:0        0 B  │   │   │   ├── krb5.config → /us
    2         18 MB  /var/lib/rpm/Packages                           -rwxrwxrwx         0:0        0 B  │   │   │   ├── libreswan.config
    2         16 MB  /var/cache/dnf/fedora-f21308f6293b3270/repodata -rwxrwxrwx         0:0        0 B  │   │   │   ├── nss.config → /usr
/9659dfc44de50563682658fd41f2ab0da60e5657e1cc4f598b50d39fb3436e0c-pr -rwxrwxrwx         0:0        0 B  │   │   │   ├── openssh.config →
...
# 从dive分析结果可以看出不但每个RUN都产生了一层镜像，而且实际上缓存并没有清理掉，提示Potential wasted space: 234 MB，潜在浪费了234MB空间

# 优化RUN命令
[root@centos7 multirun]# cat Dockerfile
FROM fedora:28
RUN dnf install -y nginx\
 && dnf clean all\
 && rm -rf /var/cache/yum
[root@centos7 multirun]# docker build -t multirun:v2 .
[root@centos7 multirun]# dive multirun:v2
┃ ● Layers ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │ Current Layer Contents ├──────────────────────────────────────────
Cmp   Size  Command                                                  Permission     UID:GID       Size  Filetree
    260 MB  FROM 9bfbc900bde2f55                                     -rwxrwxrwx         0:0        0 B  ├── bin → usr/bin
     18 MB  RUN /bin/sh -c dnf install -y nginx && dnf clean all &&  dr-xr-xr-x         0:0        0 B  ├── boot
                                                                     drwxr-xr-x         0:0        0 B  ├── dev
│ Layer Details ├─────────────────────────────────────────────────── drwxr-xr-x         0:0     1.8 MB  ├── etc
                                                                     -rw-------         0:0        0 B  │   ├── .pwd.lock
Tags:   (unavailable)                                                -rw-r--r--         0:0     4.5 kB  │   ├── DIR_COLORS
Id:     9bfbc900bde2f55fafbeeed2be52e80c9b06244dee226dc54a354f9d09ea -rw-r--r--         0:0     5.2 kB  │   ├── DIR_COLORS.256color
1910                                                                 -rw-r--r--         0:0     4.6 kB  │   ├── DIR_COLORS.lightbgcolor
Digest: sha256:17beab58d693a5e55591675c3dcc5b575a44e476d125408102ce7 -rw-r--r--         0:0       94 B  │   ├── GREP_COLORS
5348011f807                                                          drwxr-xr-x         0:0      203 B  │   ├── X11
Command:                                                             drwxr-xr-x         0:0        0 B  │   │   ├── applnk
#(nop) ADD file:42ddab590052fda98865a9ecd9dc28c8d3ba7922a6d21b1f182a drwxr-xr-x         0:0        0 B  │   │   ├── fontpath.d
0a3769419677 in /                                                    drwxr-xr-x         0:0      203 B  │   │   ├── xinit
                                                                     drwxr-xr-x         0:0      203 B  │   │   │   └── xinitrc.d
│ Image Details ├─────────────────────────────────────────────────── -rwxr-xr-x         0:0      203 B  │   │   │       └── 50-systemd-us
                                                                     drwxr-xr-x         0:0        0 B  │   │   └── xorg.conf.d
                                                                     -rw-r--r--         0:0       16 B  │   ├── adjtime
Total Image size: 278 MB                                             -rw-r--r--         0:0     1.5 kB  │   ├── aliases
Potential wasted space: 24 MB                                        drwxr-xr-x         0:0        0 B  │   ├── alternatives
Image efficiency score: 95 %                                         -rwxrwxrwx         0:0        0 B  │   │   ├── cifs-idmap-plugin → /
                                                                     -rwxrwxrwx         0:0        0 B  │   │   └── libnssckbi.so.x86_64
Count   Total Space  Path                                            drwxr-xr-x         0:0        0 B  │   ├── bash_completion.d
    2         18 MB  /var/lib/rpm/Packages                           -rw-r--r--         0:0     3.0 kB  │   ├── bashr
...
# 缓存已清理干净，有效空间占比95%，Image efficiency score: 95 %
```

### 压缩镜像层（弃用）

docker1.13版本后可以直接使用`--squash`参数压缩镜像层，之前需要单独安装`docker-squash`工具才能实现。但目前已移除了这个参数，使用也无效。

```bash
# 依旧以上面RUN为例
[root@centos7 squash]# pwd
/root/dockerfile/squash
[root@centos7 squash]# cat Dockerfile
FROM fedora:28
RUN dnf install -y nginx
RUN dnf clean all
RUN rm -rf /var/cache/yum
[root@centos7 squash]# docker build -t squash:v1 .
# 压缩前488MB
[root@centos7 squash]# docker images | grep squ
squash       v1        2666d7aa527a   45 minutes ago      488MB
# 发现提示已移除该参数，提示使用多阶段构建
[root@centos7 squash]# docker build -t squash:v2 --squash .
WARNING: experimental flag squash is removed with BuildKit. You should squash inside build using a multi-stage Dockerfile for efficiency.
[root@centos7 squash]# docker images | grep squ
squash       v1        2666d7aa527a   46 minutes ago      488MB
squash       v2        2666d7aa527a   46 minutes ago      488MB
[root@centos7 squash]# docker version
Client: Docker Engine - Community
 Version:           23.0.4
...
```

> 既然官方已弃用，那就不详细研究了。建议使用多阶段构建。原本镜像层压缩就会造成其他缓存层无法共用，失去了镜像的优势，在镜像较多的情况下无法公用，存储开销是很大的。

### 多阶段构建

scratch是一个空镜像，不能拉取也不能运行，里面什么都没有，但可以作为基础镜像构建其他镜像。

```bash
[root@centos7 go]# pwd
/root/dockerfile/go
[root@centos7 go]# cat hello.go
package main

import "fmt"

func main() {
        fmt.Println("hello world.")
}
[root@centos7 go]# cat Dockerfile
FROM golang
COPY hello.go .
RUN go build hello.go
CMD ["./hello"]
[root@centos7 go]# docker run -it --rm go:v1
hello world.
# 分阶段构建，这里指定WORKDIR，方便复制可执行文件
[root@centos7 go]# cat Dockerfile
FROM golang AS first
WORKDIR /app
COPY hello.go .
RUN go build hello.go

FROM scratch
COPY --from=first /app/hello .
CMD ["./hello"]
[root@centos7 go]# docker build -t go:v2 .
[root@centos7 go]# docker run -it --rm go:v2
hello world.
[root@centos7 go]# docker images | grep go
go           v2        d9e76072de38   29 seconds ago   1.85MB
go           v1        c5bb11851550   2 minutes ago    804MB

# 使用非scratch镜像
[root@centos7 go]# cat Dockerfile
FROM golang
WORKDIR /app
COPY hello.go .
RUN go build hello.go

FROM ubuntu
COPY --from=0 /app/hello .
CMD ["./hello"]
[root@centos7 go]# docker build -t go:v3 .
[root@centos7 go]# docker images | grep v3
go           v3        d1429ea70f10   13 seconds ago   74.6MB
```

> 注意到分阶段构建时用了`--from=first`和`--from=0`。`--from=0`表示FROM按序号算（从0开始）的阶段，可以不必指定`AS build_name`；`--from=first`这里的first就是构建阶段的别名，用AS指定别名，以供调用。

### 优化注意事项

**scratch镜像**：该镜像什么都没有，没有shell，没有程序链接库（库文件），没有调试工具（ps/top/ping等），所以如果需要这些功能，不推荐使用scratch镜像，可以选用合适的镜像构建，如：busybox、alpine、ubuntu等

**alpine镜像**：该镜像标准库是`musl libc`而不是`glibc`，程序依赖glibc库时，在alpine中编译时会因为库缺失而报错。除了标准库，alpine对文件系统也进行了精简，Python程序在alpine中运行效率非常低，所以谨慎使用，可以选择其他合适镜像构建，如：busybox:glibc、ubuntu、debian-slim和openjdk:8-jre-alpine，很多镜像发布了自己得alpine版本，也可以选择使用

生产中是否使用alpine镜像可以参考文章：[alpine镜像分析](https://ttys3.dev/post/do-not-use-alpine-in-production-environment/)

> 镜像精简固然重要，但如果以提升程序复杂度为代价是不妥的。

针对不同开发语言的镜像精简策略可以参考文章：[Docker 镜像制作教程：针对不同语言的精简策略](https://zhuanlan.zhihu.com/p/141697080)
