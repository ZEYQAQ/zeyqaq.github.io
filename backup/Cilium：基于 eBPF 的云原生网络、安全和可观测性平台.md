我问同事，你知道斯乐姆吗？

--同事说，不知道。

类似calico，搞cni的。

--同事说，不知道。

美国公司，搞网络的。

--是思科？



这个产品的母公司被思科收购了。6202年了，思科还是在买买买。



初步学习，建议装：Docker desktop、kubectl、kind、helm、cilium CLI。为什么是kind？

[Cilium 官方教程]: https://docs.cilium.io/en/stable/

默认就是 kind（Kubernetes in Docker），比 minikube 更适合做 CNI 实验（方便禁用默认 CNI）。

```
brew install kind helm cilium-cli 2>&1 | tail -30

#编辑yaml vim kind-cilium.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cilium-lab
networking:
  disableDefaultCNI: true
  kubeProxyMode: none
nodes:
  - role: control-plane
  - role: worker
  - role: worker
#创建集群
kind create cluster --config kind-cilium.yaml 2>&1 | tail -40
#安装kube-proxy的替代
kubectl get nodes 2>&1; echo '---'; API_SERVER_IP=$(docker inspect cilium-lab-control-plane -f '{{.NetworkSettings.Networks.kind.IPAddress}}'); echo "API server IP: $API_SERVER_IP"; cilium install --version 1.17.2 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=$API_SERVER_IP \
  --set k8sServicePort=6443 \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true 2>&1 | tail -20
#等待Cilium就绪
cilium status --wait --wait-duration 3m 2>&1 | tail -30
#等待容器就绪
kubectl -n kube-system get pods -o wide 2>&1 | head -20; echo '---'; kubectl -n kube-system describe pod -l k8s-app=cilium 2>&1 | grep -A5 -i "events\|warning\|error" | head -40
```





Cilium  [ˈsɪliəm]  纤毛 是CNCF的毕业项目，功能强大，可以实现：容器网络（CNI）、代替kube-proxy（实现svc）、L3-L7网络策略、服务网格（类似istio）、负载均衡（替代ingress）；特性是可以实现：网络可观测（Hubble）、透明加密（WireGuard / IPsec）、跨集群互联（Cluster Mesh）。



Cilium基于eBPF，eBPF（extended Berkeley Packet Filter）是 Linux 内核的一项技术，允许在不修改内核代码、不加载内核模块的前提下，把自定义程序"安全地"注入内核执行。可以理解为内核里的 JavaScript。

Cilium的数据包可以绕过 iptables、conntrack 甚至部分协议栈，规则更新无需重载，性能强，O(1))的时间复杂度处理数据包。



Cilium组成：

1、**Cilium Agent（cilium-agent）** 每个节点上以 DaemonSet 运行的守护进程。主要代理，干事的。

2、**Cilium Operator** 集群级别的控制器，处理跨节点的全局任务：IP 地址管理（IPAM）、CRD 状态同步、垃圾回收等。

3、**Cilium CLI（`cilium` 命令）** 管理工具，用来安装、检查状态、调试。

4、**Hubble** Cilium 的可观测性组件。



Cilium作为后起之秀，现在看完全有大一统之势：几乎把CloudNative发展到现在的所有和网络相关的概念和功能汇聚一身，同时还有内核级别的性能。

他们的开发团队不同于Istio，少了书生气但是多了一点科幻迷的理科味。

看他们的Demo就能明白：



StarWarDemo：

在我们这个受《星球大战》启发的示例中，有三个微服务应用：死星（Deathstar）、帝国（Tiefighter）和联盟（Xwing）。死星运行在80端口的HTTP Web服务，该服务以Kubernetes Service的形式暴露，用于在两个Pod副本之间进行负载均衡。死星服务为帝国的飞船提供着陆服务，以便它们可以请求着陆港。帝国Pod代表典型帝国飞船上的着陆请求客户端服务，而联盟Pod则代表联盟飞船上的类似服务。它们的存在是为了测试针对死星着陆服务的不同访问控制安全策略。
<img width="1294" height="920" alt="Image" src="https://github.com/user-attachments/assets/65ceb0ec-20aa-4cbf-be22-81e85ad535bb" />
http-sw-app.yaml中包含了三个服务的deployment，每个deployment使用k8s的label： (`org=empire, class=deathstar`), (`org=empire, class=tiefighter`),  (`org=alliance, class=xwing`)，来区分。此外文件中还定义了一个svc，给Deathstar，这个svc负责将流量分配到所有打了 (`org=empire, class=deathstar`)标签的Pod上面。

