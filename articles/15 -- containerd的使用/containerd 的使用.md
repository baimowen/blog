[^tag ]: container

## 安装



### 官方二进制文件



[containerd 1.17.28](https://github.com/containerd/containerd/releases/download/v1.7.28/containerd-1.7.28-linux-amd64.tar.gz)



### arch



```shell

sudo pacman -S --noconfirm containerd runc

sudo systemctl enable --now containerd.services

```



### nixos



```nix

{

  ...
  
  users.users.<username> = {
    isNormalUser = true;
    extraGroups = [ "wheel" "containerd" ];  # Enable ‘sudo’ for the user.
    packages = with pkgs; [];
  };
  
  virtualisation.containerd.enable = true;

}

```



临时使用



```shell

nix-shell -p containerd

```



## 使用containerd



### 镜像操作



#### 查看



```shell

ctr images list

# or

ctr i ls

```



#### 拉取



```shell

ctr images pull [--option] <repository>/<img_name>:<release>

# eg: ctr images pull docker.io/library/nginx:latest

```



> [!note]
> 
> 通过 `ctr images pull --help`  查看更多参数



#### 删除



```shell

ctr images rm <repository>/<img_name>:<release>

```



#### 挂载



```shell

ctr images mount <repository>/<img_name>:<release> <mount_dir>

# eg: ctr images mount docker.io/library/nginx:alpine /mnt

```



卸载



```shell
umount /mnt
```



#### 导出


```shell

ctr images export <custom_img_name> <repository>/<img_name>:<release>

```



#### 导入



```shell

ctr images import <custom_img_name>

```




#### 修改tag



```shell

ctr images tag <old_tag> <new_tag>

# eg: ctr images tag docker.io/library/nginx:alpine nginx:alpine

```



> [!tip]
> 
> 可以使用 `ctr images check` 命令检查对比修改前后的镜像



### 容器操作



#### 查看



```shell

ctr container ls

# or

ctr c ls

```



#### 查看任务



查看在容器当中运行中的进程



```shell

ctr task ls

# or 

ctr t ls

ctr task ps <custom_container_name>  # 查看指定容器中进程的pid以及输出示例
PID     INFO
1767    -

```



> [!tip]
> 
> 容器中的进程也是宿主机内的进程
> 
> ```shell
> ps -ef | awk 'NR==1 || /nginx/ {print}'
> UID          PID    PPID  C STIME TTY          TIME CMD
> root        1743       1  0 21:37 pts/1    00:00:00 /nix/store/2l7yrsmwbp4hf1yfzr3d2r3pllhly19p-containerd-2.1.4/bin/containerd-shim-runc-v2 -namespace default -id nginx -address /run/containerd/containerd.sock
> root        1767    1743  0 21:37 ?        00:00:00 nginx: master process nginx -g daemon off;  # 这个是主进程
> 101         1803    1767  0 21:37 ?        00:00:00 nginx: worker process  # 这个是工作进程
> 101         1804    1767  0 21:37 ?        00:00:00 nginx: worker process
> ```



#### 创建*静态*容器



```shell

ctr container create <repository>/<img_name>:<release> <custom_container_name>

```



> [!caution]
> 
> 创建容器前一定要先拉取镜像，否则无法创建



查看容器详细信息



```shell

ctr container info <custom_container_name>

```



> [!tip]
> 
> 静态容器中无任务



#### 静态容器启动为*动态*容器



```shell

ctr tast start -d <custom_container_name>

```



* `-d`: daemon，后台



> [!warning]
> 
> 直接使用 containerd 启动动态容器会报错，原因*并未使 containerd 与 runc 通信*
> 
> 解决方法：**安装 runc**




在 NixOS 的 nix-shell 中，按如下操作手动使 containerd 与 runc 通信



```shell

[nix@nixos:~]$ nix-shell -p containerd runc

[nix-shell:~]$ which containerd runc  # 类似如下输出
/nix/store/2l7yrsmwbp4hf1yfzr3d2r3pllhly19p-containerd-2.1.4/bin/containerd
/nix/store/w6yqkflfwkm5rz6xrnsh76249rdgn9na-runc-1.3.0/bin/runc

[nix-shell:~]$ mkdir -p /tmp/containerd

[nix-shell:~]$ # 创建临时配置文件并应用
cat > /tmp/containerd/config.toml << 'EOF'
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = "/nix/store/$(ls /nix/store | grep runc | head -1)/bin/runc"
EOF

[nix-shell:~]$ sudo containerd -c /tmp/containerd/config.toml &  # 指定临时配置文件

```



#### 进入运行中的容器



```shell

ctr task exec --exec-id $RANDOM -t <custom_container_name> /bin/sh

```



> [!tip]
> 
> `exec-id` 的值任意，保证唯一即可。`$RANDOM` 为随机数变量
> 
> `/bin/sh` 可以是任意 shell 命令



#### 直接创建动态容器



```shell

ctr run -d <repository>/<img_name>:<release> <custom_container_name>

```



> [!tip]
> 
> `ctr run -d --net-host` 网络模式为 host，即继承宿主机ip



#### 暂停容器



```shell

ctr task pause <custom_container_name>

```



#### 恢复容器



```shell

ctr task resume <custom_container_name>

```



> [!tip]
> 
> 恢复暂停的容器后，容器内进程 pid *不会发生变化*



#### 停止容器



```shell

ctr task kill <custom_container_name>  # 输出示例如下
ctr task ls
TASK    PID    STATUS
nginx   1767   RUNNING
nginx2  2263   STOPPED

```



删除容器内进程



```shell

ctr task delete/del/remove/rm <custom_container_name>  # 输出示例如下
ctr task ls
TASK    PID    STATUS
nginx   1767   RUNNING

```



> [!note]
> 
> `ctr task delete/del/rm/remove` 并不会删除容器，仅会删除对应容器内的进程



#### 删除容器



```shell

ctr container delete/del/remove/rm <custom_container_name>

```



> [!tip]
> 
> 和 docker 一样，在容器运行中**不可**删除容器



### 命名空间



**不同命名空间**内即使**相同容器名称**也不会冲突



```shell

ctr namespace <command> [options]

```

Command:

*   create,c    创建
*   remove,rm   移除
*   list,ls     列出
*   label       设置/清除标签



>[!note]
>
>   在**指定**命名空间内操作：`ctr -n/--namespace <command> [options] `
>
>   *eg*: `ctr -n default images ls` 



#### 共享命名空间



多个容器**逻辑上**在同一台机器中运行



##### 分类



-   共享**网络**命名空间

    容器之间可通过 localhost 互相通信

-   共享**进程**命名空间：

    容器之间进程共享



1.   共享网络



```shell

ctr run --net-host <target>

# or

ctr run --with-ns network:/proc/$pid/ns/net <target>

```



2.   共享进程



```shell

ctr run --with-ns pid:/proc/$pic/ns/pid <target>

```



>[!note]
>
>   `$pid` 获取方法：
>
>    宿主机运行：`ctr tasks ls | grep <container_name> | awk {print $2}`



### 挂载卷



将容器目录挂载到宿主机目录



```shell

ctr container create <target> <container_name> \
  --mount type=bind,src=<host_dir>,dst=<container_dir>,options=rbind,rw
  
```



## CNI 规范



CNI: Containerd Network Interface



plugins download link: 

[go-cni](https://github.com/containernetworking/cni/archive/refs/tags/v1.3.0.tar.gz)

[cni-plugins](https://github.com/containernetworking/plugins/releases/download/v1.8.0/cni-plugins-linux-amd64-v1.8.0.tgz)

流程参考：[fedora](./Untitled.txt)


### 配置



配置文件路径：`/etc/cni/net.d/*.conf`



nat网桥示例：`/etc/cni/net.d/10-cni0.conf`



```json
{
  "cniVersion": "1.0.0",
  "name": "cni0",  // 唯一标识符
  "type": "bridge",  // cni 桥接插件
  "bridge": "cni0",  // 在宿主机创建一个名为 cni0 的网桥设备
  "isGateway": true,
  "ipMasq": true,  // 容器共享主机ip，即 NAT 模式
  "ipam": {  // IP Address Manager
    "type": "host-local",  // dhcp
    "ranges": [
      [
        {
          "subnet": "10.66.0.0/16",  // 容器子网
          "gateway": "10.66.0.1"
        }
      ]
    ],
    "routes": [
      { "dst": "0.0.0.0/0" }  // 容器默认路由
    ]
  },
  "dns": {
    "nameservers": ["8.8.8.8", "114.114.114.114"]
  }
}
```



回环（127.0.0.1/localhost）网桥示例：`/etc/cni/net.d/99-loopback.conf`



```json
{
  "cniVersion": "0.4.0",
  "name": "lo",
  "type": "loopback"
}
```



>[!note]
>
>   `10-/99-` 数字前缀控制加载顺序，顺序加载。修改配置后记得**重启 containerd 服务**



检查语法问题 `jq . /etc/cni/net.d/*.conf`



修改 containerd 配置：`/etc/containerd/config.toml`



```shell
sudo vim /etc/containerd/config.toml
version = 2

[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"  # 修改为cni路径
      conf_dir = "/etc/cni/net.d"
  [plugins."io.containerd.internal.v1.opt"]
    path = "/var/lib/containerd/opt"
```



### 生成网络



生成网络的脚本文件：`cni/script/priv-net-run.sh`

>   
>
>   脚本依赖于同目录下 `exec-plugins.sh`，以及 jq，需自行安装 jq
>  
>  



运行脚本

```shell

arch@archlinux:~/cni-1.3.0/scripts$ sudo -E CNI_PATH=/home/arch/cni/cni-plugins/ ./priv-net-run.sh

# or

export CNI_PATH=/home/arch/cni/cni-plugins/
arch@archlinux:~/cni-1.3.0/scripts$ sudo -E sh ./priv-net-run.sh

```



>[!note]
>
>   默认 `CNI_PATH` 路径为 `/opt/cni/bin`，若移动 `cni-plugins/*` 到 `/opt/cni/bin` 下则无需设置环境变量
>



检查网桥是否生成： `ip addr`

删除网桥：`ip link del cni0`



#### 测试 



测试 CNI 是否成功运行：[cnitool](https://www.cni.dev/docs/cnitool/)



或使用 **bridge**



```shell

sudo -i

ip netns add testns

# add
CNI_PATH=/opt/cni/bin \
CNI_IFNAME=eth0 \
CNI_NETNS=/var/run/netns/testns \
CNI_COMMAND=ADD \
CNI_CONTAINERID=test123 \
/opt/cni/bin/bridge < /etc/cni/net.d/10-cni0.conf

```



若没有问题会看到如下输出



```json
{
    "cniVersion": "1.0.0",
    "dns": {
        "nameservers": [
            "8.8.8.8",
            "114.114.114.114"
        ]
    },
    "interfaces": [
        {
            "mac": "36:5a:f3:c0:a3:b3",
            "name": "cni0"
        },
        {
            "mac": "e6:bf:aa:d7:12:8a",
            "name": "veth40b2e7ed"
        },
        {
            "mac": "b6:28:5d:2a:54:3e",
            "name": "eth0",
            "sandbox": "/var/run/netns/testns"
        }
    ],
    "ips": [
        {
            "address": "10.66.0.2/16",
            "gateway": "10.66.0.1",
            "interface": 2
        }
    ],
    "routes": [
        {
            "dst": "0.0.0.0/0"
        }
    ]
}
```



清理测试配置



```shell
# del
CNI_PATH=/opt/cni/bin \
CNI_IFNAME=eth0 \
CNI_NETNS=/var/run/netns/testns \
CNI_COMMAND=DEL \
CNI_CONTAINERID=test123 \
/opt/cni/bin/bridge < /etc/cni/net.d/10-cni0.conf

# 清理命名空间
ip netns ls
ip netns del testns

# 清理 ipam
ls -la /var/lib/cni/networks/cni0
rm -f /var/lib/cni/networks/cni0/*

```



### 为容器配置网络


####  方法一



为*已创建*的容器配置网桥设备




```shell

export CNI_PATH=cni-plugins/
sh cni/script/exec-plugins.sh add $pid $netnspath

```



>[!note]
>
>   netnspath 路径：`netnspath=/proc/$pid/ns/net`



#### 方法二



通过 **nerdctl** 运行容器时指定网络参数



>   
>   [nerdctl2.1.6](https://github.com/containerd/nerdctl/releases/download/v2.1.6/nerdctl-2.1.6-linux-amd64.tar.gz) 从此处下载 nerdctl 二进制文件，解压后将二进制文件移动到 /usr/bin 目录下
>   



```shell

sudo nerdctl network ls
NETWORK ID      NAME      FILE
                cni0      /etc/cni/net.d/10-cni0.conf
                lo        /etc/cni/net.d/99-lo.conf

sudo nerdctl run -d --name <container_name> --network cni0 <target>

```



直接指定 `--network` 会提示在 containerd 的默认 `CNI_PATH=/opt/cni/bin` 中找不到 CNI 桥接插件



```shell

sudo -i

mkdir -p /opt/cni/bin && cp cni/cni-plugins/* /opt/cni/bin/

# 清理环境
rm -rf /etc/cni/net.d/nerdctl-bridge.conflist 
rm -rf /etc/cni/net.d/.nerdctl.lock 
rm -rf /etc/cni/net.d/default

nerdctl rm <container_name>

ip link del cni0

```



进入容器验证：



```

ctr tasks exec --exec-id $RANDOM -t <container_name> /bin/sh

/ # pinc -c 2 8.8.8.8

```



测试 http 服务



```

/ # echo "containerd net web test" > /tmp/index.html

/ # httpd -h /tmp

/ # wget -O - -q 127.0.0.1

```



>[!note]
>
>   `wget -O - -q <address>`：静默下载内容输出到终端，不保存文件
>
>   *   `-`: 下载内容输出到 **标准输出（stdout）**



## 自建仓库



```shell

ctr images pull <target>

ctr images tag <old_target> <garbor.domain/library/container_target>

ctr images push [--plain-http] -u <user_name> <passwd> <harbor.domain/library/container_target>

```



步骤：

1.   **拉取**镜像
2.   **修改**标签
3.   **推送**到仓库（仓库使用什么协议从仓库拉取镜像时就用什么协议）



>[!note]
>
>   使用自建仓库首先进行**主机名映射（/etc/hosts）**
