# 第五层：联邦身份 (Federated Identity) - 深度展开

## 1. 从第一性原理看联邦身份

### 1.1 核心问题

```
在现实世界中，身份是分散的：

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  企业环境中的身份：                                     │
│                                                         │
│    ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│    │ 企业 AD  │  │  GitHub  │  │   AWS    │           │
│    │          │  │          │  │          │           │
│    │ alice@   │  │ alice    │  │ alice    │           │
│    │ corp.com │  │          │  │ IAM User │           │
│    └──────────┘  └──────────┘  └──────────┘           │
│                                                         │
│  问题：                                                 │
│  - Alice 要记住多套密码                                 │
│  - Alice 离职要在每个系统删除账号                       │
│  - 每个系统都要维护自己的用户库                         │
│  - 没有统一的访问日志                                   │
│                                                         │
└─────────────────────────────────────────────────────────┘

联邦身份的目标：
  用一个"权威身份"访问多个系统，而不用在每个系统创建账号
```

### 1.2 联邦身份的本质

```
联邦 = 信任传递

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  身份提供者 (IdP)           服务提供者 (SP)             │
│  Identity Provider          Service Provider            │
│                                                         │
│  ┌─────────────────┐       ┌─────────────────┐         │
│  │                 │       │                 │         │
│  │  "我认证用户"   │──────►│  "我信任你的    │         │
│  │                 │ 信任  │   认证结果"     │         │
│  │  Okta          │       │                 │         │
│  │  Azure AD      │       │  AWS            │         │
│  │  Google        │       │  GitHub         │         │
│  │  GitHub        │       │  Kubernetes     │         │
│  │                 │       │                 │         │
│  └─────────────────┘       └─────────────────┘         │
│                                                         │
│  IdP 负责：认证（你是谁）                               │
│  SP 负责：授权（你能做什么）                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 1.3 为什么需要联邦身份

```
场景1：企业 SSO
──────────────
员工用企业账号登录所有系统（AWS、GitHub、Jira...）
离职时只需在企业目录删除，所有访问自动失效

场景2：CI/CD 无密钥部署
──────────────
GitHub Actions 不存储 AWS Access Key
而是用 GitHub 签发的 Token 换取 AWS 临时凭证

场景3：Kubernetes 工作负载
──────────────
K8s Pod 不存储云凭证
而是用 K8s ServiceAccount Token 换取云 IAM 权限

场景4：第三方应用集成
──────────────
SaaS 应用不需要你的 AWS 密码
而是通过 OAuth/OIDC 获得有限授权
```

---

## 2. 联邦身份的两大协议

### 2.1 SAML 2.0

```
SAML = Security Assertion Markup Language

诞生于 2005 年，主要用于：
  - 企业 SSO
  - Web 应用联邦登录

核心概念：
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  SAML Assertion（断言）                                │
│  ─────────────────────                                  │
│  IdP 签发的 XML 文档，包含：                            │
│    - 用户身份信息                                       │
│    - 认证时间                                           │
│    - 有效期                                             │
│    - IdP 的数字签名                                     │
│                                                         │
│  <saml:Assertion>                                      │
│    <saml:Subject>                                      │
│      <saml:NameID>alice@corp.com</saml:NameID>        │
│    </saml:Subject>                                     │
│    <saml:Conditions NotBefore="..." NotOnOrAfter="...">│
│    <saml:AuthnStatement AuthnInstant="...">            │
│    <ds:Signature>...</ds:Signature>                    │
│  </saml:Assertion>                                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**SAML 登录流程（SP-initiated）：**

```
┌────────┐       ┌────────┐       ┌────────┐
│  用户   │       │   SP   │       │  IdP   │
│        │       │ (AWS)  │       │ (Okta) │
└───┬────┘       └───┬────┘       └───┬────┘
    │                │                │
    │ 1. 访问 AWS    │                │
    │───────────────►│                │
    │                │                │
    │ 2. 重定向到 IdP│                │
    │◄───────────────│                │
    │                │                │
    │ 3. 跳转到 Okta │                │
    │────────────────────────────────►│
    │                │                │
    │ 4. 用户登录    │                │
    │◄───────────────────────────────►│
    │   (用户名/密码/MFA)              │
    │                │                │
    │ 5. 返回 SAML Assertion          │
    │◄────────────────────────────────│
    │   (POST 到 AWS)│                │
    │                │                │
    │ 6. 验证签名，创建会话            │
    │───────────────►│                │
    │                │                │
    │ 7. 登录成功    │                │
    │◄───────────────│                │
    │                │                │
```

