# 第四层：关键机制深入 - 深度展开

## 1. 为什么要专门讲"机制"？

```
前三层回答的是"有什么资源"、"它们做什么"。
第四层回答："它们底层如何协作"。

只有理解机制，你才能：
  - 写出符合 K8s 哲学的自定义 Controller
  - 快速排查生产问题（为什么 Pod 没起来？为什么配置没生效？）
  - 避免踩坑（为什么改 ConfigMap 没效果？为什么 Pod 不重启？）

本章四个核心机制：
  1. Label & Selector       —— K8s 的关系引擎
  2. Controller 模式         —— 声明式的执行机制
  3. Pod 生命周期 & 探针      —— 单 Pod 的状态机
  4. 网络与存储模型           —— 横跨全局的数据平面
```

---

## 2. Label & Selector - K8s 的"关系数据库"

### 2.1 为什么 K8s 不用"外键"？

```
传统数据库思维：
  Deployment.id = 1
  Pod.deployment_id = 1   ← 外键关联

K8s 的思维：
  Deployment 用 selector 匹配 Pod 的 label
  没有硬绑定，只要 label 匹配就算"我的"

为什么？
  - 灵活性：同一组 Pod 可以同时被多个 Controller/Service 关注
  - 松耦合：删除关系不影响对象本身
  - 动态性：Pod 被创建后才打 label，仍能自动加入关系
```

### 2.2 Label 的规范

```
Label = Key-Value 的字符串对

key 格式：
  [prefix/]name
  例如：app.kubernetes.io/name, env, tier

推荐的标准 label（K8s 社区约定）：
  app.kubernetes.io/name        应用名
  app.kubernetes.io/instance    实例名
  app.kubernetes.io/version     版本
  app.kubernetes.io/component   组件
  app.kubernetes.io/part-of     归属应用
  app.kubernetes.io/managed-by  管理工具（helm/kustomize）
```

### 2.3 Selector 的两种写法

```yaml
# 等值选择（老式，简单）
selector:
  app: nginx
  env: prod

# 集合选择（表达能力更强）
selector:
  matchExpressions:
    - key: tier
      operator: In
      values: [frontend, backend]
    - key: env
      operator: NotIn
      values: [dev]
    - key: gpu
      operator: Exists
    - key: legacy
      operator: DoesNotExist
```

**命令行选择器：**
```bash
kubectl get pods -l "app=nginx,env=prod"
kubectl get pods -l "tier in (frontend,backend)"
kubectl get pods -l "!legacy"
```

### 2.4 Label 被谁用？

```
┌─────────────────────────────────────────────────────────────┐
│  使用方                       Selector 的作用                │
├─────────────────────────────────────────────────────────────┤
│  Service                     找到后端 Pod                    │
│  Deployment / ReplicaSet     管理哪些 Pod                    │
│  NetworkPolicy               规则适用于哪些 Pod              │
│  PodAffinity                 和谁做邻居                      │
│  Scheduler (nodeSelector)    调度到哪类 Node                 │
│  HPA                         监控哪些 Pod 的指标             │
│  PodDisruptionBudget         保护哪些 Pod                    │
│  kubectl                     人工筛选                        │
└─────────────────────────────────────────────────────────────┘

Label 就是 K8s 的"万能粘合剂"。
```

### 2.5 Annotation - 与 Label 的区别

```
Label       用于"筛选"，有长度限制 (≤63 字符)，被 selector 使用
Annotation  用于"元信息"，可以很长，不用于 selector

典型用途：
  Label:
    app: nginx, env: prod, version: v1
  Annotation:
    build-commit: abc123def...
    kubernetes.io/change-cause: "update image to v1.2"
    prometheus.io/scrape: "true"
    deployment.kubernetes.io/revision: "5"
```

---

## 3. Controller 模式 - K8s 所有自动化的核心

### 3.1 Reconciliation 循环的三步骤

```
while true:
    1. Observe  ——  读当前状态（从 cache）
    2. Diff     ——  比较期望 vs 实际
    3. Act      ——  执行操作让实际向期望靠拢
```

### 3.2 Informer 架构

