# 第六层：工作负载身份 (Workload Identity) - 深度展开

## 1. 从第一性原理看工作负载身份

### 1.1 核心问题

```
运行在云上的应用如何安全地获取访问其他云资源的权限？

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  你的应用                          云资源               │
│                                                         │
│  ┌─────────────┐                ┌─────────────────┐    │
│  │             │                │  S3 Bucket      │    │
│  │   App       │ ─────────────► │  RDS Database   │    │
│  │   代码      │   需要凭证      │  SQS Queue      │    │
│  │             │                │  Secrets Manager│    │
│  └─────────────┘                └─────────────────┘    │
│                                                         │
│  问题：这个凭证从哪来？怎么安全地给到应用？             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 1.2 工作负载身份的本质

```
工作负载身份 = 让应用程序拥有自己的"身份证"

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  传统思维：                                             │
│    应用需要"借用"某个人或某个账号的凭证                 │
│    → 凭证是"外来的"，需要配置、存储、管理               │
│                                                         │
│  工作负载身份思维：                                     │
│    应用本身就有身份，云平台认识它                       │
│    → 身份是"内生的"，自动获取、自动轮换                │
│                                                         │
│  类比：                                                 │
│    传统 = 员工用临时访客卡进入大楼                      │
│    工作负载身份 = 员工有自己的工牌，刷卡自动识别        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 凭证管理的演进

### 2.1 阶段一：硬编码密钥

```python
# ❌ 最危险的方式：直接写在代码里

import boto3

s3 = boto3.client(
    's3',
    aws_access_key_id='AKIAIOSFODNN7EXAMPLE',        # 硬编码
    aws_secret_access_key='wJalrXUtnFEMI/K7MDENG/...' # 硬编码
)
```

**问题：**

```
┌─────────────────────────────────────────────────────────┐
│  1. 代码提交到 Git                                      │
│     → 密钥永久存在于 Git 历史                           │
│     → 即使删除也能从历史恢复                            │
│                                                         │
│  2. 被爬虫扫描                                          │
│     → GitHub 上有机器人专门扫描泄露的 AWS Key           │
│     → 几分钟内就会被发现并滥用                          │
│                                                         │
│  3. 无法轮换                                            │
│     → 要换 Key 就要改代码重新部署                       │
│     → 大家都懒得换，一个 Key 用好几年                   │
│                                                         │
│  4. 权限失控                                            │
│     → 谁有代码谁就有 Key                                │
│     → 离职员工可能还保留着代码副本                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 2.2 阶段二：环境变量/配置文件

```python
# 稍好一点：从环境变量读取

import os
import boto3

s3 = boto3.client(
    's3',
    aws_access_key_id=os.environ['AWS_ACCESS_KEY_ID'],
    aws_secret_access_key=os.environ['AWS_SECRET_ACCESS_KEY']
)
```

```yaml
# 或者从配置文件读取
# config.yaml (不提交到 Git)
aws:
  access_key_id: AKIAIOSFODNN7EXAMPLE
  secret_access_key: wJalrXUtnFEMI/K7MDENG/...
```

**改进：**
- 密钥不在代码中
- 可以在不改代码的情况下更换密钥

**仍有问题：**

```
┌─────────────────────────────────────────────────────────┐
│  1. 仍需管理密钥                                        │
│     → 密钥存在哪里？谁有权限访问？                      │
│     → Kubernetes Secret? HashiCorp Vault? AWS Secrets? │
│                                                         │
│  2. 仍有泄露风险                                        │
│     → 环境变量可能被日志打印                            │
│     → 配置文件可能被误提交                              │
│     → 内存中的密钥可能被 dump                           │
│                                                         │
│  3. 仍是长期凭证                                        │
│     → 泄露后在轮换前一直有效                            │
│     → 轮换是手动过程，容易遗忘                          │
│                                                         │
│  4. 难以最小权限                                        │
│     → 一个 Key 可能被多个应用共享                       │
│     → Key 的权限是所有应用需求的并集                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 2.3 阶段三：工作负载身份

```python
# ✅ 最佳实践：什么都不用配置

import boto3

# AWS SDK 自动从环境获取凭证
# - EC2: 从 Instance Metadata Service
# - Lambda: 从环境变量（自动注入）
# - EKS: 从 IRSA (projected service account token)

s3 = boto3.client('s3')  # 就这样，不需要任何凭证配置
s3.list_buckets()
```