### 2.2 OIDC (OpenID Connect)

```
OIDC = OAuth 2.0 + 身份层

诞生于 2014 年，更现代、更轻量：
  - 基于 JSON（不是 XML）
  - RESTful API
  - 适合移动应用、SPA、微服务

核心概念：
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  ID Token（身份令牌）                                  │
│  ─────────────────────                                  │
│  IdP 签发的 JWT（JSON Web Token），包含：               │
│    - 用户身份信息（claims）                             │
│    - 签发者（issuer）                                   │
│    - 受众（audience）                                   │
│    - 过期时间                                           │
│    - 数字签名                                           │
│                                                         │
│  eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.                 │
│  eyJpc3MiOiJodHRwczovL3Rva2VuLmFjdGlvbnMuZ2l0aHVi...  │
│  SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c          │
│    │               │                    │              │
│  Header         Payload            Signature           │
│                                                         │
│  Payload 解码后：                                       │
│  {                                                      │
│    "iss": "https://token.actions.githubusercontent.com",│
│    "sub": "repo:myorg/myrepo:ref:refs/heads/main",     │
│    "aud": "sts.amazonaws.com",                         │
│    "exp": 1705312200,                                  │
│    "iat": 1705308600                                   │
│  }                                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**OIDC 登录流程（Authorization Code Flow）：**

```
┌────────┐       ┌────────┐       ┌────────┐
│  用户   │       │  应用   │       │  IdP   │
│        │       │        │       │(Google)│
└───┬────┘       └───┬────┘       └───┬────┘
    │                │                │
    │ 1. 点击登录    │                │
    │───────────────►│                │
    │                │                │
    │ 2. 重定向到 IdP│                │
    │◄───────────────│                │
    │   (带 client_id, redirect_uri, scope)
    │                │                │
    │ 3. 跳转到 Google│               │
    │────────────────────────────────►│
    │                │                │
    │ 4. 用户登录授权│                │
    │◄───────────────────────────────►│
    │                │                │
    │ 5. 重定向回应用 │               │
    │◄────────────────────────────────│
    │   (带 authorization code)       │
    │                │                │
    │ 6. 用 code 换 token             │
    │───────────────►│───────────────►│
    │                │                │
    │ 7. 返回 ID Token + Access Token │
    │◄───────────────│◄───────────────│
    │                │                │
    │ 8. 验证 Token，登录成功         │
    │◄───────────────│                │
```

### 2.3 SAML vs OIDC 对比

```
┌─────────────────┬─────────────────┬─────────────────┐
│                 │      SAML       │      OIDC       │
├─────────────────┼─────────────────┼─────────────────┤
│ 诞生年份        │      2005       │      2014       │
├─────────────────┼─────────────────┼─────────────────┤
│ 数据格式        │      XML        │      JSON       │
├─────────────────┼─────────────────┼─────────────────┤
│ 令牌格式        │  SAML Assertion │      JWT        │
├─────────────────┼─────────────────┼─────────────────┤
│ 传输方式        │  Browser POST   │   RESTful API   │
├─────────────────┼─────────────────┼─────────────────┤
│ 主要场景        │  企业 SSO       │  现代应用/API   │
├─────────────────┼─────────────────┼─────────────────┤
│ 复杂度          │      高         │      中         │
├─────────────────┼─────────────────┼─────────────────┤
│ 移动端支持      │      差         │      好         │
├─────────────────┼─────────────────┼─────────────────┤
│ 机器对机器      │     不适合      │     适合        │
└─────────────────┴─────────────────┴─────────────────┘

选择建议：
  - 企业 SSO，对接传统 IdP → SAML
  - 现代应用，CI/CD，K8s → OIDC
```

---

## 3. AWS 联邦身份实现

### 3.1 SAML Federation

```
配置步骤：

1. 在 IdP（如 Okta）配置 AWS 作为应用
2. 下载 IdP 的 SAML 元数据 XML
3. 在 AWS 创建 SAML Identity Provider
4. 创建 IAM Role，Trust Policy 允许 SAML Provider
5. 配置属性映射（SAML Attribute → IAM Role）
```

**Trust Policy 配置：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/OktaSAML"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
```

