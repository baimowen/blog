# podman生成k8s集群

[^tag]: kubernetes container



## podman

podman 是一个 *rootless* **容器管理**工具，在使用上与 docker 命令行高度一致，学习成本低

如：

* `podman run` = `docker run`
* `podman ps` = `docker ps`
* `podman rm` = `docker rm`
* ...

podman 与 docker 的区别：
![image](https://pic.sl.al/gdrive/pic/2025-08-31/hash_83258ff7_8bab8e47a66b_deepseek_mermaid_20250831_5fce58.png)

podman 相比 docker 无守护进程、无需 root 权限，比 docker 更灵活更安全

同时 podman 也是 <u>pod + manager</u> 的组合： pod 管理器

## 通过 podman 生成 k8s 集群

首先创建一个 pod

```shell
podman pod create --name <pod_name> -p <host_port>:<container_port>
podman pod list  # 查看 pod 列表
```
> pod 内部共享同一 ip 和端口

<br>

然后启动容器并加入 pod

```shell
podman run -d \
  --pod <pod_name> \
  [--name <container_name>] \
  [-e <var>=<value>] \
  [-v <host_folder>:<container_folder>] \
  docker.io/library/<container_image>
```

查看每个容器所属 pod

```shell
podman ps --pod
```

启动/停止运行整个 pod

```shell
podman pod start/stop <pod_name>
```

导出 pod

```shell
podman generate kube <pod_name> | tee <pod_name>.yaml
```

使用 pod 通过导出的 yaml 文件导入集群

```shell
podman play kube <pod_name>.yaml
```

使用 kubectl 通过导出的 yaml 文件导入集群

```shell
kubectl create -f <pod_name>.yaml
```



## 关于如何操作 k8s



见 [[k8s 编排]]