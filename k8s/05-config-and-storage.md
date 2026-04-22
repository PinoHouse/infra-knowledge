# 第三层：核心资源对象 (三) 配置与存储 - 深度展开

## 1. 为什么要把配置和存储抽象出来？

### 1.1 十二要素应用 (The Twelve-Factor App)

```
核心原则：配置与代码分离

反模式：
  ┌─────────────────────────────────┐
  │  镜像里写死 DATABASE_URL         │
  │  → 换环境要重新打镜像             │
  │  → 密码写在代码里，泄漏风险       │
  └─────────────────────────────────┘

K8s 的解法：
  ┌─────────────────────────────────┐
  │  镜像只包含应用代码              │
  │  配置从外部注入（ConfigMap）     │
  │  敏感信息从外部注入（Secret）    │
  │  数据持久化到外部（PV/PVC）      │
  └─────────────────────────────────┘
```

### 1.2 本章要解决的三类问题

| 问题 | 资源对象 |
|------|---------|
| 非敏感配置从哪来？ | **ConfigMap** |
| 敏感信息（密码、证书）从哪来？ | **Secret** |
| 数据如何持久化？ | **Volume / PV / PVC** |
| 存储如何按需动态供给？ | **StorageClass** |

---

## 2. ConfigMap - 非敏感配置

### 2.1 定位

```
ConfigMap = 一个 Key-Value 集合，用于存放"不敏感"的配置

典型内容：
  - 环境变量（API_URL, LOG_LEVEL）
  - 配置文件（nginx.conf, application.yml）
  - 命令行参数（--workers=4）
```

### 2.2 三种创建方式

```bash
# 1. 字面量
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=WORKERS=4

# 2. 文件
kubectl create configmap nginx-config \
  --from-file=nginx.conf

# 3. YAML 声明式
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "debug"
  WORKERS: "4"
  application.yml: |        # | 表示保留换行
    server:
      port: 8080
    logging:
      level: debug
```

### 2.3 三种使用方式

```
1. 作为环境变量（单个 key）
   ─────────────────────────
   适合少量配置项。
   缺点：变了要重启 Pod。

2. 作为环境变量（整个 ConfigMap）
   ─────────────────────────
   envFrom，把所有 key 注入成环境变量。

3. 作为文件挂载到 Volume
   ─────────────────────────
   适合配置文件整体替换。
   特点：ConfigMap 更新后，文件会自动更新（有延迟，几十秒）。
        但进程不会自动重新加载！
```

**示例 1：注入环境变量**
```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
      envFrom:
        - configMapRef:
            name: app-config       # 全量注入所有 key
```

**示例 2：挂载为文件**
```yaml
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: config-volume
      configMap:
        name: nginx-config
        items:                     # 可选：只挂载部分 key
          - key: nginx.conf
            path: default.conf
```

### 2.4 更新行为对比

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   注入方式        ConfigMap 变更后的行为                     │
│   ─────────────────────────────────────                    │
│   env (环境变量)    Pod 不感知，需手动重启                   │
│   volumeMount       文件自动更新（约 10s 延迟）              │
│                    但应用要自己监听/重启才能"吃到"新配置     │
│                                                             │
│   常用做法：                                                 │
│     给 Deployment 加 checksum annotation，                  │
│     ConfigMap 变了 → annotation 变了 → 触发滚动更新          │
└─────────────────────────────────────────────────────────────┘
```

### 2.5 容量限制

```
单个 ConfigMap ≤ 1 MiB

为什么？因为所有 ConfigMap 都存在 etcd 里，
大对象会影响整个集群性能。

太大的配置怎么办？
  - 拆分成多个 ConfigMap
  - 用外部配置中心（Nacos / Consul）
  - 用 PV 挂载
```

---

## 3. Secret - 敏感配置

### 3.1 Secret vs ConfigMap

```
结构上几乎一样，关键区别：

ConfigMap:  明文 key-value
Secret:     base64 编码的 key-value (注意：不是加密！)

  data:
    password: YWRtaW4xMjM=    ← 只是 base64，任何人都能解码
                                 echo YWRtaW4xMjM= | base64 -d = admin123
