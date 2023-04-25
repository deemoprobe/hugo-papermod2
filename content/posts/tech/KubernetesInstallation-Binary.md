---
title: "KubernetesInstallation Binary"
date: 2023-04-25T18:56:52+08:00
lastmod: 2023-04-25T18:56:52+08:00
author: ["deemoprobe"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- tech
tags: # 标签
- kubernetes
description: ""
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
showbreadcrumbs: true #顶部显示路径
cover:
    image: "img/docker.png" #图片路径例如：posts/tech/123/123.png，相对路径直接相对static
    zoom: 50% # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

## 环境说明

- 宿主机系统：Windows 10
- 虚拟机版本：VMware® Workstation 16 Pro
- IOS镜像版本：CentOS Linux release 7.9.2009
- Kubernetes版本：1.26.4
- Runtime：Containerd v1.6.20
- Etcd版本：3.5.6
- 集群操作用户：root
- 更新时间：2023-04-20

CentOS7安装请参考博客文章：[LINUX之VMWARE WORKSTATION安装CENTOS-7](https://www.deemo.dev/linux/centos7install/)

## 资源分配

**网段划分**

Kubernetes集群需要规划三个网段：

- 宿主机网段：Kubernetes集群节点的网段
- Pod网段：集群内Pod的网段，相当于容器的IP
- Service网段：集群内服务发现使用的网段，service用于集群容器通信

生产环境根据申请到的IP资源进行分配即可，原则是三个网段不允许有重合IP。IP网段计算可以参考：[在线IP地址计算](http://tools.jb51.net/aideddesign/ip_net_calc/)。本文虚拟机练习环境IP地址网段分配如下：

- 宿主机网段：`192.168.43.1/24`
- Pod网段：`172.16.0.0/12`
- Service：`10.96.0.0/12`

**节点分配**

采用`3管理节点2工作节点`的高可用Kubernetes集群模式：

- k8s-master01/k8s-master02/k8s-master03 集群的Master节点
- 三个master节点同时做etcd集群
- k8s-node01/k8s-node02 集群的Node节点
- k8s-master-vip做高可用k8s-master01~03的VIP，不占用物理资源

| 主机节点名称   |       IP       | CPU核心数 | 内存大小 | 磁盘大小 |
| :------------- | :------------: | :-------: | :------: | :------: |
| k8s-master-vip | 192.168.43.200 |     /     |    /     |    /     |
| k8s-master01   | 192.168.43.201 |     2     |    2G    |   40G    |
| k8s-master02   | 192.168.43.202 |     2     |    2G    |   40G    |
| k8s-master03   | 192.168.43.203 |     2     |    2G    |   40G    |
| k8s-node01     | 192.168.43.204 |     2     |    2G    |   40G    |
| k8s-node02     | 192.168.43.205 |     2     |    2G    |   40G    |

## 操作步骤

标题后小括号注释表明操作范围：

- ALL 所有节点（k8s-master01/k8s-master02/k8s-master03/k8s-node01/k9s-node02）执行
- Master 只需要在master节点（k8s-master01/k8s-master02/k8s-master03）执行
- Node 只需要在node节点（k8s-node01/k8s-node02）执行
- 已标注的个别命令只需要在某一台机器执行，会在操作前说明
- 未标注的会在操作时说明

使用`cat << "EOF" >> file`或`cat >> file << "EOF"`添加文件内容注意cat后面的EOF一定要加上双引号（标准输入的），否则不会保留输入时的缩进格式而且会直接解析输入时的变量，进而造成文件可读性差甚至不可用；同时注意文件的`>`重写与`>>`追加。虽然单独转义输入时的变量也能避免变量被解析，但是不推荐，漏转义会造成不必要的麻烦。

### 准备工作(ALL)

添加主机信息、关闭防火墙、关闭swap、关闭SELinux、dnsmasq、NetworkManager

```bash
# 添加主机信息
cat << "EOF" >> /etc/hosts
192.168.43.200    k8s-master-vip
192.168.43.201    k8s-master01
192.168.43.202    k8s-master02
192.168.43.203    k8s-master03
192.168.43.204    k8s-node01
192.168.43.205    k8s-node02
EOF
# 关闭防火墙、dnsmasq、NetworkManager，--now参数表示关闭服务并移除开机自启
# 这些服务是否可以关闭视情况而定，本文是虚拟机实践，没有用到这些服务
systemctl disable --now firewalld
systemctl disable --now dnsmasq
systemctl disable --now NetworkManager
# 关闭swap，并注释fstab文件swap所在行
swapoff -a
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
# 关闭SELinux，并更改selinux配置文件
setenforce 0
sed -i "s/=enforcing/=disabled/g" /etc/selinux/config
```

值得注意的是`/etc/sysconfig/selinux`文件是`/etc/selinux/config`文件的软连接，用`sed -i`命令修改软连接文件会破坏软连接属性，将`/etc/sysconfig/selinux`变为一个独立的文件，即使该文件被修改了，但源文件`/etc/selinux/config`配置是没变的。此外，使用vim等编辑器编辑源文件或链接文件（编辑模式不会修改文件属性）修改也可以。软链接原理可参考博客：[LINUX之INODE详解](http://www.deemoprobe.com/yunv/inode/)

```bash
# 默认的yum源太慢，更新为阿里源，同时用sed命令删除文件中不需要的两个URL的行
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo

# 安装常用工具包
yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git -y

# 配置ntpdate，同步服务器时间
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
yum install ntpdate -y
# 同步时区和时间
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' >/etc/timezone
ntpdate time2.aliyun.com
# 可以加入计划任务，保证集群时钟是一致的
# /var/spool/cron/root文件也是crontab -e写入的文件
# crontab执行日志查看可用：tail -f /var/log/cron
cat << "EOF" >> /var/spool/cron/root
*/5 * * * * /usr/sbin/ntpdate time2.aliyun.com
EOF
# 须知：如果设置了定时任务，会经常收到提示“You have new mail in /var/spool/mail/root”
# （可选）禁用提示：echo "unset MAILCHECK" >> ~/.bashrc;source ~/.bashrc
# 禁用提示后/var/spool/mail/root文件依旧会记录root操作日志，可随时查看

# 保证文件句柄不会限制集群的可持续发展，配置limits
ulimit -SHn 65500
cat << "EOF" >> /etc/security/limits.conf
* soft nofile 65500
* hard nofile 65500
* soft nproc 65500
* hard nproc 65500
* soft memlock unlimited
* hard memlock unlimited
EOF

# 配置免密登录，k8s-master01到其他节点
# 生成密钥对（在k8s-master01节点配置即可）
ssh-keygen -t rsa
# 拷贝公钥到其他节点，首次需要认证一下各个节点的root密码，以后就可以免密ssh到其他节点
for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02;do ssh-copy-id -i .ssh/id_rsa.pub $i;done

# 克隆二进制仓库1.26分支的文件（k8s-master01上操作即可）
cd /root;git clone https://gitee.com/deemoprobe/k8s-ha-install.git -b manual-installation-v1.26.x

# 所有节点系统升级
yum update --exclude=kernel* -y
```

升级内核，4.17以下的内核cgroup存在内存泄漏的BUG，具体分析过程浏览器搜`Kubernetes集群为什么要升级内核`会有很多文章讲解

内核备用下载（下载到本地后上传到服务器，尽量不要用`wget`）：

- [kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/repo/kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm?versionId=CAEQNBiBgMDJiNbj.hciIDUyMDZlYjU5YzIwMzQ0MmNhNzBmNjBiMDY3Yjc0Y2Jl)
- [kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/repo/kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm?versionId=CAEQNBiBgMDNiNbj.hciIGQ0M2RmZDAwNDhlNjQyNjE5MTE4MDk1OGU4OThiNWY4)

```bash
# 下载4.19版本内核，如果无法下载，可以用上面提供的备用下载
cd /root
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm

# 可以在k8s-master01节点下载后，免密传到其他节点
for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02;do scp kernel-ml-* $i:/root;done

# 所有节点安装内核
cd /root && yum localinstall -y kernel-ml*
# 所有节点更改内核启动顺序
grub2-set-default  0 && grub2-mkconfig -o /etc/grub2.cfg
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
# 查看默认内核，并重启节点
grubby --default-kernel
reboot

# 确认内核版本
uname -a
# （可选）删除老版本的内核，避免以后被升级取代默认的开机4.19内核
rpm -qa | grep kernel
yum remove -y kernel-3*

# 升级系统软件包（如果跳过内核升级加参数 --exclude=kernel*）
yum update -y

# 安装IPVS相关工具，由于IPVS在资源消耗和性能上均已明显优于iptables，所以推荐开启
# 具体原因可参考官网介绍 https://kubernetes.io/zh/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/
yum install ipvsadm ipset sysstat conntrack libseccomp -y
# 加载模块，最后一条4.18及以下内核使用nf_conntrack_ipv4，4.19已改为nf_conntrack
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
# 编写参数文件
cat << "EOF" > /etc/modules-load.d/ipvs.conf 
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF
# systemd-modules-load加入开机自启
systemctl enable --now systemd-modules-load
# 自定义内核参数优化配置文件
cat << "EOF" > /etc/sysctl.d/kubernetes.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
net.ipv4.conf.all.route_localnet = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF
# 加载
sysctl --system
# 重启查看IPVS模块是否依旧加载
reboot
lsmod | grep -e ip_vs -e nf_conntrack
```

- 保证每台服务器中IPVS加载成功，以k8s-master01为例，如图：

![20220305153926](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220305153926.png)

### 部署Containerd(ALL)

Kubernetes1.24版本以后将不再支持Docker作为Runtime，本文安装使用Containerd作为Runtime。

```bash
# 配置阿里docker源
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装最新版本docker和containerd.io，安装docker是为了使用docker CLI
yum install docker-ce docker-ce-cli containerd.io -y

# （可选）也可以按需安装指定版本
yum install docker-ce-20.10.* docker-ce-cli-20.10.* containerd -y

# 配置Containerd模块
cat << "EOF" > /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
# 加载模块
modprobe -- overlay
modprobe -- br_netfilter
# 配置内核参数
cat << "EOF" > /etc/sysctl.d/containerd.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
# 加载内核参数
sysctl --system

# 生成默认配置文件
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# 更改Cgroup为Systemd，在containerd.runtimes.runc.options行后的SystemdCgroup = false修改为true，如果配置项不存在就自行添加，缩进俩空格添加SystemdCgroup = true一行
vim /etc/containerd/config.toml
          ...
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
          ...
# 将sandbox_image的Pause镜像地址改成国内：registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7
vim /etc/containerd/config.toml
    sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7"

# （可选）也可以在k8s-master01编辑/etc/containerd/config.toml文件后，将编辑后同名文件同步到其他服务器，自动覆盖
for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02;do scp /etc/containerd/config.toml $i:/etc/containerd/config.toml;done
# 查看确认是否配置成功
cat /etc/containerd/config.toml | grep -e Systemd -e sandbox_image

# 启动并加入开机自启
systemctl daemon-reload
systemctl enable --now containerd

# containerd运行时的CLI是ctr
[root@k8s-master01 ~]# ctr images ls
REF TYPE DIGEST SIZE PLATFORMS LABELS 
[root@k8s-master01 ~]# ctr version
Client:
  Version:  1.6.20
  Revision: 2806fc1057397dbaeefbea0e4e17bddfbd388f38
  Go version: go1.19.7

Server:
  Version:  1.6.20
  Revision: 2806fc1057397dbaeefbea0e4e17bddfbd388f38
  UUID: 9958928e-300c-4778-b83c-6c0073414f3e

# 配置crictl连接的运行时socket接口（指向containerd's GRPC server：/run/containerd/containerd.sock）
# crictl 默认连接到unix:///var/run/dockershim.sock
cat << "EOF" > /etc/crictl.yaml 
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
# crictl按需选择，https://github.com/kubernetes-sigs/cri-tools/releases
# containerd容器管理CLI是crictl，该命令使用和docker命令类似，下载包上传到k8s-master01
[root@k8s-master01 ~]# tar -zxvf crictl-v1.27.0-linux-amd64.tar.gz
[root@k8s-master01 ~]# mv crictl /usr/local/bin/
[root@k8s-master01 ~]# crictl version
Version:  0.1.0
RuntimeName:  containerd
RuntimeVersion:  1.6.20
RuntimeApiVersion:  v1
# 二进制可执行文件发送到其他节点
[root@k8s-master01 ~]# for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02;do scp /usr/local/bin/crictl $i:/usr/local/bin/;done
```

> crictl 是CRI（Container Runtime Interface）兼容的容器运行时的CLI（Command-Line Interface）。可以这个命令来检查和调试 Kubernetes 节点上的容器运行时和应用程序。介绍和安装方式可见：[critools](https://github.com/kubernetes-sigs/cri-tools)或[Kubernetes官方介绍](https://kubernetes.io/zh/docs/tasks/debug-application-cluster/crictl/)

### 二进制包和证书

k8s-master01节点上下载并安装Kubernetes二进制安装包（server-binaries，选择对应的架构即可）和ETCD二进制安装包，可在GitHub上查看Kubernetes1.23.x版本的信息，[官方GitHub-Kubernetes1.23版本链接](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.23.md)，[ETCD官方链接](https://github.com/etcd-io/etcd/releases)

```bash
# 以下在k8s-master01执行
# 下载太慢的话可以在相应链接网页上下载好传到服务器
wget https://dl.k8s.io/v1.26.4/kubernetes-server-linux-amd64.tar.gz
wget https://github.com/etcd-io/etcd/releases/download/v3.5.6/etcd-v3.5.6-linux-amd64.tar.gz

# 解压安装，--strip-components=N表示解压时忽略解压后的N层目录，直接获取N层目录后的目标文件。kubernetes/server/bin/是三层，etcd-v3.5.6-linux-amd64/是一层
tar -zxvf kubernetes-server-linux-amd64.tar.gz --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}
tar -zxvf etcd-v3.5.6-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin etcd-v3.5.6-linux-amd64/etcd{,ctl}

# 确认版本
kubectl version
kubelet --version
etcdctl version

# 拷贝组件到其他节点，Node节点只需要kubelet和kube-proxy
for i in k8s-master02 k8s-master03; do scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} $i:/usr/local/bin/; scp /usr/local/bin/etcd* $i:/usr/local/bin/; done
for i in k8s-node01 k8s-node02; do scp /usr/local/bin/kube{let,-proxy} $i:/usr/local/bin/; done
```

- 配置证书

```bash
# master节点创建etcd证书目录
mkdir -p /etc/etcd/ssl
# 所有节点创建pki证书目录和CNI目录（后面calico使用）
mkdir -p /etc/kubernetes/pki
mkdir -p /opt/cni/bin

# 以下在k8s-master01操作
# 安装证书生成工具，如果速度慢可以浏览器下载后上传至服务器改名即可
wget "https://pkg.cfssl.org/R1.2/cfssl_linux-amd64" -O /usr/local/bin/cfssl
wget "https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64" -O /usr/local/bin/cfssljson
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson

# 在master01节点生成etcd证书
cd /root/k8s-ha-install/pki
cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare /etc/etcd/ssl/etcd-ca
cfssl gencert \
   -ca=/etc/etcd/ssl/etcd-ca.pem \
   -ca-key=/etc/etcd/ssl/etcd-ca-key.pem \
   -config=ca-config.json \
   -hostname=127.0.0.1,k8s-master01,k8s-master02,k8s-master03,192.168.43.201,192.168.43.202,192.168.43.203 \
   -profile=kubernetes \
   etcd-csr.json | cfssljson -bare /etc/etcd/ssl/etcd

# 复制etcd证书到其他master节点
for i in k8s-master02 k8s-master03; do
    for FILE in etcd-ca-key.pem  etcd-ca.pem  etcd-key.pem  etcd.pem; do
      scp /etc/etcd/ssl/${FILE} $i:/etc/etcd/ssl/${FILE}
    done
done

# 生成Kubernetes集群ca证书
cd /root/k8s-ha-install/pki
cfssl gencert -initca ca-csr.json | cfssljson -bare /etc/kubernetes/pki/ca
# k8s service网段10.96.0.0/12，填入网段第一个IP；集群VIP地址192.168.43.200
cfssl gencert -ca=/etc/kubernetes/pki/ca.pem -ca-key=/etc/kubernetes/pki/ca-key.pem -config=ca-config.json -hostname=10.96.0.1,192.168.43.200,127.0.0.1,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.default.svc.cluster.local,192.168.43.201,192.168.43.202,192.168.43.203 -profile=kubernetes apiserver-csr.json | cfssljson -bare /etc/kubernetes/pki/apiserver

# 生成apiserver的第三方组件使用的聚合证书，告警可以忽略
cfssl gencert -initca front-proxy-ca-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-ca 
cfssl gencert -ca=/etc/kubernetes/pki/front-proxy-ca.pem -ca-key=/etc/kubernetes/pki/front-proxy-ca-key.pem -config=ca-config.json -profile=kubernetes front-proxy-client-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-client

# 生成controller-manager证书
cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   manager-csr.json | cfssljson -bare /etc/kubernetes/pki/controller-manager
# 设置集群信息，集群名：kubernetes  server：https://192.168.43.200:8443
# 如果不是高可用集群，192.168.43.200:8443改为master01的地址，8443改为apiserver的端口，默认是6443
kubectl config set-cluster kubernetes \
     --certificate-authority=/etc/kubernetes/pki/ca.pem \
     --embed-certs=true \
     --server=https://192.168.43.200:8443 \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
# 设置上下文信息，context为system:kube-controller-manager@kubernetes
kubectl config set-context system:kube-controller-manager@kubernetes \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
# 设置用户认证信息，用户为system:kube-controller-manager
kubectl config set-credentials system:kube-controller-manager \
     --client-certificate=/etc/kubernetes/pki/controller-manager.pem \
     --client-key=/etc/kubernetes/pki/controller-manager-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
# 设置默认集群环境
kubectl config use-context system:kube-controller-manager@kubernetes \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# 生成scheduler证书
cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   scheduler-csr.json | cfssljson -bare /etc/kubernetes/pki/scheduler
# 同样的，scheduler需要和controller-manager设置相同的集群上下文等信息
kubectl config set-cluster kubernetes \
     --certificate-authority=/etc/kubernetes/pki/ca.pem \
     --embed-certs=true \
     --server=https://192.168.43.200:8443 \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
     --client-certificate=/etc/kubernetes/pki/scheduler.pem \
     --client-key=/etc/kubernetes/pki/scheduler-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config set-context system:kube-scheduler@kubernetes \
     --cluster=kubernetes \
     --user=system:kube-scheduler \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config use-context system:kube-scheduler@kubernetes \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

# 配置admin证书，以及集群上下文等信息
cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   admin-csr.json | cfssljson -bare /etc/kubernetes/pki/admin

kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/pki/ca.pem --embed-certs=true --server=https://192.168.43.200:8443 --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config set-credentials kubernetes-admin --client-certificate=/etc/kubernetes/pki/admin.pem --client-key=/etc/kubernetes/pki/admin-key.pem --embed-certs=true --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config set-context kubernetes-admin@kubernetes --cluster=kubernetes --user=kubernetes-admin --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config use-context kubernetes-admin@kubernetes --kubeconfig=/etc/kubernetes/admin.kubeconfig

# 创建ServiceAccount密钥对
openssl genrsa -out /etc/kubernetes/pki/sa.key 2048
openssl rsa -in /etc/kubernetes/pki/sa.key -pubout -out /etc/kubernetes/pki/sa.pub

# 拷贝证书到其他master节点
for i in k8s-master02 k8s-master03; do 
    for FILE in $(ls /etc/kubernetes/pki | grep -v etcd); do 
        scp /etc/kubernetes/pki/${FILE} $i:/etc/kubernetes/pki/${FILE};
    done; 
    for FILE in admin.kubeconfig controller-manager.kubeconfig scheduler.kubeconfig; do 
        scp /etc/kubernetes/${FILE} $i:/etc/kubernetes/${FILE};
    done;
done

# 查看证书数量是否为23
ls /etc/kubernetes/pki/ | wc -l
23
```

### ETCD集群(Master)

> 如果Etcd集群服务器和Kubernetes集群服务器不重合（即独立的Etcd集群），需要根据实际情况配置集群IP。

```bash
# master01
cat << "EOF" > /etc/etcd/etcd.config.yml
name: 'k8s-master01'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.43.201:2380'
listen-client-urls: 'https://192.168.43.201:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.43.201:2380'
advertise-client-urls: 'https://192.168.43.201:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.43.201:2380,k8s-master02=https://192.168.43.202:2380,k8s-master03=https://192.168.43.203:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
EOF

# master02
cat << "EOF" > /etc/etcd/etcd.config.yml
name: 'k8s-master02'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.43.202:2380'
listen-client-urls: 'https://192.168.43.202:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.43.202:2380'
advertise-client-urls: 'https://192.168.43.202:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.43.201:2380,k8s-master02=https://192.168.43.202:2380,k8s-master03=https://192.168.43.203:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
EOF

# master03
cat << "EOF" > /etc/etcd/etcd.config.yml
name: 'k8s-master03'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.43.203:2380'
listen-client-urls: 'https://192.168.43.203:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.43.203:2380'
advertise-client-urls: 'https://192.168.43.203:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.43.201:2380,k8s-master02=https://192.168.43.202:2380,k8s-master03=https://192.168.43.203:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
EOF

# 在所有master节点创建etcd服务并启动
cat << "EOF" > /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Service
Documentation=https://coreos.com/etcd/docs/latest/
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.config.yml
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd3.service
EOF

# 所有Master节点创建etcd证书目录并链接证书，否则启动会失败
mkdir /etc/kubernetes/pki/etcd
ln -s /etc/etcd/ssl/* /etc/kubernetes/pki/etcd/
# 启动
systemctl daemon-reload && systemctl enable --now etcd

# 查看etcd集群状态如下即可
ETCDCTL_API=3 etcdctl --endpoints="192.168.43.203:2379,192.168.43.202:2379,192.168.43.201:2379" --cacert=/etc/kubernetes/pki/etcd/etcd-ca.pem --cert=/etc/kubernetes/pki/etcd/etcd.pem --key=/etc/kubernetes/pki/etcd/etcd-key.pem  endpoint status --write-out=table
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.43.203:2379 |   fd1372a073304e |   3.5.1 |   20 kB |     false |      false |         2 |          9 |                  9 |        |
| 192.168.43.202:2379 | 837ce9c47e0719eb |   3.5.1 |   20 kB |     false |      false |         2 |          9 |                  9 |        |
| 192.168.43.201:2379 | e9bf8d99824c9061 |   3.5.1 |   20 kB |      true |      false |         2 |          9 |                  9 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

### 高可用组件(Master)

```bash
# 所有master节点安装Keepalived和haproxy，并创建配置文件目录
yum install keepalived haproxy -y
mkdir /etc/haproxy
mkdir /etc/keepalived
# 为所有master节点添加haproxy配置，配置都一样，检查最后三行主机名和IP地址对应上就行
cat << "EOF" > /etc/haproxy/haproxy.cfg
global
  maxconn  2000
  ulimit-n  16384
  log  127.0.0.1 local0 err
  stats timeout 30s

defaults
  log global
  mode  http
  option  httplog
  timeout connect 5000
  timeout client  50000
  timeout server  50000
  timeout http-request 15s
  timeout http-keep-alive 15s

frontend monitor-in
  bind *:33305
  mode http
  option httplog
  monitor-uri /monitor

frontend k8s-master
  bind 0.0.0.0:8443
  bind 127.0.0.1:8443
  mode tcp
  option tcplog
  tcp-request inspect-delay 5s
  default_backend k8s-master

backend k8s-master
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server k8s-master01  192.168.43.201:6443  check
  server k8s-master02  192.168.43.202:6443  check
  server k8s-master03  192.168.43.203:6443  check
EOF
# keepalived配置不一样，注意区分网卡名、IP地址和虚拟IP地址
# 检查服务器网卡名
ip a 或 ifconfig
# k8s-master01 Keepalived配置
cat << "EOF" > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
    script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2  
    rise 1
}
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    mcast_src_ip 192.168.43.201
    virtual_router_id 51
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.43.200
    }
    track_script {
       chk_apiserver
    }
}
EOF
# k8s-master02 Keepalived配置
cat << "EOF" > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
    script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2  
    rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    mcast_src_ip 192.168.43.202
    virtual_router_id 51
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.43.200
    }
    track_script {
       chk_apiserver
    }
}
EOF
# k8s-master03 Keepalived配置
cat << "EOF" > /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
    script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2  
    rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    mcast_src_ip 192.168.43.203
    virtual_router_id 51
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.43.200
    }
    track_script {
       chk_apiserver
    }
}
EOF
# 所有master节点配置Keepalived健康检查脚本
cat << "EOF" > /etc/keepalived/check_apiserver.sh 
#!/bin/bash
err=0
for k in $(seq 1 3)
do
    check_code=$(pgrep haproxy)
    if [[ $check_code == "" ]]; then
        err=$(expr $err + 1)
        sleep 1
        continue
    else
        err=0
        break
    fi
