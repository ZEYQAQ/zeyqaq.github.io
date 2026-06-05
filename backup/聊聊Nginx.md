Nginx upstream 内置的负载均衡算法主要有以下几种：

**round-robin（轮询）** — 默认算法，不需要任何指令。按顺序依次把请求分发到各后端。

**weight（加权轮询）** — 本质还是轮询，通过 `weight` 参数给性能更强的机器分配更高权重。

**ip_hash** — 按客户端 IP 做哈希，同一个 IP 固定落到同一台后端，常用于简单的会话保持（session 粘性）。

**least_conn（最少连接）** — 把请求发给当前活跃连接数最少的后端，适合请求处理时长差异较大的场景。

**hash** — 基于自定义 key（如 `$request_uri`）做哈希。加上 `consistent` 参数可启用一致性哈希，后端增减时减少缓存失效。

**random** — 随机选择。可加 `two least_conn`，即随机选两台再取连接数较少的那台（power of two choices）。

**least_time** — 按平均响应时间 + 活跃连接数综合选择最快的后端。

一个常见的小提醒：`ip_hash` 和 `hash` 做会话保持时，如果后端节点数量变化（扩缩容），未用一致性哈希的话会导致大量请求重新分布



Nginx中location匹配规则

匹配优先级是 `=`（精确）> `^~`（前缀，命中后不再正则）> 正则 `~`/`~*`（区分/不区分大小写，按配置顺序）> 普通前缀匹配。

优先级从高到低：

1. `location = /path` —— **精确匹配**，命中立即停止。
2. `location ^~ /path` —— **前缀匹配且命中后不再做正则**（带 `^~` 修饰符）。
3. `location ~ /regex` / `location ~* /regex` —— **正则匹配**（`~` 区分大小写，`~*` 不区分），多个正则按**在配置文件中出现的先后顺序**匹配，命中第一个即停止。
4. `location /path` —— **普通前缀匹配**，遵循**最长前缀优先**。



Nginx中rewrite和return

return（它的性能更好）的应用：

```
server {
    listen 80;
    # 全站http强制跳转https（最优写法，优于rewrite）
    return 301 https://$host$request_uri;
}

location ~ \.(php|sh|sql)$ {
    # 拦截危险后缀访问
    return 403;
}

location /api {
    # 直接返回json
    default_type application/json;
    return 200 '{"code":0,"msg":"success"}';
}
```

rewrite的用法：

```
rewrite 正则匹配URI 替换后的URI [flag];
```

rewrite的应用：

```
#伪静态（url 美化，内部重写，地址不变）
location / {
    rewrite ^/article/(\d+)\.html$ /article.php?id=$1 last;
}

#rewrite 捕获分组 $1 $2
# /book/123 → /page?id=123
rewrite ^/book/(\d+)$ /page?id=$1 last;
#(\d+)：捕获数字，存入$1

#去掉 url 后缀.html
rewrite ^/(.*)\.html$ /$1 permanent;

#参数携带
#原地址/test?name=zhangsan → /api/test?name=zhangsan
rewrite ^/test$ /api/test?$args last;
```

last：改完路径 → **跳出当前 location，从头遍历所有 location**

break：改完路径 → **留在当前 location 继续执行 proxy_pass/root 等**

redirect：临时 302 重定向，浏览器地址栏变化

permanent：永久 301 重定向，浏览器地址栏变化



return 和 rewrite 对比选型（什么时候用谁）

|              场景               | 推荐指令 |              原因              |
| :-----------------------------: | :------: | :----------------------------: |
|     域名全站跳转 http→https     |  return  |    无正则，性能好，代码简洁    |
| 固定路径拦截（封禁目录 / 后缀） |  return  |    直接返回状态码，快速终止    |
| 接口直接返回自定义 json / 文本  |  return  |            最简配置            |
|  正则路径伪静态、URL 路径替换   | rewrite  | 需要正则捕获路径，必须 rewrite |
|   复杂多级路径拆分、参数重组    | rewrite  |       支持分组捕获 $1/$2       |



