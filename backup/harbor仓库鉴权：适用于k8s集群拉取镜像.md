针对镜像仓库拉取，经常会出现以下问题

1、我分明在内网，为什么拉不到镜像？

2、我在办公网了，怎么报错？老师给我加个临时用户密码！～

3、这个我怎么不需要密码就拉取了？

4、我k8s起容器怎么拉不到镜像，你们云服务之间还没打通吗？？



除了网络有安全组限制，其余几乎所有的核心问题都在于：failed to authorize: failed to fetch anonymous token: ... 401 Unauthorized

容器运行时尝试**匿名**拉取镜像，但harbor拒绝了（401 未授权）。说明这个镜像仓库不是公开的，需要凭证才能访问。



常见的排查与修复：

1. 手动验证能否登录和拉取

在有docker desktop的节点（最好是Mac，windows总是有各种虚拟化的问题，没人想在拉取了2层镜像后卡住吧...）上执行：

```bash
docker login <地址> -u cn-south-1@<AK> -p <登录密钥>
docker pull <地址>/<组织>/<镜像名词>
```

如果是云服务，登录密钥的生成方式：华为云控制台 → SWR → 我的镜像 → 客户端上传 → 生成临时登录指令；如果是公司自建，联系devops工程。

2、创建/更新 K8s Secret

```
kubectl get secret

NAME                              TYPE                                  DATA      AGE
default-secret                    kubernetes.io/dockerconfigjson        1         4y28d
default-token-n8vz7               kubernetes.io/service-account-token   3         4y28d
obs-脱敏脱敏脱敏脱敏脱敏脱敏脱敏-pro   cfe/secure-opaque                     2         234d
paas.elb                          cfe/secure-opaque                     1         4y28d
```

一般云服务，都会在创建集群的时候自动帮你绑定上他们自家的镜像仓库，所以看default-secret可以排查很多问题。

```
kubectl edit secret default-secret

apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyIxMDAuMTI1LjE2LjY1OjIwMjAyIjp7ImF1dGgiOiJZMjR0YzI5MWRHZ3RNVUJJVTFRelFWcEJOREpLTURrNVIxQTBRMFJRT0RveFlUQTVZMk5tT0RsaE1XVmlOekJ<脱敏>T0RNNFpXSmtNREkzT0RWbFkyVXdZV0UxWkRCaCJ9LCJzd3IuY24tc291dGgtMS5teWh1YXdlaWNsb3VkLmNvbSI6eyJhdXRoIjoiWTI0dGMyOTFkR2d0TVVCSVUxUXpRVnBCTkRKS01EazVSMUEwUTBSUU9Eb3hZVEE1WTJObU9EbGhNV1ZpTnpCaU9UZzJPREZpWXpVek1qTmhPREppWTJOaU9UUmlZV1F5WkdKa09ETTRaV0prTURJM09EVmxZMlV3WVdFMVpEQmgifX19
kind: Secret
metadata:
  annotations:
    swr-auth-may-expires-at: 2026-05-21 11:25:37.819391 +0000 UTC
  labels:
    secret-generated-by: cce
    manager: Go-http-client
    operation: Update
  name: default-secret
  namespace: default
type: kubernetes.io/dockerconfigjson

```

本质上是创建了一个特殊的 Secret，内容是 `~/.docker/config.json` 的 base64 编码，用：

```
echo "base64字符串" | base64 -d

{
    "auths": {
        "100.125.16.65:20202": {
            "auth": "Y24tc291dGgtdGVzdD1BQVU1QzFp？？？Y2Q0OWZi"
        },
        "swr.cn-south-1.myhuaweicloud.com": {
            "auth": "Y24tc291dGgtdGVz？？？Y2Q0OWZi"
        }
    }
}
```

3、绑定Secret

i）namespace 级别：在 Pod / Deployment 中引用

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      imagePullSecrets:           # ← 关键
        - name: my-registry-secret
      containers:
        - name: app
          image: swr.cn-south-1.myhuaweicloud.com/myorg/myapp:v1
```

ii）绑定到 ServiceAccount

如果一个 namespace 里所有 Pod 都要用同一个镜像仓库，挨个写 `imagePullSecrets` 很烦。可以把 Secret 绑到 ServiceAccount 上：

```bash
kubectl patch serviceaccount default \
  -n my-namespace \
  -p '{"imagePullSecrets": [{"name": "my-registry-secret"}]}'


#查看
kubectl edit serviceaccount default

apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: default
  uid: e3a670c3-3404-42f6-b4be-4316a84fecd0
secrets:
- name: <你上一部的secret>
```

之后该 namespace 下用 `default` ServiceAccount 的 Pod（不指定就是 default）会自动带上这个 Secret，不用在 Pod spec 里写。

iii）节点级别配置（不建议

直接在每个节点上配置容器运行时的凭证，所有 Pod 都能用，无需 Secret。

containerd（K8s 1.24+ 主流方式）：编辑 `/etc/containerd/config.toml`：

```toml
[plugins."io.containerd.grpc.v1.cri".registry.configs."swr.cn-south-1.myhuaweicloud.com".auth]
  username = "cn-south-1@AKxxxxx"
  password = "<登录密钥>"
```

然后 `systemctl restart containerd`。

Docker：在节点上 `docker login`，凭证写入 `/root/.docker/config.json`，kubelet 会读取。

iv）云厂商集成（IAM/IRSA 等）

云上 K8s（AWS EKS、华为云 CCE、阿里云 ACK 等）通常支持用节点的 IAM 角色或临时令牌自动认证，无需手动管理 Secret。

- AWS EKS：节点 IAM Role 带上 ECR pull 权限，kubelet 通过 `ecr-credential-provider` 自动获取临时 token
- 华为云 CCE：默认在每个 namespace 创建 `default-secret`，CCE 会自动刷新华为云 SWR 的临时凭证（24 小时一轮换）
- GKE：节点 SA 自动认证 GCR/Artifact Registry



kubelet 按顺序查找凭证：

1. Pod 的 `imagePullSecrets` 中匹配该 registry 的凭证
2. Pod 所用 ServiceAccount 的 `imagePullSecrets`
3. 节点上 kubelet 配置的 credential provider（云厂商插件）
4. 节点上 `/var/lib/kubelet/config.json`、`~/.docker/config.json` 等位置的凭证
5. 都没有 → 匿名拉取 

kubelet会每次都拉取镜像吗？这个是由工作负载配置：

```
containers:
  - name: app
    image: myrepo/app:v1
    imagePullPolicy: IfNotPresent   # Always / IfNotPresent / Never
```

| 策略           | 行为                                             |
| -------------- | ------------------------------------------------ |
| `Always`       | 每次启动 Pod 都去仓库拉（latest 标签默认是这个） |
| `IfNotPresent` | 节点本地有就用本地的（其他 tag 默认）            |
| `Never`        | 只用本地，没有就报错，不去仓库                   |

如果设成 `IfNotPresent` 且节点已有该镜像，就不会触发鉴权流程。