---
categories: [Kubernetes,Parallels]
tags: Kubernetes
---

# Kubernetes生产环境-Parallels

## 环境准备
### 系统环境
```bash
root@RockyBase ~]# cat /etc/os-release 
NAME="Rocky Linux"
VERSION="9.6 (Blue Onyx)"
ID="rocky"
ID_LIKE="rhel centos fedora"
VERSION_ID="9.6"
PLATFORM_ID="platform:el9"
PRETTY_NAME="Rocky Linux 9.6 (Blue Onyx)"
ANSI_COLOR="0;32"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:rocky:rocky:9::baseos"
HOME_URL="https://rockylinux.org/"
VENDOR_NAME="RESF"
VENDOR_URL="https://resf.org/"
BUG_REPORT_URL="https://bugs.rockylinux.org/"
SUPPORT_END="2032-05-31"
ROCKY_SUPPORT_PRODUCT="Rocky-Linux-9"
ROCKY_SUPPORT_PRODUCT_VERSION="9.6"
REDHAT_SUPPORT_PRODUCT="Rocky Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="9.6"
```
### 节点环境
|       ip        |   角色   |       hostname       |  配置  |
|:---------------:|:------:|:--------------------:|:----:|
| 192.168.10.120  | vip地址  |          -           |  -   |
| 192.168.10.121  | master | rocky-k8s-master-121 | 4c8g |
| 192.168.10.122  | master | rocky-k8s-master-121 | 4c8g |
| 192.168.10.123  | master | rocky-k8s-master-121 | 4c8g |
| 192.168.10.124  |  node  |  rocky-k8s-node-121  | 4c8g |
| 192.168.10.125  |  node  |  rocky-k8s-node-121  | 4c8g |

## 环境初始化
### 配置hosts
#### 配置192.168.10.121 hosts
```bash
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6


192.168.10.120 api.k8s.local
192.168.10.121 rocky-k8s-master-121
192.168.10.122 rocky-k8s-master-122
192.168.10.123 rocky-k8s-master-123
192.168.10.124 rocky-k8s-node-124
192.168.10.125 rocky-k8s-node-125
```

#### 分发hosts
```bash
scp /etc/hosts root@192.168.10.122:/etc
scp /etc/hosts root@192.168.10.123:/etc
scp /etc/hosts root@192.168.10.124:/etc
scp /etc/hosts root@192.168.10.125:/etc
```

### 禁用防火墙
```bash
 # 关闭firewalld
 systemctl stop firewalld.service
 systemctl disable firewalld.service
 firewall-cmd --state
 # 关闭iptables 
 iptables -F
 systemctl stop iptables.service 
 service iptables save
 systemctl disable iptables.service 
```

### 禁用 SELINUX
```bash
 sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config  
 setenforce 0
 getenforce 0
 sestatus
```


### 开启内核 ipv4 转发需要加载 br_netfilter 模块,确保Kubernetes的网络流量遵循预期的网络配置和安全策略
```bash
mkdir -p /etc/modules-load.d
```
#### 加载内核模块
```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
nf_conntrack
EOF
```
- 内核模块： 
  - overlay：用于支持overlay文件系统，这在容器运行时（如Docker）中非常常见，因为它允许在不同的层（镜像和容器）之间共享文件。
  - br_netfilter：用于支持桥接网络的包过滤。当网桥接口传输数据包时，它允许这些数据包被iptables处理。

