# kubeadm 初始化问题

[^tag ]: kubernetes



若初始化失败: 

```bash
# 重置集群
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d $HOME/.kube/config

# 重新初始化
sudo kubeadm init \
 --pod-network-cidr=10.244.0.0/16 \
 --cri-socket=unix:///run/containerd/containerd.sock \
 --apiserver-advertise-address=$(hostname -I | awk '{print $1}')  # 显式指定 API Server 地址
 --upload-certs \
 --control-plane-endpoint=$(hostname -I | awk '{print $1}')

# 重新配置 kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl cluster-info  # 应显示 API Server 地址和 CoreDNS 状态
kubectl get nodes     # 节点状态应为 Ready（安装网络插件后）
```

---

若出现ERRO[0000] validate service connection: 

```bash
sudo tee /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
sudo crictl ps -a | grep apiserver
```

---

初始化成功提示:

```text
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

 kubeadm join 192.168.99.110:6443 --token z0m3oq.gjwxzywvxdr960e8 \
       --discovery-token-ca-cert-hash sha256:845897e0cc83699623c9dddb4fc8f24e6b8f4fa1a96388b45dff102e0d8d243b \
       --control-plane --certificate-key b5c97bb1634cad385f7e4ed1ab0e554b69e99220f30751971e2c1e673d408c0c

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.99.110:6443 --token z0m3oq.gjwxzywvxdr960e8 \
       --discovery-token-ca-cert-hash sha256:845897e0cc83699623c9dddb4fc8f24e6b8f4fa1a96388b45dff102e0d8d243b
```

---

初始化后检查 `kubectl -n kube-system describe pod -l k8s-app=calico-node`, 若出现: 

Warning  Failed     104s                 kubelet            Error: failed to generate container "a65d012b3666e88c20abf7876bfc0dc3105e5ebf91333c4cb39499c1ede431c7" spec: failed to generate spec: failed to mkdir "/var/lib/calico": mkdir /var/lib/calico: file exists

检查目录中文件情况
```shell
ls -l /var/lib | grep calico
lrwxrwxrwx  1 root root      41 May 17 14:13 calico -> /var/snap/microk8s/current/var/lib/calico  # 符号链接
-rw-r--r--  1 root root     0  ... calico  # 空文件
# 删除并重建目录结构
sudo rm -f /var/lib/calico
sudo mkdir -p /var/lib/calico
sudo chown root:root /var/lib/calico
# 重启 Calico pod
kubectl -n kube-system delete pod -l k8s-app=calico-node
# 验证
kubectl get nodes
```
---
### 常见问题总结

| **现象**                        | **可能原因**    | **解决方案**           |
| :------------------------------ | :-------------- | :--------------------- |
| `kubelet` 正常但节点 `NotReady` | Calico 未就绪   | 等待或检查 Calico 日志 |
| API Server 不可达               | 防火墙/网络问题 | 开放端口或检查网络配置 |
| 节点资源不足                    | CPU/内存耗尽    | 扩容或清理资源         |

```shell
kubectl describe node worker1       # 查看节点详细事件
kubectl logs -n kube-system calico-node-g5wzv  # Calico 日志
```