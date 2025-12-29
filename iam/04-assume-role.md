# 第四层：Assume Role 机制 - 深度展开

## 1. 从第一性原理看 Assume Role

### 1.1 核心问题

```
在分布式系统中，经常遇到这样的问题：

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  身份 A 需要临时"变成"身份 B 来执行某些操作              │
│                                                         │
│  场景1：跨账号访问                                      │
│    账号 A 的用户需要访问账号 B 的资源                   │
│                                                         │
│  场景2：权限提升                                        │
│    普通用户需要临时获得管理员权限执行敏感操作            │
│                                                         │
│  场景3：服务间调用                                      │
│    Service A 需要以特定权限调用 Service B               │
│                                                         │
│  场景4：外部集成                                        │
│    CI/CD 系统需要部署到你的 AWS 账号                    │
│                                                         │
└─────────────────────────────────────────────────────────┘

传统方案：共享凭证 → 不安全、不可追溯、难以撤销

Assume Role：安全地"借用"身份
```

### 1.2 Assume Role 的本质

```
Assume Role = 身份委托 + 临时凭证

┌──────────────────────────────────────────────────────────┐
│                                                          │
│   不是"获取权限"，而是"临时成为另一个身份"               │
│                                                          │
│   Alice (IAM User)                                      │
│      │                                                   │
│      │ AssumeRole                                       │
│      ▼                                                   │
│   临时成为 AdminRole                                    │
│      │                                                   │
│      │ 以 AdminRole 的身份执行操作                       │
│      ▼                                                   │
│   操作完成，身份恢复                                     │
│                                                          │
│   关键：这段时间内，Alice 的行为被记录为 AdminRole       │
│         但 CloudTrail 会显示是 Alice assume 的           │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Assume Role 的完整流程

### 2.1 时序图

```
┌─────────┐          ┌─────────┐          ┌─────────┐
│ 请求者   │          │   STS   │          │  目标   │
│ (Alice) │          │         │          │  Role   │
└────┬────┘          └────┬────┘          └────┬────┘
     │                    │                    │
     │  1. AssumeRole     │                    │
     │  (带上自己的凭证)   │                    │
     │───────────────────►│                    │
     │                    │                    │
     │                    │  2. 验证请求者身份  │
     │                    │←──────────────────►│
     │                    │                    │
     │                    │  3. 检查 Trust     │
     │                    │     Policy         │
     │                    │     Role 允许      │
     │                    │     Alice 吗？     │
     │                    │←──────────────────►│
     │                    │                    │
     │                    │  4. 检查 Alice     │
     │                    │     是否有权限     │
     │                    │     sts:AssumeRole │
     │                    │                    │
     │  5. 返回临时凭证   │                    │
     │  (AK/SK/Token     │                    │
     │   + 过期时间)      │                    │
     │◄───────────────────│                    │
     │                    │                    │
     │  6. 用临时凭证     │                    │
     │     调用 AWS API   │                    │
     │────────────────────────────────────────►│
     │                    │                    │
```

### 2.2 代码示例

```python
import boto3

# 1. 使用当前身份（Alice）调用 STS
sts_client = boto3.client('sts')

# 2. 请求 Assume Role
response = sts_client.assume_role(
    RoleArn='arn:aws:iam::123456789012:role/AdminRole',
    RoleSessionName='alice-admin-session',
    DurationSeconds=3600,  # 1 小时
    # 可选：Session Policy 进一步限制权限
    # Policy='{"Version":"2012-10-17","Statement":[...]}'
)

# 3. 提取临时凭证
credentials = response['Credentials']
access_key = credentials['AccessKeyId']
secret_key = credentials['SecretAccessKey']
session_token = credentials['SessionToken']
expiration = credentials['Expiration']

# 4. 使用临时凭证创建新的客户端
s3_client = boto3.client(
    's3',
    aws_access_key_id=access_key,
    aws_secret_access_key=secret_key,
    aws_session_token=session_token
)

# 5. 以 AdminRole 的身份执行操作
s3_client.list_buckets()
```

```bash
# CLI 方式
aws sts assume-role \
    --role-arn arn:aws:iam::123456789012:role/AdminRole \
    --role-session-name alice-admin-session \
    --duration-seconds 3600