Nginx中proxy_pass末尾带/和不带/的区别

```
情况一：proxy_pass 带 URI（末尾有 /）
location /api/ {
    proxy_pass http://backend/;
}
请求 /api/user → 转发到 http://backend/user （去掉 /api/）

情况二：proxy_pass 不带 URI（末尾无 /）
location /api/ {
    proxy_pass http://backend;
}
请求 /api/user → 转发到 http://backend/api/user （原样保留）
```

**规律**：proxy_pass 后面如果**带了路径（包括只有一个 `/`）**，nginx 会把 location 匹配的前缀部分**替换**掉；如果**只有域名/IP 不带路径**，则把完整 URI **原样**拼接到后面。

> 注意：当 proxy_pass 使用了正则 location 或包含变量时，规则不同，不能带 URI 部分。



Nginx中常用 header 透传：`proxy_set_header Host`、`X-Real-IP`、`X-Forwarded-For` 的作用

```
location / {
    proxy_pass http://backend;
    proxy_set_header Host $host;                      # 透传原始 Host
    proxy_set_header X-Real-IP $remote_addr;          # 客户端真实 IP
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # 经过的代理链
    proxy_set_header X-Forwarded-Proto $scheme;       # 原始协议 http/https
}
```



Nginx如果想要pass到k8s到工作负载中，且Nginx不在K8S集群内，如何不给工作负载绑定LoadBalance（使用ClusterIP）？

只需要**一个对外 ELB**,挂在 Ingress Controller 上(它本身就是个 nginx)。后端所有业务 Service 都用 **ClusterIP 类型**(纯内部,不占 ELB)。Ingress 根据**域名 / 路径**把外部流量路由到不同的内部 Service。



Ingress配置实现内部工作负载互访，proxy_pass 用变量时使用 resolver+CoreDNS。

```
location / {
    resolver 10.96.0.10 valid=10s;        # K8s 中通常是 kube-dns/CoreDNS 的 ClusterIP
    set $backend "http://my-svc.default.svc.cluster.local";
    proxy_pass $backend;
}
```

- nginx 默认在**启动时**解析 upstream 域名并缓存，之后不再解析，这样可以避免每次请求都走CoreDNS解析。后端 IP 变化（如 K8s Pod 重建、Service Endpoint 变动）时不会感知，导致 502。
- 把 `proxy_pass` 写成**变量**形式并配置 `resolver`，nginx 就会在请求时按 DNS TTL 动态解析，从而跟上后端变化。
- `valid` 控制 DNS 缓存时间。



Keepalive长连接

**客户端侧：**

```nginx
keepalive_timeout 65;
keepalive_requests 1000;   # 单连接最多处理多少请求后关闭
```

**upstream 侧（很多人会漏配，后果是后端大量 TIME_WAIT）：**

```nginx
upstream backend {
    server 10.0.0.1:8080;
    keepalive 32;          # 每个 worker 与后端保持的空闲长连接数
}
location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;          # 长连接必须 1.1
    proxy_set_header Connection "";  # 清掉 Connection: close
}
```

### 

Nginx中常用调优

```
sendfile on;        # 零拷贝发送静态文件，省去内核态到用户态的数据拷贝，静态文件传输零拷贝，大幅降低 CPU。

tcp_nopush on;      # 配合 sendfile，攒满一个数据包再发（减少小包）
tcp_nodelay on;     # 长连接上禁用 Nagle 算法，及时发送小数据
#`tcp_nopush`/`tcp_nodelay` 看似矛盾，实则 nginx 会智能配合——传文件时攒包，传完最后一块时立即发出。

gzip on;            # 压缩文本类响应
gzip_types text/plain application/json application/javascript text/css;
gzip_comp_level 5;
gzip_min_length 1k;
```



Nginx中静态资源缓存配置

