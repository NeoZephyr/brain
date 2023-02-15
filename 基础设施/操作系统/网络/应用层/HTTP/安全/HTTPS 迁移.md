## 申请证书

申请证书时应当同时申请 RSA 和 ECDSA 两种证书，在 Nginx 里配置成双证书验证，这样服务器可以自动选择快速的椭圆曲线证书，同时也兼容只支持 RSA 的客户端

如果申请 RSA 证书，私钥至少要 2048 位，摘要算法应该选用 SHA-2，例如 SHA256、SHA384 等

出于安全的考虑，Let’s Encrypt 证书的有效期很短，只有 90 天，时间一到就会过期失效，所以必须要定期更新。可以在 crontab 里加个每周或每月任务，发送更新请求

## 配置 HTTPS

```nginx
listen                443 ssl;

ssl_certificate       xxx_rsa.crt;
ssl_certificate_key   xxx_rsa.key;

ssl_certificate       xxx_ecc.crt;
ssl_certificate_key   xxx_ecc.key;
```

为了提高 HTTPS 的安全系数和性能，可以强制 Nginx 只支持 TLS1.2 以上的协议，打开 Session Ticket 会话复用

```nginx
ssl_protocols               TLSv1.2 TLSv1.3;

ssl_session_timeout         5m;
ssl_session_tickets         on;
ssl_session_ticket_key      ticket.key;
```

密码套件的选择方面，建议以服务器的套件优先。这样可以避免恶意客户端故意选择较弱的套件、降低安全等级，然后密码套件向 TLS1.3 看齐，只使用 ECDHE、AES 和 ChaCha20，支持 False Start

```nginx
ssl_prefer_server_ciphers   on;
ssl_ciphers   ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-CHACHA20-POLY1305:ECDHE+AES128:!MD5:!SHA1;
```

如果服务器上使用了 OpenSSL 的分支 BorringSSL，那么还可以使用一个特殊的等价密码组（Equal preference cipher groups）特性，它可以让服务器配置一组等价的密码套件，在这些套件里允许客户端优先选择

```nginx
ssl_ciphers [ECDHE-ECDSA-AES128-GCM-SHA256|ECDHE-ECDSA-CHACHA20-POLY1305];
```

如果客户端硬件没有 AES 优化，服务器就会顺着客户端的意思，优先选择与 AES 等价的 ChaCha20 算法，让客户端能够快一点

## 服务器名称指示

在 HTTP 协议里，多个域名可以同时在一个 IP 地址上运行，这就是虚拟主机，Web 服务器会使用请求头里的 Host 字段来选择。但在 HTTPS 里，因为请求头只有在 TLS 握手之后才能发送，在握手时就必须选择虚拟主机对应的证书，TLS 无法得知域名的信息，就只能用 IP 地址来区分。所以，最早的时候每个 HTTPS 域名必须使用独立的 IP 地址，非常不方便

TLS 的扩展，给协议加个 SNI（Server Name Indication）的补充条款。它的作用和 Host 字段差不多，客户端会在 Client Hello 时带上域名信息，这样服务器就可以根据名字而不是 IP 地址来选择证书

## 重定向跳转

把不安全的 HTTP 网址用 301 或 302 重定向到新的 HTTPS 网站

```nginx
return 301 https://$host$request_uri;
rewrite ^  https://$host$request_uri permanent;
```

但这种方式有两个问题。一个是重定向增加了网络成本，多出了一次请求；另一个是存在安全隐患，重定向的响应可能会被中间人窜改，实现会话劫持，跳转到恶意网站

有一种叫 HSTS （HTTP 严格传输安全，HTTP Strict Transport Security）的技术可以消除这种安全隐患。HTTPS 服务器需要在发出的响应头里添加一个 Strict-Transport-Security 的字段，再设定一个有效期

有了 HSTS 的指示，以后浏览器再访问同样的域名的时候就会自动把 URI 里的 http 改成 https，直接访问安全的 HTTPS 网站。这样中间人就失去了攻击的机会，而且对于客户端来说也免去了一次跳转，加快了连接速度

```nginx
add_header Strict-Transport-Security max-age=15768000;
```
