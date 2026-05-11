| 目标                                | 使用机制                                                     |
| :---------------------------------- | :----------------------------------------------------------- |
| 其他工作负载不能调度上来            | Node **Taint** + 目标 Pod 的 **Toleration**                  |
| 目标工作负载只被调度到指定节点      | **NodeAffinity** 或 `nodeSelector`（给节点打标签）           |
| 同一节点上该工作负载只能跑 1 个 Pod | **Pod 反亲和** `podAntiAffinity` (topologyKey: `kubernetes.io/hostname`) |

如果要想实现：

我的某个工作负载只调度到一个指定Worker Node并且此Node只能接受此工作负载的Pod调度，拒绝其他工作负载调度，且此工作负载的多个Pod可以同时跑。那么就需要：

1、给目标节点打标签 + 打污点（`NoSchedule` 足以阻止其他 Pod 调度；若要连已运行的 Pod 也驱逐，用 `NoExecute`）：

```
# 打标签：用于亲和选择
kubectl label node <node-ip> dedicated=app-a

# 打污点：排斥其他没有对应容忍的 Pod
kubectl taint node <node-ip> dedicated=app-a:NoSchedule
```

2、工作负载yaml修改

`metadata.labels` 是给这个 Deployment 对象本身打的标签（便于检索/筛选）；
`spec.template.metadata.labels` 是给 Pod 打的标签；
`tolerations` 让 Pod 能容忍节点污点；
`nodeAffinity` 让 Pod 只去匹配的节点。

```
metadata:
  labels:
    ontoagent.kingdee.com/app: "true"

spec:
  template:
    metadata:
      labels:
        ontoagent.kingdee.com/app: "true"
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/hostname
                    operator: In
                    values:
                      - <node-ip>
      tolerations:
        - key: ontoagent.kingdee.com/app
          operator: Equal
          value: "true"
          effect: NoSchedule
```

> [!NOTE]
>
> Kubernetes 对 **标签（Label）** 和 **污点（Taint）** 的 key 有一套命名规范，合理使用「前缀（prefix）」能避免与系统保留 key 冲突，也方便团队协作时一眼看出用途。
>
> - **前缀（可选）**：必须是 DNS 子域名（如 `example.com`、`team.company.io`），长度 ≤ 253 字符
> - **名称（必填）**：字母、数字、`-`、`_`、`.`，开头结尾必须是字母或数字，长度 ≤ 63 字符
> - **保留前缀**：`kubernetes.io/`、`k8s.io/` 由系统使用，业务请勿随意占用
> - ️ **YAML 里布尔值陷阱**：`true` 在 YAML 中会被解析为布尔，而 label/taint 的值必须是字符串，所以务必写成带引号的 `"true"`；`kubectl label`/`kubectl taint` 命令行里直接写 `true` 则没问题。
>
> > 💡 官方和社区约定俗成的一个通用 key：`dedicated`（专属节点），常用于节点专用化场景，无前缀也合规。



这里用的是节点亲和（NodeAffinity）：

**作用**：让 Pod 调度到满足特定**节点标签**的节点上。

| 操作符         | 含义                             |
| :------------- | :------------------------------- |
| `In`           | 标签值在列表中                   |
| `NotIn`        | 标签值不在列表中（相当于反亲和） |
| `Exists`       | 存在该 key                       |
| `DoesNotExist` | 不存在该 key                     |
| `Gt` / `Lt`    | 大于 / 小于（整数）              |

同时也有Pod亲和（PodAffinity）：

**作用**：让 Pod 调度到**已经运行着某些 Pod**的节点（或拓扑域），实现"**凑在一起**"。

| topologyKey 值                  | 含义         |
| :------------------------------ | :----------- |
| `kubernetes.io/hostname`        | 同一个节点   |
| `topology.kubernetes.io/zone`   | 同一个可用区 |
| `topology.kubernetes.io/region` | 同一个地域   |

也有Pod 反亲和（PodAntiAffinity）：

**作用**：**避免** Pod 和某些 Pod 调度到一起，实现"**分散部署**"。