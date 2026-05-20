问题场景：在一个LLM的web应用中，为了保持长连接，需要将普通的HTTP协议升级为长连接WebSocket，从Nginx层面配置。

解决方案：

使用"WS upgrade"（WebSocket 升级）：把一个普通的 HTTP 连接"升级"为 WebSocket 连接的过程。它是 WebSocket 协议建立连接的标准握手机制。



 vim /etc/nginx/conf.d/domain.conf

```
location / {
        proxy_pass http://<ip:port>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
    }
```

 vim /etc/nginx/nginx.conf

```
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        "" close;
    }

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main; #这一行是本身就有，方便定位，以上内容为添加内容～
```



#### 为什么要"升级"——HTTP 的根本限制？

HTTP 的设计是**请求-响应**模型：客户端问一句，服务器答一句，答完连接就可以关了（或者复用给下一个请求）。这个模型有两个硬伤：

1. **服务器不能主动说话**。除非客户端先发请求，服务器没机会把"新消息来了"推给你。
2. **每个请求都要带完整的头**。Cookie、User-Agent、各种 Header 加起来动辄几百字节，对于"一句话来回几十次"的场景成本极高。

WebSocket 就是为了根除这些补丁而设计的：**一条 TCP 连接建立后，双方想什么时候说话就什么时候说话，谁也不用等谁**。



而80/443端口已经深入人心，所以WebSocket服用80/443端口，并且握手语法不变。握手成功后切换协议，这就是upgrade的本质。



#### HTTP vs WebSocket 协议层差异

| 维度         | HTTP                                 | WebSocket                    |
| ------------ | ------------------------------------ | ---------------------------- |
| 连接寿命     | 一次请求一关（或 keep-alive 短复用） | 建立后长期保持，可数小时数天 |
| 通信方向     | 客户端发起，服务器响应               | 全双工，任意一方随时发       |
| 帧/包格式    | 文本 Header + Body                   | 二进制帧，2~14 字节头        |
| 单条消息开销 | 几百字节起步                         | 最低 2 字节                  |
| 状态         | 无状态，每次请求独立                 | 有状态，连接本身就是会话     |
| URL 协议头   | `http://` / `https://`               | `ws://` / `wss://`           |
| 应用层语义   | 资源获取（GET/POST/...）             | 自由的消息通道，应用层自己定 |



#### SSE（Server-Sent Events）

SSE 是另一条路：**不升级协议，纯粹利用 HTTP 长连接的特性**。

服务器对客户端的请求**故意不结束响应**，而是用一个特殊的 `Content-Type: text/event-stream`，然后源源不断地往这条还没结束的 HTTP 响应里写数据。客户端用 `EventSource` API 接收，每收到一段就触发一次事件。



#### **SSE vs WebSocket 的关键差异**：

| 维度         | SSE                    | WebSocket                    |
| ------------ | ---------------------- | ---------------------------- |
| 方向         | 单向（服务器→客户端）  | 双向                         |
| 协议         | 就是 HTTP，没有升级    | HTTP 握手后切换协议          |
| 数据格式     | 只支持文本（UTF-8）    | 文本和二进制都支持           |
| 自动重连     | 浏览器原生支持         | 要自己写                     |
| 复杂度       | 极简，curl 就能看      | 有帧格式、掩码、ping/pong 等 |
| 客户端发消息 | 还是得用普通 HTTP 请求 | 同一条连接就能发             |
| 兼容老代理   | 偶尔有缓冲问题         | upgrade 偶尔被拦             |