**优势：**

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. 无需管理密钥                                        │
│     → 没有密钥，就没有泄露风险                          │
│     → 不用担心存在哪、谁能访问                          │
│                                                         │
│  2. 自动轮换                                            │
│     → 临时凭证自动刷新                                  │
│     → 每次都是新的，无需手动轮换                        │
│                                                         │
│  3. 最小权限                                            │
│     → 每个工作负载绑定自己的 Role                       │
│     → Role 权限精确匹配工作负载需求                     │
│                                                         │
│  4. 可审计                                              │
│     → CloudTrail 记录哪个工作负载做了什么               │
│     → 可以精确到 Pod 级别                               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 3. AWS 工作负载身份实现

### 3.1 EC2 Instance Profile

```
┌─────────────────────────────────────────────────────────────────┐
│                         EC2 Instance                            │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                      应用程序                              │ │
│  │                                                            │ │
│  │  import boto3                                             │ │
│  │  s3 = boto3.client('s3')  # SDK 自动获取凭证              │ │
│  │                                                            │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              │ HTTP 请求                        │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │           Instance Metadata Service (IMDS)                │ │
│  │                                                            │ │
│  │  http://169.254.169.254/latest/meta-data/                 │ │
│  │    iam/security-credentials/MyRole                        │ │
│  │                                                            │ │
│  │  返回：                                                    │ │
│  │  {                                                         │ │
│  │    "AccessKeyId": "ASIA...",                              │ │
│  │    "SecretAccessKey": "...",                              │ │
│  │    "Token": "...",                                        │ │
│  │    "Expiration": "2024-01-15T12:00:00Z"                  │ │
│  │  }                                                         │ │
│  │                                                            │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                              │                                  │
└──────────────────────────────│──────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Instance Profile  │
                    │         │           │
                    │         ▼           │
                    │     IAM Role        │
                    │   (附加权限策略)     │
                    └─────────────────────┘
```

**配置步骤：**

```bash
# 1. 创建 IAM Role（Trust Policy 允许 EC2 服务）
aws iam create-role --role-name MyEC2Role --assume-role-policy-document '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}'

# 2. 附加权限策略
aws iam attach-role-policy \
    --role-name MyEC2Role \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 3. 创建 Instance Profile
aws iam create-instance-profile --instance-profile-name MyEC2Profile
aws iam add-role-to-instance-profile \
    --instance-profile-name MyEC2Profile \
    --role-name MyEC2Role

# 4. 启动 EC2 时指定 Instance Profile
aws ec2 run-instances \
    --image-id ami-xxx \
    --instance-type t3.micro \
    --iam-instance-profile Name=MyEC2Profile
```

**IMDSv2 安全加固：**

```bash
# 强制使用 IMDSv2（需要 Token）
aws ec2 modify-instance-metadata-options \
    --instance-id i-xxx \
    --http-tokens required \
    --http-endpoint enabled

# 应用程序获取凭证需要先获取 Token
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/iam/security-credentials/MyRole

# AWS SDK 自动处理 IMDSv2，应用代码无需改动
```

### 3.2 Lambda Execution Role

```
┌─────────────────────────────────────────────────────────────────┐
│                       Lambda Function                           │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                      Handler 代码                          │ │
│  │                                                            │ │
│  │  def handler(event, context):                             │ │
│  │      s3 = boto3.client('s3')  # 凭证自动可用              │ │
│  │      s3.get_object(Bucket='...', Key='...')               │ │
│  │                                                            │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  环境变量（Lambda 运行时自动注入）：                            │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  AWS_ACCESS_KEY_ID=ASIA...                                │ │
│  │  AWS_SECRET_ACCESS_KEY=...                                │ │
│  │  AWS_SESSION_TOKEN=...                                    │ │
│  │  AWS_REGION=us-east-1                                     │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Execution Role    │
                    │                     │
                    │  Trust Policy:      │
                    │  lambda.amazonaws.  │
                    │    com              │
                    │                     │
                    │  Permission Policy: │
                    │  (你定义的权限)      │
                    └─────────────────────┘
```

**特点：**
- Lambda 运行时自动注入临时凭证到环境变量
- 每次调用都会刷新（如果接近过期）
- 开发者完全透明，无需任何配置

### 3.3 EKS IRSA (IAM Roles for Service Accounts)

这是最复杂也最强大的工作负载身份方案，实现 **Pod 级别**的权限隔离。

