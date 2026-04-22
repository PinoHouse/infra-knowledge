# 第三层：核心资源对象 (二) 服务发现与网络 - 深度展开

## 1. 从第一性原理看 K8s 网络

### 1.1 Pod 网络的三个基本假设

```
K8s 网络模型的"三大公理"：

1. 每个 Pod 有独立 IP
   ─────────────────
   不用 NAT，不共享端口。
   Pod 里的进程看到的自己的 IP = 外面看到的 Pod IP。

2. Pod 之间可以直接通信
   ─────────────────
   不管在不在同一个 Node，Pod A 可以直接用 Pod B 的 IP 访问。
   不需要 NAT、不需要端口映射。

3. Node 可以直接访问 Pod
   ─────────────────
   Node 上的进程（比如 kubelet）可以用 Pod IP 直接访问 Pod。
```

### 1.2 为什么要有 Service？

```
问题：Pod IP 是会变的！

  Pod 重建 → 新 IP
  扩容     → 新 Pod、新 IP
  缩容     → IP 消失

  如果 Frontend 要访问 Backend，每次 Backend Pod 重建
  都要更新配置？

解决方案：引入一个"稳定的虚拟入口"

  Frontend ──▶ Service (稳定 IP) ──▶ [Pod1, Pod2, Pod3]
                                       (动态变化的 Pod 集合)
```

### 1.3 网络的四层抽象

```
┌───────────────────────────────────────────────────────────┐
│   Layer 4: Ingress        L7 路由（HTTP 路径、域名）       │
│                                                           │
│   Layer 3: Service        L4 负载均衡（IP:Port）           │
│                                                           │
│   Layer 2: Endpoints      Service → 实际 Pod IP 的映射     │
│                                                           │
│   Layer 1: Pod 网络       每个 Pod 的 IP 连通性            │
│            (CNI 插件实现)                                  │
└───────────────────────────────────────────────────────────┘

越往上越接近业务，越往下越接近基础设施。
```

---

## 2. Service - 稳定的访问入口

### 2.1 Service 的本质

```
Service 不是一个真实的网络设备，而是一组 iptables/IPVS 规则。

┌────────────────────────────────────────────────────────────┐
│  Service: my-svc (ClusterIP: 10.96.0.100)                  │
│                                                            │
│  iptables 规则（简化）:                                     │
│  ────────────────────────────────                          │
│  dst=10.96.0.100:80  →  random pick:                       │
│                          ├─ 10.1.1.5:8080  (Pod1)         │
│                          ├─ 10.1.2.7:8080  (Pod2)         │
│                          └─ 10.1.3.9:8080  (Pod3)         │
└────────────────────────────────────────────────────────────┘

关键洞见：
  - ClusterIP 是"虚拟"的，没有任何实体绑定它
  - 数据包在内核态就被 DNAT 到真实 Pod IP
  - 访问 Service 看起来像"有个负载均衡器"，其实是每个 Node 上的
    kube-proxy 维护的规则在做分发
```

### 2.2 Service 如何找到后端 Pod？

```
两步映射：

Service  ──selector──▶  匹配的 Pod 集合
   │
   └──▶  Endpoints 对象（记录实际 Pod IP 列表）
         ↑
         │ Endpoint Controller 监听 Pod 变化，自动维护

┌──────────────────────────────────────────────────────┐
│  apiVersion: v1                                      │
│  kind: Service                                       │
│  metadata:                                           │
│    name: my-svc                                      │
│  spec:                                               │
│    selector:                                         │
│      app: web           ← 选择 label app=web 的 Pod  │
│    ports:                                            │
│      - port: 80         ← Service 对外端口           │
│        targetPort: 8080 ← Pod 容器端口               │
└──────────────────────────────────────────────────────┘
```