```

### 3.2 Secret 的"安全性"来自哪里？

```
Secret 本身没有加密保护机制，安全来自：

1. RBAC 权限控制
   ──────────────
   限制谁能读取 Secret

2. etcd 加密（需要开启）
   ──────────────
   配置 --encryption-provider-config
   让 Secret 在 etcd 里是加密存储

3. 外部密钥管理（推荐生产用法）
   ──────────────
   External Secrets Operator
   Vault / AWS Secrets Manager / AWS KMS
   真正的密钥不进 K8s，K8s 只拿临时凭证
```

### 3.3 Secret 的类型

```
K8s 把 Secret 分类，每类有对应的校验规则：

type                              用途
─────────────────────────────────────────────────
Opaque (默认)                     任意 key-value
kubernetes.io/service-account-token  ServiceAccount 的 token
kubernetes.io/dockerconfigjson    镜像仓库凭证
kubernetes.io/tls                 TLS 证书（tls.crt + tls.key）
kubernetes.io/basic-auth          用户名 + 密码
kubernetes.io/ssh-auth            SSH 私钥
```

### 3.4 TLS Secret 示例

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-tls
type: kubernetes.io/tls
data:
  tls.crt: <base64 of cert.pem>
  tls.key: <base64 of key.pem>
```

### 3.5 使用 Secret（与 ConfigMap 几乎一样）

```yaml
spec:
  containers:
    - name: app
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
      volumeMounts:
        - name: tls-vol
          mountPath: /etc/tls
          readOnly: true
  volumes:
    - name: tls-vol
      secret:
        secretName: example-tls
        defaultMode: 0400          # 只允许 owner 读
```

### 3.6 安全最佳实践

```
┌──────────────────────────────────────────────────────────┐
│  ✅ 开启 etcd 加密                                        │
│  ✅ 严格控制 RBAC（只有真正需要的 Pod 能读）              │
│  ✅ 用 External Secrets + Vault/AWS Secrets Manager       │
│  ✅ 优先 volumeMount 而不是环境变量                       │
│     （环境变量可能被 ps、crash dump 等泄漏）              │
│  ✅ 定期轮换                                              │
│  ❌ 不要把 Secret 提交到 Git                              │
│  ❌ 不要用明文 Secret（用 Sealed Secret 或 SOPS 加密后提交）│
└──────────────────────────────────────────────────────────┘
```

---

## 4. Volume - Pod 内的数据共享与持久化

### 4.1 为什么 K8s 的 Volume 不等于 Docker 的 Volume？

```
Docker Volume:
  - 绑定到容器
  - 容器删除 = Volume 是否删除看配置

K8s Volume:
  - 绑定到 Pod，容器之间共享
  - Pod 生命周期 = Volume 生命周期（除非用 PV）

┌──────────────────────────────────────────────────────┐
│                       Pod                            │
│   ┌────────────┐         ┌────────────┐              │
│   │ Container A│         │ Container B│              │
│   │            │         │            │              │
│   │  /data ────┼─┬───────┼─── /input  │              │
│   └────────────┘ │       └────────────┘              │
│                  ▼                                   │
│             ┌─────────┐                              │
│             │ Volume  │   ← Pod 级别，两个容器共享   │
│             └─────────┘                              │
└──────────────────────────────────────────────────────┘
```

### 4.2 常见 Volume 类型（按生命周期分）

```
生命周期与 Pod 绑定（Pod 消失数据也消失）：
  - emptyDir       Pod 内临时存储，两个容器共享
  - configMap      挂载 ConfigMap 为文件
  - secret         挂载 Secret 为文件
  - downwardAPI    挂载 Pod 自身的元数据

生命周期与节点绑定：
  - hostPath       挂 Node 的文件/目录（只建议给系统 Pod 用）

生命周期独立（数据持久化）：
  - persistentVolumeClaim (PVC)  ← 重点！
  - csi                          (各云厂商/存储厂商实现)
```

### 4.3 emptyDir - 最简单的共享卷

```yaml
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "while true; do date >> /data/log; sleep 5; done"]
      volumeMounts:
        - name: cache
          mountPath: /data
    - name: reader
      image: busybox
      command: ["sh", "-c", "tail -f /data/log"]
      volumeMounts:
        - name: cache
          mountPath: /data
  volumes:
    - name: cache
      emptyDir: {}
```

