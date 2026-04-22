# 第三层：核心资源对象 (四) 调度与资源管理 - 深度展开

## 1. 调度问题的本质

### 1.1 为什么调度是个难题？

```
问题拆解：
  - 一个集群有 N 个 Node，每个 Node 资源不同
  - 有 M 个待调度的 Pod，每个需求不同
  - 硬约束：必须满足（资源够、端口不冲突）
  - 软约束：最好满足（业务亲和、分散容灾）
  - 系统还在动态变化（Pod 在生老病死）

这是一个典型的"多约束优化问题"。

K8s Scheduler 的解法：
  两阶段（Filter + Score），可扩展（Framework）
```

### 1.2 本章的主角

```
调度相关资源：
  ──────────────────────────────────
  Node                集群计算资源
  Namespace           逻辑隔离边界
  ResourceQuota       命名空间总量限制
  LimitRange          单 Pod/Container 默认值与上限
  Taint / Toleration  Node 主动"排斥" Pod
  Affinity            Pod 主动"选择" Node 或其他 Pod
  PriorityClass       Pod 优先级与抢占
  PodDisruptionBudget 保证升级时的可用副本数
```

---

## 2. Node - 集群的"算力单元"

### 2.1 Node 的状态模型

```
每个 Node 对象记录：

status:
  addresses: [InternalIP, Hostname, ...]
  capacity:                ← 节点物理资源
    cpu: "8"
    memory: 32Gi
    pods: "110"
  allocatable:             ← 扣除系统预留后可用
    cpu: "7800m"
    memory: 30Gi
    pods: "110"
  conditions:
    - type: Ready          ← 节点是否可用
      status: "True"
    - type: MemoryPressure
    - type: DiskPressure
    - type: PIDPressure
    - type: NetworkUnavailable
```

### 2.2 Node 的注册与心跳

```
Node 加入集群：
  1. Kubelet 启动
  2. 自我注册到 API Server（或由管理员预先创建）
  3. 定期上报状态（心跳）

心跳机制（K8s 1.13+）：
  ────────────────────────
  1. Lease 对象：轻量心跳（每 10 秒）
     - 只更新一个小对象，减少 etcd 压力
  2. NodeStatus 更新：完整状态
     - 只在状态有变化时更新

节点失联：
  - 5 分钟没心跳 → Node 标记为 NotReady
  - 后续触发 Pod 驱逐（可通过 tolerations 调整）
```

### 2.3 Node 的资源计算

```
┌──────────────────────────────────────────────────────────┐
│  Node 32Gi 内存                                          │
│                                                          │
│  ┌────────────────┐   kubeReserved                        │
│  │  kubelet, RT   │     (1Gi)                            │
│  └────────────────┘                                      │
│                                                          │
│  ┌────────────────┐   systemReserved                     │
│  │  OS, sshd, ... │     (1Gi)                            │
│  └────────────────┘                                      │
│                                                          │
│  ┌────────────────┐   evictionThreshold                   │
│  │  eviction 区   │     (500Mi)                          │
│  └────────────────┘                                      │
│                                                          │
│  ┌────────────────┐   allocatable                        │
│  │  Pod 可用部分  │     (29.5Gi)                         │
│  └────────────────┘                                      │
└──────────────────────────────────────────────────────────┘

调度器只能用 allocatable 给 Pod 分配，不能超。
```

---

## 3. Namespace - 逻辑隔离

### 3.1 Namespace 的用途

```
Namespace 不是"安全边界"，而是"资源边界"：

  ┌─────────────────────────────────────────────┐
  │  物理上共用一个 K8s 集群                      │
  │                                             │
  │  ┌──────────────┐  ┌──────────────┐         │
  │  │ team-a       │  │ team-b       │         │
  │  │ 50 个 Pod    │  │ 100 个 Pod   │         │
  │  │ 自己的配额    │  │ 自己的配额    │         │
  │  │ 自己的 RBAC   │  │ 自己的 RBAC   │         │
  │  └──────────────┘  └──────────────┘         │
  └─────────────────────────────────────────────┘

能做：
  - 资源配额隔离（ResourceQuota）
  - 权限隔离（RBAC）
  - 网络隔离（NetworkPolicy）
  - 命名不冲突（不同 ns 可以有同名 Service）

不能做：
  - 硬隔离（同物理节点上 Pod 仍能互通，除非加 NetworkPolicy）
  - 资源完全独占（节点是共享的）
```