#### 设置sysctl参数
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
vm.swappiness=0
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-arptables = 1
EOF
```
- bridge-nf 使得 netfilter 可以对 Linux 网桥上的 IPv4/ARP/IPv6 包过滤。比如，设置net.bridge.bridge-nf-call-iptables＝1后，二层的网桥在转发包时也会被 iptables的 FORWARD 规则所过滤。常用的选项包括：

  - net.bridge.bridge-nf-call-arptables：是否在 arptables 的 FORWARD 中过滤网桥的 ARP 包
  - net.bridge.bridge-nf-call-ip6tables：是否在 ip6tables 链中过滤 IPv6 包
  - net.bridge.bridge-nf-call-iptables：是否在 iptables 链中过滤 IPv4 包
  - net.bridge.bridge-nf-filter-vlan-tagged：是否在 iptables/arptables 中过滤打了 vlan 标签的包。
  - net.bridge.bridge-nf-call-iptables 和 net.bridge.bridge-nf-call-ip6tables：这两个sysctl参数的设置允许桥接的网络流量调用iptables（IPv4和IPv6），这意味着通过网桥的流量将遵循在iptables中定义的规则。这样做是为了能对Pod之间的流量进行更细粒度的控制和安全管理
  - net.ipv4.ip_forward：此设置启用IPv4的IP转发功能。它允许主机转发到来而非目标为本机的数据包，这是容器技术中从一个网络接口转发数据包到另一个接口的关键

执行如下命令使修改生效：
```bash
sysctl -p /etc/sysctl.d/k8s.conf
```

#### 启用ip转发
```bash
# 这个命令直接启用内核的IP转发功能，即使不重启机器也可以立即生效。
# 这对初始化网络转发能力至关重要
echo 1 > /proc/sys/net/ipv4/ip_forward
```

#### 验证
##### 模块验证
```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

##### sysctl 设置验证
```bash
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

##### 系统生效
```bash
sudo sysctl --system
```


### 安装ipvs
```bash
cat <<EOF | tee /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
EOF

modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh

lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

- 使用lsmod | grep -e ip_vs -e nf_conntrack_ipv4命令查看是否已经正确加载所需的内核模块

#### 安装ipset包
```bash
yum install ipset
```

#### 便于查看 ipvs 的代理规则，最好安装一下管理工具 ipvsadm：
```bash
yum install ipvsadm
```

### 同步服务器时间
```bash
yum install chrony -y
systemctl enable chronyd
systemctl start chronyd
chronyc sources

Last metadata expiration check: 0:46:40 ago on Fri Sep  5 10:30:23 2025.
Package chrony-4.6.1-1.el9.aarch64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^+ time.neu.edu.cn               2   6   177    27  -3684us[-5566us] +/-   61ms
^- time.neu.edu.cn               2   6   177    27  -1671us[-3553us] +/-   53ms
^* 111.230.189.174               2   6   177    26  -5264us[-7146us] +/-   73ms
^+ tock.ntp.infomaniak.ch        1   6   177    28    -47ms[  -49ms] +/-  139ms
```

### 关闭swap
```bash
swapoff -a

sed -ri 's/.*swap.*/#&/' /etc/fstab
```

### 安装依赖
```bash
yum install -y dnf-utils  ipvsadm tar telnet  wget  net-tools  conntrack  ipset  jq  iptables  curl  sysstat  libseccomp  socat  nfs-utils  
yum install -y socat conntrack ebtables ipvsadm ipset jq iptables curl sysstat libseccomp wget vim net-tools git
```

### 安装Containerd
#### 导入yum源
```bash
sudo yum remove containerd.io
sudo yum install -y yum-utils

# 配置docker yum 源
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

```

#### 安装containerd
```bash
# 安装containerd
yum install -y containerd.io

# 删除日软件包默认的配置文件
mv /etc/containerd/config.toml /tmp

# 生成默认的配置文件
containerd config default > /etc/containerd/config.toml

# 修改内容SystemdCgroup 设置成true
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

##### 修改sandbox 下载地址
```bash
vim  /etc/containerd/config.toml

# 将sandbox_image下载地址改为阿里云地址 并且设置docker 镜像加速
[plugins."io.containerd.grpc.v1.cri"]
  # sandbox_image = "registry.k8s.io/pause:3.6"
  sandbox_image = "registry.aliyuncs.com/k8sxio/pause:3.9"
```

##### 设置dockerhub 镜像加速 可以不需要操作
```bash
vim  /etc/containerd/config.toml

