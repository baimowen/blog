### 检测 root 用户

  

```shell

if [ "$(id -u)" -ne 0 ]; then

    echo "请使用root用户运行此脚本！"

    exit 1

fi

# or

[ "$UID" -ne 0 ] && echo "请使用root用户运行此脚本！" && exit 1

```