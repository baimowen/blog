[^tag]: kubernetes linux

release: **Ubuntu25.04**



## 安装和部署



### 架构



![master_node](https://raw.githubusercontent.com/baimowen/blog/main/articles/17%20--%20k8s%20%E7%BC%96%E6%8E%92/master_node.svg)



>[!note]
>   对于 **containerd** 的使用以及网络设置可以查看 [[containerd 的使用]]



### 准备工作



#### 安装 containerd 和 nerdctl



从包管理安装 **containerd** 并生成默认配置



```shell
sudo apt install containerd
sudo systemctl enable --now containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
containerd --version
ctr --version
```



>[!note]
>
>   🔗参考链接：[containerd config](https://github.com/containerd/containerd/blob/main/cmd/containerd/command/config.go)



为 **containerd** 添加镜像源



```shell
awk '
/^\s*\[plugins\..io\.containerd\.grpc\.v1\.cri.\]/ { found=1 }
found && /^[[:space:]]*$/ && !added { 
   print $0
   print "    [plugins.\047io.containerd.grpc.v1.cri\047.registry.mirrors]"
   print "      [plugins.\047io.containerd.grpc.v1.cri\047.registry.mirrors.\"docker.io\"]"
   print "        endpoint = [\"https://mirror.docker.nju.edu.cn\"]"
   print ""
   added=1
   found=0
   next
}
{ print }
' /etc/containerd/config.toml > /tmp/config.toml && sudo mv /tmp/config.toml /etc/containerd/config.toml
# grep '\[plugins\..io.containerd.grpc.v1.cri.\.registry\.mirrors\]' -A3 -B1 /tmp/config.toml
```



>[!note]
>   🔗参考链接：
>
>   [1] [containerd/docs/cri/registry](https://github.com/containerd/containerd/blob/main/docs/cri/registry.md)
>
>   [2] [mirrors.nju.edu.cn](https://nju-mirror-help.njuer.org/dockerhub.html)



从 **github** 下载 **nerdctl** 二进制文件并解压



```shell
curl -LO https://github.com/containerd/nerdctl/releases/download/v2.1.6/nerdctl-2.1.6-linux-amd64.tar.gz
sudo tar -zxf nerdctl-2.1.6-linux-amd64.tar.gz -C /usr/bin
sudo nerdctl --version
```



#### 关闭内存交换



方法一：



通过修改 `/etc/fstab` 关闭内存交换



```shell
sudo sed -i.bak '/^\s*swap/ s/^/#/' /etc/fstab
```



方法二：



通过 **swapon** 关闭



```shell
# 若使用交换分区
sudo swapoff /dev/sdaX
# 若使用交换文件
sudo swapoff /path/to/swap_file
```



> [!tip]
> 查看交换分区使用 `lsblk -f`
> 
> 查看交换文件使用 `swapon --show`



检查 **swap** 状态



```shell
free -h
swapon --show
cat /proc/swaps
```



>   **临时**关闭：`sudo swapoff -a`



>[!note]
>
>   🔗参考链接：[Swap - archwiki](https://wiki.archlinuxcn.org/wiki/Swap#)



#### 加载内核模块



需要加载两个内核模块：**br_netfilter** **overlay**



* **br_netfilter**：桥接流量到 *iptables*

* **overlay**：*overlayfs*


```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```




### 安装 kubeadm、kubelet、kubectl



*   **kubeadm**

    kubeadm 是 Kubernetes 的工具之一，用于部署和管理 Kubernetes 集群。

*   **kubelet**

    kubelet 是每个节点上运行的守护进程。

*   **kubectl**

    kubectl 是 Kubernetes 的命令行客户端。



>[!note]
> 🔗参考链接：[在 Linux 系统中安装并设置 kubectl](https://kubernetes.io/zh-cn/docs/tasks/tools/install-kubectl-linux/)




#### 安装依赖



```shell
sudo apt install -y apt-transport-https ca-certificates
```



`apt-transport-https`：

-   **作用**：允许 `apt`（Ubuntu 默认的包管理工具）通过 HTTPS 协议访问软件包仓库。
-   **为什么需要**：在一些仓库中，包的下载地址是通过 HTTPS 提供的，`apt-transport-https` 是用来支持这种加密协议的。没有这个包，`apt` 就无法从 HTTPS 协议的仓库中拉取软件包。

`ca-certificates`：

-   **作用**：安装根证书，使系统能够验证 HTTPS 连接的 SSL/TLS 证书。
-   **为什么需要**：当 `apt` 通过 HTTPS 协议连接软件仓库时，需要验证服务器的证书是否可信。`ca-certificates` 包提供了一个包含公共根证书的存储库，确保你连接的仓库是安全的，不会遭遇中间人攻击。



#### 导入仓库



```shell
sudo apt install curl gpg
```



下载 kubernetes 官方标准仓库密钥



```shell
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```



添加 kubernetes 仓库到 APT 源列表



```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
```



#### 安装



```shell
sudo apt update -y
sudo apt install -y kubeadm kubelet kubectl
sudo apt-mark hold kubeadm kubelet kubectl  # 锁定版本
```



### 初始化集群



#### containerd 配置



```shell
# sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i "/\[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options\]/a\            SystemdCgroup = true" /etc/containerd/config.toml
sudo sed -i 's/disabled_plugins = \["cri"\]/disabled_plugins = []/' /etc/containerd/config.toml
sudo sed -i "/\[plugins.'io.containerd.cri.v1.images'.pinned_images\]/,/^$/ s/sandbox = .*/sandbox = 'registry.k8s.io\/pause:latest'/" /etc/containerd/config.toml  # 3.9
sudo systemctl restart containerd
```



*   `SystemdCgroup = true` → 使用 **systemd cgroup** 驱动
*   `disabled_plugins = []` → 禁用插件列表中排除 **cri(Container Runtime Interface)** 插件
*   `sandbox_image = "registry.k8s.io/pause:latest"` → 手动指定镜像版本，确保 containerd **使用预先拉取好的 pause 镜像**，避免因版本不一致或网络问题导致 Pod 无法创建。



#### kubectl 配置



配置 kubectl 的 **cgroup** 驱动



```shell
sudo mkdir -p /var/lib/kubelet
cat <<EOF | sudo tee /var/lib/kubelet/config.yaml >/dev/null
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
sudo systemctl enable --now kubelet
```



>[!note]
>
>🔗参考链接：
>
>[1] [使用 kubeconfig 文件组织集群访问](https://kubernetes.io/zh-cn/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
>
>[2] [配置 cgroup 驱动](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/)



#### 拉取镜像



拉取 kubernetes 初始化所需所有镜像：

**kube-apiserver**

**etcd**

**kube-controller-manager**

**kube-scheduler**

**kube-proxy**

**coredns**



>[!note]
>
>各组件功能见[[##kubernetes]]
>
>🔗参考链接：[Kubernetes 组件](https://kubernetes.io/zh-cn/docs/concepts/overview/components/)



```shell
sudo kubeadm config images pull \
    # --kubernetes-version=v1.30.1 \
    --image-repository=registry.aliyuncs.com/google_containers
```



拉取 **pause** 镜像



```shell
sudo nerdctl -n k8s.io pull registry.aliyuncs.com/google_containers/pause:latest  # 或使用3.9版本
```



pause 镜像的作用：**创建并维护**当前 pod 的<u>网络命名空间</u>



![node.svg](https://raw.githubusercontent.com/baimowen/blog/main/articles/17%20--%20k8s%20%E7%BC%96%E6%8E%92/node.svg)



#### 开启路由转发



启用 **ip_forward**



用于控制网络流量转发，使路由流量到容器



```shell
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.conf && sudo sysctl -p
```



#### 初始化



>[!warning]
>
>   初始化前确保宿主机**内存≥4**同时关闭**内存交换**



>[!note]
>
>   🔗参考链接：[kubeadm init](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/kubeadm-init/)



```shell
sudo kubeadm init \
  --apiserver-advertise-address=$(hostname -I | awk '{print $1}') \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --image-repository=registry.aliyuncs.com/google_containers \
  --upload-certs
```



*   `--apiserver-advertise-address` **指定 apiserver 的 ip 地址**
*   `--pod-network-cidr` **定义 pod 网络的 CIDR 范围**，在使用 calico 时是必要设置（flannel为10.244.0.0/16）
*   `--cri-socket=unix:///run/containerd/containerd.sock` **指定 containerd 作为容器运行时接口（CRI）套接字**
*   `--image-repository=registry.aliyuncs.com/google_containers` **指定镜像源仓库**（阿里源）
*   `--upload-certs` 上传证书，在集群初**始化时自动生成用于集群节点间的安全通信的证书**



>[!note]
>
>   初始化问题见：[[kubeadm 初始化问题]]



若初始化失败清理残余文件后*重新初始化*：



```shell
sudo kubeadm reset -f
sudo systemctl stop kubelet
sudo rm -rf /var/lib/kubelet/*
sudo systemctl start kubelet
sudo rm -rf /etc/cni/net.d
sudo ip link delete cni0
sudo rm -rf $HOME/.kube
sudo rm -rf /root/.kube
sudo crictl rm -a -f
sudo systemctl restart containerd
```



成功信息示例



![success_info](https://raw.githubusercontent.com/baimowen/blog/main/articles/17%20--%20k8s%20%E7%BC%96%E6%8E%92/success_info.png)



#### 加入节点到集群



根据集群初始化成功后的提示将节点加入集群



>[!warning]
>
>   准备加入集群的节点需做好节点的**[[### 准备工作]]**以及**[[### 安装 kubeadm、kubelet、kubectl]]**



```shell
kubeadm join <apiserver-advertise-address>:6443 --token ... \
  --discovery-token-ca-cert-hash sha256:...
```



### CNI



在初始化集群成功后按照提示创建对应目录结构和文件，使用 kubectl 查看 node 状态：



```shell
ubuntu@ubuntu:~$ kubectl get nodes
NAME     STATUS     ROLES           AGE     VERSION
ubuntu   NotReady   control-plane   5m44s   v1.34.1
```



以上 **NotReady** 为正常现象



使用 **Calico** 插件实现 node 间网络通信



```shell
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml
```



>[!note]
>
>🔗参考链接：
>
>[1] [kubectl](https://kubernetes.io/docs/reference/kubectl/)
>
>[2] [kubectl_apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
>
>[3] [Install Calico networking and network policy for on-premises deployments](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)



等待一分钟左右再次查看 node 状态



```shell
ubuntu@ubuntu:~$ kubectl get nodes 
NAME     STATUS   ROLES           AGE   VERSION
ubuntu   Ready    control-plane   35m   v1.34.1
```



*（可选）为 calico 换源*



```shell
sed -i 's|docker.io/calico/node|docker.m.daocloud.io/calico/node|g' calico.yaml
sed -i 's|docker.io/calico/cni|docker.m.daocloud.io/calico/cni|g' calico.yaml
sed -i 's|docker.io/calico/kube-controllers|docker.m.daocloud.io/calico/kube-controllers|g' calico.yaml
```



## kubernetes



kubernetes 是一个开源**容器编排**工具



官方架构图如下：



![Kubernetes_components](https://raw.githubusercontent.com/baimowen/blog/main/articles/17%20--%20k8s%20%E7%BC%96%E6%8E%92/Kubernetes_components.svg)



k8s 的架构是 `master-worker`，master node 负责管理整个集群，worker node 负责运行应用程序和服务



### master 节点



包含四个基本组件，其中 `kube-apiserver` 负责提供整个集群的的api接口服务，所有组件都会通过它来进行通信，是整个集群的入口，所有请求都会经过它。与 apiserver 交互可以使用 `kubectl` 客户端



其他 Control-plane 组件包括：
* Scheduler \
    调度器，负责<u>监控和调度</u> pod 的<u>资源</u>使用情况
* Controller-Manager \
    控制器管理器，负责<u>管理</u>集群中各种资源对象的<u>状态</u>，监控和检测故障并对故障进行处理
* etcd \
    键-值存储系统，用来<u>存储</u>集群中所有资源对象的<u>状态</u>信息
* Cloud-Controller-Manager \
    云控制器管理器，负责与云平台的 api 进行交互



### worker 节点



每个 worker node 都包含：



* kubelet：管理和维护 pod

* kube-proxy：为 pod 提供网络代理和负载均衡

* container-runtime：容器运行时，管理容器



>[!note]
> 常见container-runtime包括：
> * Docker-Engine
> * containerd
> * CRI-O
> * Mirantis Container Runtime



### k8s 核心组件



#### pod



**最小调度单元**，一个/多个容器的组合。具有一个内部ip地址，重建会重新分配一个ip地址



>[!tip]
>
>   一般情况下：一个pod只运行一个容器
>
>   非一般情况：日志收集、配置管理、监控等



#### svc[Service]

将**一组pod**封装成一个服务，**提供统一的入口访问**

例如：将应用程序和数据库分别封装为两个服务，应用程序通过Service访问数据库



*   内部服务：无需/无必要暴露给外部的服务

 >   eg: 缓存、消息队列、数据库等

*   外部服务：需要暴露给外部的服务

>   eg: 后端api、前端页面等



#### Ingress



管理**外部访问集群内部的入口和方式**，根据不同的转发规则，访问集群内部不同的service以及service对应的后端pod



*其他功能*：配置域名访问service、ssl证书、负载均衡等



#### ConfigMap/Secret



**封装配置信息**供service读取和使用



目的：使配置信息和service解耦，保证容器化应用的可移植性



*   ConfigMap: **明文**存储，不建议存储一些敏感信息
*   Secret: base64编码存储，需配合其他手段（如网络安全、访问控制、身份认证等）保证安全性



#### Volunmes



将数据**挂载到本地硬盘**，实现数据持久化存储



#### Deployment/StatusfulSet



将一个或多个pod封装在一起，**声明和管理Pod的生命周期**，如副本控制、滚动更新、扩容缩容等



*   Deployment
    *   适用于**无状态**服务（pod完全相同且可互换）
    *   滚动更新
    *   随机命名
*   StatusfulSet
    *   适用于**有状态**服务（需要持久化数据且具有唯一标识）
    *   稳定的网络标识符
    *   固定的命名和顺序
    *   持久化存储



Deployment ~~管理~~> replicaset ~~管理~~> pod



> replicaset 管理pod副本数量



##  管理集群




>[!note]
>
> 🔗参考链接：[kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/)
