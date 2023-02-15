```nginx
server {
    listen       443 ssl http2;
    server_name  www.pain.com;

    ssl_certificate         xxx.crt;
    ssl_certificate_key     xxx.key;
}
```

配置服务器推送特性可以使用指令 http2_push 和 http2_push_preload。不过如何合理地配置推送是个难题，如果推送给浏览器不需要的资源，反而浪费了带宽

```nginx
http2_push         /style/xxx.css;
http2_push_preload on;
```

在 TLS 的扩展里，有一个叫 ALPN（Application Layer Protocol Negotiation）的东西，用来与服务器就 TLS 上跑的应用协议进行协商

客户端在发起 Client Hello 握手的时候，后面会带上一个 ALPN 扩展，里面按照优先顺序列出客户端支持的应用协议。最优先的是 h2，其次是 http/1.1

服务器看到 ALPN 扩展以后就可以从列表里选择一种应用协议，在 Server Hello 里也带上 ALPN 扩展，告诉客户端服务器决定使用的协议
