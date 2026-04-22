# 第五层：生产实践与安全 - 深度展开

## 1. 从"能跑"到"能打"的距离

```
学会部署 Pod 只是起点，生产环境还要回答：
  - 怎么高效操作集群？（kubectl）
  - 怎么管理大量 YAML？（Helm/Kustomize）
  - 出问题怎么查？（日志/监控/调试）
  - 怎么保证安全？（RBAC、镜像、Pod 安全）
  - 怎么控制成本与稳定性？（HPA、VPA、PDB）

本章就是把前四层的知识串起来，回答"生产上怎么干"。
```

---

## 2. kubectl - 运维的瑞士军刀

### 2.1 必备的核心子命令

```
资源操作
  get             列出资源
  describe        详细信息 + 事件
  apply / create  声明式 / 命令式创建
  delete          删除
  edit            在线编辑

运行时操作
  logs            看容器日志
  exec            进入容器执行命令
  port-forward    把本地端口转发到 Pod
  cp              拷贝文件进/出容器
  top             CPU/内存使用

调试
  rollout         查看/回滚部署
  debug           启动 debug 容器
  events          查看事件

集群操作
  cordon / drain  标记节点不可调度 / 驱逐 Pod
  taint           打污点
  config          切换 context
```

### 2.2 高频排障三板斧

```bash
# 1. 这 Pod 怎么了？
kubectl describe pod <pod-name>
# 看 Events 部分，90% 的问题答案都在这里

# 2. 进程在干嘛？
kubectl logs <pod-name> -c <container> --tail=200 -f
kubectl logs <pod-name> --previous      # 上次崩溃前的日志

# 3. 直接进去看
kubectl exec -it <pod-name> -- sh
```

### 2.3 kubectl 的"瑞士军刀技巧"

```bash
# 跨所有命名空间
kubectl get pods -A

# 按 label 过滤
kubectl get pods -l app=web,env=prod

# 自定义列输出
kubectl get pods -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase'

# JSONPath 提取字段
kubectl get pods -o jsonpath='{.items[*].spec.nodeName}' | tr ' ' '\n' | sort | uniq -c

# 按 NODE 看 Pod 分布
kubectl get pods -A -o wide --sort-by=.spec.nodeName

# 看 API 资源的所有字段说明
kubectl explain pod.spec.containers.resources --recursive

# 快速 yaml 生成
kubectl create deployment web --image=nginx --dry-run=client -o yaml > web.yaml

# 临时 port-forward 测试
kubectl port-forward svc/my-svc 8080:80

# 事件按时间排序
kubectl get events --sort-by=.lastTimestamp -A

# 查某资源的 owner chain
kubectl get pod <pod> -o jsonpath='{.metadata.ownerReferences}'
```

### 2.4 效率神器

```bash
# 缩写
alias k=kubectl
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'

# 自动补全
source <(kubectl completion zsh)

# 多集群切换
kubectl config get-contexts
kubectl config use-context prod

# 插件（通过 krew 安装）
kubectl ctx      # kubectx - 切换 context
kubectl ns       # kubens  - 切换 namespace
kubectl neat     # 清理 yaml 中的 managedFields 等噪音
kubectl stern    # 多 Pod 日志流式查看
kubectl tree     # 查看资源的 owner 树
```

---

## 3. YAML 工程化 - Helm 与 Kustomize

### 3.1 为什么不能只写裸 YAML？

```
真实场景：
  - 同一个服务要部署到 dev / staging / prod
  - 每个环境镜像 tag、replicas、resource 不同
  - 有 50+ 个微服务

直接复制 YAML？
  → 改一行要改三份
  → 容易漂移（dev 和 prod 长得不一样）
  → 无法复用组件

两大方案：
  Helm       模板 + 参数（Go template）
  Kustomize  叠加 + 补丁（patch）
```

### 3.2 Helm

```
Helm Chart = 一套带参数的 YAML 模板

结构：
  mychart/
    Chart.yaml           # 元信息
    values.yaml          # 默认参数
    templates/
      deployment.yaml    # 使用 {{ .Values.xxx }}
      service.yaml
      ...

安装：
  helm install myapp ./mychart -f values-prod.yaml
  helm upgrade myapp ./mychart -f values-prod.yaml
  helm rollback myapp 2

优点：
  - 参数化灵活
  - 有仓库生态（Bitnami/Artifact Hub）
  - 一条命令安装复杂应用

缺点：
  - Go template 语法晦涩
  - 模板和结果差距大，调试痛苦
  - 变量之间的依赖难追踪
```