done

if [[ $err != "0" ]]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi
EOF
# 赋予可执行权限
chmod +x /etc/keepalived/check_apiserver.sh
# 启动haproxy和Keepalived并加入开机启动
systemctl daemon-reload && systemctl enable --now haproxy && systemctl enable --now keepalived

# 检查服务是否正常
# 这种告警可以忽略：Mar 06 14:03:35 k8s-master01 haproxy-systemd-wrapper[1981]: [WARNING] 064/140335 (1982) : config : frontend 'GLOBAL' has no 'bind'
systemctl status haproxy
systemctl status keepalived

# 测试一波
telnet k8s-master-vip 8443
ping k8s-master-vip
```

### 配置集群组件

```bash
# 所有节点创建以下集群资源目录
mkdir -p /etc/kubernetes/manifests/ /etc/systemd/system/kubelet.service.d /var/lib/kubelet /var/log/kubernetes
```

**Apiserver(Master)**

- 所有master节点配置kube-apiserver服务

```bash
# service网段为10.96.0.0/12，可自行设置，其他参数按需配置即可
# master01配置
cat << "EOF" > /usr/lib/systemd/system/kube-apiserver.service 
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
      --v=2  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --advertise-address=192.168.43.201 \
      --service-cluster-ip-range=10.96.0.0/12  \
      --service-node-port-range=30000-32767  \
      --etcd-servers=https://192.168.43.201:2379,https://192.168.43.202:2379,https://192.168.43.203:2379 \
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# master02配置
cat << "EOF" >  /usr/lib/systemd/system/kube-apiserver.service 
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
      --v=2  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --advertise-address=192.168.43.202 \
      --service-cluster-ip-range=10.96.0.0/12  \
      --service-node-port-range=30000-32767  \
      --etcd-servers=https://192.168.43.201:2379,https://192.168.43.202:2379,https://192.168.43.203:2379 \
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# master03配置
cat << "EOF" > /usr/lib/systemd/system/kube-apiserver.service 
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
      --v=2  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --advertise-address=192.168.43.203 \
      --service-cluster-ip-range=10.96.0.0/12  \
      --service-node-port-range=30000-32767  \
      --etcd-servers=https://192.168.43.201:2379,https://192.168.43.202:2379,https://192.168.43.203:2379 \
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

