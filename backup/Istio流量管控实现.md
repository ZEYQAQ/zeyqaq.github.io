1、下载x86（记得指定linux

```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.29.1 TARGET_OS=Linux TARGET_ARCH=x86_64 sh -
```

2、发送到/eddie

```
#发送到/eddie
cd /eddie
unzip istio-1.29.1.zip
cd istio-1.29.1
export PATH=$PWD/bin:$PATH
```

下载完成

----

### 平台准备

```
Istio 1.29 支持以下这些 Kubernetes 版本： 1.31, 1.32, 1.33, 1.34, 1.35。（tianti是1.31)
```

安装完成后

配置 [ELB](https://support.huaweicloud.com/intl/productdesc-elb/en-us_topic_0015479966.html) 以暴露 Istio 入口网关（如果需要，本次演示未使用）。

- [创建弹性负载均衡器](https://console.huaweicloud.com/vpc/?region=ap-southeast-1#/elbs/createEnhanceElb)

- 绑定 ELB 实例到 `istio-ingressgateway` 服务

  将 ELB 实例 ID 和 `loadBalancerIP` 设为 `istio-ingressgateway`。

  ```
  kubectl apply -f - <<EOF
  apiVersion: v1
  kind: Service
  metadata:
    annotations:
      kubernetes.io/elb.class: union
      kubernetes.io/elb.id: 4ee43d2b-cec5-4100-89eb-2f77837daa63 # ELB ID
      kubernetes.io/elb.lb-algorithm: ROUND_ROBIN
    labels:
      app: istio-ingressgateway
      install.operator.istio.io/owning-resource: unknown
      install.operator.istio.io/owning-resource-namespace: istio-system
      istio: ingressgateway
      istio.io/rev: default
      operator.istio.io/component: IngressGateways
      operator.istio.io/managed: Reconcile
      operator.istio.io/version: 1.9.0
      release: istio
    name: istio-ingressgateway
    namespace: istio-system
  spec:
    clusterIP: 10.247.7.192
    externalTrafficPolicy: Cluster
    loadBalancerIP: 119.8.36.132     ## ELB EIP
    ports:
    - name: status-port
      nodePort: 32484
      port: 15021
      protocol: TCP
      targetPort: 15021
    - name: http2
      nodePort: 30294
      port: 80
      protocol: TCP
      targetPort: 8080
    - name: https
      nodePort: 31301
      port: 443
      protocol: TCP
      targetPort: 8443
    - name: tcp
      nodePort: 30229
      port: 31400
      protocol: TCP
      targetPort: 31400
    - name: tls
      nodePort: 32028
      port: 15443
      protocol: TCP
      targetPort: 15443
    selector:
      app: istio-ingressgateway
      istio: ingressgateway
    sessionAffinity: None
    type: LoadBalancer
  EOF
  
  ```

---

### Istio 使用的端口

Istio Sidecar 代理（Envoy）使用以下端口和协议。



为避免与 Sidecar 发生端口冲突，应用程序不应使用 Envoy 所使用的任何端口。

| 端口  | 协议 | 描述                                                  | 仅限 Pod 内部 |
| ----- | ---- | ----------------------------------------------------- | ------------- |
| 15000 | TCP  | Envoy 管理端口（命令/诊断）                           | 是            |
| 15001 | TCP  | Envoy 出站                                            | 否            |
| 15002 | TCP  | 故障检测侦听端口                                      | 是            |
| 15004 | HTTP | 调试端口                                              | 是            |
| 15006 | TCP  | Envoy 入站                                            | 否            |
| 15008 | H2   | HBONE mTLS 隧道端口                                   | 否            |
| 15020 | HTTP | 从 Istio 代理、Envoy 和应用程序合并的 Prometheus 遥测 | 否            |
| 15021 | HTTP | 健康检查                                              | 否            |
| 15053 | DNS  | DNS 端口，如果启用了捕获                              | 是            |
| 15090 | HTTP | Envoy Prometheus 遥测                                 | 否            |

Istio 控制平面（istiod）使用以下端口和协议。

| 端口  | 协议  | 描述                                        | 仅限本地主机 |
| ----- | ----- | ------------------------------------------- | ------------ |
| 443   | HTTPS | Webhook 服务端口                            | 否           |
| 8080  | HTTP  | 调试接口（已弃用，仅限容器端口）            | 否           |
| 15010 | GRPC  | XDS 和 CA 服务（纯文本，仅用于安全网络）    | 否           |
| 15012 | GRPC  | XDS 和 CA 服务（TLS 和 mTLS，推荐用于生产） | 否           |
| 15014 | HTTP  | 控制平面监控                                | 否           |
| 15017 | HTTPS | Webhook 容器端口，从 443 转发               | 否           |

---

```
istioctl install
```

![Image](https://github.com/user-attachments/assets/be11471f-e743-4ea1-badd-ad9ebcf9d7e9)


---

### 卸载 Istio

要从集群中完整卸载 Istio，运行下面命令：

```
$ istioctl uninstall --purge
```

可选的 `--purge` 参数将移除所有 Istio 资源，包括可能被其他 Istio 控制平面共享的、集群范围的资源。

或者，只移除指定的 Istio 控制平面，运行以下命令：

```
$ istioctl uninstall <your original installation options>
```

或

```
$ istioctl manifest generate <your original installation options> | kubectl delete --ignore-not-found=true -f -
```

控制平面的命名空间（例如：`istio-system`）默认不会被移除。 如果确认不再需要，用下面命令移除该命名空间：

```
$ kubectl delete namespace istio-system
```

---

### 体验流量镜像（或叫影子流量）

流量镜像，也称为影子流量，是一种以尽可能低的风险允许负责功能特性的团队改动生产环境的强大理念。 镜像会将实时流量的副本发送到镜像服务。镜像流量发生在主服务的关键请求路径之外。

> [!TIP]
>
> Kubernetes Gateway API CRD
>
> Gateway API 是 Kubernetes 官方推出的**下一代流量管理标准**，用来替代传统的 Ingress 资源。CRD（Custom Resource Definition）是它的自定义资源定义。
>
> Gateway API 可以类比ingress，不过ingress本质上是nginx，协议基本只有http；而Gateway API是GatewayClass → Gateway → Route，职责分离，支持HTTP、TCP、TLS、gRPC 等，原生支持流量分流、Header 匹配、镜像等，在不同厂商云服务统一标准，跨实现可移植。
>
> 也就是说在后续的k8s中，istio是作为标准组件的一环，api也不再是单独的istio api，而是更“标准”的gateway api。当然，这里仍然使用istio api演示～

部署 `httpbin-v1`：

```
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF
```

部署 `httpbin-v2`：

```
kubectl create -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v2
  template:
    metadata:
      labels:
        app: httpbin
        version: v2
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF
```

查看已经部署的资源：

```
kubectl get all -l app=httpbin


NAME                              READY     STATUS    RESTARTS   AGE
pod/httpbin-v1-84bf94b48-wtv2g    1/1       Running   0          4m1s
pod/httpbin-v2-585bf46f77-qdmj5   1/1       Running   0          3m16s

NAME              TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)    AGE
service/httpbin   ClusterIP   192.168.200.129   <none>        8000/TCP   2m33s

#Service 会把流量轮询分发到这两个 Pod 上。Istio 做版本路由的基础——先让多个版本的 Pod 都挂在同一个 Service 下。



NAME                                    DESIRED   CURRENT   READY     AGE
replicaset.apps/httpbin-v1-84bf94b48    1         1         1         4m1s
replicaset.apps/httpbin-v2-585bf46f77   1         1         1         3m16s

#ReplicaSet是 Deployment 自动管理的中间层。Deployment 做滚动更新时，会创建新的 ReplicaSet 启动新 Pod，同时缩减旧 ReplicaSet 的 Pod，以此实现平滑升级。
```

部署用于向 `httpbin` 服务发送请求的 `curl` 工作负载：

```
cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      labels:
        app: curl
    spec:
      containers:
      - name: curl
        image: curlimages/curl
        command: ["/bin/sleep","3650d"]
        imagePullPolicy: IfNotPresent
EOF
```

*为这个curl工作负载注入sidecar*

```
# 确保 namespace 开启了自动注入
kubectl label namespace default istio-injection=enabled --overwrite

# 重建 curl Pod（让 sidecar 注入）
kubectl delete pod curl-64b5f558b9-m9zdh
```

创建一个默认路由规则，将所有流量路由到服务的 `v1` 版本：

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
```

现在所有流量都指向 `httpbin:v1` 服务，并向此服务发送请求：

```
kubectl exec deploy/curl -c curl -- curl -sS http://httpbin:8000/headers

#如果版本较低，不支持deployment一键执行，可以用pod
kubectl exec <pod名> -c curl -- curl -sS http://httpbin:8000/headers
```

查看 `httpbin-v1` 和 `httpbin-v2` 这 2 个 Pod 的日志。 您应可以看到 `v1` 版本的访问日志条目，而 `v2` 版本没有日志：

```
kubectl logs deploy/httpbin-v1 -c httpbin


[2026-04-14 01:19:14 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2026-04-14 01:19:14 +0000] [1] [INFO] Listening at: http://0.0.0.0:80 (1)
[2026-04-14 01:19:14 +0000] [1] [INFO] Using worker: sync
[2026-04-14 01:19:14 +0000] [9] [INFO] Booting worker with pid: 9
172.16.1.77 - - [14/Apr/2026:02:25:45 +0000] "GET /headers HTTP/1.1" 200 775 "-" "curl/8.19.0"

*如果这里的ip为198.0.0.0/8网段，你很有可能没有注入sidecar*
```

```
kubectl logs deploy/httpbin-v2 -c httpbin

[2026-04-14 01:19:49 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2026-04-14 01:19:49 +0000] [1] [INFO] Listening at: http://0.0.0.0:80 (1)
[2026-04-14 01:19:49 +0000] [1] [INFO] Using worker: sync
[2026-04-14 01:19:49 +0000] [9] [INFO] Booting worker with pid: 9
```

修改路由规则，将流量镜像发送至 `httpbin-v2`：

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
    mirror:
      host: httpbin
      subset: v2
    mirrorPercentage:
      value: 100.0
EOF
```

发送流量：

```
kubectl exec deploy/curl -c curl -- curl -sS http://httpbin:8000/headers

#如果版本较低，不支持deployment一键执行，可以用pod
kubectl exec <pod名> -c curl -- curl -sS http://httpbin:8000/headers
```

日志验证：

```
[root@gz-hw-tianti-test-ngx-euler01 istio-1.29.1]# kubectl logs deploy/httpbin-v1 -c httpbin
172.16.1.77 - - [14/Apr/2026:02:25:45 +0000] "GET /headers HTTP/1.1" 200 775 "-" "curl/8.19.0"

[root@gz-hw-tianti-test-ngx-euler01 istio-1.29.1]# kubectl logs deploy/httpbin-v2 -c httpbin
172.16.1.77 - - [14/Apr/2026:02:25:45 +0000] "GET /headers HTTP/1.1" 200 808 "-" "curl/8.19.0"


时间对得上～
```



清理：

```
kubectl delete virtualservice httpbin
kubectl delete destinationrule httpbin

kubectl delete deploy httpbin-v1 httpbin-v2 curl
kubectl delete svc httpbin

```

---

### Bookinfo展示特性

![Image](https://github.com/user-attachments/assets/b32d9589-7d3d-443d-bafc-207dd7977139)
请移步下一章节，以下内容暂未跑通（Istio 控制器会**自动生成** Service 和 Deployment，控制不了 Service 的细节（比如 ipFamilyPolicy），导致单栈集群不能正常建立svc）：



下载yaml（Gateway api crd）

```
curl -Lo standard-install.yaml https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

放入/eddie

```
cd /eddie
kubectl apply -f standard-install.yaml



[root@gz-hw-tianti-test-ngx-euler01 eddie]# kubectl apply -f standard-install.yaml
customresourcedefinition.apiextensions.k8s.io/backendtlspolicies.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
```

至此就配置安装好了gateway api

```
cd /eddie/istio-1.29.1
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

*上面这条命令会启动 bookinfo 应用架构图中显示的全部四个服务。 也会启动三个版本的 reviews 服务：v1、v2 以及 v3。*
```

验证你的服务已经正常启动：

```
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

创建loadbalance，暴露服务到外部，我们要通过浏览器访问：

```
kubectl apply -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml
```

因为创建 Kubernetes `Gateway` 资源也会[[部署关联的代理服务](https://istio.io/latest/zh/docs/tasks/traffic-management/ingress/gateway-api/#automated-deployment)](https://istio.io/latest/zh/docs/tasks/traffic-management/ingress/gateway-api/#automated-deployment)， 所以运行以下命令等待网关就绪：

```
kubectl wait --for=condition=programmed gtw bookinfo-gateway
```

*因为*svc*中关于多栈（ipv4和v6）的配置是*

```yaml
ipFamilyPolicy: PreferDualStack
```

*虽然实际只分配了 IPv4，但华为云的 webhook 看到 `PreferDualStack` 就认为涉及 IPv6，共享型 ELB 直接拒绝，导致无法修改。*

修改istio环境变量

```
kubectl set env deployment/istiod -n istio-system PILOT_IPFAMILYPOLICY=SingleStack
```

重建网关

```
kubectl delete gateway bookinfo-gateway
kubectl apply -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml
```

---

### Bookinfo展示特性

注入sidecar

```
kubectl label namespace default istio-injection=enabled
```

部署应用

```
cd /eddie/istio-1.29.1
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

要确认 Bookinfo 应用正在运行，请从某个 Pod 中（例如从 `ratings` 中）用 `curl` 命令对此应用发送一条请求：

```
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

<title>Simple Bookstore App</title>
```

确认istio有 IngressGateway 组件

```bash
kubectl get svc -n istio-system istio-ingressgateway
kubectl get pods -n istio-system -l istio=ingressgateway
```

创建istio原生gateway

```
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

kubectl get gateway.networking.istio.io

export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

```

登陆华为云，进入集群，切换到istio-system命名空间，更新istio-ingressgateway的服务挂在共享型elb上。（这里因为tianti test的443被占用了，所以我暴露到了8443，另外8080的我暴露到了50001，其余没变。

```
export INGRESS_NAME=istio-ingressgateway
export INGRESS_NS=istio-system
kubectl get svc "$INGRESS_NAME" -n "$INGRESS_NS"
```

```
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

```
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
curl -s "http://${GATEWAY_URL}/productpage" | grep -o "<title>.*</title>"

*你就应该看到<title>Simple Bookstore App</title>*
```

访问http://192.168.100.24:50001/productpage

![Image](https://github.com/user-attachments/assets/97e56b81-8426-4c3f-8bf8-425c28bf1c62)
创建默认目标规则

```
kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml


[root@gz-hw-tianti-test-ngx-euler01 istio-1.29.1]# kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
destinationrule.networking.istio.io/productpage created
destinationrule.networking.istio.io/reviews created
destinationrule.networking.istio.io/ratings created
destinationrule.networking.istio.io/details created

*你可以用以下命令来查看当前规则*
kubectl get destinationrules -o yaml
```

---

### 配置请求路由

Istio 使用 Virtual Service 来定义路由规则。 运行以下命令以应用 Virtual Service， 在这种情况下，Virtual Service 将所有流量路由到每个微服务的 `v1` 版本。

```
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

使用以下命令显示已定义的路由：

```
kubectl get virtualservices -o yaml
```

![Image](https://github.com/user-attachments/assets/cf3d36e6-6f20-48d2-8ec2-99fcde3db965)
此时再看，就是没有星星了。

公网也可以访问：

![Image](https://github.com/user-attachments/assets/4a0381ab-f7eb-4127-9f89-b620b2eb2e8c)
运行以下命令以启用基于用户的路由：

```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

在 Bookinfo 应用程序的 `/productpage` 上，以用户 `jason` 身份登录。

![Image](https://github.com/user-attachments/assets/5c017cf2-65a8-4bc8-b179-3f78d89beaaa)
通过浏览器标头来分离流量：

```
cp samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml samples/bookinfo/networking/virtual-service-reviews-test-v2.1.yaml

vim samples/bookinfo/networking/virtual-service-reviews-test-v2.1.yaml
```



```
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - match:
    - headers:
        user-agent:
          exact: 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36'
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1

```

```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.1.yaml
```

![Image](https://github.com/user-attachments/assets/a97c2704-6978-496a-ac04-3000bb49f346)


----

### 金丝雀发布

华为云上ASM服务可以输入灰度版本号，输入流量控制策略就可以灰度、回滚、完全更新。

如果是自建，那么就是复制deployment，用subset的label（如matchLabels. version: v2)，进到istio进行流量管控。hpa最好，如果业务场景不适合hpa，就增加资源。由于erp涉及数据库操作，回滚很难，流量控制也不合适，所以在这里不再赘述erp行业使用istio的实践。