```
┌──────────────────────────────────────────────────────────────────┐
│                        EKS Cluster                               │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Namespace: production                                      │ │
│  │                                                             │ │
│  │  ┌─────────────────┐      ┌─────────────────┐              │ │
│  │  │   Pod: app-a    │      │   Pod: app-b    │              │ │
│  │  │                 │      │                 │              │ │
│  │  │ ServiceAccount: │      │ ServiceAccount: │              │ │
│  │  │   sa-app-a      │      │   sa-app-b      │              │ │
│  │  │       │         │      │       │         │              │ │
│  │  │       ▼         │      │       ▼         │              │ │
│  │  │ IAM Role:       │      │ IAM Role:       │              │ │
│  │  │ role-app-a      │      │ role-app-b      │              │ │
│  │  │ (只能访问S3)    │      │ (只能访问DynamoDB)│             │ │
│  │  └─────────────────┘      └─────────────────┘              │ │
│  │                                                             │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  关键：同一个 Node 上的不同 Pod 有不同的 IAM 权限               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**IRSA 工作原理：**

```
┌─────────────────────────────────────────────────────────────────┐
│                          Pod 内部                               │
│                                                                 │
│  1. Kubernetes 自动挂载 Projected ServiceAccount Token         │
│                                                                 │
│     /var/run/secrets/eks.amazonaws.com/serviceaccount/token    │
│     (这是一个 JWT，由 K8s 签发，包含 ServiceAccount 信息)       │
│                                                                 │
│  2. 环境变量自动注入                                            │
│                                                                 │
│     AWS_ROLE_ARN=arn:aws:iam::123:role/my-role                │
│     AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/.../token    │
│                                                                 │
│  3. AWS SDK 检测到这些环境变量                                  │
│     自动调用 AssumeRoleWithWebIdentity                         │
│     用 K8s Token 换取 AWS 临时凭证                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                               │
                               │ AssumeRoleWithWebIdentity
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                         AWS STS                                 │
│                                                                 │
│  1. 验证 JWT 签名                                               │
│     → 用 EKS OIDC Provider 的公钥验证                          │
│                                                                 │
│  2. 检查 Trust Policy                                          │
│     → JWT 的 sub 是否匹配？                                    │
│     → system:serviceaccount:namespace:sa-name                  │
│                                                                 │
│  3. 返回临时凭证                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**完整配置流程：**

```bash
# 1. 获取 EKS 集群的 OIDC Provider URL
OIDC_PROVIDER=$(aws eks describe-cluster --name my-cluster \
    --query "cluster.identity.oidc.issuer" --output text | sed 's|https://||')

# 2. 创建 IAM OIDC Identity Provider（如果还没有）
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# 3. 创建 IAM Role 和 Trust Policy
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:production:my-app-sa",
          "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

aws iam create-role --role-name my-app-role \
    --assume-role-policy-document file://trust-policy.json

# 4. 附加权限策略
aws iam attach-role-policy --role-name my-app-role \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 5. 创建 Kubernetes ServiceAccount
kubectl create serviceaccount my-app-sa -n production

kubectl annotate serviceaccount my-app-sa -n production \
    eks.amazonaws.com/role-arn=arn:aws:iam::${ACCOUNT_ID}:role/my-app-role
```

```yaml
# 6. 在 Deployment 中使用 ServiceAccount
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: my-app-sa  # 使用带 IAM 角色的 SA
      containers:
        - name: app
          image: my-app:latest
          # 无需任何 AWS 凭证配置！
```

### 3.4 ECS Task Role

```
┌─────────────────────────────────────────────────────────────────┐
│                         ECS Task                                │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                     Container                              │ │
│  │                                                            │ │
│  │  应用程序使用 AWS SDK                                      │ │
│  │  SDK 自动从 Task Metadata Endpoint 获取凭证               │ │
│  │                                                            │ │
│  │  http://169.254.170.2/v2/credentials/{id}                 │ │
│  │                                                            │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  AWS_CONTAINER_CREDENTIALS_RELATIVE_URI=/v2/credentials/xxx   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │     Task Role       │
                    │                     │
                    │  每个 Task 可以有   │
                    │  独立的 IAM Role    │
                    └─────────────────────┘
```

---

## 4. GCP 工作负载身份

### 4.1 GCE Service Account

```
┌─────────────────────────────────────────────────────────────────┐
│                        GCE Instance                             │
│                                                                 │
│  应用程序                                                        │
│      │                                                           │
│      │ 使用 Google Cloud SDK                                    │
│      ▼                                                           │
│  Metadata Server                                                │
│  http://metadata.google.internal/computeMetadata/v1/            │
│    instance/service-accounts/default/token                      │
│                                                                 │
│  返回 Access Token                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Service Account   │
                    │   xxx@project.iam.  │
                    │   gserviceaccount.  │
                    │   com               │
                    └─────────────────────┘
```

