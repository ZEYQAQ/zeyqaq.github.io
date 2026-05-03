古人云：如果一层中间件解决不了问题，那就再加一层中间件。

而我们就遇到了这样的一个问题：我们的prometheus的防火墙配置的过于繁琐，同时prometheus.yml不规范；每次修改prometheus.yml都需要reload，无法批量化暴露node的指标。我们就引入了Consul，这个服务发现工具，在每次安装node_exporter的时候，顺带安装Consul的Client。这样我们只需要在prometheus.yml中配置Consul的server，而由Consul管理需要监控的node主机。免去了人工修改的低效与误操作的风险。



下面是我整理的操作流程及一键部署脚本，供参考。



## 安装node_exporter

1、下载

2、解压安装移动

```
 tar -zvxf node_exporter-1.11.1.linux-amd64.tar.gz
 mv node_exporter-1.11.1.linux-amd64/node_exporter /usr/local/bin/
 chmod +x /usr/local/bin/node_exporter 
```

3、创建systemd

```
cat > /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now node_exporter
systemctl status node_exporter
```



## 在consul中注册

```
#web访问：
http://<server_ip>:8500/ui/dc1/services
http://10.101.128.98:8500/ui/dc1/services
```

1、安装consul客户端（在被监控的主机

```
cd /tmp
wget https://releases.hashicorp.com/consul/1.21.5/consul_1.21.5_linux_amd64.zip
unzip consul_1.21.5_linux_amd64.zip
mv consul /usr/local/bin/
chmod +x /usr/local/bin/consul
consul version   # 验证
```

2、写 agent 主配置 `/etc/consul.d/consul.hcl`:

i）agent 配置加上 ACL/加密相关项(如果 server 端开了的话)（1.21 版本默认推荐启用 ACL 和 gossip 加密。）

查看加密配置：

```bash
# 在 server 上看
consul info | grep -E 'acl|encrypted'
```

如果 server 端开了 ACL 和加密,client agent 配置要补全。完整版 `/etc/consul.d/consul.hcl`:

```hcl
datacenter   = "dc1"
data_dir     = "/var/lib/consul"
node_name    = "<node的名称>"
bind_addr    = "<node的IP>"
client_addr  = "127.0.0.1"
server       = false
retry_join   = ["你的Consul Server IP"]

# 如果 server 启用了 gossip 加密(consul keygen 生成的那个 key)
encrypt = "你的加密key=="

# 如果 server 启用了 ACL
acl {
  enabled        = true
  default_policy = "deny"
  enable_token_persistence = true
  tokens {
    agent   = "agent-token-uuid"      # 注册服务用的 token
    default = "default-token-uuid"    # 可选
  }
}

# 1.21 推荐显式开 telemetry(可选)
telemetry {
  disable_hostname = true
}
```

如果 server 是裸跑没开 ACL/加密,这一步跳过。

ii）没开加密使用：

```
vim /etc/consul.d/consul.hcl
```

```
datacenter   = "dc1"
data_dir     = "/var/lib/consul"
node_name    = "<node的名称>"
bind_addr    = "<node的IP>"
client_addr  = "127.0.0.1"
server       = false
retry_join   = ["你的Consul Server IP"] 
```

```
vim /etc/systemd/system/consul.service
```

```
[Unit]
Description=Consul Agent
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/consul agent -config-dir=/etc/consul.d
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```
systemctl daemon-reload
systemctl enable --now consul

# 验证已加入集群
consul members
```

3、注册 node_exporter 服务

在client上

```
vim /etc/consul.d/node-exporter.json
```

```
{
  "service": {
    "id": "node-exporter-<node的IP>",
    "name": "node-exporter",
    "tags": ["prometheus", "metrics"],
    "address": "<node的IP>",
    "port": 9100,
    "meta": {
      "job": "prod"
    },
    "check": {
      "id": "node-exporter-http",
      "name": "node_exporter HTTP check",
      "http": "http://<node的IP>:9100/metrics",
      "method": "GET",
      "interval": "15s",
      "timeout": "3s"
    }
  }
}
```

```
consul reload
```

4、验证结果：

1、client上：

```
# 看服务列表
curl -s http://127.0.0.1:8500/v1/agent/services | python3 -m json.tool

# 看健康检查状态
curl -s http://127.0.0.1:8500/v1/health/service/node-exporter?passing | python3 -m json.tool
```

2、在 Consul UI 上看

浏览器打开 `http://你的Consul:8500/ui`,Services 列表里应该出现 `node-exporter`

3、prometheus上验证

```
curl -s http://localhost:9090/api/v1/targets | grep -A3 '<client_ip>'
```



## 在prometheus里配置consul

```
 vim /usr/local/prometheus/prometheus.yml
```

```
scrape_configs:

  # 1. 监控 Consul 集群自身
  - job_name: 'consul'
    static_configs:
      - targets: ['10.101.128.100:8500', '10.101.128.98:8500', '10.101.128.99:8500']
    metrics_path: /v1/agent/metrics
    params:
      format: ['prometheus']

  # 2. 从 Consul 自动发现node-exporter
  - job_name: 'node-exporter'
    consul_sd_configs:
      - server: '127.0.0.1:8500'
        datacenter: 'dc1'
        services: ['node-exporter']
    relabel_configs:
      # 只保留健康实例
      - source_labels: [__meta_consul_health]
        action: keep
        regex: passing
      # 用 IP:PORT 作为 instance 标签
      - source_labels: [__meta_consul_service_address, __meta_consul_service_port]
        separator: ':'
        target_label: instance
      - source_labels: [__meta_consul_dc]
        target_label: datacenter
      - source_labels: [__meta_consul_node]
        target_label: consul_node
      # 保留原始 job tag (sbx/prod) 作为额外标签，方便按环境筛选
      - source_labels: [__meta_consul_service_metadata_job]
        target_label: env

```