### 3.3 Kustomize

```
Kustomize 的哲学：不模板化，只"叠加"

结构：
  base/
    deployment.yaml
    service.yaml
    kustomization.yaml
  overlays/
    dev/
      kustomization.yaml       # 引用 base + dev 的补丁
      patch-replicas.yaml
    prod/
      kustomization.yaml
      patch-replicas.yaml

使用：
  kubectl apply -k overlays/prod

优点：
  - 无模板，调试容易（kustomize build 看结果）
  - K8s 原生支持（kubectl -k）
  - 组合性强

缺点：
  - 复杂参数传递不如 Helm
```

### 3.4 选型建议

```
选 Helm：
  - 部署第三方应用（Prometheus / Istio / PostgreSQL）
  - 需要分发给其他团队使用

选 Kustomize：
  - 自家业务的多环境部署
  - 喜欢"所见即所得"

也可以组合：Helm 渲染出 base → Kustomize 再 overlay
```

---

## 4. 日志 - 谁在乎 stdout？

### 4.1 K8s 的日志假设

```
应用只需做一件事：把日志写到 stdout / stderr

容器运行时：把 stdout 重定向到 Node 上的文件
  /var/log/pods/<namespace>_<pod>/<container>/0.log

kubectl logs 就是读这个文件。
```

### 4.2 生产日志方案

```
Node 级 DaemonSet 采集：

┌────────────────────────────────────────────────────────┐
│  每个 Node 跑一个 fluentd/fluent-bit/filebeat Pod       │
│  ├─ 读 /var/log/pods/*/*/*.log                         │
│  ├─ 解析 + 打标签（pod_name, namespace, ...）          │
│  └─ 发送到 Elasticsearch / Loki / CloudWatch           │
└────────────────────────────────────────────────────────┘
```

### 4.3 最佳实践

```
✅ 所有日志输出到 stdout/stderr（不要写本地文件）
✅ 结构化日志（JSON 格式），方便解析和查询
✅ 带 trace ID / request ID，跨服务追踪
✅ 区分级别（ERROR/WARN/INFO），生产环境关 DEBUG
❌ 不要把敏感信息（密码、token）写进日志
❌ 不要在容器里写到持久卷然后自己管理日志
```

---

## 5. 监控 - 看得见才能管理

### 5.1 监控的四个黄金指标

```
Google SRE 四个黄金指标：
  1. Latency         延迟
  2. Traffic         流量
  3. Errors          错误率
  4. Saturation      饱和度（资源使用率）

监控分层：
  基础设施   Node CPU/Mem/Disk/Network
  容器       Pod CPU/Mem/重启次数
  应用       QPS/Latency/Error rate
  业务       下单成功率、GMV...
```

### 5.2 Prometheus + Grafana 体系

```
┌────────────────────────────────────────────────────────────┐
│                                                            │
│   ┌─────────────┐                                          │
│   │ Node        │ ◀── DaemonSet，采集 Node 指标             │
│   │ Exporter    │                                          │
│   └─────────────┘                                          │
│                                                            │
│   ┌─────────────┐                                          │
│   │ kube-state- │ ◀── Deployment，把 K8s 对象状态转成指标   │
│   │ metrics     │     (Pod count, Deployment.replicas...)  │
│   └─────────────┘                                          │
│                                                            │
│   ┌─────────────┐                                          │
│   │ 应用自身      │ ◀── 暴露 /metrics 端点                  │
│   │ (Prom SDK)  │                                          │
│   └─────────────┘                                          │
│                                                            │
│        ↑ scrape                                            │
│        │                                                   │
│   ┌─────────────┐      ┌─────────────┐                    │
│   │ Prometheus  │ ───▶ │  Grafana    │                    │
│   │ (存储+查询) │      │  (可视化)    │                    │
│   └─────────────┘      └─────────────┘                    │
│        │                                                   │
│        ▼                                                   │
│   ┌─────────────┐                                          │
│   │ Alertmanager│ ──▶ 企业微信 / Slack / PagerDuty         │
│   └─────────────┘                                          │
└────────────────────────────────────────────────────────────┘
```