# 设置dockerhub 镜像加速 可以不需要操作   
[plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"
```
```bash
# docker hub镜像加速
mkdir -p /etc/containerd/certs.d/docker.io
cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://dockerproxy.com"]
  capabilities = ["pull", "resolve"]

[host."https://docker.m.daocloud.io"]
  capabilities = ["pull", "resolve"]

[host."https://reg-mirror.qiniu.com"]
  capabilities = ["pull", "resolve"]

[host."https://registry.docker-cn.com"]
  capabilities = ["pull", "resolve"]

[host."http://hub-mirror.c.163.com"]
  capabilities = ["pull", "resolve"]

[host."https://docker.1ms.run"]
  capabilities = ["pull", "resolve"]
[host."https://docker.mybacc.com"]
  capabilities = ["pull", "resolve"]
[host."https://dytt.online"]
  capabilities = ["pull", "resolve"]
[host."https://lispy.org"]
  capabilities = ["pull", "resolve"]
[host."https://docker.xiaogenban1993.com"]
  capabilities = ["pull", "resolve"]
[host."https://docker.yomansunter.com"]
  capabilities = ["pull", "resolve"]
[host."https://aicarbon.xyz"]
  capabilities = ["pull", "resolve"]
[host."https://666860.xyz"]
  capabilities = ["pull", "resolve"]
[host."https://docker.zhai.cm"]
  capabilities = ["pull", "resolve"]
[host."https://a.ussh.net"]
  capabilities = ["pull", "resolve"]
[host."https://hub.littlediary.cn"]
  capabilities = ["pull", "resolve"]
[host."https://hub.rat.dev"]
  capabilities = ["pull", "resolve"]
[host."https://docker.m.daocloud.io"]
  capabilities = ["pull", "resolve"]

EOF

# registry.k8s.io镜像加速
mkdir -p /etc/containerd/certs.d/registry.k8s.io
tee /etc/containerd/certs.d/registry.k8s.io/hosts.toml << 'EOF'
server = "https://registry.k8s.io"

[host."https://k8s.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# docker.elastic.co镜像加速
mkdir -p /etc/containerd/certs.d/docker.elastic.co
tee /etc/containerd/certs.d/docker.elastic.co/hosts.toml << 'EOF'
server = "https://docker.elastic.co"

[host."https://elastic.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# gcr.io镜像加速
mkdir -p /etc/containerd/certs.d/gcr.io
tee /etc/containerd/certs.d/gcr.io/hosts.toml << 'EOF'
server = "https://gcr.io"

[host."https://gcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# ghcr.io镜像加速
mkdir -p /etc/containerd/certs.d/ghcr.io
tee /etc/containerd/certs.d/ghcr.io/hosts.toml << 'EOF'
server = "https://ghcr.io"

[host."https://ghcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# k8s.gcr.io镜像加速
mkdir -p /etc/containerd/certs.d/k8s.gcr.io
tee /etc/containerd/certs.d/k8s.gcr.io/hosts.toml << 'EOF'
server = "https://k8s.gcr.io"

[host."https://k8s-gcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# mcr.m.daocloud.io镜像加速
mkdir -p /etc/containerd/certs.d/mcr.microsoft.com
tee /etc/containerd/certs.d/mcr.microsoft.com/hosts.toml << 'EOF'
server = "https://mcr.microsoft.com"

[host."https://mcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# nvcr.io镜像加速
mkdir -p /etc/containerd/certs.d/nvcr.io
tee /etc/containerd/certs.d/nvcr.io/hosts.toml << 'EOF'
server = "https://nvcr.io"

[host."https://nvcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# quay.io镜像加速
mkdir -p /etc/containerd/certs.d/quay.io
tee /etc/containerd/certs.d/quay.io/hosts.toml << 'EOF'
server = "https://quay.io"

[host."https://quay.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# registry.jujucharms.com镜像加速
mkdir -p /etc/containerd/certs.d/registry.jujucharms.com
tee /etc/containerd/certs.d/registry.jujucharms.com/hosts.toml << 'EOF'
server = "https://registry.jujucharms.com"

[host."https://jujucharms.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# rocks.canonical.com镜像加速
mkdir -p /etc/containerd/certs.d/rocks.canonical.com
tee /etc/containerd/certs.d/rocks.canonical.com/hosts.toml << 'EOF'
server = "https://rocks.canonical.com"

[host."https://rocks-canonical.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```

##### 创建存储挂载
```bash
mkdir -p /var/lib/containerd/ mkdir -p /apps/containerd/ 
# 改成你大硬盘路径 
/etc/fstab 

echo "/apps/containerd /var/lib/containerd none defaults,bind,nofail 0 0" >>/etc/fstab 

systemctl daemon-reload 

# 挂载 
mount -a 

# 查看是否挂载 
mount | grep containerd /dev/vda3 on /var/lib/containerd type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota) 
```

##### 启动
```bash
systemctl daemon-reload
systemctl enable containerd --now
```

##### 可选
```bash
# 可选

sudo vim /etc/containerd/config.toml
 
## 注释掉 or 将cri从数组中去除：disabled_plugins = ["cri"]
 
# 重启containerd
sudo systemctl restart containerd
```

##### 安装crictl
```bash
VERSION="v1.34.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-arm64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-arm64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-arm64.tar.gz
```

##### 创建crictl配置
```bash
cat <<EOF | tee /etc/crictl.yaml
runtime-endpoint: "unix:///var/run/containerd/containerd.sock"
image-endpoint: "unix:///var/run/containerd/containerd.sock"
timeout: 10
debug: false
pull-image-on-create: true
disable-pull-on-run: false
EOF
```

##### 查看配置是否生效 
```bash
crictl info|  grep sandboxImage
     "sandboxImage": "registry.aliyuncs.com/google_containers/pause:3.6",
crictl info|  grep SystemdCgroup
            "SystemdCgroup": true 
```


## 负载均衡
- 为 apiserver 提供负载均衡器有很多方法，比如传统的 haproxy+keepalived，或者使用 nginx 代理也可以，这里我们使用一个比较新颖的工具 kube-vip。
- kube-vip 可以在你的控制平面节点上提供一个 Kubernetes 原生的 HA 负载均衡，我们不需要再在外部设置 HAProxy 和 Keepalived 来实现集群的高可用了。
- 在以前我们在私有环境下创建 Kubernetes 集群时，我们需要准备一个硬件/软件的负载均衡器来创建多控制面集群，更多的情况下我们会选择使用 HAProxy + Keepalived 来实现这个功能。一般情况下我们创建2个负载均衡器的虚拟机，然后分配一个 VIP，然后使用 VIP 为负载均衡器提供服务，通过 VIP 将流量重定向到后端的某个 Kubernetes 控制器平面节点上。
</br>不使用vip
<img src="https://picdn.youdianzhishi.com/images/20210616142006.png">
使用vip
<img src="https://picdn.youdianzhishi.com/images/20210616142207.png">
- kube-vip 可以通过静态 pod 运行在控制平面节点上，这些 pod 通过 ARP 会话来识别每个节点上的其他主机，我们可以选择 BGP 或 ARP 来设置负载平衡器，这与 Metal LB 比较类似。在 ARP 模式下，会选出一个领导者，这个节点将继承虚拟 IP 并成为集群内负载均衡的 Leader，而在 BGP 模式下，所有节点都会通知 VIP 地址。
- 集群中的 Leader 将分配 vip，并将其绑定到配置中声明的选定接口上。当 Leader 改变时，它将首先撤销 vip，或者在失败的情况下，vip 将直接由下一个当选的 Leader 分配。当 vip 从一个主机移动到另一个主机时，任何使用 vip 的主机将保留以前的 vip <-> MAC 地址映射，直到 ARP 过期（通常是30秒）并检索到一个新的 vip <-> MAC 映射，这可以通过使用无偿的 ARP 广播来优化。
- kube-vip 可以被配置为广播一个无偿的 arp（可选），通常会立即通知所有本地主机 vip <-> MAC 地址映射已经改变。
<img src="https://picdn.youdianzhishi.com/images/20210616142509.png">

### 部署vip
使用 kube-vip 来实现集群的高可用，首先在 master1 节点上生成基本的 Kubernetes 静态 Pod 资源清单文件
#### 创建lb master1 节点
```bash

# 官方文档: https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#kube-vip
# 可使用镜像  juestnow/kube-vip:v0.6.4
# KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
export KVVERSION='v0.6.4'
export VIP=192.168.10.120
export INTERFACE='enp0s5'

# 下载镜像
ctr images pull docker.io/plndr/kube-vip:$KVVERSION
 
# 执行命令创建yaml
ctr run --rm --net-host docker.io/plndr/kube-vip:$KVVERSION vip \
/kube-vip manifest pod \
--interface $INTERFACE \
--vip $VIP \
--controlplane \
--services \
--arp \
--leaderElection | tee  /etc/kubernetes/manifests/kube-vip.yaml


# 修改后内容
cat  /etc/kubernetes/manifests/kube-vip.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-vip
  namespace: kube-system
spec:
  containers:
  - args:
    - manager
    env:
    - name: vip_arp
      value: "true"
    - name: port
      value: "6443"
    - name: vip_interface
      value: enp0s5
    - name: vip_cidr
      value: "32"
    - name: cp_enable
      value: "true"
    - name: cp_namespace
      value: kube-system
    - name: vip_ddns
      value: "false"
    - name: svc_enable
      value: "true"
    - name: svc_leasename
      value: plndr-svcs-lock
    - name: vip_leaderelection
      value: "true"
    - name: vip_leasename
      value: plndr-cp-lock
    - name: vip_leaseduration
      value: "5"
    - name: vip_renewdeadline
      value: "3"
    - name: vip_retryperiod
      value: "1"
    - name: vip_address
      value: 192.168.10.120
    - name: prometheus_server
      value: :2112
    image: ghcr.io/kube-vip/kube-vip:v0.6.4
    imagePullPolicy: Always
    name: kube-vip
    resources: {}
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_RAW
    volumeMounts:
    - mountPath: /etc/kubernetes/admin.conf
      name: kubeconfig
  hostAliases:
  - hostnames:
    - kubernetes
    ip: 127.0.0.1
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/admin.conf
    name: kubeconfig
status: {}
```

## 初始化控制平面
### 配置k8s仓库
```bash
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/repodata/repomd.xml.key
EOF
```

### 安装k8s组件
```bash
yum install -y kubelet-1.28.15 kubeadm-1.28.15 kubectl-1.28.15
```

### master 节点设置开机启动
```bash
systemctl enable --now kubelet
```

### 初始化配置
```bash
kubeadm config print init-defaults --component-configs KubeletConfiguration > kubeadm.yaml
```
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.10.121  # 指定当前节点内网IP
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/containerd/containerd.sock  # 使用 containerd的Unix socket 地址
  imagePullPolicy: IfNotPresent
  name: rocky-k8s-master-121   # 修改节点名称
  taints: 
  - effect: "NoSchedule"
    key: "node-role.kubernetes.io/master"
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs  # kube-proxy 模式
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.28.15
controlPlaneEndpoint: api.k8s.local:6443  # 设置控制平面Endpoint地址
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
  certSANs:    # 添加其他master节点的相关信息
  - api.k8s.local
  - rocky-k8s-master-121
  - rocky-k8s-master-122
  - rocky-k8s-master-123
  - 192.168.10.121
  - 192.168.10.122
  - 192.168.10.123
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16    # 指定 pod 子网
scheduler: {}
---
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
containerRuntimeEndpoint: ""
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
logging:
  flushFrequency: 0
  options:
    json:
      infoBufferSize: "0"
  verbosity: 0
memorySwap: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
rotateCertificates: true
runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
```
这里需要注意的是我们在 ClusterConfiguration 块的配置中新增了控制平面的地址以及将 api.k8s.local 这个域名加入到了证书签名中，该域名将映射到 vip：

```yaml
controlPlaneEndpoint: api.k8s.local:6443  # 设置控制平面Endpoint地址
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
  certSANs:    # 添加其他master节点的相关信息
  - api.k8s.local
  - rocky-k8s-master-121
  - rocky-k8s-master-122
  - rocky-k8s-master-123
  - 192.168.10.121
  - 192.168.10.122
  - 192.168.10.123
```

### 执行初始化操作
```bash
[root@rocky-k8s-master-121 ~]# kubeadm config images pull --config kubeadm.yaml
W0905 14:04:54.104802    5694 initconfiguration.go:307] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta3", Kind:"ClusterConfiguration"}: strict decoding error: yaml: unmarshal errors:
  line 16: key "apiServer" already set in map
W0905 14:04:54.106354    5694 initconfiguration.go:120] Usage of CRI endpoints without URL scheme is deprecated and can cause kubelet errors in the future. Automatically prepending scheme "unix" to the "criSocket" with value "/var/run/containerd/containerd.sock". Please update your configuration!
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-apiserver:v1.28.15
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-controller-manager:v1.28.15
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-scheduler:v1.28.15
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-proxy:v1.28.15
[config/images] Pulled registry.aliyuncs.com/google_containers/pause:3.9
[config/images] Pulled registry.aliyuncs.com/google_containers/etcd:3.5.15-0
[config/images] Pulled registry.aliyuncs.com/google_containers/coredns:v1.10.1
[root@rocky-k8s-master-121 ~]# kubeadm init --upload-certs --config kubeadm.yaml
W0905 14:11:10.298064    6192 initconfiguration.go:307] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta3", Kind:"ClusterConfiguration"}: strict decoding error: yaml: unmarshal errors:
  line 16: key "apiServer" already set in map
W0905 14:11:10.299116    6192 initconfiguration.go:120] Usage of CRI endpoints without URL scheme is deprecated and can cause kubelet errors in the future. Automatically prepending scheme "unix" to the "criSocket" with value "/var/run/containerd/containerd.sock". Please update your configuration!
[init] Using Kubernetes version: v1.28.15
[preflight] Running pre-flight checks
        [WARNING Hostname]: hostname "node" could not be reached
        [WARNING Hostname]: hostname "node": lookup node on 192.168.10.1:53: no such host
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W0905 14:11:10.587782    6192 checks.go:835] detected that the sandbox image "registry.aliyuncs.com/k8sxio/pause:3.9" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.aliyuncs.com/google_containers/pause:3.9" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [api.k8s.local kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local node rocky-k8s-master-121 rocky-k8s-master-122 rocky-k8s-master-123] and IPs [10.96.0.1 192.168.10.121 192.168.10.122 192.168.10.123]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost node] and IPs [192.168.10.121 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost node] and IPs [192.168.10.121 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 6.620684 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
9610d3d78e962b12dc45dccd3f2fd18e1839c8c9aedcb9144d9ed918ada91601
[mark-control-plane] Marking the node node as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node node as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: abcdef.0123456789abcdef
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join api.k8s.local:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:3842d92bc1517acc7918415cc52c0d8b58e9c4e2df62c71b7c94438f1d93f1ee \
        --control-plane --certificate-key 9610d3d78e962b12dc45dccd3f2fd18e1839c8c9aedcb9144d9ed918ada91601

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join api.k8s.local:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:3842d92bc1517acc7918415cc52c0d8b58e9c4e2df62c71b7c94438f1d93f1ee
```

### 添加master节点
```bash
kubeadm join api.k8s.local:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:3842d92bc1517acc7918415cc52c0d8b58e9c4e2df62c71b7c94438f1d93f1ee \
        --control-plane --certificate-key 9610d3d78e962b12dc45dccd3f2fd18e1839c8c9aedcb9144d9ed918ada91601
```

### 添加node节点
```bash
kubeadm join api.k8s.local:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:3842d92bc1517acc7918415cc52c0d8b58e9c4e2df62c71b7c94438f1d93f1ee
```

### 本地权限
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 安装calico
```bash
# 页面访问地址
https://github.com/projectcalico/calico/blob/v3.27.5/manifests/calico.yaml

# 手动下载
calico/manifests/calico.yaml

# 上传至master
kubectl apply -f calico.yaml
```

### 在其他master节点部署vip
```bash
export KVVERSION='v0.6.4'
export VIP=192.168.10.120
export INTERFACE='enp0s5'

# 下载镜像
ctr images pull docker.io/plndr/kube-vip:$KVVERSION

ctr run --rm --net-host docker.io/plndr/kube-vip:$KVVERSION vip \
/kube-vip manifest pod \
--interface $INTERFACE \
--vip $VIP \
--controlplane \
--services \
--arp \
--leaderElection | tee  /etc/kubernetes/manifests/kube-vip.yaml
```

## 安装Metallb和Ingress
### Metallb
```bash
kubectl edit configmap -n kube-system kube-proxy

mode: "" --> mode: "ipvs"
strictARP: fasle --> strictARP: true

wget https://github.com/metallb/metallb/blob/v0.14.9/config/manifests/metallb-native.yaml


```


## 请理
```bash
kubeadm reset
ifconfig cni0 down && ip link delete cni0
ifconfig flannel.1 down && ip link delete flannel.1
rm -rf /var/lib/cni/
```



 

 
