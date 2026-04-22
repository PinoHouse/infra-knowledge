# 第三层：核心资源对象 (一) 工作负载层 - 深度展开

## 1. 为什么需要多种工作负载资源？

### 1.1 应用的本质分类

```
从"应用特征"推导出资源类型：

┌─────────────────────────────────────────────────────────────┐
│  问题：我的应用长什么样？                                    │
│                                                             │
│    无状态、可水平扩展?      →  Deployment                   │
│    有状态、需要稳定身份?    →  StatefulSet                  │
│    每个节点跑一个?          →  DaemonSet                    │
│    跑一次就结束?            →  Job                          │
│    定时跑?                  →  CronJob                      │
│                                                             │
│  K8s 不是"一种资源走天下"，而是为不同场景提供专用抽象         │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 资源对象的层级关系

```
┌──────────────────────────────────────────────────────────────┐
│                   抽象层级（高 → 低）                         │
│                                                              │
│    Deployment / StatefulSet / DaemonSet / Job                │
│            │  (定义应用部署策略)                              │
│            ▼                                                 │
│        ReplicaSet                                            │
│            │  (保证 N 个副本)                                │
│            ▼                                                 │
│           Pod                                                │
│            │  (最小调度单元)                                 │
│            ▼                                                 │
│        Container(s)                                          │
│            │  (真正的进程)                                   │
│            ▼                                                 │
│         进程 + 命名空间 + cgroup                             │
└──────────────────────────────────────────────────────────────┘

上层资源是下层资源的"管理者"，下层资源是上层资源的"执行者"。
```

---

## 2. Pod - 最小调度单元

### 2.1 为什么不是容器，而是 Pod？

```
核心洞见：有些容器天然需要"绑在一起"

场景：主应用 + 日志收集 sidecar
┌─────────────────────────────────────────────────┐
│                     Pod                         │
│                                                 │
│   ┌──────────┐           ┌───────────────┐     │
│   │  App     │  写日志    │  Fluentd      │     │
│   │ Container│──────────▶│  Sidecar      │     │
│   │          │  共享卷    │  (收集 → ES)  │     │
│   └──────────┘           └───────────────┘     │
│                                                 │
│   共享：                                         │
│     - Network (同一个 IP、localhost 互通)        │
│     - Volume (共享挂载)                          │
│     - IPC (进程间通信)                           │
│                                                 │
│   不共享：                                       │
│     - PID (默认各自独立)                         │
│     - 文件系统 (容器镜像各自隔离)                 │
└─────────────────────────────────────────────────┘

如果 K8s 以"容器"为最小单位，这种紧耦合关系就没法表达。
```

### 2.2 Pod 的本质：一组共享 Linux Namespace 的容器

```
Pod 启动过程：

Step 1: 创建一个 "pause" 容器
  ──────────────────────────
  - 持有 network namespace、IPC namespace
  - 作为"占位符"，其他容器加入它的 namespace

Step 2: 启动业务容器，加入 pause 的 namespace
  ──────────────────────────
  - 共享网络栈（同 IP、同端口空间）
  - 共享 IPC（可用 SHM 通信）

┌─────────────────────────────────────────────┐
│  Pod                                        │
│  ┌────────┐  ┌────────┐  ┌────────┐         │
│  │ pause  │  │  app   │  │sidecar │         │
│  │ (根)   │←─│        │←─│        │         │
│  └────────┘  └────────┘  └────────┘         │
│       共享 netns / ipcns / (可选 pidns)      │
└─────────────────────────────────────────────┘
```

### 2.3 Pod 的使用方式

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web         # Label 是 Service/Controller 的选择依据
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:      # 调度依据（最少需要多少）
          cpu: 100m
          memory: 128Mi
        limits:        # 硬上限（超出会被限流/OOM Kill）
          cpu: 500m
          memory: 512Mi
```

### 2.4 重要：Pod 是"一次性"的

