跨站脚本攻击（XSS）是前端常见而且危害巨大的安全漏洞，攻击可以在页面上注入恶意脚本来窃取cookies或者劫持页面发送恶意请求。所以CSP（Content Security Policy，内容安全策略）应运而生，CSP是由服务器通过HTTP响应头（或者html标签）下发给浏览器的一套安全策略，告诉浏览器，这个页面只能从哪些域名加载资源，只允许执行哪些经过认证的脚本。



CSP头部的格式通常有多个指令（Directives）组成，以;分割，举例：

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://qaq-cdn.example.com; img-src *;
```

这里的3个指令，default-src、script-src、img-src的作用分别是：兜底策略（未单独指定的资源类型均遵循此规则）、限制 JS 脚本的执行与加载来源（最核心）、限制图片的加载来源。

这里的self和https://qaq-cdn.example.com * 都是「配置」，其中self比较特殊，指的是同源（Same-Origin），也就是说浏览器只允许加载和执行与当前页面所在地址“协议、域名、端口”完全一致的资源。

举例，如果你访问的是https://www.example.com/home‌（默认 HTTPS 端口为 443），当配置了 script-src 'self' 或 default-src 'self' 时，浏览器对不同资源的判定如下：

| 请求资源 URL                            | 是否属于 `'self'` | 原因说明                              |
| --------------------------------------- | ----------------- | ------------------------------------- |
| `https://www.example.com/static/app.js` | ✅ **允许**        | 协议、域名、端口完全一致              |
| `https://www.example.com/api/getUser`   | ✅ **允许**        | 协议、域名、端口完全一致              |
| `http://www.example.com/app.js`         | ❌ **拦截**        | **协议不同**（HTTP HTTPS）            |
| `https://api.example.com/data`          | ❌ **拦截**        | **域名不同**（子域名也不属于同源）    |
| `https://cdn.example.com/app.js`        | ❌ **拦截**        | **域名不同**（静态资源 CDN 域名不同） |
| `https://www.example.com:8080/app.js`   | ❌ **拦截**        | **端口不同**（8080 443）              |





在启用严格的在启用严格的 script-src 时，浏览器默认禁止执行任何内联脚本（如 <script>console.log('hi')</script> 或 HTML 属性 onclick="..."），因为浏览器无法在没有规则的情况下区分内联脚本是“业务正常代码”还是“攻击者注入的恶意脚本”。

业界解决内联脚本执行合法性主要有两种主流机制：

一是基于Nonce动态令牌的方式，在CSP头下发随机的token，只有带相同nonce的script才能执行。

二是彻底消除内联执行，将script改为type="application/json"，作为数据节点而非脚本。



基于Nonce的，在页面构建/网关注入时，会给<script>加上一个占位符nonce="xxx-csp-token"，当用户请求到达接入层网关（如LB）时，网关动态生成一个高强度随机token，比如（ZeYQa0）并完成1、在 HTTP 响应头中下发：Content-Security-Policy: script-src 'nonce-ZeYQa0' ...2、在返回的 HTML 中将占位符替换为真实 Token：<script nonce="ZeYQa0">...3、浏览器校验 HTML 中的 nonce与响应头一致，允许执行。

好处在于：改造成本几乎为0



基于 Non-Executable Data Tag / 数据岛机制的，平台将注入脚本标签的type改为application/json，浏览器解析到该标签时，认为它只是一个“纯数据块”，不会当作 JavaScript 解释执行，因此完全不受 script-src 规则的拦截。