### 3.2 常见 Namespace 划分

```
按用途：
  kube-system      K8s 自身组件
  kube-public      对所有用户可见的数据
  kube-node-lease  Node Lease
  default          用户未指定时的默认 ns

按业务：
  prod / staging / dev
  team-a / team-b / ...
  app-foo / app-bar / ...
```

### 3.3 跨 Namespace 访问

```
Service DNS 全称：
  <service>.<namespace>.svc.cluster.local

同 ns：直接用 service-name
跨 ns：用 service-name.namespace
```

---

## 4. 资源模型：Requests & Limits

### 4.1 两个概念的根本区别

```
Requests = 调度依据（"我至少要这么多"）
  - Scheduler 用它判断 Node 能不能放下这个 Pod
  - 计入 Node 的"已分配"，即使实际没用那么多

Limits = 运行时上限（"我最多能用这么多"）
  - CPU 超过 → 被限流 (throttle)
  - 内存超过 → 被 OOM Killed

示例：
  requests: { cpu: 100m, memory: 128Mi }
  limits:   { cpu: 500m, memory: 512Mi }

  调度器：按 100m CPU / 128Mi 内存找 Node
  运行时：允许最多用 500m CPU / 512Mi 内存
```

### 4.2 CPU 的单位

```
CPU 是"可压缩资源"：超限就限流，不会死。

1 CPU = 1 个 vCPU / 1 核
1000m (millicore) = 1 CPU

  100m = 0.1 核
  500m = 0.5 核
  2    = 2 核
```

### 4.3 内存的单位

```
内存是"不可压缩资源"：超限就 OOM Kill，会死。

  Mi / Gi    二进制（1 Mi = 1024 * 1024 B）← 推荐
  M  / G     十进制（1 M = 1000 * 1000 B）

生产建议：一定要设 memory limits，防止内存泄漏打爆 Node。
```

### 4.4 QoS 分级

```
K8s 按 Requests/Limits 的设置把 Pod 分三个级别：

┌──────────────┬─────────────────────────────────┬───────────┐
│ QoS          │ 规则                            │ 驱逐顺序  │
├──────────────┼─────────────────────────────────┼───────────┤
│ Guaranteed   │ 所有容器 requests == limits     │ 最后驱逐  │
│              │ 且都设置了                      │           │
│ Burstable    │ 至少一个设了 requests/limits    │ 中间      │
│              │ 但不满足 Guaranteed             │           │
│ BestEffort   │ 没有任何 requests/limits        │ 最先驱逐  │
└──────────────┴─────────────────────────────────┴───────────┘

节点内存不够时：BestEffort → Burstable → Guaranteed 顺序驱逐。

生产建议：关键业务设成 Guaranteed（requests = limits）。
```

---

## 5. ResourceQuota - 命名空间配额

### 5.1 作用

```
限制一个 namespace 里所有资源的"总量"

防止一个团队/业务把整个集群资源吃光。
```

### 5.2 YAML 示例

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "100"         # 所有 Pod requests 总和上限
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"
    pods: "500"                 # Pod 数量上限
    persistentvolumeclaims: "50"
    services.loadbalancers: "2" # 最多创建 2 个 LoadBalancer Service
    count/deployments.apps: "20"
```

### 5.3 重要规则

```
⚠️ 一旦设置了 ResourceQuota，该 namespace 下的 Pod
   必须设置 resources.requests/limits，否则创建失败！

  解决方案：搭配 LimitRange 设默认值
```

---

## 6. LimitRange - 默认值 & 单对象上限

### 6.1 作用

```
ResourceQuota 限"总量"，LimitRange 限"单个"：