**属性映射示例：**

```xml
<!-- IdP 返回的 SAML Assertion 中的属性 -->
<Attribute Name="https://aws.amazon.com/SAML/Attributes/Role">
  <AttributeValue>
    arn:aws:iam::123456789012:role/AdminRole,
    arn:aws:iam::123456789012:saml-provider/OktaSAML
  </AttributeValue>
</Attribute>

<Attribute Name="https://aws.amazon.com/SAML/Attributes/RoleSessionName">
  <AttributeValue>alice@corp.com</AttributeValue>
</Attribute>

<!-- AWS 根据这些属性决定用户可以 Assume 哪个 Role -->
```

### 3.2 OIDC Federation

```
配置步骤：

1. 在 AWS 创建 OIDC Identity Provider
   - Provider URL（如 https://token.actions.githubusercontent.com）
   - Audience（如 sts.amazonaws.com）
2. 创建 IAM Role，Trust Policy 允许 OIDC Provider
3. 配置 Condition 限制（如只允许特定 repo）
```

**创建 OIDC Provider（CLI）：**

```bash
# 获取 IdP 的 thumbprint（用于验证签名）
# GitHub Actions 的 thumbprint
THUMBPRINT="6938fd4d98bab03faadb97b34396831e3780aea1"

aws iam create-open-id-connect-provider \
    --url https://token.actions.githubusercontent.com \
    --client-id-list sts.amazonaws.com \
    --thumbprint-list $THUMBPRINT
```

**Trust Policy 配置（GitHub Actions）：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:*"
        }
      }
    }
  ]
}
```

**Condition 的重要性：**

```
┌─────────────────────────────────────────────────────────┐
│  不加 Condition 的危险：                                │
│                                                         │
│  如果只写：                                             │
│  "Principal": {                                        │
│    "Federated": "arn:...:oidc-provider/token.actions..."│
│  }                                                      │
│                                                         │
│  任何 GitHub 仓库都能 Assume 你的 Role！               │
│  包括攻击者控制的仓库                                   │
│                                                         │
│  必须加 Condition：                                     │
│  "StringLike": {                                        │
│    "token.actions.githubusercontent.com:sub":          │
│      "repo:YOUR_ORG/YOUR_REPO:*"                       │
│  }                                                      │
│                                                         │
│  限制只有你的仓库才能 Assume                            │
└─────────────────────────────────────────────────────────┘
```

---

## 4. GitHub Actions OIDC 详解

### 4.1 工作原理

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  GitHub Actions Runner                                         │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Workflow 运行时                                         │   │
│  │                                                          │   │
│  │  1. 请求 OIDC Token                                     │   │
│  │     → GitHub 签发 JWT                                   │   │
│  │     → Token 包含 repo、branch、workflow 等信息          │   │
│  │                                                          │   │
│  │  2. 用 JWT 调用 AWS STS                                 │   │
│  │     → AssumeRoleWithWebIdentity                         │   │
│  │     → AWS 验证 JWT 签名（用 GitHub 的公钥）             │   │
│  │     → AWS 检查 Trust Policy 条件                        │   │
│  │                                                          │   │
│  │  3. 获得 AWS 临时凭证                                   │   │
│  │     → 用于后续 AWS 操作                                 │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 JWT Token 结构

```json
// GitHub Actions 签发的 OIDC Token 内容
{
  // 签发者
  "iss": "https://token.actions.githubusercontent.com",

  // 主体：包含 repo、ref 等信息
  "sub": "repo:myorg/myrepo:ref:refs/heads/main",

  // 受众
  "aud": "sts.amazonaws.com",

  // 过期时间（通常很短，如 5 分钟）
  "exp": 1705312200,
  "iat": 1705308600,

  // 额外的 claims（可用于 Condition）
  "repository": "myorg/myrepo",
  "repository_owner": "myorg",
  "ref": "refs/heads/main",
  "sha": "abc123...",
  "workflow": "Deploy",
  "actor": "alice",
  "event_name": "push",
  "run_id": "1234567890",
  "run_number": "42",
  "environment": "production"
}
```

### 4.3 完整配置示例

```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS

on:
  push:
    branches: [main]

permissions:
  id-token: write   # 必须：允许请求 OIDC Token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
          # 可选：指定会话名称
          role-session-name: github-actions-${{ github.run_id }}

      - name: Deploy
        run: |
          aws s3 sync ./dist s3://my-bucket/
          aws cloudfront create-invalidation --distribution-id XXX --paths "/*"
