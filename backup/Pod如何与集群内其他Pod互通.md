根据 Kubernetes 的网络模型要求：所有 Pod 都可以与任何其他 Pod 通信，且无需使用 NAT（网络地址转换）。

宿主机上的 `kubelet` 调用 CNI 插件，CNI插件从 IPAM（IP地址管理）连接池分配 IP 并配置给容器驱动，CNI 将分配好的 IP 返回给 `kubelet` ， `kubelet` 异步更新到 API Server，最终写入 Etcd。

查询Pod的IP：

```
kubectl get pods -o wide
```

Endpoints（或 EndpointSlice） 是 Kubernetes 里的一个独立资源对象，它记录的是 Service（服务）后端对应的 Pod 容器 IP + 端口。如果容器绑定了SVC，还可以使用

```
kubectl get endpoints
```



Pod之间相互通信是CNI实现的。通常有以下几种类型：

1、Overlay（覆盖网络）

通过在现有的物理/虚拟网络（Underlay）之上构建一层虚拟的逻辑网络来实现 Pod 间互通。当 Pod A 发送数据给 Pod B 时，CNI 会将原始的 Pod IP 报文进行二次封装（例如封装在 UDP、VXLAN 或 Geneve 报文里），外层使用宿主机的 IP 作为源和目的地址。数据到达目的宿主机后，再由 CNI 进行解封装，将原始报文投递给 Pod B。VXLAN（最常用）：将二层以太网帧封装在三层 UDP 报文中（端口一般为 4789）。

所以node之间的4789端口需要互通，且因为有封包和解包的过程，会带来一定的 CPU 开销和网络延迟，且由于增加了报头，实际的 MTU 变小了。

2、Underlay（路由/直通模式）

与 Overlay 相反，Underlay 模式不进行任何报文封装，而是直接利用底层物理网络设备（如交换机、路由器）来传输 Pod 的流量。Pod 的 IP 在整个物理网络中是直接可见且可路由的。CNI 插件（如 Calico 的 BGP 模式）将每个宿主机变成一个路由器。宿主机会向周围的物理交换机或其它节点宣告自己持有的 Pod 网段。当流量跨节点传输时，直接通过标准的 IP 路由转发到达目的宿主机。

通常云厂商原生CNI都选用这种方式，如 AWS-CNI、阿里云-Terway、华为云-CCE 的 VPC 路由模式。比如华为云的容器网段就是在大的VPC中划分的子网。举例： 容器网段是10.16.0.0/16，PodIP为10.16.2.33，它与其他Pod联通走的就是VPC网。

3、Host-Gateway 模式（直连路由）

一种介于 Overlay 和完全 Underlay 之间的折中方案（如 Flannel 的 host-gw 模式，或 Calico 的 DirectRouting）。它不使用 BGP 这种复杂的路由协议，也不封包。它直接在每个宿主机的操作系统路由表里，添加其它节点 Pod 网段的静态路由，并将下一跳（Next Hop）指向对应节点的宿主机 IP。

因为只存了下一跳的宿主机ip，所以必须要求所有宿主机在同一个二层网络（同一个交换机/同一个 VLAN）下。如果宿主机跨了三层网段（比如中间隔了路由器），这种静态下一跳的路由就会失效。

4、基于 eBPF 的高级路由模式（如 Cilium）

得益于Linux 内核技术的发展， eBPF 网络插件既可以跑在 Overlay 模式下，也可以跑在 Native Routing（类似 Underlay）模式下。它的核心特性在于绕过了 Linux 传统的 TCP/IP 内核协议栈和 iptables/ipvs。通过直接在内核网卡驱动的挂载点（XDP/TC）运行 eBPF 字节码，在最底层直接将报文重定向到目的 Pod 的虚拟网卡（veth）。即使在 Overlay 模式下，性能也极高，极大降低了内核上下文切换的开销。天然支持高效的 Service 负载均衡和 NetworkPolicy（网络策略）细粒度控制。