### 2.3 Service 的四种类型

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   ClusterIP (默认)                                          │
│   ─────────────────                                         │
│   只能集群内部访问                                            │
│   适用：后端服务、数据库                                      │
│                                                             │
│   NodePort                                                  │
│   ─────────────────                                         │
│   每个 Node 上开一个端口（30000-32767）                      │
│   Node:Port → Service                                       │
│   适用：开发测试、临时暴露                                    │
│                                                             │
│   LoadBalancer                                              │
│   ─────────────────                                         │
│   云厂商为你创建一个真实的外部 LB                             │
│   External LB → Node:Port → Service                         │
│   适用：生产环境暴露服务                                      │
│                                                             │
│   ExternalName                                              │
│   ─────────────────                                         │
│   DNS CNAME，把 Service 名映射到外部域名                      │
│   my-svc → my-db.rds.amazonaws.com                          │
│   适用：集群内统一访问外部资源                                │
└─────────────────────────────────────────────────────────────┘
```

### 2.4 图解四种类型

```
ClusterIP：
  集群内 Pod ──▶ ClusterIP ──▶ Pod

NodePort：
  外部 ──▶ NodeIP:30080 ──▶ ClusterIP ──▶ Pod
          (每个 Node 都能接)

LoadBalancer：
  外部 ──▶ Cloud LB (公网IP) ──▶ NodeIP:30080 ──▶ Pod

ExternalName：
  Pod 访问 my-db ──▶ DNS 返回 my-db.rds.amazonaws.com
  (根本不经过 kube-proxy)
```

### 2.5 Headless Service - 特殊的"无 IP" Service

```
clusterIP: None

意义：不做负载均衡，DNS 直接返回所有 Pod IP

普通 Service:
  nslookup my-svc → 10.96.0.100 (一个虚拟IP)

Headless Service:
  nslookup my-svc → 10.1.1.5, 10.1.2.7, 10.1.3.9 (所有 Pod IP)

用途：
  - StatefulSet 需要按 Pod 名称寻址
  - 客户端想自己做负载均衡（比如 gRPC 客户端）
  - 想要所有实例的列表而非一个入口
```

---

## 3. Endpoints / EndpointSlice - Service 的"后端列表"

### 3.1 Endpoints 的角色

```
Service 只是定义了"入口"，Endpoints 记录"后端是谁"。

apiVersion: v1
kind: Endpoints
metadata:
  name: my-svc          # ← 与 Service 同名
subsets:
  - addresses:
      - ip: 10.1.1.5
        targetRef:
          kind: Pod
          name: web-xxx
      - ip: 10.1.2.7
      - ip: 10.1.3.9
    ports:
      - port: 8080

谁维护 Endpoints？
  Endpoint Controller：
    - Watch Service 和 Pod
    - Pod 变化 → 自动更新 Endpoints
    - 只把 Ready 的 Pod 放进去（readinessProbe 通过）
```

### 3.2 EndpointSlice - 大规模下的优化

```
Endpoints 的问题：
  一个 Service 如果有 5000 个 Pod，所有 Pod 信息放在一个对象里。
  任何 Pod 变动 → 整个对象重写 → 广播到所有 Node → 网络风暴

EndpointSlice（K8s 1.17+ 默认）：
  把 Endpoints 切成多个小片（每片 ≤ 100 个）
  变动只影响一个片，大幅降低通知开销

Service (my-svc)
  ├─ EndpointSlice my-svc-abc (Pod 1-100)
  ├─ EndpointSlice my-svc-def (Pod 101-200)
  └─ EndpointSlice my-svc-ghi (Pod 201-300)
```

---

## 4. DNS - 服务发现的核心

### 4.1 CoreDNS - 集群内置 DNS

```
每个 Service 创建后，CoreDNS 自动注册 DNS 记录：

Service my-svc 在 namespace default
  ↓
DNS 记录：
  my-svc.default.svc.cluster.local  →  10.96.0.100

缩写规则（Pod 内的 /etc/resolv.conf 配置了 search 域）：
  my-svc                                        ← 同 namespace 可省略
  my-svc.default                                ← 跨 namespace 带上
  my-svc.default.svc.cluster.local              ← 全称
```

### 4.2 Pod DNS 配置原理

```
Pod 启动时，kubelet 注入 /etc/resolv.conf：

  nameserver 10.96.0.10               ← CoreDNS 的 ClusterIP
  search default.svc.cluster.local \  ← 搜索域
         svc.cluster.local \
         cluster.local
  options ndots:5                     ← . 数 < 5 时，先拼接搜索域

所以 curl my-svc 的查询顺序：
  1. my-svc.default.svc.cluster.local   ← 命中！
  2. my-svc.svc.cluster.local
  3. my-svc.cluster.local
  4. my-svc (当作绝对名查外网)