```
特点：
  - Pod 创建时创建，Pod 删除时清空
  - 默认在 Node 磁盘上
  - 可指定 medium: Memory 使用 tmpfs（内存）
```

---

## 5. PV / PVC - 持久化存储抽象

### 5.1 为什么要"两层"？职责分离

```
痛点：开发者不关心"这块盘是 EBS 还是 NFS"，他们只要"100GB 可读写"

解决方案：解耦"提供"与"消费"

  ┌──────────────────────────────────────────────────────┐
  │   管理员视角                                          │
  │                                                      │
  │     PV (PersistentVolume)                            │
  │     "我这有一块 100GB 的 EBS，谁要用？"               │
  │                                                      │
  └──────────────────────────────────────────────────────┘
                            ▲
                            │ 绑定 (Binding)
                            ▼
  ┌──────────────────────────────────────────────────────┐
  │   开发者视角                                          │
  │                                                      │
  │     PVC (PersistentVolumeClaim)                      │
  │     "我要一块 100GB 可读写的存储"                     │
  │                                                      │
  └──────────────────────────────────────────────────────┘
```

### 5.2 静态供给 vs 动态供给

```
静态供给（古老方式）：
  1. 管理员手动创建一批 PV
  2. 用户创建 PVC 去"认领"匹配的 PV

动态供给（现代方式）：
  1. 用户创建 PVC，指定 StorageClass
  2. StorageClass 自动创建一个 PV（调用云厂商 API 建 EBS 等）
  3. PVC 自动绑定到新 PV
```

### 5.3 完整流程（动态供给）

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  1. 集群预先定义好 StorageClass                               │
│     apiVersion: storage.k8s.io/v1                            │
│     kind: StorageClass                                       │
│     metadata:                                                │
│       name: fast-ssd                                         │
│     provisioner: ebs.csi.aws.com                             │
│     parameters:                                              │
│       type: gp3                                              │
│                                                              │
│  2. 用户声明需求                                              │
│     apiVersion: v1                                           │
│     kind: PersistentVolumeClaim                              │
│     metadata:                                                │
│       name: data-claim                                       │
│     spec:                                                    │
│       accessModes: [ReadWriteOnce]                           │
│       storageClassName: fast-ssd                             │
│       resources:                                             │
│         requests:                                            │
│           storage: 100Gi                                     │
│                                                              │
│  3. CSI Controller 看到新 PVC                                │
│     调用 AWS API 创建 EBS 卷                                 │
│     创建 PV 并绑定 PVC                                       │
│                                                              │
│  4. Pod 挂载 PVC                                             │
│     spec:                                                    │
│       containers:                                            │
│         - name: app                                          │
│           volumeMounts:                                      │
│             - name: data                                     │
│               mountPath: /data                               │
│       volumes:                                               │
│         - name: data                                         │
│           persistentVolumeClaim:                             │
│             claimName: data-claim                            │
│                                                              │
│  5. Kubelet 调用 CSI 把 EBS 挂到 Node，再 bind mount 到容器   │
└──────────────────────────────────────────────────────────────┘
```

### 5.4 访问模式 (AccessModes)

```
┌───────────────┬───────────────────────────────────────────┐
│ Mode          │ 含义                                      │
├───────────────┼───────────────────────────────────────────┤
│ ReadWriteOnce │ 一个 Node 读写（EBS、大多数块存储）        │
│ ReadWriteMany │ 多 Node 同时读写（NFS、CephFS、EFS）       │
│ ReadOnlyMany  │ 多 Node 只读                              │
│ ReadWriteOncePod │ 单 Pod 读写（更严格，K8s 1.22+）        │
└───────────────┴───────────────────────────────────────────┘

⚠️ 选错访问模式是生产事故高发点：
   - 块存储（EBS）只能 RWO → 想给多个 Pod 共享会失败
   - 需要 RWX → 只能用 NFS/EFS/CephFS 类文件存储
```

### 5.5 回收策略 (ReclaimPolicy)

```
PV 上的 persistentVolumeReclaimPolicy：

Retain (保留)
  PVC 删除后，PV 和数据保留（需手动清理）
  ← 生产环境推荐，防误删

