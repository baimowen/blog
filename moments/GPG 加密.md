## 对称加密

  

```shell

gpg -c -a <file/text>

```

`-a`：生成base64编码的文本

  

### 非对称加密

  

```shell

gpg -e -a -r <secret_key_hash> <file/text>

```

  

### 解密

  

```shell

gpg -d

```

  

### 生成密钥对



```shell

gpg --full-gen-key

```

  

### 查看密钥对

  

```shell

gpg --list-keys

gpg --list-secret-keys

```