# 返回：
{
    "Credentials": {
        "AccessKeyId": "ASIAIOSFODNN7EXAMPLE",
        "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/...",
        "SessionToken": "FwoGZXIvYXdzEBY...",
        "Expiration": "2024-01-15T12:30:00Z"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROA3XFRBF535EXAMPLE:alice-admin-session",
        "Arn": "arn:aws:sts::123456789012:assumed-role/AdminRole/alice-admin-session"
    }
}
```

---

## 3. 三层权限控制

### 3.1 权限的交集原理

```
Assume Role 涉及三层权限，最终权限是它们的交集：

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Layer 1: Trust Policy（信任策略）                      │
│  ─────────────────────────────────────                  │
│  谁可以 Assume 这个 Role？                              │
│  如果不在信任列表中，连 Assume 都做不了                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼ 通过
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Layer 2: Permission Policy（权限策略）                 │
│  ─────────────────────────────────────                  │
│  这个 Role 本身有什么权限？                             │
│  Assume 后最多只能有这些权限                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼ 交集
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Layer 3: Session Policy（会话策略）[可选]              │
│  ─────────────────────────────────────                  │
│  本次 Assume 进一步限制什么权限？                       │
│  可以比 Permission Policy 更严格，但不能更宽松          │
│                                                         │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
                    最终权限
```

**示意图：**

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│   Permission Policy          Session Policy                  │
│   (Role 有的权限)             (本次限制)                      │
│                                                               │
│   ┌─────────────────┐        ┌─────────────────┐             │
│   │    s3:*         │        │  s3:GetObject   │             │
│   │    ec2:*        │   ∩    │  s3:ListBucket  │             │
│   │    rds:*        │        │                 │             │
│   └─────────────────┘        └─────────────────┘             │
│                                                               │
│           ↓                          ↓                        │
│   ┌───────────────────────────────────────────┐              │
│   │            最终权限                        │              │
│   │                                           │              │
│   │     s3:GetObject + s3:ListBucket         │              │
│   │     (只有交集部分)                        │              │
│   └───────────────────────────────────────────┘              │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 3.2 Trust Policy 详解

```json
// Trust Policy 的基本结构
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        // 谁可以 Assume
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        // 什么条件下允许
      }
    }
  ]
}
```

**不同场景的 Trust Policy：**

```json
// 场景1：允许同账号的特定用户
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:user/alice"
  },
  "Action": "sts:AssumeRole"
}

// 场景2：允许同账号的特定角色（角色链）
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/ServiceRole"
  },
  "Action": "sts:AssumeRole"
}

// 场景3：允许另一个账号（跨账号）
{
  "Principal": {
    "AWS": "arn:aws:iam::999888777666:root"
  },
  "Action": "sts:AssumeRole"
}

// 场景4：允许 AWS 服务（EC2、Lambda 等）
{
  "Principal": {
    "Service": "ec2.amazonaws.com"
  },
  "Action": "sts:AssumeRole"
}

// 场景5：允许 OIDC 身份提供者（GitHub、Google 等）
{
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    },
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
    }
  }
}

// 场景6：允许 SAML 身份提供者（企业 SSO）
{
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
```

**Trust Policy 的 Condition 限制：**

```json
// 限制：必须使用 MFA
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:user/alice"
  },
  "Action": "sts:AssumeRole",
  "Condition": {
    "Bool": {
      "aws:MultiFactorAuthPresent": "true"
    }
  }
}

// 限制：只允许特定 IP
{
  "Principal": {
    "AWS": "arn:aws:iam::999888777666:root"
  },
  "Action": "sts:AssumeRole",
  "Condition": {
    "IpAddress": {
      "aws:SourceIp": "203.0.113.0/24"
    }
  }
}