# 启动kube-apiserver
systemctl daemon-reload && systemctl enable --now kube-apiserver
```

**ControllerManager(Master)**

所有Master节点配置kube-controller-manager服务，配置都一样

```bash
# Pod网段是172.16.0.0/12，可按需更改，不要和其他在用网段冲突即可
# 给所有master节点配置服务
cat << "EOF" > /usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
      --v=2 \
      --root-ca-file=/etc/kubernetes/pki/ca.pem \
      --cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem \
      --cluster-signing-key-file=/etc/kubernetes/pki/ca-key.pem \
      --service-account-private-key-file=/etc/kubernetes/pki/sa.key \
      --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig \
      --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \
      --leader-elect=true \
      --use-service-account-credentials=true \
      --node-monitor-grace-period=40s \
      --node-monitor-period=5s \
      --pod-eviction-timeout=2m0s \
      --controllers=*,bootstrapsigner,tokencleaner \
      --allocate-node-cidrs=true \
      --cluster-cidr=172.16.0.0/12 \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem \
      --node-cidr-mask-size=24
      
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

# 启动
systemctl daemon-reload && systemctl enable --now kube-controller-manager
```

**Scheduler(Master)**

所有Master节点配置kube-scheduler服务，配置都一样

```bash
cat << "EOF" > /usr/lib/systemd/system/kube-scheduler.service 
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
      --v=2 \
      --leader-elect=true \
      --authentication-kubeconfig=/etc/kubernetes/scheduler.kubeconfig \
      --authorization-kubeconfig=/etc/kubernetes/scheduler.kubeconfig \
      --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