```
┌──────────────────────────────────────────────────────┐
│  关键心态：Pod 是 Cattle，不是 Pet                    │
│                                                      │
│   - Pod 挂了不会"复活"，而是"重建"（新 Pod，新 IP）   │
│   - 不要直接用 Pod，要用 Controller 来管理 Pod        │
│   - 数据放在 Pod 里 = 数据会丢                        │
└──────────────────────────────────────────────────────┘
```

---

## 3. ReplicaSet - 副本保证

### 3.1 定位

```
ReplicaSet 的唯一职责：保证有 N 个匹配标签的 Pod 在运行

  期望 replicas: 3
       │
       ├─ 实际 0 个？ → 创建 3 个
       ├─ 实际 2 个？ → 创建 1 个
       ├─ 实际 5 个？ → 删除 2 个
       └─ 实际 3 个？ → 什么都不做
```

### 3.2 ReplicaSet 是如何找到它管理的 Pod 的？

```
答案：通过 Label Selector（不是通过"创建关系"）

apiVersion: apps/v1
kind: ReplicaSet
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx          # ← 所有 label app=nginx 的 Pod 都归我管
  template:               # ← Pod 模板（用于创建新 Pod）
    metadata:
      labels:
        app: nginx        # ← 新 Pod 会带这个 label
    spec:
      containers: ...
```

```
⚠️ 注意：如果你手动创建一个 label=nginx 的 Pod，
        ReplicaSet 会把它当成"自己的"，甚至可能删掉它以满足副本数。
```

### 3.3 一般不直接用 ReplicaSet

```
ReplicaSet 的不足：
  - 只能保证副本数，不支持"更新镜像后滚动替换"
  - 一次性、无版本管理

因此实际中 99% 都用 Deployment，由 Deployment 来创建和管理 ReplicaSet。
```

---

## 4. Deployment - 无状态应用的标准部署方式

### 4.1 Deployment 的增值：版本管理 + 滚动更新

```
Deployment vs ReplicaSet：

Deployment = ReplicaSet + 更新策略 + 历史版本

┌──────────────────────────────────────────────────────────┐
│  Deployment: web                                         │
│                                                          │
│   ┌─────────────────┐    ┌─────────────────┐            │
│   │ ReplicaSet: v1  │    │ ReplicaSet: v2  │            │
│   │ replicas: 0     │    │ replicas: 3     │  ← 当前版本 │
│   │ (保留用于回滚)  │    │                 │            │
│   └─────────────────┘    └─────────────────┘            │
│                                                          │
│   更新镜像时：                                            │
│     1. 创建新 RS (v2)，逐步扩容                           │
│     2. 旧 RS (v1) 逐步缩容                                │
│     3. 直到 v2 达到期望副本，v1 为 0                      │
└──────────────────────────────────────────────────────────┘
```

### 4.2 滚动更新策略

```yaml
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%          # 更新期间最多超出期望副本多少
      maxUnavailable: 25%    # 更新期间最多允许多少不可用
```

```
滚动过程示例 (replicas=10, maxSurge=25%, maxUnavailable=25%)：

时刻  v1(旧)  v2(新)  总数    状态
────  ──────  ──────  ────    ─────
 t0    10      0      10      正常
 t1    10      2      12      v2 先起来（surge 2）
 t2    8       2      10      v1 下掉 2
 t3    8       4      12
 t4    6       4      10
 ...    (循环：新增 → 删除)
 tN    0       10     10      完成
```

### 4.3 更新触发条件

```
只有 spec.template 变化才会触发滚动更新：
  - 镜像变了 ✓
  - 环境变量变了 ✓
  - resources 变了 ✓
  - replicas 变了 ✗ (只是扩缩容，不发版)

⚠️ 改 ConfigMap 不会触发 Deployment 更新！
   需要配合 checksum annotation 或重启 Pod。
```

### 4.4 回滚