```
open_file_cache max=10000 inactive=60s;   # 缓存文件描述符、大小、修改时间
open_file_cache_valid 60s;
open_file_cache_min_uses 2;
open_file_cache_errors on;
```



Nginx中TLS握手过程（简述

1. Client Hello：客户端发送支持的 TLS 版本、加密套件、随机数。
2. Server Hello：服务端选定套件、发送证书、随机数。
3. 客户端校验证书，生成 pre-master secret，用服务端公钥加密发送。
4. 双方用两个随机数 + pre-master 生成对称会话密钥。
5. 之后用对称密钥加密通信。

TLS 1.3 简化了握手（1-RTT，甚至 0-RTT），更快更安全。



Nginx中限流实现

**请求速率限流 limit_req（漏桶算法）**

```nginx
# http 块定义
limit_req_zone $binary_remote_addr zone=req_limit:10m rate=10r/s;

location /api/ {
    limit_req zone=req_limit burst=20 nodelay;
}
```

- `rate=10r/s`：平均每秒 10 个请求（漏桶恒定出水）。
- `burst=20`：允许 20 个请求的突发排队缓冲。
- `nodelay`：突发请求立即处理（不强制按速率延迟），但仍占用 burst 名额；不加 nodelay 则超出 rate 的请求会被排队延迟发出。
- 超过限制返回 503（可用 `limit_req_status` 改）。

**并发连接限流 limit_conn**

```nginx
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

location /download/ {
    limit_conn conn_limit 5;   # 单 IP 最多 5 个并发连接
}
```



Nginx中安全配置

**访问控制与防盗链**

```nginx
# IP 黑白名单
location /admin/ {
    allow 192.168.1.0/24;
    deny all;
}

# 防盗链
location ~* \.(jpg|png|gif)$ {
    valid_referers none blocked example.com *.example.com;
    if ($invalid_referer) { return 403; }
}
```

**基础安全加固**

- 隐藏版本号：`server_tokens off;`
- 限制请求体大小：`client_max_body_size 10m;`（防大包攻击 / 控制上传）
- 超时设置防慢连接攻击：`client_body_timeout`、`client_header_timeout`。
- 配合 WAF（如 ModSecurity）做应用层防护。



使用keepalived做Nginx高可用

- 两台 nginx，keepalived 基于 **VRRP** 协议维护一个 **VIP（虚拟 IP）**。
- 正常时 VIP 在主节点；主节点宕机或 nginx 挂掉（通过脚本检测），VIP 漂移到备节点，对外 IP 不变，实现故障自动切换。
- 可做主主模式（两个 VIP 互为主备）提升资源利用率。



Nginx配置健康检查

- **被动健康检查（开源版自带）**：

  ```nginx
  upstream backend {
      server 10.0.0.1 max_fails=3 fail_timeout=30s;
  }
  ```

  在 `fail_timeout` 时间内失败达到 `max_fails` 次，则该节点被标记为不可用，暂停 `fail_timeout` 后再试。缺点：只有真实请求失败才发现，有延迟。

- **主动健康检查**：开源版需第三方模块 `nginx_upstream_check_module`，或用 NGINX Plus 的 `health_check` 指令，主动定时探测后端。



平滑升级Nginx本身，实现业务不中断

**二进制热升级**：发 `USR2` 信号启动新版本 master 和 worker（与旧的并存），验证无误后发 `QUIT` 让旧进程优雅退出，实现不停机升级 nginx 本身。



Nginx常见状态码

| 状态码                  | 含义               | 常见原因                                   |
| ----------------------- | ------------------ | ------------------------------------------ |
| **499**                 | 客户端主动断开连接 | 客户端超时/用户取消，常因后端响应太慢      |
| **502 Bad Gateway**     | 网关收到无效响应   | 后端进程挂了、连接被拒、后端返回非法响应   |
| **504 Gateway Timeout** | 网关超时           | 后端处理太慢，超过 `proxy_read_timeout` 等 |
| **413**                 | 请求体过大         | 超过 `client_max_body_size`                |