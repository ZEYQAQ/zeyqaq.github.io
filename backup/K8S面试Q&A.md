1、

Q：

如果此时你发送一个命令“运行一个 Web 应用”，这个指令进入工厂后，你认为这三个组件分别应该先做什么，后做什么呢？

A：

首先是apiserver收到了kubectl apply的请求，将此次数据分析为具体指令并将记录存入etcd；scheduler和controllermanager会不断查询etcd，一旦查询到数据，scheduler会检查哪些node存在资源以供调度，然后kubelet会将其镜像拉取后按照yaml启动，controllermanager会不断检查pod状态，从pending直到running

2、

Q：

你了解 Deployment 和 ReplicaSet 的区别吗？在实际生产中，为什么我们几乎总是直接使用 Deployment，而不是手动去创建 ReplicaSet？

A：

deployment因为有滚动部署，在更新时可以理解为多个replicaset。而replicaset只是简单的维护统一版本镜像的副本数量，缺少则补充。

3、

Q：

在执行滚动更新时，如果新版本的镜像有问题（比如镜像拉取失败或程序启动后一直报错），Deployment 是如何处理的？集群会一直处于一个“半更新”的状态吗？

A：

在所有更新中，deployment都会新启一个replicaset，保留原有Pod。deployment首先会经历pending状态，kubelet会尝试从scheduler维护的etcd数据中判断哪个node存在资源，然后尝试调度镜像到node。镜像拉取如果存在问题会报错ImagePullBackOff，此时controlermanager会检测到ImagePullBackOff状态，然后显示出来。如果是程序启动有问题则报错CrashLoopBackOff，Kubelet 发现进程挂了，就会根据 RestartPolicy（默认是 Always）尝试重启。但是k8s集群为了避免无限循环造成的资源浪费，一般会在600秒（progressDeadlineSeconds）后标记为Failed。集群会一直处于更新失败再更新的循环，直到问题解决后清除原有的pod。

4、

Q：

如果我们正在执行滚动更新，为了防止新版本发布导致大面积故障，我们通常会配置 `maxUnavailable` 和 `maxSurge` 这两个参数。你知道这两个参数分别代表什么意义，以及如果把 `maxSurge` 设置为 0 会发生什么吗？

A：

maxUnavailable (最大不可用副本数)定义在滚动更新过程中，同一时刻允许有多少个 Pod 处于不可用状态。它可以是具体数字，也可以是百分比（例如 25%）。maxSurge(最大激增副本数)定义在滚动更新过程中，除了预期的副本数之外，还可以额外创建多少个新 Pod。如果我们将 maxSurge 设置为 0，当滚动更新的时候，则会先删除已有的一个pod，然后补充一个新版本的pod。一般在资源极度受限的情况下使用，但是这样会让稳定性大打折扣。

5、

Q：

你知道 K8s 中的 Liveness Probe 和 Readiness Probe 的区别吗？如果一个容器的 `Liveness Probe` 频繁报错，K8s 会采取什么动作？

A；

Liveness Probe存活探针，会通过检查URL到状态码或者通过执行指定脚本来确定pod内应用是否存活。readiness是就绪探针，和存活探针类似，也是检查状态码，不过这个探针的用途是来确认新容器是否已经可以接受请求。如果三次（一般）存活探针都没有通过，超时10s（一般）就会杀死容器，在原有pod内新启容器。 

6、

Q：

假设你有一个业务系统，在启动时需要加载大量的缓存数据，耗时可能长达 2 分钟。如果直接配置 `Liveness Probe`，很容易在启动期间就因为超时而导致容器被反复重启。你会如何利用 `initialDelaySeconds` 或者其他方案来解决这个问题，以保证 Pod 能正常启动并对外提供服务？

A：

存活探针（livenessProbe）和就绪探针（readinessProbe）都在容器启动后，各自等待自己的 `initialDelaySeconds` 结束后开始检查，互不干扰。initialDelaySeconds是各种探针的一个参数，代表在容器启动到第一次检测的等待的时间间隔。

7、

Q：

你了解 Kubernetes 的 Service 吗？为什么有了 Pod IP，我们还需要 Service？如果 Pod 被删除重建导致 IP 变了，Service 是如何保证流量依然能准确转发到新 Pod 的？

A：

pod重建后ip会变化，svc会建立一个虚拟ip（Virtual IP），SVC会维护这个虚拟ip对应的后面的podip。访问 Service 的 IP 时，请求会被转发到后端某一个正在运行的 Pod 的 IP 上。

其中CoreDNS负责解析SVC的名字为虚拟IP、网络插件（CNI）负责Pod 的实际 IP 分配和网桥搭建。

8、

Q：在 Kubernetes 中，有一个名为 Endpoints（或者对于较新版本是 EndpointSlice）的资源对象。你认为当 Controller Manager 检测到旧 Pod 被销毁、新 Pod 启动并就绪时，它会通过什么方式去更新这个 Endpoints 对象，从而让 Service 能准确感知到新 Pod 的存在呢？

A：

Endpoint Controller（是Controller Manager的一部份）会时刻盯着API Server（使用 List-Watch 机制）

（为什么不是读etcd？因为**etcd (存储层)** <---> **API Server (逻辑层/权限/缓存)** <---> **各个控制器 (如 Endpoint Controller)**，etcd的权限比较高，不是谁都能读写，根据RBAC权限控制，需要通过API Server查完后提供给Controller）