### 4.2 GKE Workload Identity

```
类似 EKS IRSA，实现 K8s ServiceAccount 到 GCP Service Account 的映射

┌──────────────────────────────────────────────────────────────────┐
│                        GKE Cluster                               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  K8s ServiceAccount                                          ││
│  │  metadata:                                                   ││
│  │    annotations:                                              ││
│  │      iam.gke.io/gcp-service-account: my-sa@project.iam...   ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│                              │ Workload Identity                 │
│                              ▼                                   │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  GCP Service Account│
                    │  (拥有 GCP 资源权限) │
                    └─────────────────────┘
```

**配置示例：**

```bash
# 1. 启用 Workload Identity
gcloud container clusters update my-cluster \
    --workload-pool=my-project.svc.id.goog

# 2. 创建 GCP Service Account
gcloud iam service-accounts create my-app-sa

# 3. 授予权限
gcloud projects add-iam-policy-binding my-project \
    --member="serviceAccount:my-app-sa@my-project.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"

# 4. 绑定 K8s SA 到 GCP SA
gcloud iam service-accounts add-iam-policy-binding \
    my-app-sa@my-project.iam.gserviceaccount.com \
    --role="roles/iam.workloadIdentityUser" \
    --member="serviceAccount:my-project.svc.id.goog[production/my-k8s-sa]"

# 5. 注解 K8s ServiceAccount
kubectl annotate serviceaccount my-k8s-sa \
    --namespace production \
    iam.gke.io/gcp-service-account=my-app-sa@my-project.iam.gserviceaccount.com
```

---

## 5. Azure 工作负载身份

### 5.1 Managed Identity

```
Azure 提供两种 Managed Identity：

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  System-assigned Managed Identity                              │
│  ─────────────────────────────────                              │
│  - 与资源（VM、App Service）生命周期绑定                        │
│  - 资源删除时身份也删除                                         │
│  - 一个资源一个身份                                             │
│                                                                 │
│  User-assigned Managed Identity                                │
│  ─────────────────────────────────                              │
│  - 独立创建和管理                                               │
│  - 可以分配给多个资源                                           │
│  - 资源删除后身份仍存在                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 AKS Workload Identity

```
类似 EKS IRSA 和 GKE Workload Identity

Pod → K8s ServiceAccount → Federated Credential → Managed Identity → Azure 资源
```

---

## 6. 工作负载身份最佳实践

### 6.1 权限最小化

```yaml
# ❌ 错误：一个 Role 给所有应用使用
# 结果：每个应用都有所有权限的并集

apiVersion: v1
kind: ServiceAccount
metadata:
  name: shared-sa  # 所有应用共用
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/super-role  # 权限很大

---
# ✅ 正确：每个应用独立的 ServiceAccount 和 Role

apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-a-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/app-a-role  # 只有 S3 读权限
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-b-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/app-b-role  # 只有 DynamoDB 权限
```

### 6.2 Trust Policy 精确匹配

```json
// ❌ 危险：允许整个集群的任何 ServiceAccount
{
  "Condition": {
    "StringLike": {
      "oidc.eks....:sub": "system:serviceaccount:*:*"
    }
  }
}

// ✅ 正确：精确匹配 namespace 和 ServiceAccount
{
  "Condition": {
    "StringEquals": {
      "oidc.eks....:sub": "system:serviceaccount:production:my-app-sa"
    }
  }
}
```

### 6.3 凭证刷新处理

```python
# AWS SDK 自动处理凭证刷新，但有些场景需要注意

# 场景：长时间运行的连接（如数据库连接池）
# 问题：连接创建时的凭证可能过期

# 解决方案1：使用支持凭证刷新的库
import boto3
from botocore.credentials import RefreshableCredentials

# 解决方案2：定期重建连接
# 解决方案3：使用 IAM 数据库认证（RDS）

import boto3

rds = boto3.client('rds')
token = rds.generate_db_auth_token(
    DBHostname='mydb.xxx.rds.amazonaws.com',
    Port=3306,
    DBUsername='myuser'
)
# token 有效期 15 分钟，需要定期刷新
```

### 6.4 调试工作负载身份

```bash
# EKS IRSA 调试