```bash
kubectl rollout history deployment/web
kubectl rollout undo deployment/web --to-revision=2
```

```
回滚的本质：把历史 ReplicaSet 的 replicas 重新扩上去。
```

---

## 5. StatefulSet - 有状态应用

### 5.1 "有状态"意味着什么？

```
无状态应用（Deployment）：
  - Pod 之间完全等价
  - 随便杀、随便扩，谁都一样

有状态应用（StatefulSet）：
  - Pod 之间有身份区别（db-0 是主，db-1 是从）
  - 数据绑定到特定 Pod（db-0 的数据只能 db-0 用）
  - 启动/缩容有顺序（先起 0，再起 1）
```

### 5.2 StatefulSet 的三大特性

```
1. 稳定的网络标识
   ─────────────────
   Pod 名称：<statefulset>-<ordinal>
   例如：mysql-0, mysql-1, mysql-2

   DNS：<pod-name>.<service-name>.<namespace>.svc.cluster.local
   例如：mysql-0.mysql.default.svc.cluster.local

   就算 Pod 重建，名称和 DNS 也不变。

2. 稳定的存储
   ─────────────────
   每个 Pod 有独立的 PVC（通过 volumeClaimTemplates 自动创建）
   mysql-0 ↔ pvc-mysql-0 ↔ 某块 EBS
   mysql-1 ↔ pvc-mysql-1 ↔ 另一块 EBS

   即使 Pod 重建，还是挂回同一块磁盘。

3. 有序部署/缩容
   ─────────────────
   启动：0 → 1 → 2（必须等前一个 Ready 才起下一个）
   缩容：2 → 1 → 0（逆序）
   更新：默认逆序滚动
```

### 5.3 需要 Headless Service 配合

```yaml
# 必须有一个 clusterIP: None 的 Service
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None          # ← Headless Service
  selector:
    app: mysql
  ports:
    - port: 3306

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql       # ← 关联 Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template: ...
  volumeClaimTemplates:    # ← 每个 Pod 自动创建独立 PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

```
为什么需要 Headless Service？
  普通 Service：DNS 返回 ClusterIP（一个虚拟 IP 做负载均衡）
  Headless Service：DNS 返回所有 Pod IP 列表

  StatefulSet 需要按 Pod 名称寻址，所以要 Headless Service
  提供 per-pod DNS。
```

### 5.4 典型场景

| 应用 | 为什么用 StatefulSet |
|------|---------------------|
| MySQL / PostgreSQL | 主从有明确角色，数据绑定 |
| Redis Cluster | 节点身份稳定、分片数据绑定 |
| Elasticsearch | 节点名稳定、shard 数据持久 |
| Kafka / ZooKeeper | 需要稳定的 quorum 成员 |
| Milvus / Etcd | 集群节点需要稳定身份 |

---

## 6. DaemonSet - 每节点一个 Pod

### 6.1 使用场景

```
DaemonSet 的哲学：每台 Node 上必须跑一份

典型用途：
  ┌──────────────────────────────────────────────────┐
  │                                                  │
  │  日志收集   →  Fluentd / Filebeat               │
  │  监控采集   →  Node Exporter / DataDog Agent    │
  │  网络插件   →  Calico / Cilium                  │
  │  存储插件   →  CSI Node Driver                  │
  │  安全扫描   →  Falco                            │
  │                                                  │
  │  这些都是"基础设施"级别，每台机器都要有            │
  └──────────────────────────────────────────────────┘
```

### 6.2 特性

```
1. 每个匹配的 Node 上恰好一个 Pod
   新加 Node？自动起一个。
   Node 下线？Pod 自动跟随消失。

2. 一般不设 replicas（由 Node 数量决定）

3. 通常需要较高权限（hostNetwork、hostPath 等）