```
┌──────────────────────────────────────────────────────────────┐
│                     API Server                               │
│                         │                                    │
│                    Watch (长连接)                             │
│                         ▼                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                    Reflector                         │    │
│  │   调用 API Server 的 Watch，收事件                    │    │
│  └──────────┬───────────────────────────────────────────┘    │
│             ▼                                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                    Delta FIFO                        │    │
│  │   事件队列：Added / Updated / Deleted                 │    │
│  └──────────┬───────────────────────────────────────────┘    │
│             ▼                                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                  Indexer / Store                     │    │
│  │   本地缓存（内存），按 namespace/name 索引             │    │
│  └──────────┬───────────────────────────────────────────┘    │
│             ▼                                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                  Event Handlers                      │    │
│  │   OnAdd / OnUpdate / OnDelete                        │    │
│  └──────────┬───────────────────────────────────────────┘    │
│             ▼                                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                  Work Queue                          │    │
│  │   只入队 "namespace/name"，去重                       │    │
│  └──────────┬───────────────────────────────────────────┘    │
│             ▼                                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                  Reconcile Loop                      │    │
│  │   从 queue 取一个 key，读 cache，对比，执行           │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### 3.3 为什么要用本地 cache + WorkQueue？

```
用 cache 的原因：
  - 每个 Controller Reconcile 都查 API Server → 压力爆炸
  - 本地 cache 通过 Watch 事件增量更新 → 一次全量，后续增量
  - 读操作永远命中本地，零延迟

用 WorkQueue 的原因：
  - 去重：同一个资源短时间多次变更只处理一次
  - 限流：RateLimiter 防止失败重试打爆下游
  - 并发：多个 worker 并发处理，key 级别互斥
  - 失败重试：requeue 机制 + 指数退避
```

### 3.4 Reconcile 的幂等性要求

```
核心要求：Reconcile(key) 必须幂等

为什么？
  - 同一个 key 可能被重复处理
  - 处理到一半崩溃重启，会再次处理
  - 不幂等 = 灾难

正确做法：
  每次从"现状"出发，决定"怎么调整到期望"
  而不是"我刚才做了什么，接着做"
```

### 3.5 CRD + Operator 模式

```
K8s 强大在于可扩展：

CRD (CustomResourceDefinition)
  ────────────────────────────
  定义新的资源类型
  例如：MySQLCluster、RedisCluster、MilvusCluster

Operator
  ────────────────────────────
  自定义 Controller，Reconcile 新资源类型
  把"领域知识"编码成控制器

典型 Operator 流程：
  用户 kubectl apply MySQLCluster
       ↓
  Operator 监听到
       ↓
  创建：StatefulSet + Service + ConfigMap + Secret
       ↓
  持续调和：扩缩容、主从切换、备份...
```

---

## 4. Pod 生命周期 - 单 Pod 的状态机

### 4.1 Pod Phase

```
┌──────────────────────────────────────────────────────────┐
│  Pending     已创建对象，但还没所有容器运行                 │
│              (镜像拉取中、调度中、等资源...)                │
│     │                                                    │
│     ▼                                                    │
│  Running     至少一个容器在跑                              │
│     │                                                    │
│     ├────▶ Succeeded  所有容器正常退出，不会重启（Job 常见） │
│     │                                                    │
│     └────▶ Failed     所有容器都退出，至少一个异常退出      │
│                                                          │
│  Unknown     节点失联，状态不明                            │
└──────────────────────────────────────────────────────────┘
```

### 4.2 完整启动流程

```
Scheduler 绑定 Pod 到 Node
          ↓
Kubelet 创建 Pod Sandbox (pause 容器 + CNI)
          ↓
挂载 Volumes（PVC attach → mount → bind mount 进容器）
          ↓
┌─────────────────────────────────────┐
│  按顺序启动 initContainers           │
│                                     │
│  init-1 → 退出 0 → init-2 → 退出 0  │
│                    ↓                │
│  若任一失败 → Pod 进入 Init:Error   │
│  (有 restartPolicy 决定是否重启)    │
└─────────────────────────────────────┘
          ↓
启动所有主容器（并行）
          ↓
postStart Hook（如有）
          ↓
┌─────────────────────────────────────┐
│  探针开始工作                        │
│    startupProbe                     │
│      通过 → liveness/readiness 激活 │
│    livenessProbe                    │
│      失败 → 重启容器                │
│    readinessProbe                   │
│      失败 → Service 不转流量        │
└─────────────────────────────────────┘
          ↓
Pod Ready
```

### 4.3 initContainers - 做启动准备

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ["sh", "-c", "until nc -z db 3306; do sleep 2; done"]
    - name: init-schema
      image: my-app:migrator
      command: ["./migrate.sh"]
  containers:
    - name: app
      image: my-app:1.0
```

