containerd为了保障系统内核不被随意篡改，对容器的脚本执行、挂载、有严格的要求，但是有些环境需要拥有高权限。
所以containerd提供了 --privileged 模式运行的容器。普通容器默认通过Linux内核的多种安全机制被隔离和限制，而特权容器则解除了这些限制，几乎拥有与宿主机root用户相同的权限。

特权容器可以：
授予所有Linux capabilities：普通容器默认只有一个有限的capabilities子集（如 CAP_NET_BIND_SERVICE），特权容器拥有全部capabilities（如 CAP_SYS_ADMIN、CAP_SYS_MODULE、CAP_NET_ADMIN 等）。
解除seccomp/AppArmor/SELinux限制：默认的系统调用过滤和 LSM 策略被禁用。
允许访问宿主机所有设备：/dev下的设备节点全部可见，cgroup 设备白名单不再生效。
允许挂载文件系统：可以执行mount、pivot_root等操作。
可以操作内核模块、修改内核参数（如sysctl）。

为什么Agent Sandbox需要特权容器？
Agent sandbox（比如让 LLM Agent 执行任意代码的沙箱环境）通常需要在容器里再做一层隔离或执行复杂操作，这就经常碰到普通容器权限不够的场景：
1. 容器内运行 Docker / 容器（Docker-in-Docker, DinD）
Agent 可能需要自己拉镜像、起容器、跑测试。Docker daemon 启动时需要操作 cgroup、网络命名空间、overlay 文件系统、iptables 等，这些都需要 CAP_SYS_ADMIN 和设备访问权限。
2. 嵌套虚拟化或更强隔离
比如在 sandbox 内跑 KVM、Firecracker、gVisor、Kata Containers，需要访问 /dev/kvm、修改网络栈、挂载文件系统。
3. 自定义网络/防火墙
Agent 可能要配置 iptables 规则限制出网（避免泄漏、防止外联恶意服务），这需要 CAP_NET_ADMIN。
4. 文件系统操作（最重要）
挂载 overlayfs 做写时复制快照，让每个 task 有独立可回滚的文件系统视图——需要挂载权限。
5. 系统调用追踪 / 调试
ptrace、bpf、seccomp 规则等，用来观察 Agent 跑的代码做了什么。
6. cgroup 资源限制
精细控制 Agent 子进程的 CPU、内存、PID 数等，需要写 cgroup 文件系统。

所以我们的Agent产品在没有配置特权容器时，不能够生成PPT。1、没有挂盘2、没有操作文件系统的高级管理权限

如何配置？
spec:
  containers:
  - securityContext:
      privileged: true