1. 给没设 resources 的 Pod 自动填默认值
2. 限制单个 Pod/Container 的 min/max
3. 限制 PVC 的大小范围
```

### 6.2 YAML 示例

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
    - type: Container
      default:                    # 默认 limits（没写就用这个）
        cpu: 500m
        memory: 512Mi
      defaultRequest:             # 默认 requests
        cpu: 100m
        memory: 128Mi
      max:                        # 单容器上限
        cpu: "4"
        memory: 8Gi
      min:                        # 单容器下限
        cpu: 10m
        memory: 16Mi
    - type: PersistentVolumeClaim
      max:
        storage: 100Gi
      min:
        storage: 1Gi
```

---

## 7. Taint & Toleration - Node 主动排斥

### 7.1 核心语义

```
Taint（污点）：Node 上打标签，"我不欢迎谁"
Toleration（容忍）：Pod 上声明，"我不在乎你的污点"

默认行为：Pod 不能调度到有 taint 的 Node
除非：Pod 声明了匹配的 toleration
```

### 7.2 Taint 的三种 effect

```
NoSchedule     新 Pod 不会被调度过来（旧的不动）
PreferNoSchedule   尽量不调度，硬找不到其他 Node 时也能来
NoExecute      新 Pod 不来，旧的也会被驱逐
```

### 7.3 用法示例

**给节点打 taint：**
```bash
kubectl taint nodes node-1 dedicated=gpu:NoSchedule
```

**Pod 声明 toleration：**
```yaml
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
```

### 7.4 常见用途

```
1. 专用节点
   ────────────
   GPU 节点打 dedicated=gpu:NoSchedule
   只有带对应 toleration 的 AI Pod 能上

2. 维护节点
   ────────────
   kubectl cordon + drain 的背后原理就是加 taint

3. 隔离异常节点
   ────────────
   Node 出问题 → Node Controller 自动加 taint → 驱逐 Pod

4. 主控节点防业务 Pod
   ────────────
   Master 节点天生有 taint：
     node-role.kubernetes.io/control-plane:NoSchedule
   所以普通 Pod 不会调度到 Master
```

---

## 8. Affinity / Anti-Affinity - Pod 主动选择

### 8.1 两种 Affinity

```
nodeAffinity
  ────────────
  Pod 要调度到什么样的 Node 上
  （nodeSelector 的升级版，支持复杂表达式）

podAffinity / podAntiAffinity
  ────────────
  Pod 要和谁做邻居/不做邻居
```

### 8.2 硬要求 vs 软偏好

```
requiredDuringSchedulingIgnoredDuringExecution
  硬要求：不满足不调度

preferredDuringSchedulingIgnoredDuringExecution
  软偏好：满足了加分，不满足也没关系

(IgnoredDuringExecution = 调度后条件变了不会迁移 Pod)
```

### 8.3 Node Affinity 示例

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["us-west-2a", "us-west-2b"]
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: node-type
                operator: In
                values: ["high-memory"]
```

### 8.4 Pod Anti-Affinity - 打散副本

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: web                    # 不要和 app=web 的 Pod 在一起
          topologyKey: kubernetes.io/hostname   # 按 Node 维度打散
```

```
实际效果：多副本 web Pod 分布到不同 Node。

常用 topologyKey：
  kubernetes.io/hostname                  按 Node 打散
  topology.kubernetes.io/zone             按可用区打散
  topology.kubernetes.io/region           按 region 打散
```

### 8.5 Topology Spread Constraints（更推荐）

```
podAntiAffinity 的问题：
  - 语义是"不要一起"，但无法表达"均匀分布"
  - 大规模下性能不佳

Topology Spread Constraints（K8s 1.19+）：
  - 直接表达"在某个拓扑维度下均匀分布"
```

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1                            # 最大不均度
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule      # 不满足时不调度
      labelSelector:
        matchLabels:
          app: web
```

```
效果：
  3 个 web Pod 分布在 3 个 AZ 各一个
  而不是挤在同一个 AZ
```

---

## 9. PriorityClass - 优先级与抢占

### 9.1 使用场景

```
生产集群资源紧张时：
  - 关键业务 Pod 应该优先调度
  - 实在没资源时，可以"抢占"低优先级 Pod 的位置

定义优先级：
```

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "for critical services"
```

```yaml
spec:
  priorityClassName: high-priority
  containers: ...
```