# 启动
systemctl daemon-reload && systemctl enable --now kube-scheduler
```

**确认集群组件状态**

```bash
# 所有Master节点均检查，-l参数输出完整信息
# 如果有E开头的报错，需要排查解决一下，常见问题是IP冲突、证书错误
# W告警可暂时忽略
systemctl status kube-apiserver -l
systemctl status kube-controller-manager -l
systemctl status kube-scheduler -l

```

### 配置TLS Bootstrapping

只需在master01节点配置，TLS Bootstrapping的官方说明请见：[TLS Bootstrapping](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)

```bash
cd /root/k8s-ha-install/bootstrap

# 查看一下secret，name后要和token-id一致，token-id.token-secret和集群设置--token=对应上
[root@k8s-master01 bootstrap]# cat bootstrap.secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-c8ad9c
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  description: "The default bootstrap token generated by 'kubelet '."
  token-id: c8ad9c
  token-secret: 2e4d610cf3e7426e
  ...

kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/pki/ca.pem --embed-certs=true --server=https://192.168.43.200:8443 --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config set-credentials tls-bootstrap-token-user --token=c8ad9c.2e4d610cf3e7426e --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config set-context tls-bootstrap-token-user@kubernetes --cluster=kubernetes --user=tls-bootstrap-token-user --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config use-context tls-bootstrap-token-user@kubernetes --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