```
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.19.3/examples/minikube/http-sw-app.yaml
```

如果你的k8s访问不了，那么使用：

```
---
apiVersion: v1
kind: Service
metadata:
  name: deathstar
  labels:
    app.kubernetes.io/name: deathstar
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    org: empire
    class: deathstar
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deathstar
  labels:
    app.kubernetes.io/name: deathstar
spec:
  replicas: 2
  selector:
    matchLabels:
      org: empire
      class: deathstar
  template:
    metadata:
      labels:
        org: empire
        class: deathstar
        app.kubernetes.io/name: deathstar
    spec:
      containers:
      - name: deathstar
        # renovate: datasource=docker depName=quay.io/cilium/starwars
        image: quay.io/cilium/starwars@sha256:896dc536ec505778c03efedb73c3b7b83c8de11e74264c8c35291ff6d5fe8ada # v2.3
---
apiVersion: v1
kind: Pod
metadata:
  name: tiefighter
  labels:
    org: empire
    class: tiefighter
    app.kubernetes.io/name: tiefighter
spec:
  containers:
  - name: spaceship
    image: quay.io/cilium/json-mock:v1.3.8@sha256:5aad04835eda9025fe4561ad31be77fd55309af8158ca8663a72f6abb78c2603
---
apiVersion: v1
kind: Pod
metadata:
  name: xwing
  labels:
    app.kubernetes.io/name: xwing
    org: alliance
    class: xwing
spec:
  containers:
  - name: spaceship
    image: quay.io/cilium/json-mock:v1.3.8@sha256:5aad04835eda9025fe4561ad31be77fd55309af8158ca8663a72f6abb78c2603
```

等待一会儿初始化：

```
kubectl get pods,svc
```

每个Pod在Cilium里都会被表示为本地Cilium代理的一个endpoint，可以在Cilium的Pod内调用cilium-dbg来列出他们：

```
kubectl -n kube-system exec ds/cilium -- cilium-dbg endpoint list
```

这时候我们验证：没策略时，谁都能停靠

```
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

更改L3/L4策略，只允许帝国战机停靠：

```
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.19.3/examples/minikube/sw_l3_l4_policy.yaml
```

```
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L3-L4 policy to restrict deathstar access to empire ships only"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

<img width="1308" height="926" alt="Image" src="https://github.com/user-attachments/assets/e2448be5-ea30-4c67-a246-819e6be27db6" />
再试一下尝试降落：

```
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
#Ship landed
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

会发现只有帝国可降落～

这时候我们再次检查入口策略：

```
kubectl -n kube-system exec cilium-1c2cz -- cilium-dbg endpoint list
```

如果要看详情：

```
kubectl get cnp
#更进一步：
kubectl describe cnp rule1
```

以上是四层网络的转发，当你想更进一步，比如仅仅向开放一些特定的API（比如限制*tiefighter*只能发起 POST /v1/request-landing API 调用，而禁止所有其他调用（包括 PUT /v1/exhaust-port）。）那就可以继续：

<img width="1354" height="968" alt="Image" src="https://github.com/user-attachments/assets/8c29eff0-0eb5-40b7-8c00-983ea81412e2" />


```
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.19.3/examples/minikube/sw_l3_l4_l7_policy.yaml
```

```
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L7 policy to restrict access to specific HTTP call"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/v1/request-landing"
```

现在我们可以重新运行上述相同的测试，但会看到不同的结果：

```
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed
```

和

```
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Access denied
```

由于此规则建立在身份感知规则之上，因此来自没有标签的 pod 的流量 `org=empire`将继续被丢弃，导致连接超时：

```
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