# 1. 检查 ServiceAccount 注解
kubectl get sa my-app-sa -n production -o yaml

# 2. 检查 Pod 中的环境变量
kubectl exec -it my-pod -n production -- env | grep AWS

# 3. 检查 Token 文件是否存在
kubectl exec -it my-pod -n production -- \
    cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token

# 4. 手动测试 AssumeRoleWithWebIdentity
kubectl exec -it my-pod -n production -- bash -c '
  aws sts assume-role-with-web-identity \
    --role-arn $AWS_ROLE_ARN \
    --web-identity-token file://$AWS_WEB_IDENTITY_TOKEN_FILE \
    --role-session-name test-session
'

# 5. 检查 Trust Policy
aws iam get-role --role-name my-app-role \
    --query 'Role.AssumeRolePolicyDocument'
```

---

## 7. 迁移指南：从 Access Key 到工作负载身份

### 7.1 迁移步骤

```
┌─────────────────────────────────────────────────────────────────┐
│                        迁移路线图                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  阶段1：评估                                                    │
│  ─────────                                                      │
│  - 列出所有使用 Access Key 的应用                              │
│  - 确定每个应用运行在什么环境（EC2、Lambda、K8s...）           │
│  - 分析每个应用需要什么 AWS 权限                               │
│                                                                 │
│  阶段2：准备                                                    │
│  ─────────                                                      │
│  - 为每个应用创建专用 IAM Role                                 │
│  - 配置最小权限的 Permission Policy                            │
│  - 配置正确的 Trust Policy                                     │
│                                                                 │
│  阶段3：并行测试                                                │
│  ─────────                                                      │
│  - 在测试环境验证工作负载身份                                  │
│  - 确保应用功能正常                                            │
│  - 检查 CloudTrail 日志确认权限足够                            │
│                                                                 │
│  阶段4：灰度切换                                                │
│  ─────────                                                      │
│  - 先切换非关键应用                                            │
│  - 监控是否有问题                                               │
│  - 逐步扩大范围                                                 │
│                                                                 │
│  阶段5：清理                                                    │
│  ─────────                                                      │
│  - 删除不再使用的 Access Key                                   │
│  - 更新文档和 runbook                                          │
│  - 设置告警：检测是否有新的 Access Key 创建                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 常见迁移场景

```
场景1：EC2 上的应用
──────────────────
Before: 应用从配置文件读取 Access Key
After:  附加 Instance Profile，删除配置中的 Key

场景2：Lambda 函数
──────────────────
Before: 使用环境变量传入 Access Key
After:  配置 Execution Role（Lambda 本就支持）

场景3：EKS 中的应用
──────────────────
Before: K8s Secret 存储 Access Key
After:  配置 IRSA，删除 Secret

场景4：CI/CD 流水线
──────────────────
Before: GitHub Secrets 存储 Access Key
After:  配置 OIDC Federation，删除 Secrets
```

---

## 8. 总结

```
┌─────────────────────────────────────────────────────────────────┐
│                   工作负载身份核心概念                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  演进路径：                                                     │
│    硬编码密钥 → 环境变量 → 工作负载身份                        │
│    (最危险)     (稍好)     (最佳实践)                          │
│                                                                 │
│  核心优势：                                                     │
│    无需管理密钥  ─  没有密钥就没有泄露                         │
│    自动轮换      ─  临时凭证自动刷新                           │
│    最小权限      ─  每个工作负载独立权限                       │
│    可审计        ─  精确追踪到 Pod 级别                        │
│                                                                 │
│  各云实现：                                                     │
│    AWS   ─  Instance Profile / IRSA / Task Role               │
│    GCP   ─  Service Account / Workload Identity               │
│    Azure ─  Managed Identity / AKS Workload Identity          │
│                                                                 │
│  最佳实践：                                                     │
│    每个应用独立身份  ─  不共享 ServiceAccount                  │
│    Trust Policy 精确  ─  不用通配符                            │
│    定期审查权限       ─  删除不需要的权限                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **核心洞见：工作负载身份的本质是让应用"天生"就有身份，而不是"借用"别人的凭证。这不仅更安全，而且更简单——最好的安全方案往往也是最简单的方案。**

---

## 思考题

1. 为什么 IRSA 比使用 Node 的 Instance Profile 更安全？
2. 如果一个 Pod 被攻破，攻击者能获取什么权限？如何最小化影响？
3. 设计一个方案：让开发环境的应用只能访问开发环境的资源，即使代码部署错误也不能访问生产资源。