如果缺少iptables监控模块（那就在client的机器上：

```
# 1. 加载模块
sudo modprobe nf_conntrack

# 2. 验证已加载
lsmod | grep nf_conntrack
# 应该看到 nf_conntrack 这一行

# 3. 验证 /proc 接口出现了
cat /proc/sys/net/netfilter/nf_conntrack_count
# 输出一个数字(当前连接跟踪数)

# 4. 验证 exporter 现在能暴露指标了
curl -s http://localhost:9100/metrics | grep nf_conntrack
# 应该看到 node_nf_conntrack_entries 等

# 5. 设置开机自动加载(避免重启后又没了)
echo "nf_conntrack" | sudo tee /etc/modules-load.d/nf_conntrack.conf
```



---

Claude：当你已经配置好prometheus和ConsulServer之后，你可以在node上使用本脚本，一键安装并配置。

```
#!/usr/bin/env bash
#Author:ZEYQAQ
set -euo pipefail

NODE_EXPORTER_VERSION="${NODE_EXPORTER_VERSION:-1.11.1}"
CONSUL_VERSION="${CONSUL_VERSION:-1.21.5}"
CONSUL_SERVER_IP="${CONSUL_SERVER_IP:-<这是你的Prometheus到机器的IP>}"
CLIENT_IP="${CLIENT_IP:-$(ip route get "${CONSUL_SERVER_IP}" 2>/dev/null | awk '{for(i=1;i<=NF;i++) if($i=="src"){print $(i+1); exit}}')}"
CLIENT_IP="${CLIENT_IP:?cannot auto-detect CLIENT_IP, please pass it explicitly}"
echo "==> CLIENT_IP=${CLIENT_IP}"
DATACENTER="${DATACENTER:-dc1}"
NODE_NAME="${NODE_NAME:-kworkclient-${CLIENT_IP//./-}}"
PKG_HOST="${PKG_HOST:-<这是你的存有文件的机器IP>}"
PKG_USER="${PKG_USER:-root}"
PKG_PASS="${PKG_PASS:-<这是你的存有文件的机器明文密码>}"
PKG_PATH="${PKG_PATH:-/root}"
if [[ $EUID -ne 0 ]]; then
  echo "must run as root" >&2
  exit 1
fi
PKG_URL="${PKG_URL:-http://${PKG_HOST}:8000}"
echo "==> [1/6] fetch & install node_exporter ${NODE_EXPORTER_VERSION} from ${PKG_URL}"
cd /tmp
NE_TGZ="node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"
if [[ ! -f "$NE_TGZ" ]]; then
  curl -fsSLO "${PKG_URL}/${NE_TGZ}"
fi
tar -zxf "$NE_TGZ"
mv -f "node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter" /usr/local/bin/
chmod +x /usr/local/bin/node_exporter
echo "==> [2/6] systemd unit for node_exporter"
cat > /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now node_exporter
systemctl --no-pager status node_exporter | head -n 5 || true
echo "==> [3/6] install consul ${CONSUL_VERSION}"
cd /tmp
CONSUL_ZIP="consul_${CONSUL_VERSION}_linux_amd64.zip"
if [[ ! -f "$CONSUL_ZIP" ]]; then
  curl -fsSLO "${PKG_URL}/${CONSUL_ZIP}"
fi
command -v unzip >/dev/null 2>&1 || { (command -v dnf >/dev/null && dnf install -y unzip) || yum install -y unzip; }
unzip -o "$CONSUL_ZIP" >/dev/null
mv -f consul /usr/local/bin/
chmod +x /usr/local/bin/consul
consul version
echo "==> [4/6] consul agent config (no ACL / no encryption)"
mkdir -p /etc/consul.d /var/lib/consul
cat > /etc/consul.d/consul.hcl <<EOF
datacenter   = "${DATACENTER}"
data_dir     = "/var/lib/consul"
node_name    = "${NODE_NAME}"
bind_addr    = "${CLIENT_IP}"
client_addr  = "127.0.0.1"
server       = false
retry_join   = ["${CONSUL_SERVER_IP}"]
EOF

cat > /etc/systemd/system/consul.service <<'EOF'
[Unit]
Description=Consul Agent
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/consul agent -config-dir=/etc/consul.d
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now consul
sleep 3
consul members || true
echo "==> [5/6] register node_exporter service"
cat > /etc/consul.d/node-exporter.json <<EOF
{
  "service": {
    "id": "node-exporter-${CLIENT_IP}",
    "name": "node-exporter",
    "tags": ["prometheus", "metrics"],
    "address": "${CLIENT_IP}",
    "port": 9100,
    "meta": {
      "job": "prod"
    },
    "check": {
      "id": "node-exporter-http",
      "name": "node_exporter HTTP check",
      "http": "http://${CLIENT_IP}:9100/metrics",
      "method": "GET",
      "interval": "15s",
      "timeout": "3s"
    }
  }
}
EOF
consul reload
echo "==> [6/6] load nf_conntrack module"
modprobe nf_conntrack || true
echo "nf_conntrack" > /etc/modules-load.d/nf_conntrack.conf
echo "==> done. verify:"
echo "    curl -s http://127.0.0.1:8500/v1/agent/services | python3 -m json.tool"
echo "    curl -s 'http://127.0.0.1:8500/v1/health/service/node-exporter?passing' | python3 -m json.tool"

```