# 创建配置目录，拷贝admin.kubeconfig文件为config，授权kubectl，使得当前用户可以使用kubectl创建资源
# 其他master节点如果需要使用kubectl命令创建资源，也可以拷贝文件过去
# 下面是没有授权的情况
# [root@k8s-master01 bootstrap]# kubectl get cs
# The connection to the server localhost:8080 was refused - did you specify the right host or port?
mkdir -p /root/.kube;cp /etc/kubernetes/admin.kubeconfig /root/.kube/config

# 授权后查看集群状态
[root@k8s-master01 bootstrap]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true","reason":""}
etcd-1               Healthy   {"health":"true","reason":""}
etcd-2               Healthy   {"health":"true","reason":""}

# 创建bootstrap-secret
[root@k8s-master01 bootstrap]# kubectl apply -f bootstrap.secret.yaml 
secret/bootstrap-token-c8ad9c created
clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created
clusterrolebinding.rbac.authorization.k8s.io/node-autoapprove-bootstrap created
clusterrolebinding.rbac.authorization.k8s.io/node-autoapprove-certificate-rotation created
clusterrole.rbac.authorization.k8s.io/system:kube-apiserver-to-kubelet created
clusterrolebinding.rbac.authorization.k8s.io/system:kube-apiserver created
```

### 配置Kubelet

```bash
# 从master01拷贝证书文件到其他节点
cd /etc/kubernetes/
for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02; do
    ssh $i mkdir -p /etc/kubernetes/pki
    for FILE in pki/ca.pem pki/ca-key.pem pki/front-proxy-ca.pem bootstrap-kubelet.kubeconfig; do
      scp /etc/kubernetes/$FILE $i:/etc/kubernetes/${FILE}
    done
done

