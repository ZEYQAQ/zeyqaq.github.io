Anthropic在我的主观视角里总以一种“精确的工程师”的形象出现。

我看到这个status.claude.com/的时候有两种想法：
1、Anthropic想到了一种更直观的展示服务状态的判断机制。错误和宕机不可避免，但是 只有Claude一家AI敢于直面自己的错误。因此，真正发生服务不可用时，极大可能地避免了客户不明是自己本地的还是Anthropic的问题的疑问。敢于承认错误不仅没有降低他们的SLA，相反，这个页面会让人感觉到“即使Claude挂了也是可靠的”
2、Claude真的很容易崩。

status.claude.com的样式：

![Image](https://github.com/user-attachments/assets/2a6bdf63-eb8d-4d6e-8719-244de46c45b1)

Claude的手绘风格也加深了我对其公司的“工程师”的主观印象。相对于OpenAI的偏平现代和Google Gemini坚持用Google Fonts的安卓味道，Anthropic的风格相对讨喜。

于是我打算从Prometheus的学习入手，设计一个类似于Claude风格的对www.zeyqaq.com可用性的网站：https://status.zeyqaq.com

我初步的计划为：
1、选用独立于www.zeyqaq.com的服务器作为status.zeyqaq.com的服务器，这样能尽可能避免两个网站一起崩掉。
2、Ruohai为我提供了一个思路：使用Cloudflare免费提供的反向代理，将status.zeyqaq.com的443端口路由至我带有公网IPv4地址的腾讯云服务器。
3、Vibe一个前端，前端形象参考status.claude.com的样式。
4、在服务器上安装Prometheus，使用拨测或使用成熟的exporter，检测网站可用性。
5、网站前端通过公网调取Prometheus的数据，不使用Grafana，仅调用Prometheus的Graph接口。

以下是我的提示词：
/plan
你需要为我构建一个类似status.claude.com的风格的页面，用于监控www.zeyqaq.com的拨测状态，制作前端页面（建议使用react框架），电脑有docker你可以选用nginx来打包整个前端。拨测的结果数据通过公网访问另一台服务器的Prometheus/garph，这个数据来源地址我需要你以环境变量的形式预留（PromUrl="https://url:12345")。其中nginx容器需要监听80和443端口，但是我会在最后以443端口以nodeport形式暴露到k8s。