// 限制：必须设置特定的 ExternalId（防止混淆代理攻击）
{
  "Principal": {
    "AWS": "arn:aws:iam::999888777666:root"
  },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "unique-id-from-partner"
    }
  }
}
```

### 3.3 Session Policy 详解

```python
# Session Policy 在 Assume 时传入
response = sts_client.assume_role(
    RoleArn='arn:aws:iam::123456789012:role/AdminRole',
    RoleSessionName='limited-session',
    Policy=json.dumps({
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject"
                ],
                "Resource": [
                    "arn:aws:s3:::my-bucket/readonly/*"
                ]
            }
        ]
    })
)
```

**Session Policy 的作用：**

```
┌─────────────────────────────────────────────────────────┐
│  场景：同一个 Role 被不同任务使用，需要不同权限         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  DeployRole 的 Permission Policy:                      │
│    - s3:*                                              │
│    - lambda:*                                          │
│    - cloudformation:*                                  │
│                                                         │
│  任务 A（只需要上传 artifact）:                         │
│    Session Policy: 只允许 s3:PutObject                 │
│    最终权限: s3:PutObject                              │
│                                                         │
│  任务 B（只需要更新 Lambda）:                           │
│    Session Policy: 只允许 lambda:UpdateFunctionCode   │
│    最终权限: lambda:UpdateFunctionCode                 │
│                                                         │
│  好处：一个 Role，多个最小权限的使用方式                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Permission Boundary（权限边界）

### 4.1 什么是 Permission Boundary

```
Permission Boundary 是附加在 IAM User 或 Role 上的"天花板"

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Permission Policy（权限策略）：你被授予的权限          │
│  Permission Boundary（权限边界）：你最多能有的权限      │
│                                                         │
│  最终权限 = Permission Policy ∩ Permission Boundary    │
│                                                         │
└─────────────────────────────────────────────────────────┘

类比：
  Permission Policy = 你的工资卡余额
  Permission Boundary = 每日取款限额

  余额有 10 万，但每日限额 1 万
  → 今天最多取 1 万
```

### 4.2 为什么需要 Permission Boundary

```
场景：委托权限管理

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  问题：                                                 │
│  - 管理员 Admin 可以创建 IAM User/Role                 │
│  - Admin 可能创建一个权限比自己还大的 Role              │
│  - 或者创建一个 Role 给自己更多权限（权限提升攻击）      │
│                                                         │
│  解决方案：Permission Boundary                         │
│  - 规定 Admin 创建的任何 User/Role 必须附加边界        │
│  - 边界限制了创建出来的身份最多能有什么权限             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**实际示例：**

```json
// 1. 定义一个 Permission Boundary
// 名称：DeveloperBoundary
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "dynamodb:*",
        "lambda:*",
        "logs:*"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "iam:*",
        "organizations:*",
        "account:*"
      ],
      "Resource": "*"
    }
  ]
}

// 2. 给开发团队 Lead 权限创建 Role，但必须附加边界
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DeveloperBoundary"
        }
      }
    }
  ]
}

// 效果：
// - Lead 可以创建 Role
// - 但创建的 Role 必须附加 DeveloperBoundary
// - 即使 Lead 给 Role 附加 AdministratorAccess
//   实际权限也被 Boundary 限制
```

### 4.3 完整权限评估流程

```
请求到达
    │
    ▼
┌─────────────────────────────┐
│ 1. Organization SCP        │  ← 组织层面的限制
└─────────────────────────────┘
    │ 通过
    ▼
┌─────────────────────────────┐
│ 2. Permission Boundary     │  ← 身份的权限天花板
└─────────────────────────────┘
    │ 通过
    ▼
┌─────────────────────────────┐
│ 3. Identity Policy         │  ← 身份被授予的权限
│    + Resource Policy       │
└─────────────────────────────┘
    │ 通过
    ▼
┌─────────────────────────────┐
│ 4. Session Policy          │  ← 会话级别的限制
│    (如果是 Assume Role)    │
└─────────────────────────────┘
    │
    ▼
  Allow

任何一层拒绝 → 最终拒绝
所有层通过 → 最终允许
```

---

## 5. 跨账号访问模式

### 5.1 基本跨账号模式

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Account A (123456789012)         Account B (999888777666)    │
│                                                                 │
│   ┌─────────────┐                  ┌─────────────────────────┐ │
│   │  IAM User   │                  │      IAM Role           │ │
│   │   Alice     │                  │    CrossAccountRole     │ │
│   │             │                  │                         │ │
│   │  Policy:    │                  │  Trust Policy:          │ │
│   │  Allow      │   AssumeRole     │  Allow Account A        │ │
│   │  sts:       │ ───────────────► │                         │ │
│   │  AssumeRole │                  │  Permission Policy:     │ │
│   │  on B's Role│                  │  Allow s3:GetObject     │ │
│   │             │                  │  on my-bucket/*         │ │
│   └─────────────┘                  └─────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

步骤：
1. Account B 创建 Role，Trust Policy 允许 Account A
2. Account A 的 Alice 有权限 sts:AssumeRole 到 B 的 Role
3. Alice 调用 AssumeRole，获取 Account B 的临时凭证
4. Alice 用临时凭证访问 Account B 的 S3
```