# 所有节点配置kubelet服务
cat << "EOF" > /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kubelet

Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
# Runtime为Containerd，kubelet服务的配置文件
cat << "EOF" > /etc/systemd/system/kubelet.service.d/10-kubelet.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig"
Environment="KUBELET_SYSTEM_ARGS=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
Environment="KUBELET_CONFIG_ARGS=--config=/etc/kubernetes/kubelet-conf.yml"
Environment="KUBELET_EXTRA_ARGS=--node-labels=node.kubernetes.io/node='' "
ExecStart=
ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_SYSTEM_ARGS $KUBELET_EXTRA_ARGS
EOF
# 创建kubelet配置文件，对应上面Environment配置KUBELET_CONFIG_ARGS
# clusterDNS配置为service网段的第十个地址
cat << "EOF" > /etc/kubernetes/kubelet-conf.yml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
cgroupsPerQOS: true
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s
EOF

# 启动服务
systemctl daemon-reload && systemctl enable --now kubelet

# 以k8s-master01为例查看运行日志和状态，只有CNI一处报错表示配置正确，后面CNI配置好后就会正常
tail -f /var/log/messages
...
Apr 21 16:15:56 k8s-master01 kubelet: E0421 16:15:56.608747    7169 kubelet.go:2475] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
...
[root@k8s-master01 ~]# systemctl status kubelet
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubelet.conf
   Active: active (running) since 五 2023-04-21 16:09:00 CST; 6min ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 7169 (kubelet)
    Tasks: 11
   Memory: 40.6M
   CGroup: /system.slice/kubelet.service
           └─7169 /usr/local/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig --kubeconfig=/etc/kuberne...

4月 21 16:14:26 k8s-master01 kubelet[7169]: E0421 16:14:26.578931    7169 kubelet.go:2475] "Container runtime network not read...alized"
4月 21 16:14:31 k8s-master01 kubelet[7169]: E0421 16:14:31.580077    7169 kubelet.go:2475] "Container runtime network not read...alized"
....

# 并且此时节点状态应该是可查询且处于NotReady，CNI配置后就会ready
[root@k8s-master01 ~]# kubectl get node
NAME           STATUS     ROLES    AGE     VERSION
k8s-master01   NotReady   <none>   7m12s   v1.26.4
k8s-master02   NotReady   <none>   7m11s   v1.26.4
k8s-master03   NotReady   <none>   7m10s   v1.26.4
k8s-node01     NotReady   <none>   7m24s   v1.26.4
k8s-node02     NotReady   <none>   7m12s   v1.26.4
```

### 配置Kube-proxy

```bash
# 只需在k8s-master01上执行
cd /root/k8s-ha-install
kubectl -n kube-system create serviceaccount kube-proxy

kubectl create clusterrolebinding system:kube-proxy --clusterrole system:node-proxier --serviceaccount kube-system:kube-proxy

SECRET=$(kubectl -n kube-system get sa/kube-proxy \
    --output=jsonpath='{.secrets[0].name}')

JWT_TOKEN=$(kubectl -n kube-system get secret/$SECRET \
--output=jsonpath='{.data.token}' | base64 -d)

PKI_DIR=/etc/kubernetes/pki
K8S_DIR=/etc/kubernetes

kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/pki/ca.pem --embed-certs=true     --server=https://192.168.43.200:8443 --kubeconfig=${K8S_DIR}/kube-proxy.kubeconfig

kubectl config set-credentials kubernetes --token=${JWT_TOKEN} --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config set-context kubernetes --cluster=kubernetes --user=kubernetes --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config use-context kubernetes --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

# 从k8s-master01将kubeconfig拷贝到其他节点
for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02; do
     scp /etc/kubernetes/kube-proxy.kubeconfig $i:/etc/kubernetes/kube-proxy.kubeconfig
done

# 所有节点配置kube-proxy服务
cat << "EOF" > /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --config=/etc/kubernetes/kube-proxy.yaml \
  --v=2

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
# 所有节点配置kube-proxy.yaml,clusterCIDR为pod网段
cat << "EOF" > /etc/kubernetes/kube-proxy.yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 172.16.0.0/12 
configSyncPeriod: 15m0s
conntrack:
  max: null
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  masqueradeAll: true
  minSyncPeriod: 5s
  scheduler: "rr"
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms
EOF
# 启动
systemctl daemon-reload && systemctl enable --now kube-proxy
```

### 配置Calico

```bash
# k8s-master01上执行，更改calico中Pod网段为自己的
cd /root/k8s-ha-install/calico/
sed -i "s#POD_CIDR#172.16.0.0/12#g" calico.yaml
# 检查是否更改成功
[root@k8s-master01 calico]# grep "IPV4POOL_CIDR" calico.yaml  -A 1
            - name: CALICO_IPV4POOL_CIDR
              value: "172.16.0.0/12"
# 应用calico
[root@k8s-master01 calico]# kubectl apply -f calico.yaml
poddisruptionbudget.policy/calico-kube-controllers created
poddisruptionbudget.policy/calico-typha created
serviceaccount/calico-kube-controllers created
serviceaccount/calico-node created
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
service/calico-typha created
daemonset.apps/calico-node created
deployment.apps/calico-kube-controllers created
deployment.apps/calico-typha created

# （中间状态）在calico生效过程中查看各节点污点状态，发现均有不可调度污点，不必管他，等待calico生效即可，这种污点是kubernetes集群保护机制，在节点处于not ready状态时，节点不可调度
[root@k8s-master01 calico]# kubectl describe node| grep Taints:
Taints:             node.kubernetes.io/not-ready:NoSchedule
Taints:             node.kubernetes.io/not-ready:NoSchedule
Taints:             node.kubernetes.io/not-ready:NoSchedule
Taints:             node.kubernetes.io/not-ready:NoSchedule
Taints:             node.kubernetes.io/not-ready:NoSchedule