Delete
  PVC 删除后，PV 和底层存储都删除
  ← 测试环境方便

Recycle (已废弃)
  简单清空后复用
```

---

## 6. StorageClass - 动态供给的"模板"

### 6.1 StorageClass 的作用

```
StorageClass 定义：
  - 用哪个 provisioner（CSI 驱动）
  - 底层存储的参数（磁盘类型、副本数、加密等）
  - 回收策略、volumeBindingMode 等

一个集群通常有多个 StorageClass：
  - fast-ssd     (生产用，gp3)
  - slow-hdd     (归档用，st1)
  - shared-nfs   (需要 RWX 时用)
```

### 6.2 关键参数：volumeBindingMode

```
Immediate (立即绑定)：
  创建 PVC → 立即创建 PV → 立即绑定
  问题：Pod 还没调度，PV 可能建在和 Pod 不同的可用区！

WaitForFirstConsumer (等首个消费者)：
  创建 PVC → 等到 Pod 调度时 → 在 Pod 所在可用区创建 PV
  ← 推荐！特别是跨可用区集群
```

### 6.3 默认 StorageClass

```bash
kubectl patch storageclass fast-ssd \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

```
PVC 没指定 storageClassName 时，自动使用默认 StorageClass。
```

---

## 7. CSI - 容器存储接口

### 7.1 CSI 的定位

```
类似 CNI 之于网络、CRI 之于容器运行时：
  CSI = Container Storage Interface

让 K8s 核心代码不需要内置各家存储厂商的逻辑，
外部实现标准接口即可集成。

┌──────────────────────────────────────────────────────┐
│                 K8s Core                             │
│                     │                                │
│                   CSI API                            │
│                     │                                │
│     ┌───────────────┼───────────────┐                │
│     ▼               ▼               ▼                │
│  AWS EBS CSI     Ceph CSI       Azure Disk CSI       │
│  (独立进程)       (独立进程)     (独立进程)          │
└──────────────────────────────────────────────────────┘
```

### 7.2 CSI 插件的两个组件

```
1. Controller 组件（以 Deployment 跑在集群里一份）
   ────────────────────────────────────────
   负责"创建/删除/扩容"底层存储
   调用云厂商 API

2. Node 组件（以 DaemonSet 跑在每个 Node 上）
   ────────────────────────────────────────
   负责"挂载/卸载"存储到本机
```

---

## 8. 生产实践要点

### 8.1 ConfigMap / Secret

```
✅ 用 Kustomize 的 configMapGenerator 自动加 hash 后缀
   → 改配置自动触发 Pod 滚动

✅ 大型配置用外部配置中心，ConfigMap 只放启动参数

✅ Secret 用 External Secrets Operator 从 Vault/AWS SM 同步

✅ 敏感配置文件挂载时用 defaultMode: 0400

❌ 不要在镜像里写死任何环境相关的配置
```

### 8.2 存储

```
✅ StorageClass 用 WaitForFirstConsumer
✅ 生产环境 ReclaimPolicy: Retain
✅ StatefulSet 搭配 volumeClaimTemplates，一 Pod 一盘
✅ 监控 PV 使用率，避免写满
✅ 定期备份（Velero / 云厂商快照）

❌ 不要用 hostPath 做生产存储（Pod 换节点就丢）
❌ 不要用 emptyDir 存"不能丢"的数据
❌ 不要把数据库放在 Deployment + PVC 上（用 StatefulSet）
```

---

## 9. 总结：配置与存储的抽象分层

```
层次                   职责                     关键资源
────────────────────────────────────────────────────────────
配置抽象              注入而非硬编码            ConfigMap, Secret
卷抽象                Pod 内共享/临时存储       Volume (emptyDir...)
持久化抽象            解耦"需求"与"供给"       PVC / PV
动态供给              按需自动创建存储          StorageClass
存储插件接口          解耦 K8s 与存储厂商       CSI
```

> **核心洞见：配置和存储的设计哲学，本质都是"解耦"。ConfigMap/Secret 把配置从代码解耦，PVC 把存储需求从具体实现解耦。这让你可以把同一份 YAML 跑在任何集群、任何云上。**