### 5.3 kube-prometheus-stack

```
生产环境推荐一条命令装上整套：
  helm install monitoring prometheus-community/kube-prometheus-stack

包含：
  - Prometheus Operator
  - Prometheus + Alertmanager
  - Grafana + 预置 Dashboard
  - Node Exporter + kube-state-metrics
  - 常用告警规则
```

---

## 6. 故障排查思路

### 6.1 分层排查法

```
从"症状"向下推导：

  业务报警：用户下单失败
       ↓
  应用层：QPS 降为 0？Latency 飙升？5xx 变多？
       ↓
  K8s 层：Pod 全挂了？Ready 数量骤降？HPA 扩容卡住？
       ↓
  基础设施：Node 失联？磁盘满了？网络分区？
```

### 6.2 Pod 启动不了的排查清单

```
┌──────────────────────────────────────────────────────────┐
│   症状                       大概率原因                    │
├──────────────────────────────────────────────────────────┤
│   Pending                    调度失败（资源、nodeSelector）│
│                              PVC 无法绑定                  │
│   ContainerCreating          镜像拉取、卷挂载、CNI 分 IP    │
│   ImagePullBackOff           镜像名错、仓库凭证缺失         │
│   CrashLoopBackOff           应用起来就挂（看 logs）       │
│   OOMKilled                  内存 limits 太小              │
│   Error                      command 写错 / Init 失败      │
│   Terminating 卡住           finalizer 未处理 / 卷卸载失败  │
└──────────────────────────────────────────────────────────┘
```

### 6.3 排查命令串

```bash
# 1. Pod 整体状态
kubectl get pod <pod> -o wide

# 2. 事件与条件
kubectl describe pod <pod>

# 3. 当前日志 / 上次崩溃日志
kubectl logs <pod> [-c <container>] [--previous]

# 4. 实时进程、挂载
kubectl exec -it <pod> -- sh
# 里面 ps / cat /etc/resolv.conf / env / mount

# 5. 网络连通性
kubectl run -it --rm debug --image=nicolaka/netshoot -- bash
# 里面：nslookup / dig / curl / tcpdump

# 6. 节点资源
kubectl top node
kubectl top pod -A --sort-by=memory

# 7. 事件（集群级问题）
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

### 6.4 Ephemeral Container（kubectl debug）

```
痛点：业务镜像精简（distroless），没 sh、没 curl，怎么调试？

方案：kubectl debug 往 Pod 里临时加一个调试容器，
      共享进程空间、网络空间

# 复制一个 Pod 然后进去调试
kubectl debug <pod> -it --image=nicolaka/netshoot --share-processes --copy-to=<pod>-debug

# 直接给运行中的 Pod 加容器（K8s 1.23+ 默认开启）
kubectl debug -it <pod> --image=busybox --target=<container>
```

---

## 7. RBAC - 权限控制的基石

### 7.1 四个核心对象

```
┌───────────────────────────────────────────────────────────┐
│                                                           │
│   Role  /  ClusterRole                                    │
│       "能做什么" (动作清单)                                │
│       Role 是 namespace 级，ClusterRole 是集群级           │
│                                                           │
│   RoleBinding  /  ClusterRoleBinding                      │
│       "谁能做" (把 Role 绑给 User/Group/ServiceAccount)    │
│       RoleBinding 是 namespace 级                         │
│       ClusterRoleBinding 是集群级                         │
│                                                           │
└───────────────────────────────────────────────────────────┘

授权 = Subject (谁) + Verb (做什么) + Resource (对什么)
```

### 7.2 典型 Role 定义

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-a
  name: pod-reader
rules:
  - apiGroups: [""]            # core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
    resourceNames: ["web"]     # 只针对特定对象（可选）
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: team-a
  name: read-pods
subjects:
  - kind: User
    name: alice@example.com
  - kind: ServiceAccount
    name: my-app
    namespace: team-a
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 7.3 常见 Verb

```
get / list / watch       查
create / update / patch  写
delete / deletecollection 删
impersonate              模拟其他身份
*                        所有
```

### 7.4 ServiceAccount - Pod 的身份

```
每个 Pod 都跑着一个"身份"（默认是 namespace 的 default SA）

