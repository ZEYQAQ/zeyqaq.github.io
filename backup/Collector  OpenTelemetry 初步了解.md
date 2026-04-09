**它是什么？**

简单理解就是一个独立的数据管道服务。应用产生的三种遥测数据（Traces 链路、Metrics 指标、Logs 日志）先发给 Collector，Collector 处理完再转发到后端存储。而不是让应用直接连后端。

**为什么需要它？**

没有 Collector 的话，每个应用要直接对接后端存储，如果你有 50 个服务，后端从 Jaeger 换成 Tempo，就得改 50 个服务的配置。有了 Collector 作为中间层，应用只管往 Collector 发数据，后端怎么换、数据怎么处理，都在 Collector 里统一配置就行。

**它的三段式架构：**

数据流是 `Receivers → Processors → Exporters`，对应接收、处理、导出。

```
# otel-collector-config.yaml

receivers:      http:
        endpoint: 0.0.0.0:4318

processors:
  # 批量处理，攒一批再发，减少网络开销
  batch:
    timeout: 5s
    send_batch_size: 1024

  # 内存限制，防止 OOM
  memory_limiter:
    limit_mib: 512
    check_interval: 5s

exporters:
  # 链路数据发到 Tempo
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

  # 指标数据发到 Prometheus
  prometheus:
    endpoint: 0.0.0.0:8889

  # 日志数据发到 Loki
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

  # 调试用，输出到控制台
  debug:
    verbosity: detailed

service:
  pipelines:
    # 链路管道
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/tempo]

    # 指标管道
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]

    # 日志管道
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loki]
```

**配置的核心逻辑就是在最下面的 `service.pipelines` 里把三段串起来：** 这条管道用什么 receiver 接收、经过哪些 processor 处理、最后用哪个 exporter 导出到哪里。

**部署方式：**

最简单的方式用 Docker 跑：

```bash
docker run -d \
  --name otel-collector \
  -p 4317:4317 \
  -p 4318:4318 \
  -p 8889:8889 \
  -v $(pwd)/otel-collector-config.yaml:/etc/otelcol/config.yaml \
  otel/opentelemetry-collector-contrib:latest
```

**应用端只需要配一个环境变量指向 Collector：**

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://collector地址:4317
```

----

#### 开发如何用？

**总结一下：**

| 方面     | Java               | Go              |
| -------- | ------------------ | --------------- |
| 接入方式 | javaagent 自动注入 | 手动集成 SDK    |
| 改代码   | 不需要             | 需要            |
| 配置方式 | 启动参数/环境变量  | 代码 + 环境变量 |
| 上手难度 | 很低               | 中等            |

-----

#### 尝试一次模拟配置（https://opentelemetry.io/zh/docs/collector/quick-start/）

系统是 macOS arm64，已安装 Homebrew 但未安装 Go。我将用 Homebrew 安装 Go，然后在 `~/.zshrc` 中配置 `GOPATH` 和 `GOBIN`。

```
which go 2>/dev/null; go version 2>/dev/null; uname -m; echo "SHELL=$SHELL"; ls ~/.zshrc 2>/dev/null && echo "zshrc exists"

brew install go
go install github.com/open-telemetry/opentelemetry-collector-contrib/cmd/telemetrygen@latest

```

```
docker run -p 127.0.0.1:4317:4317 -p 127.0.0.1:4318:4318 -p 127.0.0.1:55679:55679 otel/opentelemetry-collector-contrib:0.149.0 2>&1 | tee collector-output.txt # 可选择将输出保存，以便后续搜索
```

![PixPin_2026-04-09_10-24-05](/Users/zeyqaq/Library/Mobile Documents/com~apple~CloudDocs/读书会分享/Collector  OpenTelemetry 初步了解.assets/PixPin_2026-04-09_10-24-05.jpg)



## 配置与所在位置（https://opentelemetry.io/zh/docs/collector/configuration/）（此文档包含最多运维侧存在的问题！）

默认情况下，Collector 配置文件位于 `/etc/<otel-directory>/config.yaml`，其中 `<otel-directory>` 可以是 `otelcol`、`otelcol-contrib` 或其他值，具体取决于你使用的 Collector 版本或发行版。

任何 Collector 配置文件的结构都由四类访问遥测数据的管道组件组成：

\* __接收器__ * __处理器__ * __导出器__ * __连接器__

OpenTelemetry Collector 的架构和 ELK 那套（Filebeat → Logstash → Elasticsearch）的思路不一样。在 ELK 里，采集、处理、存储确实是不同的独立服务。但在 Collector 里，这四个组件都是**配置文件里的不同配置段**，跑在同一个进程中。

拿**ELK 做类比：**

| Collector 组件      | 作用                                                         | 类比 ELK（但都在 Collector 一个进程里） |
| ------------------- | ------------------------------------------------------------ | --------------------------------------- |
| Receiver（接收器）  | 接收数据，支持 OTLP、Kafka、Prometheus 等多种协议            | 类似 Logstash 的 input                  |
| Processor（处理器） | 对数据做过滤、采样、添加属性、批量打包等                     | 类似 Logstash 的 filter                 |
| Exporter（导出器）  | 把数据发到后端存储，比如 Tempo、Prometheus、Loki、ClickHouse | 类似 Logstash 的 output                 |
| Connector（连接器） | 连接两条管道，比如从 Trace 数据中提取出 Metrics              |                                         |

---

OpenTelemetry（OTel）**只包含数据的产生、采集和传输**，不包含后端存储和展示。

具体来说，OTel 项目包含的是：

- **SDK**（各语言的埋点库，Java/Go/Python 等）
- **javaagent**（Java 的自动注入探针）
- **Collector**（接收、处理、导出的中转服务）
- **OTLP 协议**（统一的数据传输协议）
- **telemetrygen**（测试工具）

**Prometheus、Tempo、Loki、Jaeger、Grafana 这些全都不属于 OTel**，它们是独立的项目，OTel 只是能把数据发给它们。

**用快递来类比：**

OTel 就是一套**快递物流体系**——打包（SDK 埋点）、收件（Receiver）、分拣（Processor）、派送（Exporter）。但它不是仓库，也不是收货人。

- **Prometheus** → 仓库，存指标
- **Tempo/Jaeger** → 仓库，存链路
- **Loki/ClickHouse** → 仓库，存日志
- **Grafana** → 收货人，负责看