# calico Pod处于Running后集群网络将建立，node将处于Ready状态，calico建立网络取决于电脑性能（硬件和网络环境），一般几分钟即可完成，性能差的可能会花费更长时间
[root@k8s-master01 calico]# kubectl get po -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS      AGE
kube-system   calico-kube-controllers-6bd6b69df9-5nwkl   1/1     Running   0             10m
kube-system   calico-node-8cvcd                          1/1     Running   0             10m
kube-system   calico-node-96mx8                          1/1     Running   1 (36s ago)   10m
kube-system   calico-node-f49rb                          1/1     Running   0             10m
kube-system   calico-node-rgs7f                          1/1     Running   0             10m
kube-system   calico-node-vzrls                          1/1     Running   0             10m
kube-system   calico-typha-77fc8866f5-h9bsj              1/1     Running   0             10m
# 查看集群状态和资源
[root@k8s-master01 calico]# kubectl get node
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    <none>   23m   v1.26.4
k8s-master02   Ready    <none>   23m   v1.26.4
k8s-master03   Ready    <none>   23m   v1.26.4
k8s-node01     Ready    <none>   23m   v1.26.4
k8s-node02     Ready    <none>   23m   v1.26.4
```

### 配置CoreDNS

- 在k8s-master01操作

```bash
# 如果更改了k8s service的网段需要将coredns的serviceIP改成k8s service网段的第十个IP
cd /root/k8s-ha-install/
COREDNS_SERVICE_IP=`kubectl get svc | grep kubernetes | awk '{print $3}'`0
sed -i "s#KUBEDNS_SERVICE_IP#${COREDNS_SERVICE_IP}#g" CoreDNS/coredns.yaml
kubectl apply -f CoreDNS/coredns.yaml 

[root@k8s-master01 k8s-ha-install]# kubectl get pod -A | grep coredns
kube-system   coredns-5db5696c7-tsqts                    1/1     Running   0          80s
```

## 配置Metrics

- 在k8s-master01操作

在新版的Kubernetes中系统资源的采集均使用Metrics-server，可以通过Metrics采集节点和Pod的内存、磁盘、CPU和网络的使用率。

```bash
cd /root/k8s-ha-install/metrics-server
kubectl  create -f . 

# 等待metrics-server部署好后，便可使用
[root@k8s-master01 metrics-server]# kubectl top nodes
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master01   186m         9%     1094Mi          58%       
k8s-master02   192m         9%     1176Mi          62%       
k8s-master03   176m         8%     1123Mi          60%       
k8s-node01     72m          3%     463Mi           24%       
k8s-node02     66m          3%     472Mi           25%  
```

## 配置Dashboard

Dashboard是一个展示Kubernetes集群资源和Pod日志，甚至可以执行容器命令的web控制台。

```bash
# 直接部署即可
cd /root/k8s-ha-install/dashboard/
kubectl apply -f .
```

```bash
# 查看dashboard端口，默认是NodePort模式，访问集群内任意节点的32486端口即可
[root@k8s-master01 ~]# kubectl get svc -A | grep dash
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.111.39.8      <none>        8000/TCP                 12m
kubernetes-dashboard   kubernetes-dashboard        NodePort    10.98.143.126    <none>        443:32486/TCP            12m
```

访问dashboard：<https://集群内任意节点IP:32486>

发现提示隐私设置错误的问题，解决方法是在Chrome浏览器启动参数加入`--test-type --ignore-certificate-errors`，再访问就没有这个提示

![20211213154024](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20211213154024.png)

```bash
# 获取登陆令牌（token）
[root@k8s-master01 dashboard]# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-cj2kt
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: c86fbde2-36ea-4dd3-94fd-8ce8012fdf22

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1411 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImFPeklobHBkNVRzZzZYVF9nbG5BMTgwOHdvMUNkV2FGbW1wdmUzZzdJRXcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWNqMmt0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJjODZmYmRlMi0zNmVhLTRkZDMtOTRmZC04Y2U4MDEyZmRmMjIiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.YgxIsaxR-hfyT9YLGdszggQ0Rvoc4SvyqswgvHz2ySc27q8lAQ7EJxhze3bhrdTL79z3J30T6vmuA5Be3kq_c2r42r2Iy-pC92t8xTISlPWEl7JfSg8GSbX2-UxUM_wqCmbMO3RWGYW5FpzrJ2pSVaeIGlu2JmYTugtS50LCFi87DmP2tDAKLQfh1NRylpEPI1AJPbl41E2wyDBUlS86YF_glUnQxyDCyrf2wJ2Akjqe7If2b9tAXHbSZBcQJFGHENymYhdBW6QObmTRxUsaOX9wdTToFcoHr-FaE4LcP9KuXhxP-gNNyVN7HN0k0WbhAp6CBIoypFCVLIN96EvNIg
```

- 选择`ALL namespace`，可以查看如下图

[![1][1]][1]

[1]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20230421165240.png

## 集群优化(可选)

Docker可在`/etc/docker/daemon.json`自定义优化配置，所有配置可见：[官方docker configuration](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)，docker常用优化配置见下方注释说明。

```bash
# （！！！如果使用docker作为Runtime的话）优化docker配置
# /etc/docker/daemon.json文件，按需配置，不需要全部都照抄，使用时删除注释，因为JSON文件不支持注释
{
  "exec-opts": ["native.cgroupdriver=systemd"], # cgroups驱动
  "registry-mirrors": ["https://ynirk4k5.mirror.aliyuncs.com"], # 镜像加速器地址
  "allow-nondistributable-artifacts": [],
  "api-cors-header": "",
  "authorization-plugins": [],
  "bip": "",
  "bridge": "",
  "cgroup-parent": "",
  "cluster-advertise": "",
  "cluster-store": "",
  "cluster-store-opts": {},
  "containerd": "/run/containerd/containerd.sock",
  "containerd-namespace": "docker",
  "data-root": "", # 数据根目录，大量docker镜像可能会占用较大存储，可以设置系统盘外的挂载盘
  "debug": true,
  "default-address-pools": [
    {
      "base": "172.30.0.0/16",
      "size": 24
    },
    {
      "base": "172.31.0.0/16",
      "size": 24
    }
  ],
  "default-cgroupns-mode": "private",
  "default-gateway": "",
  "default-gateway-v6": "",
  "default-runtime": "runc",
  "default-shm-size": "64M",
  "default-ulimits": {
    "nofile": {
      "Hard": 64000,
      "Name": "nofile",
      "Soft": 64000
    }
  },
  "dns": [],
  "dns-opts": [],
  "dns-search": [],
  "exec-root": "",
  "experimental": false,
  "features": {},
  "fixed-cidr": "",
  "fixed-cidr-v6": "",
  "group": "",
  "hosts": [],
  "icc": false,
  "init": false,
  "init-path": "/usr/libexec/docker-init",
  "insecure-registries": [],
  "ip": "0.0.0.0",
  "ip-forward": false,
  "ip-masq": false,
  "iptables": false,
  "ip6tables": false,
  "ipv6": false,
  "labels": [],
  "live-restore": true, # docker进程宕机时容器依然保持存活
  "log-driver": "json-file", # 日志格式
  "log-level": "", # 日志级别
  "log-opts": { # 日志优化
    "cache-disabled": "false",
    "cache-max-file": "5",
    "cache-max-size": "20m",
    "cache-compress": "true",
    "env": "os,customer",
    "labels": "somelabel",
    "max-file": "5", # 最大日志数量
    "max-size": "10m" # 保存的最大日志大小
  },
  "max-concurrent-downloads": 3, # pull下载并发数
  "max-concurrent-uploads": 5, # push上传并发数
  "max-download-attempts": 5,
  "mtu": 0,
  "no-new-privileges": false,
  "node-generic-resources": [
    "NVIDIA-GPU=UUID1",
    "NVIDIA-GPU=UUID2"
  ],
  "oom-score-adjust": -500,
  "pidfile": "",
  "raw-logs": false,
  "runtimes": {
    "cc-runtime": {
      "path": "/usr/bin/cc-runtime"
    },
    "custom": {
      "path": "/usr/local/bin/my-runc-replacement",
      "runtimeArgs": [
        "--debug"
      ]
    }
  },
  "seccomp-profile": "",
  "selinux-enabled": false,
  "shutdown-timeout": 15,
  "storage-driver": "",
  "storage-opts": [],
  "swarm-default-advertise-addr": "",
  "tls": true,
  "tlscacert": "",
  "tlscert": "",
  "tlskey": "",
  "tlsverify": true,
  "userland-proxy": false,
  "userland-proxy-path": "/usr/libexec/docker-proxy",
  "userns-remap": ""
}

