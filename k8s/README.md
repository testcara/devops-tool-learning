1. 修改 Deployment 副本数 1→2，发生了什么？

    用户通过 kubectl、Helm 执行更新操作，向 kube-apiserver 发送变更请求。
    apiserver 完成权限和语法校验后，将副本数为 2 的期望状态持久化到 etcd。
    Deployment Controller 持续监听集群资源，发现集群当前实际副本数与 etcd 期望状态不一致，触发调谐，通过 ReplicaSet 生成新的 Pod 对象。
    K8s Scheduler 为待调度 Pod 筛选匹配合适的 Node 节点并完成绑定。
    目标节点上的 kubelet 负责创建、启动 Pod，等待 Pod 就绪。
    最终集群实际状态对齐 etcd 期望状态，调谐完成。

2. 外网访问 Service-Pod 完整流程（精准落地版）

    集群节点分为 Master 节点与 Worker 节点。Master 节点仅负责集群管理、状态调度，不承接业务流量；外网用户请求只会进入 Worker 节点。
    Ingress 控制器并非集群节点自带策略，是独立部署在 Worker 节点上的七层反向代理 Pod（底层基于 Nginx/Traefik）。
    外网流量先抵达集群 Worker 节点，再交由 Ingress 控制器做七层域名、路径匹配，转发至对应 Service，最终通过节点内核规则分发至后端业务 Pod。

    详细的过程为：
    外网用户发起请求，DNS 解析域名，指向集群节点公网 IP
    流量打到集群 Worker 节点（不会走 Master 节点，master 不承接业务流量）
    节点将流量交给部署在当前节点的 Ingress 控制器 Pod（本质是独立的 Nginx 服务，七层应用）
    Ingress 控制器实时监听集群内的 Ingress 规则资源，根据域名、URL 路径匹配对应的后端 ClusterIP Service
    节点内核的 kube-proxy 通过 iptables/IPVS 规则，匹配 Service 关联的健康 Pod，完成四层负载均衡转发
    Pod 内业务容器处理请求，请求原路返回，完成一次外网访问

3. kubectl /kube-apiserver/kubeadm 关系

    kubeadm：集群搭建工具，只负责初始化集群，搭建后不参与运行
    kube-apiserver：集群唯一入口，所有增删改查请求必经，校验权限、读写 etcd
    kubectl：客户端工具，封装 http 请求，帮用户和 apiserver 交互

    串联：kubeadm 搭集群 → kubectl 发命令 → apiserver 存 etcd → 控制器落地资源
4. K8s / K3s / K9s 区别

    K8s：标准生产级容器编排，解决容器无法集群调度、自愈、滚动更新、高可用问题，用于生产微服务
    K3s：轻量版 K8s，资源占用极低，去掉冗余组件，用于本地测试、边缘部署
    K9s：可视化终端运维工具，替代手写 kubectl 命令，提升排查效率
5. 