```

### 4.3 StatefulSet 的 Pod DNS

```
Headless Service + StatefulSet 的组合：

  mysql-0.mysql.default.svc.cluster.local  → 10.1.1.5
  mysql-1.mysql.default.svc.cluster.local  → 10.1.2.7
  mysql-2.mysql.default.svc.cluster.local  → 10.1.3.9

这就是为什么可以在 MySQL 集群配置里写死 "mysql-0" 作为主节点。
```

---

## 5. Ingress - HTTP 层路由

### 5.1 为什么需要 Ingress？

```
只用 LoadBalancer Service 的问题：

  Service A (LoadBalancer) → 一个公网 IP
  Service B (LoadBalancer) → 一个公网 IP
  Service C (LoadBalancer) → 一个公网 IP
                             (每个都要花钱)

你其实想要：
  一个入口 → 按域名/路径分发到不同 Service

  api.example.com     ──▶ api-service
  web.example.com     ──▶ web-service
  example.com/admin   ──▶ admin-service
```

### 5.2 Ingress 的两个组成部分

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   Ingress 资源对象  =  路由规则（声明性配置）              │
│                                                          │
│   Ingress Controller =  真正执行路由的进程                │
│                        （Nginx / Traefik / HAProxy...）   │
│                                                          │
│   关键：K8s 本身不内置 Ingress Controller，             │
│         需要自己安装（比如 ingress-nginx）                │
└──────────────────────────────────────────────────────────┘
```

### 5.3 Ingress YAML 示例

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
      secretName: example-tls        # 证书放在 Secret 里
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
    - host: example.com
      http:
        paths:
          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

### 5.4 流量走向

```
外部请求 (api.example.com)
      │
      ▼
┌─────────────────┐
│  Cloud LB       │  ← LoadBalancer Service 暴露 Ingress Controller
└────────┬────────┘
         ▼
┌─────────────────┐
│ Ingress         │  ← 读 Ingress 对象，生成 nginx.conf
│ Controller      │     按 host/path 分发
│ (nginx pod)     │
└────────┬────────┘
         ▼
┌─────────────────┐
│  Service        │  ← ClusterIP，做 L4 负载均衡
│  (api-service)  │
└────────┬────────┘
         ▼
     [Pods]
```

### 5.5 Ingress 不够用怎么办？

```
Ingress 的局限：
  - 只管 HTTP/HTTPS
  - 高级功能（重试、限流、mTLS、金丝雀）靠 annotation，不标准
  - 不同 Controller 的能力参差不齐

解决方案：
  1. Gateway API（新一代标准，K8s 官方力推）
  2. Service Mesh（Istio / Linkerd）
```

---

## 6. Gateway API - Ingress 的继任者

### 6.1 为什么 Ingress 要被替代？

```
Ingress 的设计问题：
  - 配置能力弱，各家 Controller 靠 annotation 扩展 → 不可移植
  - 缺乏角色分离（平台 vs 开发者应该关注不同的对象）
  - 不支持非 HTTP 协议（gRPC、TCP、UDP）

Gateway API 的解法：
  - 明确分层：GatewayClass / Gateway / HTTPRoute
  - 标准化高级功能（流量拆分、Header 操作等）
  - 多协议支持
```

### 6.2 资源分层

```
┌──────────────────────────────────────────────────────────┐
│ GatewayClass   平台管理员维护（"集群支持哪些类型的网关"）   │
│     ↑                                                    │
│     │                                                    │
│ Gateway        基础设施工程师维护（一个具体的网关实例）     │
│     ↑                                                    │
│     │                                                    │
│ HTTPRoute      应用开发者维护（路由规则）                  │
│ TCPRoute                                                 │
│ GRPCRoute                                                │
└──────────────────────────────────────────────────────────┘
```

---

## 7. NetworkPolicy - Pod 级防火墙

### 7.1 默认网络模型的问题

```
K8s 默认：任何 Pod 都可以访问任何 Pod
         ↓
         生产环境的安全风险：
           前端 Pod 不应该能直接访问数据库
           开发环境 Pod 不应该能访问生产环境 Pod
```

### 7.2 NetworkPolicy 的作用