### 5.2 多账号架构模式

```
常见的企业多账号架构：

┌─────────────────────────────────────────────────────────────────┐
│                        Organization                             │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                  Management Account                      │  │
│   │                  (账单、组织管理)                         │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                  │
│          ┌───────────────────┼───────────────────┐             │
│          │                   │                   │             │
│          ▼                   ▼                   ▼             │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     │
│   │  Security   │     │   Shared    │     │    Log      │     │
│   │  Account    │     │  Services   │     │   Account   │     │
│   │             │     │             │     │             │     │
│   │ - IAM管理   │     │ - 共享服务  │     │ - CloudTrail│     │
│   │ - 安全工具  │     │ - 网络      │     │ - 日志归档  │     │
│   └─────────────┘     └─────────────┘     └─────────────┘     │
│                              │                                  │
│          ┌───────────────────┼───────────────────┐             │
│          │                   │                   │             │
│          ▼                   ▼                   ▼             │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     │
│   │    Dev      │     │   Staging   │     │    Prod     │     │
│   │  Account    │     │   Account   │     │   Account   │     │
│   │             │     │             │     │             │     │
│   │ - 开发环境  │     │ - 测试环境  │     │ - 生产环境  │     │
│   └─────────────┘     └─────────────┘     └─────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 跨账号访问的两种方式

```
方式1：Resource-based Policy（资源策略）
─────────────────────────────────────────

