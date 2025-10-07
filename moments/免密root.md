### 配置免密 root

  

#### 方法一：

  

```shell

echo "<username> ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/10-nopasswd-user

chmod 440 /etc/sudoers.d/10-nopasswd-user

```

  

#### 方法二：

  

```shell

sed -i '/^#\s*%wheel\s*ALL=(ALL)\s*NOPASSWD:\s*ALL/s/^#\s*//' /etc/sudoers

```

  

#### 验证

  

```shell

visudo -c || { echo "语法错误，请检查！"; rm -f /etc/sudoers.d/10-nopasswd-user; exit 1; }

sudo -v

```