如您所见，借助 Cilium L7 安全策略，我们能够仅允许 *tiefighter*访问*deathstar*上所需的 API 资源，从而为微服务之间的通信实现“最小权限”安全方法。请注意，它`path`匹配的是精确的 URL，例如，如果您想允许访问 /v1/ 下的所有内容，则需要使用正则表达式：

```
path: "/v1/.*"
```

可以通过以下方式查看 L7 策略：

```
$ kubectl describe ciliumnetworkpolicies
Name:         rule1
Namespace:    default
Labels:       <none>
Annotations:  API Version:  cilium.io/v2
Description:  L7 policy to restrict access to specific HTTP call
Kind:         CiliumNetworkPolicy
Metadata:
  Creation Timestamp:  2020-06-15T14:06:48Z
  Generation:          2
  Managed Fields:
    API Version:  cilium.io/v2
    Fields Type:  FieldsV1
    fieldsV1:
      f:description:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:endpointSelector:
          .:
          f:matchLabels:
            .:
            f:class:
            f:org:
        f:ingress:
    Manager:         kubectl
    Operation:       Update
    Time:            2020-06-15T14:10:46Z
  Resource Version:  3445
  Self Link:         /apis/cilium.io/v2/namespaces/default/ciliumnetworkpolicies/rule1
  UID:               eb3a688b-b3aa-495c-b20a-d4f79e7c088d
Spec:
  Endpoint Selector:
    Match Labels:
      Class:  deathstar
      Org:    empire
  Ingress:
    From Endpoints:
      Match Labels:
        Org:  empire
    To Ports:
      Ports:
        Port:      80
        Protocol:  TCP
      Rules:
        Http:
          Method:  POST
          Path:    /v1/request-landing
Events:            <none>
```

也可以进cilium-dbg的容器，使用cli：

```
$ kubectl -n kube-system exec cilium-qh5l2 -- cilium-dbg policy get
[
  {
    "endpointSelector": {
      "matchLabels": {
        "any:class": "deathstar",
        "any:org": "empire",
        "k8s:io.kubernetes.pod.namespace": "default"
      }
    },
    "ingress": [
      {
        "fromEndpoints": [
          {
            "matchLabels": {
              "any:org": "empire",
              "k8s:io.kubernetes.pod.namespace": "default"
            }
          }
        ],
        "toPorts": [
          {
            "ports": [
              {
                "port": "80",
                "protocol": "TCP"
              }
            ],
            "rules": {
              "http": [
                {
                  "path": "/v1/request-landing",
                  "method": "POST"
                }
              ]
            }
          }
        ]
      }
    ],
    "labels": [
      {
        "key": "io.cilium.k8s.policy.derived-from",
        "value": "CiliumNetworkPolicy",
        "source": "k8s"
      },
      {
        "key": "io.cilium.k8s.policy.name",
        "value": "rule1",
        "source": "k8s"
      },
      {
        "key": "io.cilium.k8s.policy.namespace",
        "value": "default",
        "source": "k8s"
      },
      {
        "key": "io.cilium.k8s.policy.uid",
        "value": "eb3a688b-b3aa-495c-b20a-d4f79e7c088d",
        "source": "k8s"
      }
    ]
  }
]
Revision: 11

```

清理所有Demo：

```
kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/1.19.3/examples/minikube/http-sw-app.yaml 
kubectl delete cnp rule1
```



这个小的演示流程展示了Cilium在七层路由的特点，它可以实现类似istio的基于互联网协议的流量管控。

不过不同于istio的服务网格：在每个 Pod 里都注入一个 Envoy 代理容器,所有进出流量都被 Envoy 拦截,由它来做流量的加密、路由、监控。

Cilium的能力更强：Pod IP 会变(扩缩容、重启),如果策略规则写"允许 IP 10.244.1.5 访问 10.244.2.8",Pod 一重建就失效了。如果给每对 Pod IP 都写 iptables 规则,集群一大规则爆炸。基于identity进行流量管理，把一组标签相同的 Pod 归为一个 identity,分配一个数字 ID(比如 class=tiefighter 的所有 Pod 都是 identity 12345)。每个数据包在 Cilium 里都带上来源 identity(通过 eBPF 写在包元数据里),节点收到包后查一下 identity 之间是否允许通信就行。Pod 数量变化不影响规则数量,只跟标签组合数有关,可扩展性好得多。