用途：
  Pod 里运行的程序调用 K8s API 时需要身份
  例如：Prometheus 要读取 Pod 列表，Ingress Controller
        要读取 Service 列表

YAML:
spec:
  serviceAccountName: my-app   # 不指定则用 default

Token 自动注入到 Pod：
  /var/run/secrets/kubernetes.io/serviceaccount/token
  /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  /var/run/secrets/kubernetes.io/serviceaccount/namespace
```

### 7.5 最小权限原则

```
✅ 为每类工作负载创建专用 ServiceAccount
✅ Role 用白名单（明确允许什么），避免 verbs: ["*"]
✅ 只绑定真正需要的 Role
✅ 禁用默认 SA 的 token 自动挂载（automountServiceAccountToken: false）
   如果业务不需要调 K8s API

❌ 不要把 cluster-admin 随便给人
❌ 不要用 * 匹配所有资源/动作
```

---

## 8. Pod Security - 容器级安全

### 8.1 SecurityContext - Pod/Container 级配置

```yaml
spec:
  securityContext:                    # Pod 级
    runAsNonRoot: true                # 不以 root 运行
    runAsUser: 1000
    fsGroup: 2000                     # 挂载卷的属主
    seccompProfile:
      type: RuntimeDefault            # 启用 seccomp 过滤
  containers:
    - name: app
      securityContext:                # Container 级
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true  # 根文件系统只读
        capabilities:
          drop: ["ALL"]               # 默认丢弃所有特殊能力
          add: ["NET_BIND_SERVICE"]   # 只加必须的
```

### 8.2 Pod Security Standards (PSS)

```
K8s 定义了三个预设档位：

Privileged  完全不限制（只给系统组件用）
Baseline    最小限制（防明显的提权）
Restricted  严格限制（推荐生产）

K8s 1.25+ 用 Pod Security Admission 强制执行：

apiVersion: v1
kind: Namespace
metadata:
  name: app
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 8.3 镜像安全

```
供应链安全是生产环境的重点：

✅ 使用可信仓库（私有 Harbor / ECR）
✅ 镜像扫描（Trivy / Clair）
✅ 签名验证（cosign / Sigstore）
✅ 不用 latest tag，固定 SHA256
✅ 用最小基础镜像（distroless / Alpine）
✅ 不在镜像里放密钥、SSH key、.aws/credentials
```

### 8.4 Admission Webhook - 自定义准入

```
在 API Server 写入 etcd 之前拦截：

Mutating Webhook:  修改对象
  例如：Istio 自动注入 sidecar

Validating Webhook: 校验对象
  例如：OPA Gatekeeper 检查策略
        Kyverno 执行合规检查

生产环境典型策略：
  - 必须设置 resources
  - 禁止 latest 标签
  - 禁止 hostPath
  - 必须带 owner label
```

---

## 9. 弹性与可用性

### 9.1 HPA - 水平自动扩缩

```
Horizontal Pod Autoscaler

原理：监控指标 → 计算期望副本数 → 调整 Deployment.replicas
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60    # 平均 CPU 超 60% 就扩
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:                          # 扩缩行为控制
    scaleUp:
      stabilizationWindowSeconds: 0
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容更保守
```

### 9.2 VPA - 垂直自动扩缩

```
Vertical Pod Autoscaler：自动调整 requests/limits

适用：
  - 单实例应用，无法水平扩展
  - 资源请求值难以估算

模式：
  Off            只推荐，不自动改
  Initial        创建时生效
  Auto           自动改，重启 Pod 应用
```

### 9.3 Cluster Autoscaler

```
前两个是"Pod 层"的伸缩，还需要"Node 层"：

Cluster Autoscaler
  - Pod Pending 因为资源不够 → 加 Node
  - Node 上 Pod 少且可迁移 → 减 Node

和云厂商集成：
  - AWS：调整 ASG
  - GCP：调整 MIG
  - Azure：调整 VMSS

新一代替代：Karpenter（AWS 主推，更灵活）
```

### 9.4 PodDisruptionBudget 回顾

```
确保 voluntary disruption 时有足够可用副本，
与 HPA 搭配形成"弹性 + 稳定"双保险。
```

### 9.5 多可用区部署

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: web
```

```
效果：副本均匀分布到 3 个 AZ，单个 AZ 挂掉还有 2/3 可用。
```

---

## 10. 成本与效率

### 10.1 常见浪费

```
1. 过度请求（requests 远大于实际使用）
   → 资源利用率低，集群规模虚增