```
NetworkPolicy = 基于 Label 的 L3/L4 防火墙规则

┌───────────────────────────────────────────────────────┐
│   规则以 Pod 为主语，定义：                             │
│                                                       │
│   - ingress (入站流量)：谁可以访问我                   │
│   - egress  (出站流量)：我可以访问谁                   │
│                                                       │
│   匹配维度：                                           │
│   - podSelector     (按 label)                        │
│   - namespaceSelector (按 namespace label)            │
│   - ipBlock         (按 CIDR)                         │
└───────────────────────────────────────────────────────┘
```

### 7.3 YAML 示例

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: database      # 这个策略保护 app=database 的 Pod
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend    # 只允许 app=backend 访问
      ports:
        - protocol: TCP
          port: 5432
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53       # 允许 DNS 查询
```

### 7.4 重要提醒

```
⚠️ NetworkPolicy 需要 CNI 插件支持才能生效！

  支持：Calico / Cilium / Weave
  不支持：Flannel (默认配置下)

⚠️ 一旦给 Pod 定义了 NetworkPolicy，就是"白名单"模式
  没写明允许的流量都会被拒绝。
```

### 7.5 生产实践：默认拒绝

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}           # 匹配 namespace 下所有 Pod
  policyTypes:
    - Ingress
    - Egress
# 空规则 = 全部拒绝
```

```
典型策略分层：
  1. 先 default-deny-all
  2. 允许 DNS 查询
  3. 按业务维度明确允许的流量
```

---

## 8. CNI - 容器网络接口

### 8.1 CNI 的定位

```
K8s 不自己实现 Pod 网络，而是定义了一个接口：CNI

Kubelet 启动 Pod 时：
  1. 创建 pause 容器（获得 network namespace）
  2. 调用 CNI 插件配置网络：
     - 分配 IP
     - 插网卡（veth pair）
     - 配路由
  3. 业务容器加入这个 namespace

不同 CNI 实现：
  - Flannel   (简单，overlay)
  - Calico    (BGP 路由，支持 NetworkPolicy)
  - Cilium    (eBPF，高性能，可观测性强)
  - AWS VPC CNI (Pod 直接用 VPC IP)
```

### 8.2 CNI 插件的两类模型

```
1. Overlay 网络 (封包)
   ─────────────────
   Pod IP 是"虚拟"的，跨 Node 用 VXLAN/IPIP 封包
   优点：简单，不依赖底层网络
   缺点：有封包开销，性能损耗

      Pod(10.1.1.5) → 封包 → Node → 解包 → Pod(10.2.1.7)

2. Underlay / 路由 网络
   ─────────────────
   Pod IP 是真实的网络地址，靠路由器/BGP 转发
   优点：性能好，没封包开销
   缺点：需要底层网络支持

      Pod → 直接走路由表 → Pod
```

---

## 9. 完整数据流：外部请求到 Pod

```
时间线：用户访问 https://api.example.com/users

1. DNS 解析
   api.example.com → Cloud LB 公网 IP

2. Cloud LB → Node
   LB 把请求转到某个 Node 的 NodePort

3. kube-proxy (iptables/IPVS)
   NodePort → Ingress Controller Service 的 ClusterIP
           → DNAT 到某个 Ingress Controller Pod

4. Ingress Controller (nginx)
   按 Host/Path 匹配到后端 Service
   api.example.com/users → user-service

5. kube-proxy 再次介入
   user-service ClusterIP → DNAT 到某个 user Pod

6. CNI 网络
   数据包通过 CNI 配置的网络到达目标 Pod

7. 业务 Pod 处理请求，返回响应（原路返回）
```

---

## 10. 总结：网络抽象的分层智慧

```
层次                     解决的问题                    关键资源
──────────────────────────────────────────────────────────────
CNI (L3 网络)            Pod IP 连通性                -
Service (L4 LB)          稳定入口 + 负载均衡          Service, Endpoints
DNS                      名字 → IP 的映射             CoreDNS
Ingress (L7 路由)        域名/路径分发                Ingress, Gateway
安全 (L3/L4 ACL)         细粒度访问控制              NetworkPolicy
```

> **核心洞见：K8s 网络的精髓是"抽象解耦"。Pod 不关心谁访问它（Service 解决）、Service 不关心怎么 HTTP 路由（Ingress 解决）、Ingress 不关心底层通信（CNI 解决）。每一层只做一件事，组合起来就是一个弹性可扩展的网络栈。**