# 无注释版
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://ynirk4k5.mirror.aliyuncs.com"],
  "containerd-namespace": "docker",
  "data-root": "",
  "debug": true,
  "default-cgroupns-mode": "private",
  "default-gateway": "",
  "default-gateway-v6": "",
  "default-runtime": "runc",
  "default-shm-size": "64M",
  "default-ulimits": {
    "nofile": {
      "Hard": 64000,
      "Name": "nofile",
      "Soft": 64000
    }
  },
  "init-path": "/usr/libexec/docker-init",
  "live-restore": true,
  "log-driver": "json-file",
  "log-level": "",
  "log-opts": {
    "cache-disabled": "false",
    "cache-max-file": "5",
    "cache-max-size": "20m",
    "cache-compress": "true",
    "env": "os,customer",
    "labels": "somelabel",
    "max-file": "5",
    "max-size": "10m"
  },
  "max-concurrent-downloads": 3,
  "max-concurrent-uploads": 5,
  "max-download-attempts": 5,
  "mtu": 0,
  "no-new-privileges": false,
  "oom-score-adjust": -500,
  "pidfile": "",
  "raw-logs": false,
  "runtimes": {
    "cc-runtime": {
      "path": "/usr/bin/cc-runtime"
    },
    "custom": {
      "path": "/usr/local/bin/my-runc-replacement",
      "runtimeArgs": [
        "--debug"
      ]
    }
  },
  "seccomp-profile": "",
  "selinux-enabled": false,
  "shutdown-timeout": 15,
  "storage-driver": "",
  "storage-opts": [],
  "swarm-default-advertise-addr": "",
  "userland-proxy-path": "/usr/libexec/docker-proxy",
  "userns-remap": ""
}


# 设置证书有效期
[root@k8s-master01 ~]# vim /usr/lib/systemd/system/kube-controller-manager.service
... # 加入下面配置
--experimental-cluster-signing-duration=876000h0m0s
...
[root@k8s-master01 ~]# systemctl daemon-reload
[root@k8s-master01 ~]# systemctl restart kube-controller-manager

# kubelet优化加密算法，默认的算法容易被漏洞扫描；增长镜像下载周期，避免有些大镜像未下载完成就被动死亡退出
# --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
# --image-pull-progress-deadline=30m
[root@k8s-master01 ~]# vim /etc/systemd/system/kubelet.service.d/10-kubelet.conf
... # 下面这行中KUBELET_EXTRA_ARGS=后加入配置
Environment="KUBELET_EXTRA_ARGS=--tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 --image-pull-progress-deadline=30m"
...

# 集群配置优化，详见https://kubernetes.io/zh/docs/tasks/administer-cluster/reserve-compute-resources/
[root@k8s-master01 ~]# vim /etc/kubernetes/kubelet-conf.yml
# 文件中添加如下配置
rotateServerCertificates: true
allowedUnsafeSysctls: # 允许在修改内核参数，此操作按情况选择，用不到就不用设置
 - "net.core*"
 - "net.ipv4.*"
kubeReserved: # 为Kubernetes集群守护进程组件预留资源，例如：kubelet、Runtime等
  cpu: "100m"
  memory: 100Mi
  ephemeral-storage: 1Gi
systemReserved: # 为系统守护进程预留资源，例如：sshd、cron等
  cpu: "100m"
  memory: 100Mi
  ephemeral-storage: 1Gi

# 为集群节点打标签，删除标签把 = 换成 - 即可
kubectl label nodes k8s-node01 node-role.kubernetes.io/node=
kubectl label nodes k8s-node02 node-role.kubernetes.io/node=
kubectl label nodes k8s-master01 node-role.kubernetes.io/master=
kubectl label nodes k8s-master02 node-role.kubernetes.io/master=
kubectl label nodes k8s-master03 node-role.kubernetes.io/master=
# 添加标签后查看集群状态
[root@k8s-master01 ~]# kubectl get node
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    master   35m   v1.26.4
k8s-master02   Ready    master   35m   v1.26.4
k8s-master03   Ready    master   35m   v1.26.4
k8s-node01     Ready    node     35m   v1.26.4
k8s-node02     Ready    node     35m   v1.26.4
```

生产环境建议ETCD集群和Kubernetes集群分离，而且使用高性能数据盘存储数据，根据情况决定是否将Master节点也作为Pod调度节点。

## 测试集群

```bash
# 测试namespace
kubectl get namespace
kubectl create namespace test
kubectl get namespace
kubectl delete namespace test

# 创建nginx实例并开放端口
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
# 查看调度状态和端口号
[root@k8s-master01 ~]# kubectl get po,svc -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
pod/nginx-748c667d99-dmtn6   1/1     Running   0          9m55s   172.25.244.194   k8s-master01   <none>           <none>

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        69m   <none>
service/nginx        NodePort    10.110.18.105   <none>        80:31687/TCP   9s    app=nginx
```

在浏览器输入<http://任意节点IP:31687/> 访问nginx，访问结果如图

![20230421165824](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20230421165824.png)

至此，基于二进制方式的Kubernetes高可用集群部署并验证成功。