```

**更精细的 Condition 控制：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          // 只允许特定环境
          "token.actions.githubusercontent.com:environment": "production"
        },
        "StringLike": {
          // 只允许特定仓库的 main 分支
          "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

---

## 5. Kubernetes OIDC Federation

### 5.1 为什么 K8s 需要联邦身份

```
传统方式的问题：

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  方式1：在 Pod 中挂载 Access Key（Secret）              │
│                                                         │
│  ❌ 问题：                                              │
│    - Secret 可能被其他 Pod 访问                        │
│    - Secret 需要手动轮换                                │
│    - 难以实现最小权限（多个 Pod 共享一个 Key）          │
│    - 凭证泄露风险                                       │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  方式2：使用 Node 的 Instance Profile                  │
│                                                         │
│  ❌ 问题：                                              │
│    - Node 上所有 Pod 共享同样的权限                    │
│    - 无法实现 Pod 级别的最小权限                        │
│    - 一个 Pod 被攻破，可以访问所有权限                  │
│                                                         │
└─────────────────────────────────────────────────────────┘

解决方案：Pod 级别的身份

  每个 Pod 有自己的身份（ServiceAccount）
  每个 ServiceAccount 映射到特定的 IAM Role
  Pod 只能获得自己 ServiceAccount 对应的权限
```

### 5.2 IRSA 工作原理 (IAM Roles for Service Accounts)

```
┌──────────────────────────────────────────────────────────────────┐
│                        EKS Cluster                               │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                         Pod                                 │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │  Container                                            │  │ │
│  │  │                                                       │  │ │
│  │  │  环境变量（自动注入）：                                │  │ │
│  │  │  AWS_ROLE_ARN=arn:aws:iam::123:role/MyRole           │  │ │
│  │  │  AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/...    │  │ │
│  │  │                                                       │  │ │
│  │  │  挂载的文件：                                         │  │ │
│  │  │  /var/run/secrets/eks.amazonaws.com/serviceaccount/  │  │ │
│  │  │    └── token  (K8s 签发的 JWT)                       │  │ │
│  │  │                                                       │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  │                            │                                │ │
│  │                            │ AWS SDK 自动读取               │ │
│  │                            ▼                                │ │
│  └────────────────────────────────────────────────────────────┘ │
│                               │                                  │
│                               │ AssumeRoleWithWebIdentity       │
│                               ▼                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                     AWS STS                                 │ │
│  │                                                             │ │
│  │  1. 验证 JWT 签名（用 EKS OIDC Provider 的公钥）           │ │
│  │  2. 检查 Trust Policy                                      │ │
│  │  3. 返回临时凭证                                           │ │
│  │                                                             │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 5.3 IRSA 配置步骤

```bash
# 1. 确认 EKS 集群的 OIDC Provider
aws eks describe-cluster --name my-cluster \
    --query "cluster.identity.oidc.issuer" --output text
# 输出: https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E

# 2. 创建 IAM OIDC Provider（如果还没有）
eksctl utils associate-iam-oidc-provider \
    --cluster my-cluster \
    --approve

# 3. 创建 IAM Role 和 ServiceAccount
eksctl create iamserviceaccount \
    --cluster my-cluster \
    --namespace default \
    --name my-service-account \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
    --approve
```

**手动配置 Trust Policy：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub":
            "system:serviceaccount:default:my-service-account",
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:aud":
            "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

**ServiceAccount 配置：**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/MyRole

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: my-service-account  # 使用这个 ServiceAccount
      containers:
        - name: app
          image: my-app:latest
          # 无需任何 AWS 凭证配置
          # SDK 会自动使用 IRSA
```

### 5.4 K8s Token 的结构

```json
// K8s ServiceAccount Token 内容
{
  "iss": "https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E",
  "sub": "system:serviceaccount:default:my-service-account",
  "aud": ["sts.amazonaws.com"],
  "exp": 1705398600,
  "iat": 1705312200,
  "kubernetes.io": {
    "namespace": "default",
    "pod": {
      "name": "my-app-xxx-yyy",
      "uid": "abc-123-..."
    },
    "serviceaccount": {
      "name": "my-service-account",
      "uid": "def-456-..."
    }
  }
}
```

---

## 6. GCP Workload Identity Federation

### 6.1 概述

```
GCP 的联邦身份实现：

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  外部身份                    GCP                        │
│                                                         │
│  ┌─────────────────┐       ┌─────────────────────────┐ │
│  │  AWS            │       │  Workload Identity Pool │ │
│  │  Azure          │──────►│                         │ │
│  │  GitHub Actions │       │  映射外部身份到          │ │
│  │  Kubernetes     │       │  GCP Service Account    │ │
│  │  任何 OIDC IdP  │       │                         │ │
│  └─────────────────┘       └─────────────────────────┘ │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 6.2 配置示例（GitHub Actions → GCP）

