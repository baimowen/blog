# NixOS 安装

[^tag]: nix linux

## 准备工作

在 [nixos.org](https://channels.nixos.org/nixos-25.05/latest-nixos-minimal-x86_64-linux.iso) 下载最新版本的镜像包

准备一个 **8g** 以上的u盘，通过 [rufus](https://rufus.ie/zh/) 或 [ventory](https://www.ventoy.net/cn/) 刷写u盘

## 从u盘启动

将u盘插入电脑，在开机时点按 **F2/F8/F10/Del** 键~根据主板型号不同，进入主板BIOS对应按键不同~，进入bios后在启动（boot）选项卡中选择 **USB Devices** 或 **Removable Devices**

## 安装

可自选使用图形化安装和最小化安装，推荐最小化安装更加自由以及熟悉系统

### 重置root密码（可选）

切换到 root 用户和环境

```shell
sudo -i
```

使用 `passwd` 命令重置密码，默认为重置当前用户密码

```shell
passwd [user_name]
```

### 联网

电脑使用有线连接一般通过 dhcp 自动获取ip地址，使用 `ping -c 3 baidu.com` 检查网络连通性

无线网络使用 wpa_cli 进行连接

#### 有线连接

查看默认网卡

```bash
ip r | awk 'NR==1{print $5}'
```

查看当前ip

```bash
ip r | awk 'NR==1{print $9}'
```

格式化显示默认网卡、当前ip以及网关信息

```bash
# 方法一 ip
ip r | awk '/default/ {g=$3} /src/ {ip=$(NF-2)} END {print "interface: ens33\nipaddr:", ip, "\ngateway:", g}'
interface:
ipaddr:
gateway:

# 方法二 ifconfig+ip
ifconfig ens33 | awk '/inet / {ip=$2} END {print "interface: ens33\nipaddr:", ip}'; ip r | awk '/default/ {print "gateway:", $3}'
interface:
ipaddr:
gateway:
```

使用 `ifconfig` 修改ip

```bash
# ifconfig ens33 <ipaddr> netmask 255.255.255.0  # 设置ip和子网掩码
# route add default gw <gateway>  # 设置网关
ip addr add <ipaddr>/24 dev <interface>  # 设置静态ip和子网掩码
ip link set <interface> up  # 启用网卡
ip route del default || true  # 删除旧的默认路由
ip route add default via <gateway> dev <interface>  # 设置默认网关
echo "nameserver 8.8.8.8" | tee -a /etc/resolv.conf  # 设置dns
systemctl stop dhcpcd
# 验证
ip route
```

>   若修改ip后查看ip存在两个ip，使用以下命令删除
>
>   ```shell
>   ip addr del <ipaddr> dev <interface>
>   ```

#### 无线连接

扫描无线网络

```shell
wpa_cli -i wlan0 scan && sleep 2
```

查看扫描结果

```shell
wpa_cli -i wlan0 scan_results
```

配置无线网络

```shell
wpa_cli -i wlan0 set_network <network_id> ssid '"WiFi_Name"'
wpa_cli -i wlan0 set_network <network_id> psk '"WiFi_Password"'
wpa_cli -i wlan0 set_network <network_id> key_mgmt WPA-PSK
```

启用并保存

```shell
wpa_cli -i wlan0 enable_network <network_id>
wpa_cli -i wlan0 save_config
```

### 分区

创建 GPT 分区表

```bash
parted /dev/nvme0n1 -- mklabel gpt
```

创建 EFI 分区（512MB，FAT32）

```bash
parted /dev/nvme0n1 -- mkpart ESP fat32 1MiB 513MiB
parted /dev/nvme0n1 -- set 1 esp on
```

创建 Btrfs 根分区

```
parted /dev/nvme0n1 -- mkpart primary btrfs 513MiB -1GiB
```

创建 Swap 分区（1GB）

```
parted /dev/nvme0n1 -- mkpart primary linux-swap -1GiB 100%
```

格式化 EFI 分区（FAT32）

```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

格式化 Btrfs 分区

```bash
mkfs.btrfs /dev/nvme0n1p2
```

启用 Swap 分区

```bash
mkswap /dev/nvme0n1p3
swapon /dev/nvme0n1p3
```

挂载 Btrfs 分区

```bash
mount /dev/nvme0n1p2 /mnt
```

创建子卷

```bash
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@nix
btrfs subvolume list /mnt  # 查看子卷
```

卸载临时挂载

```bash
umount /mnt
```

重新挂载子卷

```bash
mount -t btrfs -o subvol=@,compress=zstd /dev/nvme0n1p2 /mnt
mkdir -p /mnt/{home,nix,boot}
mount -t btrfs -o subvol=@home,compress=zstd /dev/nvme0n1p2 /mnt/home
mount -t btrfs -o subvol=@nix,compress=zstd /dev/nvme0n1p2 /mnt/nix
mount /dev/nvme0n1p1 /mnt/boot
```

检查分区

```bash
lsblk -f
swapon --show
mount | grep /dev/nvme0n1
```

>[!note]
>
>   或者使用 cfdisk（tui）、gdisk（交互式）进行分区由于较为简单直观此处不表


>[!note]
>
>   有关**分区加密**见：[[btrfs on LUKS]]


### 生成配置

为root生成默认配置

```bash
nixos-generate-config --root /mnt/
```

检查硬件配置文件 `/mnt/etc/nixos/hardware-configuration.nix` 确保挂载点正确，以下为示例，uuid为硬盘分区的唯一标识符，根据自己情况修改

```bash
fileSystems."/" =
{ device = "/dev/disk/by-uuid/7de5e3d3-d2a1-4017-9eea-4d113910094c";
  fsType = "btrfs";
  options = [ "subvol=@" ];
};

fileSystems."/home" =
{ device = "/dev/disk/by-uuid/7de5e3d3-d2a1-4017-9eea-4d113910094c";
  fsType = "btrfs";
  options = [ "subvol=@home" ];
};

fileSystems."/nix" =
{ device = "/dev/disk/by-uuid/7de5e3d3-d2a1-4017-9eea-4d113910094c";
  fsType = "btrfs";
  options = [ "subvol=@nix" ];
};

fileSystems."/boot" =
{ device = "/dev/disk/by-uuid/BDE9-0613";
  fsType = "vfat";
  options = [ "fmask=0022" "dmask=0022" ];
};
```

### 修改主配置文件

配置文件位置：`/mnt/etc/nixos/configuration.nix`

#### 修改引导

*   grub

```bash
# Use the GRUB 2 boot Loader.
boot.loader = {
    grub = {
      enable = true;
      device = "nodev";
      efiSupport = true;
      # timeout = 0;  # 设置等待时间为 0 秒
    };
    efi = {
      canTouchEfiVariables = true;
      efiSysMountPoint = "/boot";
    };
};
```

*   systemd-boot

```bash
# Use the systemd-boot EFI boot Loader.
boot.loader = {
    efi = {
      canTouchEfiVariables = true;
      efiSysMountPoint = "/boot";  # 确保与 EFI 分区挂载点一致
    };
    systemd-boot = {
      enable = true;
      # timeout = 0;  # 设置等待时间为 0 秒
    };
};
```

#### 修改主机名

```shell
sed -i '/networking\.hostName/ s/^\(\s*\)#\s*/\1/; s/"\([^"]*\)"/"nixos"/' /mnt/etc/nixos/configuration.nix
```

#### 开启networkmanager

```shell
sed -i '/networking\.networkmanager\.enable/ s/^\(\s*\)#\s*/\1/; s/false/true/' /mnt/etc/nixos/configuration.nix
```

#### 修改时区

```bash
sed -i '/time\.timeZone/s/^.*$/  time.timeZone = "Asia\/Shanghai";/' /mnt/etc/nixos/configuration.nix
```

#### 修改语言

```bash
sed -i '/i18n\.defaultLocale/s/^.*$/  i18n.defaultLocale = "en_US.UTF-8";/' /mnt/etc/nixos/configuration.nix
cat /mnt/etc/nixos/configuration.nix | grep -B 1 'i18n.defaultLocale'
  # Select internationalisation properties.
  i18n.defaultLocale = "en_US.UTF-8";  # systemc default language
```

#### 用户配置

```nix
...
users.users.<username> = {
  isNormalUser = true;
  extraGroups = [ "wheel" "networkmanager"]; # Enable ‘sudo’ for the user.
  packages = with pkgs; [];
};
```

#### 系统配置

```bash
sed -i '/^  # environment.systemPackages/,/^  # ];/ s/^  # /  /' /mnt/etc/nixos/configuration.nix
cat /mnt/etc/nixos/configuration.nix | grep -A6 "List packages installed in system profile"
  # List packages installed in system profile.
  # You can use https://search.nixos.org/ to find more packages (and options).
  environment.systemPackages = with pkgs; [
    networkmanager
    vim # Do not forget to add an editor to edit configuration.nix! The Nano editor is also installed by default.
  ];
```

#### 开启openssh

```bash
sed -i '/^  # services.openssh.enable = true;/ s/^  # /  /' /mnt/etc/nixos/configuration.nix
# 或使用以下匹配
# sed -i '/services\.openssh\.enable/s/^[[:space:]]*#[[:space:]]*/  /' /mnt/etc/nixos/configuration.nix
cat /mnt/etc/nixos/configuration.nix | grep -B 1 'services.openssh'
  # Enable the OpenSSH daemon.
  services.openssh.enable = true;
```

添加openssh密钥认证以及关闭密码登录（可选）

```nix
...
services.openssh = {
    enable = true;
    # Disable SSH password login
    passwordAuthentication = false;
};
...
users.users.<name> = {
    extraGroups = [ "wheel" "networkmanager" ];
    packages = with pkgs; [];
    # Add ssh public keys
    openssh.authorizedKeys.keys = [
        "ssh-ed25519 ... hostname"
    ];
};
```

#### 放行指定端口

```shell
sed -i '/# Open ports in the firewall./a \  networking.firewall.allowedTCPPorts = [ 22 ];' /mnt/etc/nixos/configuration.nix
```

### 系统安装

```shell
sudo nixos-install --option substituters "https://mirrors.tuna.tsinghua.edu.cn/nix-channels/store"
```

>   [!note]
>
>   安装过程中会提示设置root密码

安装完成后可以通过 `nixos-enter` 修改普通用户密码

```shell
nixos-enter --root /mnt -c 'passwd <username>'
```

## 重启

卸载临时挂载并重启

```shell
umount -R /mnt
reboot
```



关于 **homemanager** 和 **flake**：[[HomeManager 和 Flake]]