Account B 的 S3 Bucket Policy 直接允许 Account A：

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::bucket-in-b/*"
    }
  ]
}

特点：
  ✓ 简单直接
  ✓ 不需要 AssumeRole
  ✗ 只有支持 Resource Policy 的服务可用
  ✗ 难以集中管理


方式2：AssumeRole（角色假扮）
─────────────────────────────────────────

Account B 创建 Role，允许 Account A assume：

Trust Policy:
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:root"
  },
  "Action": "sts:AssumeRole"
}

Permission Policy:
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::bucket-in-b/*"
}

特点：
  ✓ 所有服务都支持
  ✓ 可以精细控制权限
  ✓ 有完整的审计日志
  ✓ 可以添加 MFA、ExternalId 等条件
  ✗ 需要额外的 AssumeRole 调用
```

### 5.4 External ID：防止混淆代理攻击

```
问题场景：第三方服务代替你访问 AWS

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   你的账号                    第三方服务                        │
│   (Victim)                   (如监控服务)                       │
│                                                                 │
│   ┌─────────────┐            ┌─────────────────┐               │
│   │ IAM Role    │            │   监控服务       │               │
│   │             │◄───────────│                 │               │
│   │ Trust:      │  Assume    │ 用你的 Role     │               │
│   │  监控服务   │            │ 访问你的资源     │               │
│   └─────────────┘            └─────────────────┘               │
│                                     ▲                           │
│                                     │                           │
│                              ┌──────┴──────┐                   │
│                              │   攻击者     │                   │
│                              │             │                   │
│                              │ 也用同一个   │                   │
│                              │ 监控服务     │                   │
│                              │             │                   │
│                              │ 欺骗服务说   │                   │
│                              │ "访问Victim"│                   │
│                              └─────────────┘                   │
│                                                                 │
│   问题：攻击者可以让监控服务代他访问你的资源！                  │
│   这就是"混淆代理"攻击                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

解决方案：External ID

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   你的 Role 的 Trust Policy:                                   │
│                                                                 │
│   {                                                             │
│     "Principal": {                                             │
│       "AWS": "arn:aws:iam::MONITOR_ACCOUNT:role/MonitorRole"   │
│     },                                                          │
│     "Action": "sts:AssumeRole",                                │
│     "Condition": {                                              │
│       "StringEquals": {                                         │
│         "sts:ExternalId": "your-unique-secret-id-12345"        │
│       }                                                         │
│     }                                                           │
│   }                                                             │
│                                                                 │
│   监控服务只有知道你的 ExternalId 才能 Assume                   │
│   攻击者不知道这个 ID，无法冒充                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Role Chaining（角色链）

### 6.1 什么是 Role Chaining

```
Role A → Assume → Role B → Assume → Role C

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   User                                                         │
│     │                                                           │
│     │ Assume                                                   │
│     ▼                                                           │
│   Role A (Dev 账号)                                            │
│     │                                                           │
│     │ Assume                                                   │
│     ▼                                                           │
│   Role B (Staging 账号)                                        │
│     │                                                           │
│     │ Assume                                                   │
│     ▼                                                           │
│   Role C (Prod 账号)                                           │
│                                                                 │
│   用途：从开发账号一步步访问到生产账号                          │
│   每一步都有审计记录                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Role Chaining 的限制

```
限制1：最大会话时长
──────────────────
第一次 Assume：可以设置最长 12 小时
链式 Assume：最长只有 1 小时

原因：安全考虑，链越长风险越大


限制2：凭证刷新
──────────────────
链式 Assume 获得的凭证不能自动刷新
必须重新走一遍链


限制3：追溯复杂度
──────────────────
CloudTrail 会记录每一次 Assume
但追踪"最初是谁"需要关联多条日志
```

---

## 7. 常见问题与陷阱

### 7.1 Trust Policy 配置错误

```
错误1：Trust Policy 过于宽松

❌ 危险：
{
  "Principal": "*",
  "Action": "sts:AssumeRole"
}
→ 任何人都能 Assume！

❌ 危险：
{
  "Principal": {"AWS": "*"},
  "Action": "sts:AssumeRole"
}
→ 任何 AWS 账号都能 Assume！

✅ 正确：明确指定 Principal


错误2：忘记 Principal 也需要权限

Account A 的 User 想 Assume Account B 的 Role

只配置 B 的 Trust Policy 允许 A ❌ 不够

还需要：A 的 User 有 sts:AssumeRole 权限

两边都要配置！
```

### 7.2 Session 时长陷阱

```
场景：长时间运行的任务

问题：
  - Assume Role 获取的凭证最长 12 小时
  - 任务运行超过 12 小时
  - 凭证过期，任务失败

解决方案：
  1. 使用 Instance Profile（EC2）
     → 自动刷新凭证

  2. 实现凭证刷新逻辑
     → 在凭证过期前重新 Assume

  3. 拆分长任务
     → 每个子任务独立获取凭证
```

### 7.3 调试权限问题

```bash
# 查看当前身份
aws sts get-caller-identity

# 输出：
{
    "UserId": "AROA3XFRBF535:my-session",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/MyRole/my-session"
}

# 模拟权限检查（不实际执行）
aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::123456789012:role/MyRole \
    --action-names s3:GetObject \
    --resource-arns arn:aws:s3:::my-bucket/file.txt

# 查看 Role 的 Trust Policy
aws iam get-role --role-name MyRole

# 查看 Role 的 Permission Policies
aws iam list-attached-role-policies --role-name MyRole
aws iam list-role-policies --role-name MyRole  # inline policies
```

---

## 8. 总结

```
┌─────────────────────────────────────────────────────────────────┐
│                     Assume Role 核心概念                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  本质：                                                         │
│    临时"成为"另一个身份，获取其权限                             │
│                                                                 │
│  三层控制：                                                     │
│    Trust Policy      ─  谁能 Assume                            │
│    Permission Policy ─  Assume 后能做什么                       │
│    Session Policy    ─  本次会话进一步限制                      │
│                                                                 │
│  权限边界：                                                     │
│    Permission Boundary ─  权限的"天花板"                       │
│                                                                 │
│  跨账号访问：                                                   │
│    Resource Policy  ─  资源直接授权（简单，有限）               │
│    Assume Role      ─  角色假扮（通用，推荐）                   │
│                                                                 │
│  安全要点：                                                     │
│    External ID      ─  防止混淆代理攻击                         │
│    MFA Condition    ─  敏感角色要求 MFA                         │
│    Session Duration ─  合理设置会话时长                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **核心洞见：Assume Role 不是"获取权限"，而是"临时成为另一个身份"。理解这个本质，就理解了为什么它比共享凭证更安全——因为每次"变身"都有记录，都有时限，都可以精确控制。**

---

## 思考题

1. 为什么 Role Chaining 的最大会话时长被限制为 1 小时？
2. 设计一个方案：开发者可以临时获得生产环境只读权限，但必须 MFA + 审批。
3. External ID 是用来防止什么攻击？如果不设置会有什么风险？