```
特点：
  - 顺序执行
  - 必须全部成功才跑主容器
  - 挂了按 restartPolicy 重启
  - 资源计算：取 max(max(init), sum(main))
                 （因为 init 和 main 不同时运行）
```

### 4.4 三种探针

```
┌─────────────┬──────────────────────────────────────────┐
│ 探针        │ 作用                                     │
├─────────────┼──────────────────────────────────────────┤
│ startup     │ 应用启动期间独占，通过后让位             │
│             │ 解决："慢启动应用被 liveness 误杀"问题   │
├─────────────┼──────────────────────────────────────────┤
│ liveness    │ "进程还活着吗？"                         │
│             │ 失败 → 重启容器                          │
├─────────────┼──────────────────────────────────────────┤
│ readiness   │ "我能接流量吗？"                         │
│             │ 失败 → 从 Endpoints 摘除（不重启）       │
└─────────────┴──────────────────────────────────────────┘
```

**三种检查方式：**
```yaml
containers:
  - name: app
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30        # 最长等 30 * 10s = 5 分钟
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 0      # startup 已兜底，这里无需延时
      periodSeconds: 10
      timeoutSeconds: 1
      failureThreshold: 3
    readinessProbe:
      exec:
        command: ["cat", "/tmp/ready"]
      periodSeconds: 5
```

### 4.5 探针踩坑清单

```
❌ 没设 startupProbe，导致 liveness 在启动阶段就开始检查
   → 慢启动应用反复重启

❌ liveness 检查依赖下游（DB 连得上才返回 200）
   → 下游抖动时一堆 Pod 被杀，雪崩

❌ liveness = readiness（都查 /health）
   → 下游抖动 → readiness 失败摘流 + liveness 失败重启
   → 应该：liveness 只检查进程本身

✅ liveness  —— 进程级：我还活着
✅ readiness —— 业务级：我能处理请求
✅ startup   —— 启动期：给足初始化时间
```

### 4.6 Pod 终止流程

```
kubectl delete pod / 滚动更新 / 驱逐
         ↓
1. API Server 标记 deletionTimestamp
2. Pod 状态变 Terminating
3. Endpoint Controller 把 Pod 从 Endpoints 摘除  ──┐
   kube-proxy 更新规则（需要时间传播到各 Node）    │
                                                  │
4. Kubelet 并行执行：                              │
   - preStop Hook                                  │
   - 容器收到 SIGTERM（有 grace period，默认 30s） │
                                                  │
     优雅关闭期间 Pod IP 可能还在接流量！          │
     所以需要：                                    │
       a. preStop sleep 10s                       │
       b. 或应用收到 SIGTERM 后再处理一会          │
                                                  │
5. grace period 到 → SIGKILL                       │
6. 清理 Volume、CNI                                │
7. Pod 对象从 API Server 删除                      │
```

### 4.7 优雅关闭的正确姿势

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 15"]    # 等摘流生效
      # 应用本身：收到 SIGTERM 后停止接新请求，
      #           处理完存量请求后退出
```

---

## 5. K8s 网络模型回顾与深化

### 5.1 网络模型的三条公理（再次强调）

```
1. 每个 Pod 有独立 IP，不用 NAT
2. 所有 Pod 之间可以直接通信（跨 Node 也行）
3. Node 可以直接访问 Pod

为什么这么设计？
  简化应用层：应用看到的是"一张大二层网络"
  解耦：应用不需要知道 NAT、端口映射这些细节
```

### 5.2 四类网络通信

```
1. Container ↔ Container (同 Pod)
   ────────────────────────────
   共享 netns，用 localhost 通信

2. Pod ↔ Pod (同 Node)
   ────────────────────────────
   通过 Node 上的 bridge / host-routes 直连

3. Pod ↔ Pod (跨 Node)
   ────────────────────────────
   由 CNI 插件实现：
     Overlay (VXLAN/IPIP) → 封包
     或 Routing (BGP) → 原生路由

4. Pod ↔ Service (虚拟 IP)
   ────────────────────────────
   kube-proxy 的 iptables/IPVS 规则 DNAT 到真实 Pod IP
```

### 5.3 DNS 作为"服务发现中枢"

```
CoreDNS 不只解析 Service 名：

Service:
  my-svc.default.svc.cluster.local → ClusterIP

Pod (仅 StatefulSet 自动启用 / 其他需 hostname):
  pod-0.my-svc.default.svc.cluster.local → Pod IP

