# Kubernetes 学习脉络 - 从第一性原理出发

## 第一层：理解"为什么" (Why)

### 1. 容器化的本质问题
- 单机容器 (Docker) 解决了什么？→ 环境一致性、进程隔离
- 单机容器解决不了什么？→ **多机编排、故障恢复、弹性扩缩**
- K8s 的定位：**分布式容器编排系统**

### 2. K8s 的核心设计哲学
```
声明式 > 命令式
期望状态 (Desired State) vs 实际状态 (Current State)
控制循环 (Control Loop) 不断调和两者
```

---

## 第二层：核心架构 (What)

### 3. 两个视角理解架构

**物理视角（组件）：**
```
┌─────────────────────────────────────────────────┐
│  Control Plane (Master)                         │
│  ┌──────────┐ ┌────────────┐ ┌───────────────┐ │
│  │API Server│ │ Scheduler  │ │Controller Mgr │ │
│  └────┬─────┘ └────────────┘ └───────────────┘ │
│       │                                         │
│  ┌────▼─────┐                                   │
│  │  etcd    │  ← 唯一的真相来源                  │
│  └──────────┘                                   │
└─────────────────────────────────────────────────┘
        │
┌───────▼─────────────────────────────────────────┐
│  Worker Node                                    │
│  ┌─────────┐  ┌───────────┐  ┌──────────────┐  │
│  │ Kubelet │  │ Kube-proxy│  │ Container RT │  │
│  └─────────┘  └───────────┘  └──────────────┘  │
└─────────────────────────────────────────────────┘
```

**逻辑视角（控制循环）：**
```
用户 → API Server → etcd (存储期望状态)
                       ↓
              Controller 监听变化
                       ↓
              对比期望 vs 实际
                       ↓
              执行调和动作
```

---

## 第三层：核心资源对象 (按依赖关系学习)

### 4. 工作负载层 (从简单到复杂)

| 顺序 | 资源 | 本质 |
|------|------|------|
| 1 | **Pod** | 最小调度单元，一组共享网络/存储的容器 |
| 2 | **ReplicaSet** | 保证 N 个 Pod 副本运行 |
| 3 | **Deployment** | 管理 ReplicaSet，支持滚动更新 |
| 4 | StatefulSet | 有状态应用（稳定标识、顺序部署） |
| 5 | DaemonSet | 每个节点运行一个 Pod |
| 6 | Job/CronJob | 一次性/定时任务 |

### 5. 服务发现与网络层

| 资源 | 本质 |
|------|------|
| **Service** | 稳定的访问入口（ClusterIP/NodePort/LoadBalancer） |
| **Ingress** | HTTP 层路由（类似反向代理） |
| Endpoints | Service 背后的实际 Pod IP 列表 |
| NetworkPolicy | 网络层 ACL |

### 6. 配置与存储层

| 资源 | 用途 |
|------|------|
| **ConfigMap** | 非敏感配置 |
| **Secret** | 敏感配置（Base64，非加密） |
| **PV/PVC** | 持久化存储抽象 |
| StorageClass | 动态存储供给 |

### 7. 调度与资源管理

- **Node** - 工作节点
- **Namespace** - 资源隔离
- **ResourceQuota / LimitRange** - 资源限制
- **Taint/Toleration, Affinity** - 调度约束

---

## 第四层：关键机制深入

### 8. 必须理解的机制

1. **Label & Selector** - K8s 的"关系型数据库查询"
2. **Controller 模式** - Informer → WorkQueue → Reconcile
3. **网络模型** - 每个 Pod 有独立 IP，跨节点可达
4. **存储模型** - PV/PVC 解耦存储供给与消费

### 9. 生命周期管理
```
Pod 生命周期:
  Init Containers → Main Containers → Termination

探针:
  - livenessProbe  → 是否重启
  - readinessProbe → 是否接流量
  - startupProbe   → 启动检测
```

---

## 第五层：生产实践

### 10. 运维核心技能
- `kubectl` 熟练使用（get/describe/logs/exec/port-forward）
- YAML 编写与调试
- 日志与监控（Prometheus + Grafana）
- 故障排查思路

### 11. 安全与权限
- **RBAC** - Role/ClusterRole/RoleBinding
- ServiceAccount
- Pod Security Standards

---

## 推荐学习路径

```
Week 1-2: 概念 + 单机实验 (minikube/kind)
    └── Pod, Deployment, Service 反复练习

Week 3-4: 深入工作负载
    └── 有状态应用、配置管理、持久化

Week 5-6: 网络与服务治理
    └── Service/Ingress/DNS/NetworkPolicy

Week 7-8: 生产化能力
    └── 监控、日志、RBAC、资源管理
```

---

## 核心心法

> **记住一句话：K8s 就是一个"声明式的分布式状态机"。**
>
> 你告诉它你想要什么（YAML），Controller 不断努力让现实靠近期望。
