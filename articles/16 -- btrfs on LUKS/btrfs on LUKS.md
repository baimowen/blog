[^tag]: linux

### btrfs on LUKS

本文是在 LUKS 加密容器内创建 btrfs 文件系统，重点是进行全盘加密操作

>   在 btrfs 文件系统上创建 LUKS 加密容器自行百度或AI



**步骤**：分盘 -> 加密 -> 创建文件系统 -> 挂载

---

#### partition



```shell
parted /dev/sda -- mklabel gpt
parted /dev/sda -- mkpart ESP fat32 1MiB 512MiB
parted /dev/sda -- set 1 esp on
parted /dev/sda -- mkpart primary 512MiB 100%
```

---

#### crypt



```shell
cryptsetup luksFormat /dev/sdaX  # 加密
cryptsetup open /dev/sdaX cryptroot  # 解密
```



`cryptroot`	给解锁后的设备起的名字，这个名字会用作映射设备的标识。解锁后系统会生成：`/dev/mapper/cryptroot`



关于 crypt 的一些其他命令

```shell
cryptsetup open /dev/sdaX cryptroot  # 解密
cryptsetup luksDump /dev/sdaX  # 查看加密信息
cryptsetup luksAddKey /dev/sdaX  # 添加新密码
cryptsetup luksRemoveKey /dev/sdaX  # 删除旧密码
cryptsetup luksChangeKey /dev/sdaX  # 修改密码
```



可选参数

`--key-file <file>`	从文件读取解密密钥，而不是交互式输入。



取消加密

```shell
cryptsetup luksOpen /dev/sdaX cryptroot  # 解密
mount /dev/mapper/cryptroot /mnt  # 挂载分区
mount -t ext4 /dev/sdaXX /mnt2  # 挂载未加密分区
rsync -aAXHv --progress /mnt/ /mnt2/  # 复制数据
diff -r /mnt /mnt2  # 检查数据差异
umount /mnt /mnt2
cryptsetup luksClose cryptroot
mkfs.btrfs /dev/sdaX
mount -t btrfs /dev/sdaX /mnt
btrfs subvol create /mnt/@
umount /mnt
mount --mkdir -t btrfs -o subvol=@,compress=zstd:3,noatime /dev/sdaX /mnt
```

**步骤**：解密 -> 复制数据到未加密的分区 -> 格式化 -> 写回原分区

rsync 参数：

| 参数         | 含义                                                         |
| ------------ | ------------------------------------------------------------ |
| `-a`         | **archive**（归档模式）。保留尽可能多的文件属性，包括：权限、时间戳、符号链接、设备文件等。等价于：`-rlptgoD`（recursive + links + perms + times + group + owner + devices）。用于递归复制目录并保持原有属性。 |
| `-A`         | **保留ACL**（Access Control Lists）。复制文件的访问控制列表，适用于需要保留精确权限的系统（例如 Linux + SELinux）。 |
| `-X`         | **保留扩展属性**（Extended Attributes）。复制文件的扩展属性，比如 SELinux 安全上下文、文件系统特定的元数据等。 |
| `-H`         | **保留硬链接**                                               |
| `-v`         | **verbose**（详细模式）。显示详细过程，方便监控进度和调试。  |
| `--progress` | **显示进度**                                                 |

---

#### btrfs



```shell
mkfs.btrfs [-L <target>] /dev/mapper/cryptroot  # 格式化文件系统
mount /dev/mapper/cryptroot /mnt  # 挂载
btrfs subvol create /mnt/@  # 创建子卷
umount /mnt
mount --mkdir -t btrfs -o subvol=@,compress=zstd:3,noatime /dev/mapper/cryptroot /mnt
```