PTR 反查:
  10.1.1.5 → <pod>.default.svc.cluster.local
```

---

## 6. K8s 存储模型回顾与深化

### 6.1 挂载的完整链路

```
┌──────────────────────────────────────────────────────────────┐
│  CSI Controller：在云厂商创建卷（EBS/Disk/NFS share）        │
│                         │                                    │
│                    Attach to Node                             │
│                         ▼                                    │
│  CSI Node Driver：在 Node 上发现设备，mount 到 staging 路径   │
│                         │                                    │
│                    Publish                                    │
│                         ▼                                    │
│  Kubelet：从 staging 路径 bind mount 到 Pod 容器              │
│                         │                                    │
│                         ▼                                    │
│  Container 看到 /data 就是这块卷                              │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 Pod 迁移时存储怎么办？

```
ReadWriteOnce (EBS 类)：
  Pod 从 Node A 迁到 Node B
   1. Node A 上先 umount + detach
   2. Node B 上 attach + mount
   3. 新 Pod 起来，挂上

ReadWriteMany (NFS/EFS)：
  多 Node 可同时 mount，Pod 漂移更简单
```

### 6.3 "StatefulSet + PVC 不会丢数据"的原理

```
StatefulSet 的 Pod 与 PVC 有"稳定绑定关系"：

  mysql-0  ↔  pvc-mysql-0   ←  PV  ←  EBS-xxxxx
  mysql-1  ↔  pvc-mysql-1   ←  PV  ←  EBS-yyyyy

Pod 删了：只删 Pod，不删 PVC
重建 Pod：按名字 mysql-0 重新创建，挂回 pvc-mysql-0
数据：完整保留
```

---

## 7. 核心事件流：从 kubectl apply 到容器运行

结合前面几章，把端到端的事件链串起来：

```
┌──────────────────────────────────────────────────────────────┐
│  kubectl apply -f deployment.yaml                            │
│         │                                                    │
│         ▼                                                    │
│  API Server                                                  │
│    认证 → 授权 → Mutating Admission → Validation              │
│    → Validating Admission → 写 etcd                          │
│         │                                                    │
│         ▼ (Watch 事件)                                       │
│  Deployment Controller                                       │
│    发现新 Deployment，创建 ReplicaSet                        │
│         │                                                    │
│         ▼ (Watch 事件)                                       │
│  ReplicaSet Controller                                       │
│    发现新 RS，按 replicas 数创建 N 个 Pod                     │
│         │                                                    │
│         ▼ (Watch 事件)                                       │
│  Scheduler                                                   │
│    Filter + Score → 给每个 Pod 挑 Node                       │
│    更新 pod.spec.nodeName                                    │
│         │                                                    │
│         ▼ (Watch 事件)                                       │
│  Kubelet (被调度到的 Node)                                   │
│    1. 创建 pause 容器                                        │
│    2. 调用 CNI 分 IP                                          │
│    3. 调用 CSI 挂载卷                                         │
│    4. 跑 initContainers                                      │
│    5. 跑业务容器                                              │
│    6. 启动探针                                                │
│         │                                                    │
│         ▼ (Watch 事件)                                       │
│  Endpoint Controller                                         │
│    Pod Ready → 加入 Endpoints                                │
│         │                                                    │
│         ▼ (Watch 事件)                                       │
│  kube-proxy (所有 Node)                                      │
│    更新 iptables/IPVS 规则，Service 把流量带到新 Pod          │
│                                                              │
│                        完成 ✓                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. 总结：机制层的智慧

```
机制                   核心思想                             价值
────────────────────────────────────────────────────────────────
Label & Selector       关系数据库思维                      动态松耦合
Controller + Informer  Watch + Cache + Reconcile           自愈与一致性
Pod 生命周期 & 探针    清晰的状态机                        可预测行为
网络模型               "一张大二层"的抽象                  应用透明
存储模型               PVC 与 Pod 解耦绑定                 数据持久化
事件驱动架构           所有组件只与 API Server 对话        松耦合 + 可扩展
```

> **核心洞见：K8s 的所有"魔法"本质上是同一种设计模式的反复应用——声明期望 + Watch 变化 + 调和现实。理解了这一点，你就理解了为什么 K8s 的扩展点（CRD、Operator、Admission Webhook）能如此自然地融入。整个系统就是一张"所有玩家都监听同一个 API Server、协作调整共同状态"的大棋盘。**