```bash
# 1. 创建 Workload Identity Pool
gcloud iam workload-identity-pools create "github-pool" \
    --location="global" \
    --display-name="GitHub Actions Pool"

# 2. 创建 Provider
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
    --location="global" \
    --workload-identity-pool="github-pool" \
    --display-name="GitHub Provider" \
    --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
    --issuer-uri="https://token.actions.githubusercontent.com"

# 3. 绑定 Service Account
gcloud iam service-accounts add-iam-policy-binding "my-sa@my-project.iam.gserviceaccount.com" \
    --role="roles/iam.workloadIdentityUser" \
    --member="principalSet://iam.googleapis.com/projects/123456/locations/global/workloadIdentityPools/github-pool/attribute.repository/myorg/myrepo"
```

```yaml
# GitHub Actions workflow
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: 'projects/123456/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
    service_account: 'my-sa@my-project.iam.gserviceaccount.com'
```

---

## 7. 常见问题与陷阱

### 7.1 Token 验证失败

```
问题：AssumeRoleWithWebIdentity 失败

排查步骤：

1. 检查 Token 是否过期
   → JWT 通常有效期很短（5-15 分钟）

2. 检查 Audience 是否匹配
   → Trust Policy 中的 aud 必须与 Token 中的一致

3. 检查 Subject 是否匹配
   → StringLike 的模式是否正确

4. 检查 OIDC Provider 配置
   → Thumbprint 是否正确
   → URL 是否正确（注意末尾不要有斜杠）
```

### 7.2 权限过宽

```
❌ 危险的配置：

{
  "Condition": {
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:myorg/*"
    }
  }
}

→ myorg 下所有仓库都能 Assume！包括 fork 的仓库？


✅ 安全的配置：

{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
    }
  }
}

→ 只有特定仓库的特定分支
```

### 7.3 Session 名称冲突

```
问题：多个并发的 Assume 会冲突吗？

答案：不会，只要 RoleSessionName 不同

最佳实践：使用唯一标识符作为 session name

GitHub Actions:
  role-session-name: github-${{ github.run_id }}-${{ github.run_attempt }}

K8s:
  Session name 自动包含 pod 名称
```

---

## 8. 总结

```
┌─────────────────────────────────────────────────────────────────┐
│                     联邦身份核心概念                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  本质：                                                         │
│    用外部 IdP 认证，云平台授权                                  │
│    信任传递：IdP 证明你是谁，云平台决定你能做什么               │
│                                                                 │
│  两大协议：                                                     │
│    SAML  ─  企业 SSO，XML 格式，较复杂                         │
│    OIDC  ─  现代应用，JWT 格式，更轻量                         │
│                                                                 │
│  关键场景：                                                     │
│    企业 SSO        ─  员工用企业账号访问云                     │
│    GitHub Actions  ─  CI/CD 无密钥部署                         │
│    K8s IRSA        ─  Pod 级别的云权限                         │
│                                                                 │
│  安全要点：                                                     │
│    Condition 必须精确 ─  限制 repo、branch、environment        │
│    Token 有效期短    ─  减少泄露风险                           │
│    审计日志完整      ─  可追溯谁用什么身份做了什么              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **核心洞见：联邦身份的本质是"信任传递"——你信任 IdP 的认证结果，IdP 信任用户的凭证。这条信任链使得用户无需在每个系统创建账号，也使得机器无需存储长期密钥。**

---

## 思考题

1. 为什么 OIDC Token 的有效期设计得很短（通常 5-15 分钟）？
2. 如果你的 GitHub 仓库被 fork，fork 的仓库能 Assume 你的 AWS Role 吗？如何防止？
3. 设计一个方案：允许 K8s 中的某个 Pod 只能访问特定的 S3 bucket 前缀。