4. 通过 nodeSelector / tolerations 控制在哪些 Node 上跑
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
        - operator: Exists            # 容忍所有 taint，连 master 也跑
      containers:
        - name: fluentd
          image: fluentd:v1.16
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:                   # 挂 Node 的文件系统
            path: /var/log
```

---

## 7. Job / CronJob - 任务型工作负载

### 7.1 Job - 跑一次就结束

```
Deployment 与 Job 的根本区别：

Deployment: Pod 挂了就重建，永远不结束
Job:        Pod 正常退出 (exit 0) 就认为成功，不再重启

用途：
  - 数据迁移
  - 批量计算
  - 一次性运维任务
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 5           # 需要成功完成 5 次
  parallelism: 2           # 同时并行 2 个 Pod
  backoffLimit: 3          # 失败重试上限
  template:
    spec:
      restartPolicy: OnFailure    # 或 Never
      containers:
        - name: migrator
          image: my-migrator:1.0
```

### 7.2 Job 的三种模式

```
1. 非并行 Job (completions=1, parallelism=1)
   ────────────────────────────────────────
   跑一个 Pod，成功就结束。
   最常见的"一次性任务"。

2. 固定完成次数 Job (completions=N, parallelism=M)
   ────────────────────────────────────────
   要成功跑 N 次，同时最多 M 个并行。
   用于"把 100 个文件各处理一次"之类。

3. 工作队列 Job (completions 不设, parallelism=M)
   ────────────────────────────────────────
   M 个 Pod 并行消费外部队列，直到队列空。
   每个 Pod 自己决定是否还有活可干。
```

### 7.3 CronJob - 定时触发 Job

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"           # 每天凌晨 2 点
  successfulJobsHistoryLimit: 3   # 保留最近 3 次成功记录
  failedJobsHistoryLimit: 1
  concurrencyPolicy: Forbid       # 上一次没跑完就不起新的
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: my-backup:1.0
```

```
CronJob 到点会创建一个 Job，Job 再创建 Pod。

层级：CronJob ──▶ Job ──▶ Pod
```

### 7.4 concurrencyPolicy 选项

| 值 | 语义 |
|----|------|
| `Allow` (默认) | 允许同时跑多个 Job |
| `Forbid` | 如果上一次还在跑，跳过本次 |
| `Replace` | 取消上一次，用新的替代 |

---

## 8. 工作负载选型决策树

```
                   ┌─────────────────────┐
                   │ 是持续运行的服务吗？ │
                   └──────┬──────────────┘
                          │
                ┌─────────┴─────────┐
                │                   │
               否                  是
                │                   │
                ▼                   ▼
         ┌───────────┐       ┌──────────────┐
         │ 一次性？  │       │ 每节点一个？ │
         └─────┬─────┘       └──────┬───────┘
               │                    │
         ┌─────┴──────┐      ┌──────┴──────┐
         │            │      │             │
        否           是     是            否
         │            │      │             │
         ▼            ▼      ▼             ▼
     ┌───────┐   ┌──────┐ ┌───────────┐ ┌─────────────┐
     │CronJob│   │ Job  │ │DaemonSet │ │有状态/有序? │
     └───────┘   └──────┘ └───────────┘ └──────┬──────┘
                                               │
                                       ┌───────┴───────┐
                                       │               │
                                      是              否
                                       │               │
                                       ▼               ▼
                                ┌─────────────┐  ┌────────────┐
                                │StatefulSet │  │ Deployment │
                                └─────────────┘  └────────────┘
```

---

## 9. 总结：工作负载资源的设计哲学

```
设计维度                  对应资源
─────────────────────────────────────────────
副本保证                  ReplicaSet
版本管理 + 滚动更新       Deployment
稳定身份 + 顺序           StatefulSet
每节点一份                DaemonSet
一次性任务                Job
定时任务                  CronJob
```

> **核心洞见：K8s 不是给你一个"万能容器"，而是把"应用部署模式"抽象成了若干种标准形态。选对资源类型，等于说对了 K8s 的"方言"。**
