### 虚拟化全家桶
一、内核层（唯一在内核）

KVMLinux 内核虚拟化模块，提供硬件级加速，所有高性能虚拟机都依赖它

二、用户态・虚拟机引擎（真正创建虚拟机的程序）

    QEMU
    通用全功能虚拟机模拟器，上层应用软件；模拟完整电脑硬件，搭配 KVM 跑常规虚拟机（Windows/Linux）。
    Firecracker
    AWS 开源轻量微型虚拟机引擎，同是上层应用；极致精简、启动极快、占用低，用于云原生 / Serverless 强隔离。

三、统一管理层（中间调度）

libvirt上层服务 / 工具，非内核模块；统一管理 QEMU、虚拟机生命周期（开机、快照、网络、存储），提供接口给上层平台调用。

四、上层操作 / 集群平台（人直接用的）

    virt-manager
    Linux 桌面图形小工具，基于 libvirt，单机少量虚拟机自用。
    PVE（Proxmox VE）
    轻量中小型集群虚拟化平台，Web 界面；底层 QEMU+libvirt，兼顾虚拟机 + LXC 容器，中小企业 / 自建机房常用。
    OpenStack
    大型企业级私有云平台，组件重、复杂；面向多租户、大规模算力调度，底层依赖 Nova+libvirt+QEMU。

最终层级链
物理硬件 → KVM (内核) → QEMU/Firecracker (虚拟机引擎) → libvirt (管理) → virt-manager / PVE / OpenStack (上层平台)

### 容器化全家桶
一、核心底层内核技术（容器的根基，Linux 自带）
1. Namespace 【隔离】
把系统资源切分成独立小空间

    隔离：PID、网络、挂载、用户、主机名

    效果：容器里看不到宿主机、看不到其他容器进程

2. Cgroup 【限制】
限制硬件资源上限

    限制：CPU、内存、磁盘 IO、网速
    
    效果：容器不能乱抢占宿主机资源，防止崩机

3. OverlayFS 【联合文件系统】
容器镜像分层只读、高效复用的核心

    下层：镜像只读层（共用，省空间）
    
    上层：容器可读写临时层
    
    优势：秒起容器、镜像体积小、多层复用

二、容器运行时栈（从上到下，标准层级）
1. runc 【最底层执行器】

    真正启动容器的最小工具

    直接调用内核 Namespace+Cgroup
    
    所有容器底层最终都靠 runc 跑起来

2. containerd 【容器运行时】

    行业标准容器 runtime（K8s 默认标配）

    职责：镜像管理、容器生命周期、调用 runc 启动容器

    定位：中间核心层，稳、轻、标准

3. Docker 【上层工具 / 客户端】

    早期大一统容器工具，包含：客户端 + 镜像 + 网络 + 存储

    底层：Docker 内部封装了 containerd + runc

    现状：K8s 弃用 Docker，改用原生 containerd

4. Podman 【Docker 替代品】

    无守护进程、可无 root 运行

    命令和 Docker 完全兼容，无缝替换

    安全更强，OpenShift / 企业环境常用

三、一条层级链（直接背，永久不乱）

内核：Namespace（隔离）+ Cgroup（限制）+ OverlayFS（分层存储）→ runc（底层启动容器）→ containerd（标准运行时，管容器 / 镜像）→ Docker / Podman（上层操作工具）

四、一句话极简总结

    Namespace：看不见别人（隔离）
    Cgroup：不能乱吃资源（限制）
    OverlayFS：镜像分层、省空间
    runc：底层真正启动容器
    containerd：K8s 标准容器运行时
    Docker：老牌一体化容器工具，底层依赖 containerd
    Podman：安全版、无守护进程的 Docker 替代工具