2. Pod 密度低
   → Node CPU/Mem 使用率 < 30%

3. 非生产环境常驻
   → dev/staging 跑着空载 Pod

4. LoadBalancer 太多
   → 每个都花钱，应合并到 Ingress

5. 未合理使用 Spot/抢占式实例
   → 成本可降 70%+
```

### 10.2 优化手段

```
✅ 用 VPA 的 recommendation 模式调整 requests
✅ 生产用 Guaranteed，离线任务用 BestEffort
✅ 非关键业务跑在 Spot 实例节点（加 taint + toleration）
✅ 夜间缩容开发环境（CronJob + kubectl scale）
✅ 监控 Node 利用率，Cluster Autoscaler 自动下线空闲节点
✅ 大镜像分层缓存，多个服务共享 base image
```

---

## 11. 生产级 Checklist

### 11.1 应用上线前

```
[ ] 镜像固定 tag（非 latest），镜像已扫描
[ ] 设置了 requests 和 limits
[ ] 配置了 livenessProbe + readinessProbe (+ startupProbe)
[ ] preStop + 合理的 terminationGracePeriodSeconds
[ ] 多副本 + PodDisruptionBudget
[ ] 跨可用区 TopologySpread
[ ] 非 root 运行，丢弃不必要的 capabilities
[ ] ConfigMap/Secret 注入配置，不硬编码
[ ] Secret 来自外部密钥管理（Vault/External Secrets）
[ ] 日志输出到 stdout
[ ] 暴露 /metrics 给 Prometheus
[ ] 有对应的 Grafana Dashboard 和告警规则
[ ] NetworkPolicy 限制访问
[ ] HPA 已配置（如适用）
```

### 11.2 集群建设

```
[ ] 至少 3 个 master 节点（高可用）
[ ] etcd 定期备份（Velero 或云厂商快照）
[ ] 日志采集上了 DaemonSet
[ ] 监控栈（kube-prometheus-stack）部署
[ ] Ingress Controller（nginx / ALB / Istio Gateway）
[ ] CNI 支持 NetworkPolicy（Calico/Cilium）
[ ] CSI + 默认 StorageClass
[ ] Cluster Autoscaler / Karpenter
[ ] RBAC 分级（admin / developer / read-only）
[ ] Pod Security Standard 启用（restricted on 业务 ns）
[ ] Admission Webhook（Kyverno/Gatekeeper）
[ ] 定期升级策略（每半年跟 K8s 新版本）
[ ] 灾备方案（Velero 备份 + 恢复演练）
```

---

## 12. 总结：生产思维的五个维度

```
维度                   关注点                        典型工具
──────────────────────────────────────────────────────────────
操作效率               快速做对事                     kubectl + 插件
部署工程化             版本化、可重复、多环境          Helm / Kustomize
可观测性               看得见 + 查得出                 Prom/Grafana/Loki
安全                   身份、权限、网络、供应链        RBAC/PSS/镜像扫描
弹性与成本             自动伸缩 + 降本                 HPA/VPA/Karpenter
```

> **核心洞见：生产 K8s 不是一个"高阶 YAML 练习"，而是一门系统工程。它要求你同时思考业务、开发体验、安全、成本、可靠性五条主线。真正的 K8s 高手，不是能写多复杂的 YAML，而是知道什么时候什么都不做——让简单的东西保持简单，只在必要处引入复杂度。**

---

## 至此，五层学习脉络全部完成

```
Layer 1 (Why)          01-why-kubernetes.md        ✓
Layer 2 (What/架构)    02-architecture.md          ✓
Layer 3 (核心资源)     03 工作负载                  ✓
                       04 服务与网络                ✓
                       05 配置与存储                ✓
                       06 调度与资源管理            ✓
Layer 4 (关键机制)     07-core-mechanisms.md       ✓
Layer 5 (生产实践)     08-production.md            ✓
```

继续下去，推荐方向：
  - CRD + Operator 开发（controller-runtime / Kubebuilder）
  - Service Mesh（Istio / Linkerd / Cilium Service Mesh）
  - GitOps（ArgoCD / Flux）
  - 多集群管理（Cluster API / Karmada）
  - 边缘/异构计算（KubeEdge）
