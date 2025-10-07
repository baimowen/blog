# Nginx

[^tag ]: nginx



### https 配置

首先设置重定向，将对80端口的请求重定向到443端口
```toml
server {
    listen 80;
    server_name localhost;
    return 301 https://$host$request_uri;
}
```
修改监听端口为443，并添加证书和密钥的路径
```shell
listen 443 ssl;
server_name localhost;
# 证书和密钥
ssl_certificate "/path/to/cacert.pem";
ssl_certificate_key "/path/to/private.key";
```
设置ssl超时、加密和算法
```toml
ssl_session_timeout 60m;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!rc4:!DHE;
ssl_prefer_server_ciphers on;
```

### 反向代理

正向代理为代理客户端请求服务端

反向代理则将代理配置在服务端，多个服务共享一个地址

反向代理示例如下：
```toml
server {
    listen          80;
    listen          [::]:80;

    server_name     <your_server_name>;
    rewrite ^(.*)$  https://$host$1 permanent;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen  443       ssl;
    listen  [::]:443  ssl;
    http2   on;

    server_name         <your_server_name>;

    ssl_certificate     /path/to/ssl_cert;
    ssl_certificate_key /path/to/ssl_cert_key;

    location / {
        proxy_set_header    Host                $host;
        proxy_set_header    X-Real-IP           $remote_addr;
        proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto   $scheme;
        proxy_http_version  1.1;
        proxy_set_header    Upgrade             $http_upgrade;
        proxy_set_header    Connection          $connection_upgrade;
        proxy_pass          http://127.0.0.1:9000/;
    }
}
```

### 负载均衡

通过 `upstream` 将多个后端服务器分组，通过 `weight` 值设置权重进行负载均衡

```toml
http {
    upstream <server_name> {
        ip hash;
        server 192.168.1.10:1234 weight=3;
        server 192.168.1.11:1235 weight=2;
        server 192.168.1.12:1236 weight=1;
    }
    ...
    server {
        location /app {
            proxy_pass https://<server_name>;
        }
        ...
    }
}
```
> [!note]
> `ip hash` 可以确保同一个client请求会被发送到同一台服务器