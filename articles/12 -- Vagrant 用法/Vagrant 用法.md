# Vagrant 用法

[^tag]: linux



## 使用 Vagrant 批量部署虚拟机

常见 linux 发行版不像 nixos 一样可以通过一个文件描述整个系统，做到和原系统相近可以通过 vagrant 复现

1.下载并安装 virtualbox \
下载地址：https://www.virtualbox.org/wiki/Downloads

2.下载并安装 vagrant \
下载地址：https://www.vagrantup.com/downloads.html

### Vagrant 基本命令

在空文件夹初始化虚拟机

```shell
vagrant init <box-name>
```

在初始化完的文件夹内启动虚拟机

```shell
vagrant up [--provider virtualbox]
```

ssh 登录启动的虚拟机
```shell
vagrant ssh
```
> [!note]
> vagrant 创建的虚拟机默认用户为 vagrant，密码为 vagrant，并且配置**免密 root**
> 
> **免密root**见：[[免密root]]


挂起启动的虚拟机

```shell
vagrant suspend
```

重启虚拟机

```shell
vagrant reload
```

关闭虚拟机

```shell
vagrant halt
```

查找虚拟机的运行状态

```shell
vagrant status
```

销毁当前虚拟机

```shell
vagrant destroy
```

列出本地环境中所有的box

```shell
vagrant box list
```

添加box到本地vagrant环境

```shell
vagrant box add <box-name>
```

更新本地环境中指定的box

```shell
vagrant box update <box-name>
```

删除本地环境中指定的box

```shell
vagrant box remove <box-name>
```

重新打包本地环境中指定的box

```shell
vagrant box repackage <box-name>
```

打包分发指定 box

```shell
vagrant package --output <new_box_name>
```
> [!note]
> 打包本地盒子时，应注意：
> 
> 1. 创建 vagrant 用户，密码为 vagrant 并且配置免密 root（重要！！！）
>
> 2. 清理虚拟机内的**缓存**以达到减小盒子体积的目的  
> 
> 3. 网络接口默认配置为 **DHCP**
> 
> 4. 设置一个通用主机名（可选）


> [!note]
> box 可以在此处查找：https://app.vagrantup.com/boxes/search

### Vagrantfile

#### 基本信息

```
Vagrant.configure("2") do |config|
    # info
    config.vm.box = "generic/arch"  # box_name
    config.vm.box_version = "4.3.12"  # box_version
    config.vm.hostname = "arch"  # 虚拟机主机名
    config.vm.box_check_update = false  # 检查 box 更新
    config.ssh.forward_agent = true  # 开启 ssh 代理
end
```

#### 分配硬件资源

```
config.vm.provider "virtualbox" do |vb|
  vb.name = "ubuntu-imooc"
  vb.memory = "1024"
  vb.cpus = 2
end
```

#### 网络配置


1. **Private network(私有网络)**
```
config.vm.network "private_network", ip: "192.168.1.10"  # 固定IP
config.vm.network "private_network", type: "dhcp"  # 自动分配
```

2. **Public network(公有网络)**
```
config.vm.network "public_network", ip: "192.168.1.10"  # 固定IP
config.vm.network "public_network", [bridge: "en1: Wi-Fi (AirPort)"]  # 桥接网卡
```

3. **设置端口转发**

```
config.vm.network :forwarded_port, guest: 80, host: 8080
```

私有网络和共有网络的区别是<u>外部主机</u>是否能访问该虚拟机（不包括宿主机）

#### 共享文件夹

```
Vagrant.configure("2") do |config|
  # other config here

  config.vm.synced_folder "src/", "/srv/website"
end
```

#### 自动脚本

在虚拟机第一次启动时**自动运行的脚本**

```shell
config.vm.provision "shell", inline: <<-SHELL
    pwd
    ls
SHELL
```

或**上传文件**

```shell
config.vm.provision "file", source: "script.sh", destination: "~/script.sh"
```

或启动后的**提示信息**

```shell
config.vm.post_up_message = <<-MESSAGE
    hello world!
MESSAGE
```