查看是不是有新的deployment，如有，则Controller去匹配带这个SVC标签的所有的Ready的Pod。加入到Endpoint对象中。而每个node上的kube-proxy会实时监控这个endpoint对象，一旦列表更新，就会更新iptables或者IPVS，确保新流量到新的Pod。

9、

Q：

ControllerManager有什么组件？

A：有数十个，举例

**Node Controller**：监控节点健康状态。

**ReplicaSet Controller**：确保副本数量达标。

**Deployment Controller**：处理滚动更新和版本回滚。

**Endpoint Controller**：专门负责监控 Service 和 Pod 的映射关系。

**ServiceAccount Controller**：负责自动创建默认的服务账号等。

10、

Q：iptables和IPVS的区别？

A：

iptables是线性的，每个流量进来都要从头到尾根据规则链一条一条匹配，时间负责度为O(n)，而IPVS是根据哈希表，利用Linux内核专门用于负载均衡的模块来存储转发规则，时间复杂度接近O(1)。

11、

Q：你刚才提到通过 `kubectl` 操作资源，如果某天 API Server 挂了，你觉得已经在运行的 Pod 会受到影响吗？ 或者说，如果你此时手动删掉一个节点上的 kube-proxy，会有什么后果？

A；

短时间没什么影响？

API Server负责新增删去扩容更新配置，除此之外的node层面上的业务一切正常。

因为kubeproxy只是负责根据endpoint配置Linux转发规则，并不实际操作流量的转发，真正负责转发的是iptables、IPVS。kubeproxy负责转发规则的改写，这也代表，不能再新增、删除任何Pod，因为新的Pod IP的已经无法维护到SVC。

12、

Q：

我们一直在说 API Server 存储数据。如果我要查看一个 Pod 的详细信息，发现它的 `Status` 部分显示 `ContainerCreating`，这通常意味着什么组件在此时最忙碌？或者说，这个状态是因为哪个环节卡住了？ （提示：想想是谁负责从镜像仓库拉取镜像并启动容器的？）

A：

kubelet从镜像仓库拉取镜像，调用Runtime创建容器，调用CNI分配Pod IP并配置网络，挂在PV、CM、Secret。kubelet在轮询查看Pod状态（为什么不是controllermanager？因为一旦它把这个新 Pod 创建出来并分配给一个节点（Binding），它的核心工作就完成了。 它并不直接管这个 Pod 里的容器镜像有没有拉下来，也不管容器有没有启动。直到最后，Controller Manager 再次“轮询”API Server，看到新的工作负载状态变了，它就确信目标已达成，从而停止干预。--------------Controller Manager (如 ReplicaSet Controller) 负责的是集群层面的‘最终一致性’，确保逻辑上的副本数符合预期；而 Kubelet 负责的是节点层面的‘运行时一致性’，处理具体容器的拉取、启动和健康状态。Controller Manager 并不直接管理容器的启动细节，它通过 API Server 接收 Kubelet 上报的状态，从而实现全链路的闭环。”）

13、
Q：

污点与容忍、亲和与反亲和

A：

Taint（污点）：给节点打上标记，表示“我不欢迎某些 Pod”。

*例子*：`key=dedicated, value=special, effect=NoSchedule` (拒绝没有特殊容忍度的 Pod 进入)。

Toleration（容忍）：给 Pod 打上标记，表示“虽然这个节点有污点，但我可以忍受，请让我进去”。



Node Affinity（节点亲和性）：比如“我想去有 `disktype=ssd` 标签的机器”。

Pod Affinity（Pod 亲和性/反亲和性）：这是个高级玩法，比如：

- *亲和性*：要求这两个 Pod 必须部署在同一台机器上（为了低延迟）。
- *反亲和性*（podAntiAffinity）：要求这两个 Pod 绝对不要部署在同一台机器上（为了高可用，防止一台机器坏了导致整个服务挂掉）。

14、

Q：

如果业务必须要在 2 个节点上部署 3 个副本怎么办？这三个副本还不能在同一台node上。

A：

将 `requiredDuringSchedulingIgnoredDuringExecution` 改为 `preferredDuringSchedulingIgnoredDuringExecution`（软约束）。

调度器会“尽量”满足反亲和性，但如果实在没地方放了，它会退而求其次，将第三个副本挤在已有的节点上，而不是让 Pod 一直 Pending。

//拓扑感知（Topology Spread Constraints）：比亲和更加优秀，可以实现这种。允许定义“分布偏差（maxSkew）”允许一定程度的不均匀，但尽量保持平衡。

15、

Q：

如何确认一个新服务的Requests和Limits？

A：

i）建设监控系统，将数据采集到Prometheus，`container_memory_working_set_bytes`（内存工作集）和 `container_cpu_usage_seconds_total`（CPU 平均使用率）查看所需内容，观察一个业务周期。

ii）使用VPA，Vertical Pod Autoscaler垂直扩缩容，VPA 可以设置为 `Recommender` 模式，它不直接调整资源，而是通过分析历史数据，告诉你：“你设置了 1GB 内存，但实际运行中只需要 400MB”。

iii）压力测试

在生产环境中，最完美的资源配置策略是 “Requests = Limits”

