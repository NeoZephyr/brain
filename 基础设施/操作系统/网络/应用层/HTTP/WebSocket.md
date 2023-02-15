## 握手

WebSocket 的握手是一个标准的 HTTP GET 请求，但要带上两个协议升级的专用头字段：
1. Connection: Upgrade，表示要求协议升级
2. Upgrade: websocket，表示要升级成 WebSocket 协议

另外，为了防止普通的 HTTP 消息被意外识别成 WebSocket，握手消息还增加了两个额外的认证用头字段：
1. Sec-WebSocket-Key：一个 Base64 编码的 16 字节随机数，作为简单的认证密钥
2. Sec-WebSocket-Version：协议的版本号，当前必须是 13

服务器收到 HTTP 请求报文，看到上面的四个字段，就知道是 WebSocket 的升级请求，于是构造一个特殊的 101 Switching Protocols 响应报文，通知客户端，接下来全改用 WebSocket 协议通信

WebSocket 的握手响应报文也是有特殊格式的，要用字段 Sec-WebSocket-Accept 验证客户端请求报文，同样也是为了防止误连接。具体的做法是把请求头里 Sec-WebSocket-Key 的值，加上一个专用的 UUID，再计算 SHA-1 摘要。客户端收到响应报文，就可以用同样的算法，比对值是否相等，如果相等，就说明返回的报文确实是刚才握手时连接的服务器，认证成功