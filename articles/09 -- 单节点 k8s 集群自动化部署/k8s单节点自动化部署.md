[^tag]: kubernetes

部署详细过程：[[k8s 编排]]



## 单节点

单节点 k8s 集群可通过脚本自动化部署，后续再向集群内添加节点即可（无 ansible）

完整脚本如下：
```shell
#!/bin/bash

# tip
echo "If the deployment is carried out with root privileges, the cluster configuration will be located in /root/.kube/config. \nUse 'sudo kubectl get nodes' to check the status of the nodes."

# error
set -e

# package
sudo apt update && sudo apt upgrade

if ! command -v kubelet >/dev/null 2>&1 || \
   ! command -v kubeadm >/dev/null 2>&1 || \
   ! command -v kubectl >/dev/null 2>&1; then
    echo "ERROR: Kubernetes components (kubelet, kubeadm, or kubectl) not installed properly."
    echo "Run env.sh to deploy the node environment"
    exit 1
fi

sudo systemctl enable --now kubelet

read -p "Confirm whether it is the master node (Y/n): " confirm
confirm=${confirm:-Y}
case "$confirm" in
    [Yy]|"")
        echo "Proceeding with master node configuration..."
        ;;
    [Nn])
        echo "This is not a master node. Exiting."
        exit 1
        ;;
    *)
        echo "Invalid input. Exiting."
        exit 1
        ;;
esac

if [ ! -d /var/lib/kubelet ]; then
    sudo mkdir -p /var/lib/kubelet
fi
if [ ! -f /var/lib/kubelet/config.yaml ] || ! grep -q 'cgroupDriver: systemd' /var/lib/kubelet/config.yaml 2>/dev/null; then
    cat <<EOF | sudo tee /var/lib/kubelet/config.yaml >/dev/null
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
    echo "Configured cgroupDriver in /var/lib/kubelet/config.yaml"
fi
if grep -q 'cgroupDriver: systemd' /var/lib/kubelet/config.yaml 2>/dev/null; then
    echo "Verification: cgroupDriver is properly set to systemd"
else
    echo "error: cgroupDriver configuration not found" >&2
    exit 1
fi

# kubernetes master node
echo "pull kubernetes images..."
for i in {1..3}; do
    if pull_output=$(sudo kubeadm config images pull \
      --image-repository=registry.aliyuncs.com/google_containers \
      --kubernetes-version=v1.30.1 2>&1) && \
      echo "$pull_output" | grep -q "Pulled registry.aliyuncs.com/google_containers/"; then
        echo "$pull_output"
        break
    fi
    if [ $i -eq 3 ]; then
        echo "error: can not pull kubernetes images"
        exit 1
    fi
    echo "pull images failed, retry ($i/3)..."
    sleep 5
done

echo "init kubeadm..."
sudo nerdctl -n k8s.io pull registry.aliyuncs.com/google_containers/pause:3.9
sudo nerdctl -n k8s.io tag registry.aliyuncs.com/google_containers/pause:3.9 registry.k8s.io/pause:3.9
sudo kubeadm init \
  --apiserver-advertise-address=$(hostname -I | awk '{print $1}') \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --image-repository=registry.aliyuncs.com/google_containers \
  --kubernetes-version=v1.30.1 \
  --upload-certs 2>&1 | tee /tmp/kubeadm-init.log
if grep -qi "error" /tmp/kubeadm-init.log; then
    echo "error: kubeadm init failed."
    echo "the initialization log has been saved to /tmp/kubeadm-init.log."
    echo "reset the cluster..."
    sudo kubeadm reset -f
    sudo systemctl stop kubelet
    sudo rm -rf /var/lib/kubelet/*
    sudo systemctl start kubelet
    sudo rm -rf /etc/cni/net.d
    sudo ip link delete cni0
    sudo rm -rf $HOME/.kube
    sudo rm -rf /root/.kube
    sudo crictl rm -a -f
    sudo crictl rmi -a
    sudo systemctl restart containerd
    exit 1
fi
sleep 5
echo "create $HOME/.kube ..."
# mkdir /home/ubuntu/.kube
mkdir $HOME/.kube
if [ ! -d "$HOME/.kube" ]; then
    echo "create $HOME/.kube failed, please create it manually."
    exit 1
fi
echo "locate: $HOME/.kube"
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "export KUBECONFIG=$HOME/.kube/config" | sudo tee -a /etc/profile
echo "create $HOME/.kube/config sucessfully."


# Calico
echo "=== installing Calico ==="
if ! curl -fSL -o "calico.yaml" "https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml"; then
    echo "Error: Failed to download calico.yaml from GitHub."
    exit 1
fi
echo "config calico.yaml"
sed -i 's@docker.io/calico/node@docker.m.daocloud.io/calico/node@g' calico.yaml
sed -i 's@docker.io/calico/cni@docker.m.daocloud.io/calico/cni@g' calico.yaml
sed -i 's@docker.io/calico/kube-controllers@docker.m.daocloud.io/calico/kube-controllers@g' calico.yaml
kubectl apply -f calico.yaml
echo "waiting for calico to be ready..."
sleep 120
node_status=$(kubectl get nodes -o jsonpath='{.items[0].status.conditions[?(@.type=="Ready")].status}')
if [ "$node_status" != "True" ]; then
    echo "error: Node status is abnormal, rolling back Calico configuration."
    kubectl delete -f calico.yaml
    exit 1
else
    echo "The node status is normal, continue."
fi


# crictl
echo "=== installing crictl ==="
if ! curl -fSL -o "crictl-v1.33.0-linux-amd64.tar.gz" "https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.33.0/crictl-v1.33.0-linux-amd64.tar.gz"; then
    echo "Error: Failed to download crictl-v1.33.0-linux-amd64.tar.gz from GitHub."
    exit 1
fi
sudo tar -C /usr/local/bin -xzf crictl-v1.33.0-linux-amd64.tar.gz
rm crictl-v1.33.0-linux-amd64.tar.gz
echo "config crictl"
cat <<EOF | sudo tee /etc/crictl.yaml >/dev/null
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF


echo "=== Kubernetes deployment completed. ==="
kubectl get nodes
```


## 添加节点

添加节点的脚本如下：

```shell
#! /bin/bash

read -p "Enter the worker node hostname to join the cluster: " WORKER_HOST
if [ -z "$WORKER_HOST" ]; then
    echo "Error: Worker hostname cannot be empty"
    exit 1
fi
# if ! ssh -o ConnectTimeout=5 "$WORKER_HOST" true &> /dev/null; then
#     echo "Error: Cannot connect to worker node $WORKER_HOST via SSH"
#     echo "Make sure:"
#     echo "1. SSH passwordless login is configured"
#     echo "2. The hostname is correct and reachable"
#     exit 1
# fi

echo "print join command"
JOIN_COMMAND=$(kubeadm token create --print-join-command 2>/dev/null)
if [ -z "$JOIN_COMMAND" ]; then
    echo "error: cannot create join command"
    exit 1
fi
JOIN_COMMAND="sudo $JOIN_COMMAND"
echo "join command: $JOIN_COMMAND"
echo "Joining worker node $WORKER_HOST to the cluster"
ssh "$WORKER_HOST" "$JOIN_COMMAND"
if [ $? -eq 0 ]; then
    echo "Success: $WORKER_HOST has joined the cluster"
else
    echo "Error: Failed to join $WORKER_HOST to the cluster"
    exit 1
fi

# kubectl label node 192.168.99.12 node-role.kubernetes.io/worker=worker
# kubectl label node 192.168.99.13 node-role.kubernetes.io/worker=worker
```