### 9.2 抢占机制

```
新 Pod P 调度失败 → Scheduler 查找：
  "驱逐哪些低优先级 Pod 后，P 能被调度？"
  ├─ 找到候选 Node
  ├─ 挑选要驱逐的 Pod（必须优先级低于 P）
  └─ 发送驱逐请求
      被驱逐 Pod 会按 PodDisruptionBudget 处理
      驱逐后 P 调度上去

可选禁用：
  preemptionPolicy: Never
  表示该 Pod 只调度自己不抢占
```

### 9.3 系统保留优先级

```
K8s 预定义：
  system-cluster-critical   (2000000000)
  system-node-critical      (2000001000)

这些留给 DaemonSet、kube-system 组件使用，
业务 Pod 不要用。
```

---

## 10. PodDisruptionBudget - 可用性保障

### 10.1 解决什么问题？

```
场景：管理员 drain 一个 Node（维护/升级）
      节点上的 Pod 被驱逐
      如果一个服务所有副本碰巧都在这个 Node 上 → 全挂

PDB 的作用：
  "无论发生什么 voluntary disruption，
   这个服务至少要有 N 个可用副本"
```

### 10.2 Voluntary vs Involuntary

```
Voluntary Disruption（PDB 能管）：
  - kubectl drain
  - 节点升级
  - 集群缩容

Involuntary Disruption（PDB 管不了）：
  - 硬件故障
  - 内核 panic
  - OOM Kill
```

### 10.3 YAML 示例

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2        # 或 maxUnavailable: 1
  selector:
    matchLabels:
      app: web
```

```
效果：任何时刻 drain Node 时，web Pod 可用数不能低于 2。
      如果 drain 会破坏约束，drain 会一直等。
```

---

## 11. 调度的全貌流程

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  1. Pod 创建，nodeName 为空                                   │
│                                                              │
│  2. Scheduler Watch 到未调度 Pod                              │
│                                                              │
│  3. Filter 阶段：排除不满足的 Node                             │
│     - PodFitsResources (资源够吗？)                           │
│     - PodMatchNodeSelector                                   │
│     - NoVolumeZoneConflict                                   │
│     - PodToleratesNodeTaints                                 │
│     - CheckNodeAffinity                                      │
│     - CheckPodAffinity                                       │
│     ... 十几个 Filter Plugin                                  │
│                                                              │
│  4. Score 阶段：给剩余 Node 打分                               │
│     - LeastRequestedPriority   (资源剩余多的优先)              │
│     - BalancedResourceAllocation (CPU/Memory 均衡优先)         │
│     - ImageLocalityPriority    (镜像已在的优先)                │
│     - PodAffinityPriority                                    │
│     - NodeAffinityPriority                                   │
│     - TopologySpreadPriority                                 │
│     ... 加权求和                                              │
│                                                              │
│  5. 选最高分的 Node，Bind：更新 pod.nodeName                   │
│                                                              │
│  6. 如果 Filter 阶段找不到任何 Node：                          │
│     → 尝试抢占（Preemption）                                  │
│     → 还不行 → Pod Pending，等下次调度                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 12. 总结：调度的设计哲学

```
维度                       工具                       典型场景
──────────────────────────────────────────────────────────────
资源供给                   Node, kubelet              集群扩缩容
资源边界                   Namespace                  多租户隔离
资源配额                   ResourceQuota              防团队吃光集群
单对象限制                 LimitRange                 默认值 + 上限
Node 主动排斥              Taint/Toleration           专用节点、维护
Pod 主动选择               Affinity, NodeSelector     业务亲和、GPU 调度
均匀分布                   TopologySpreadConstraints  多 AZ 容灾
优先级保障                 PriorityClass              关键业务抢占
可用性保障                 PodDisruptionBudget        无中断维护
```

> **核心洞见：K8s 调度不是简单的"装箱问题"，而是一个"约束满足 + 多目标优化"问题。Requests/Limits 定义硬约束，Affinity/Taint 表达业务规则，PriorityClass/PDB 保障 SLO。理解调度器就是理解 K8s 如何在"资源利用率"和"应用稳定性"之间做权衡。**