当然，这里提及一点：

##### 寻址 ≠ 鉴权,这是两件事

网络通信里有两个独立的问题:

1. **寻址(routing)**:这个包要送到哪台机器?走什么路径?
2. **鉴权(policy)**:这个包**允许**送过去吗?

- **IP** 解决问题 1
- **identity** 解决问题 2

Cilium 没有发明新的寻址机制,它**仍然用 IP 路由**把包送到目标 Pod。identity 只在判断"允不允许"的时候用。



### 完整流程:帝国 → deathstar

假设帝国 Pod(IP `10.244.1.5`)要访问 deathstar Service(ClusterIP `10.96.0.5:80`),后端 Pod IP 是 `10.244.2.8`。

**步骤 1:Service 解析(还是 IP 在干活)**

Cilium(或 kube-proxy)做 DNAT,把目标地址 `10.96.0.5:80` 改成真实 Pod IP `10.244.2.8:80`。

**步骤 2:发送端打标签**

包从帝国 Pod 出来时,**源节点**的 eBPF 程序查表:

> "这个 Pod IP `10.244.1.5` 属于哪个 identity?" → identity `12345`(帝国)

然后把 identity `12345` 这个数字**编码进数据包**(通过 VXLAN header、IPSec/WireGuard 字段、或者特殊 IP 选项,取决于你的 datapath 模式)。

**步骤 3:正常 IP 路由送过去**

包按照**普通的 IP 路由**(VXLAN 隧道 / 直接路由 / 云厂商 CNI)送到目标 Pod 所在的节点。这一段**和普通 K8s 网络没区别**,没有什么"identity 寻路"。

**步骤 4:接收端鉴权**

包到达目标节点时,eBPF 程序拆出包里的 identity:

> 来源 identity `12345` → 本地 Pod `10.244.2.8`(identity `67890`,deathstar)
>
> 查策略表:`12345 → 67890:80/TCP` 允许吗?

允许就放行交给 Pod,不允许就 drop。

**步骤 5:Pod 收到包**

deathstar 容器收到一个普通的 TCP 包,从它的视角看一切都是标准 IP 网络。

##### 关键点:identity 是"乘客信息",不是"地址"

IP → identity 的对应表：Cilium 在每个节点维护一份映射表(`cilium endpoint list` 可以看到):

```
kubectl -n kube-system exec ds/cilium -- cilium endpoint list
kubectl -n kube-system exec ds/cilium -- cilium identity list


ENDPOINT   IDENTITY   IPv4              LABELS
1234       12345      10.244.1.5        class=tiefighter
5678       67890      10.244.2.8        app=deathstar
```

这份表通过 Cilium agent 之间(或经由 kvstore / Kubernetes API)同步,所以**每个节点都知道"哪个 IP 属于哪个 identity"**。





Cilium能大一统K8S的网络，不只是靠他的identity，也在于它有一个强大的可视化应用，可以帮助运维同事看到已经配置的网络策略和实际的业务流量。

精巧的GUI，你可以清晰的看到不同namespace下的流量流向：


```
cilium hubble ui 
```

<img width="1611" height="929" alt="Image" src="https://github.com/user-attachments/assets/86eebb1e-507d-4c14-8c48-fa3fd2039e22" />
不够炫酷？

```
cilium connectivity test
```

官方提供了一键test命令，使用之后会产生很多拟真Pod和对应的流量，试试看！

<img width="1611" height="929" alt="Image" src="https://github.com/user-attachments/assets/082ebb3a-08fd-4e1f-8370-386360e500b1" />



对于我司的业务：华为云上CCE仍然用的原来的那一套，如果要用Cilium的话，加钱上CCE Turbo就好了。战未来嘛～



最后，

想好了说再见？

```
kind delete cluster --name cilium-lab  
```

