## 安装 SSL

```bash
./configure \
--with-http_ssl_module

make
make install
```

## SSL 配置

```nginx
server {
    listen 443;
    server_name www.pain.blue;

    # 开启 ssl
    ssl on;
    ssl_certificate blue.crt;
    ssl_certificate_key blue.key;

    # ssl 会话 cache
    ssl_session_cache shared:SSL:1m;
    # ssl 会话超时时间
    ssl_session_timeout 5m;

    # 加密套件
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
}
```