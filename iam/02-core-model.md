# 第二层：核心模型 (What) - 深度展开

## 1. 策略的本质：一个判定函数

### 1.1 回到第一性原理

每当一个请求到达云平台，系统需要做一个判定：

```
┌─────────────────────────────────────────────────────────┐
│                       请求                              │
│                                                         │
│   "我是 Alice，想对 my-bucket/data.csv 执行下载"        │
│                                                         │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                     IAM 引擎                            │
│                                                         │
│   evaluate(request) → Allow | Deny                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
                    Allow 或 Deny
```

这个判定需要哪些输入？

```
请求的完整上下文：

┌──────────────────────────────────────────────────────────┐
│                                                          │
│  WHO     →  谁在请求？           →  Alice (IAM User)    │
│                                                          │
│  WHAT    →  想做什么操作？       →  s3:GetObject        │
│                                                          │
│  WHICH   →  对哪个资源？         →  arn:aws:s3:::       │
│                                     my-bucket/data.csv  │
│                                                          │
│  WHEN    →  什么时候？           →  2024-01-15 10:30    │
│                                                          │
│  WHERE   →  从哪里请求？         →  IP: 203.0.113.50    │
│                                                          │
│  HOW     →  通过什么方式？       →  MFA: Yes, TLS: 1.3  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 1.2 策略的数学表达

IAM 策略本质上是一个 **谓词逻辑表达式**：

```
Allow IF:
    Principal ∈ {allowed_principals}
    AND Action ∈ {allowed_actions}
    AND Resource ∈ {allowed_resources}
    AND Conditions are satisfied

用代码表达：

def evaluate(request):
    for policy in get_all_applicable_policies(request.principal):
        if matches(policy, request):
            if policy.effect == "Deny":
                return DENY           # 显式拒绝，立即返回
            else:
                found_allow = True    # 记录找到允许

    if found_allow:
        return ALLOW
    else:
        return DENY                   # 默认拒绝
```

---

## 2. 四大要素：Principal / Action / Resource / Condition

### 2.1 Principal：谁在请求

```
Principal 回答：这个请求是谁发起的？

┌─────────────────────────────────────────────────────────┐
│                   Principal 类型                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  AWS Account     "Principal": {"AWS": "123456789012"}  │
│  (整个账号)                                             │
│                                                         │
│  IAM User        "Principal": {                        │
│  (具体用户)         "AWS": "arn:aws:iam::123456789012  │
│                            :user/alice"               │
│                  }                                      │
│                                                         │
│  IAM Role        "Principal": {                        │
│  (角色)             "AWS": "arn:aws:iam::123456789012  │
│                            :role/MyRole"              │
│                  }                                      │
│                                                         │
│  AWS Service     "Principal": {                        │
│  (AWS 服务)         "Service": "ec2.amazonaws.com"     │
│                  }                                      │
│                                                         │
│  Federated       "Principal": {                        │
│  (联邦身份)         "Federated": "arn:aws:iam::...     │
│                            :saml-provider/Okta"       │
│                  }                                      │
│                                                         │
│  Everyone        "Principal": "*"                      │
│  (任何人)         ⚠️ 极度危险，谨慎使用                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**关键理解：**

```
Identity-based Policy（附加在身份上）
  → 不需要写 Principal，因为"谁"已经确定了
  → 这个策略附加在 Alice 身上，Principal 就是 Alice

Resource-based Policy（附加在资源上）
  → 必须写 Principal，指明"允许谁"
  → 这个策略附加在 S3 Bucket 上，需要说明允许谁访问
```

### 2.2 Action：想做什么

```
Action 回答：请求想执行什么操作？

AWS 的 Action 命名规范：
┌─────────────────────────────────────────────────────────┐
│                                                         │
│    <service>:<operation>                               │
│        │          │                                     │
│        │          └── 具体操作（API 名称）               │
│        └───────────── 服务名                            │
│                                                         │
│    示例：                                               │
│    s3:GetObject           读取 S3 对象                  │
│    s3:PutObject           上传 S3 对象                  │
│    s3:DeleteObject        删除 S3 对象                  │
│    ec2:StartInstances     启动 EC2 实例                 │
│    ec2:TerminateInstances 终止 EC2 实例                 │
│    iam:CreateUser         创建 IAM 用户                 │
│    sts:AssumeRole         假扮角色                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**通配符使用：**

```json
{
  "Action": "s3:*"              // S3 的所有操作
}

{
  "Action": "s3:Get*"           // S3 的所有 Get 开头的操作
}

{
  "Action": [                   // 多个具体操作
    "s3:GetObject",
    "s3:PutObject"
  ]
}

{
  "Action": "*"                 // ⚠️ 所有服务的所有操作
}                               // 极度危险
```

**Action 的分类思考：**

```
按风险等级分类：

低风险（读取类）          中风险（修改类）         高风险（管理类）
─────────────────       ─────────────────       ─────────────────
s3:GetObject            s3:PutObject            s3:DeleteBucket
s3:ListBucket           ec2:StartInstances      ec2:TerminateInstances
ec2:DescribeInstances   rds:ModifyDBInstance    iam:CreateUser
logs:GetLogEvents       lambda:UpdateFunction   iam:AttachRolePolicy

最小权限原则：只给需要的 Action，不要图省事给 *
```

### 2.3 Resource：对什么操作

```
Resource 回答：操作的目标是什么？

AWS 使用 ARN (Amazon Resource Name) 标识资源：

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  arn:partition:service:region:account:resource         │
│   │      │        │      │      │        │              │
│   │      │        │      │      │        └── 资源标识   │
│   │      │        │      │      └─────────── 账号 ID    │
│   │      │        │      └────────────────── 区域       │
│   │      │        └───────────────────────── 服务名     │
│   │      └────────────────────────────────── 分区       │
│   └───────────────────────────────────────── 固定前缀   │
│                                                         │
└─────────────────────────────────────────────────────────┘

实际例子：

S3 Bucket:
  arn:aws:s3:::my-bucket

S3 Object:
  arn:aws:s3:::my-bucket/path/to/file.txt

EC2 Instance:
  arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0

IAM Role:
  arn:aws:iam::123456789012:role/MyRole

Lambda Function:
  arn:aws:lambda:us-east-1:123456789012:function:my-function
```

**通配符使用：**

```json
// 特定 bucket 下的所有对象
{
  "Resource": "arn:aws:s3:::my-bucket/*"
}

// 特定前缀下的对象
{
  "Resource": "arn:aws:s3:::my-bucket/logs/*"
}

// 所有以 dev- 开头的 bucket
{
  "Resource": "arn:aws:s3:::dev-*"
}

// ⚠️ 所有资源（极度危险）
{
  "Resource": "*"
}
```

**常见陷阱：**

```
陷阱：Bucket 和 Object 是不同的资源！

想法：允许读取 my-bucket 里的文件
错误写法：
{
  "Action": ["s3:ListBucket", "s3:GetObject"],
  "Resource": "arn:aws:s3:::my-bucket"
}
→ GetObject 会失败！因为 Object 需要 my-bucket/*

正确写法：
{
  "Action": "s3:ListBucket",
  "Resource": "arn:aws:s3:::my-bucket"      ← Bucket 级别
},
{
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*"    ← Object 级别
}
```

### 2.4 Condition：在什么条件下

```
Condition 回答：还需要满足什么额外条件？

这是 IAM 最强大也最容易被忽视的特性。

┌─────────────────────────────────────────────────────────┐
│                    Condition 结构                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  "Condition": {                                        │
│      "操作符": {                                        │
│          "条件键": "条件值"                              │
│      }                                                  │
│  }                                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**常用条件操作符：**

```
字符串操作符：
  StringEquals          精确匹配
  StringNotEquals       不等于
  StringLike            支持通配符 (* 和 ?)
  StringNotLike

数值操作符：
  NumericEquals         等于
  NumericNotEquals      不等于
  NumericLessThan       小于
  NumericGreaterThan    大于

日期操作符：
  DateEquals            日期等于
  DateLessThan          日期早于
  DateGreaterThan       日期晚于

布尔操作符：
  Bool                  布尔值匹配

IP 地址操作符：
  IpAddress             IP 在范围内
  NotIpAddress          IP 不在范围内

存在性操作符：
  Null                  键是否存在
```

**常用条件键：**

```
全局条件键（所有服务可用）：

aws:SourceIp              请求来源 IP
aws:CurrentTime           当前时间
aws:SecureTransport       是否使用 HTTPS
aws:MultiFactorAuthPresent 是否使用 MFA
aws:PrincipalArn          请求者的 ARN
aws:RequestedRegion       请求的区域
aws:TagKeys               资源标签键
aws:ResourceTag/tag-key   资源的特定标签

服务特定条件键：

s3:prefix                 S3 对象前缀
s3:x-amz-acl              S3 ACL
ec2:InstanceType          EC2 实例类型
ec2:ResourceTag/tag-key   EC2 资源标签
```

**实际场景示例：**

```json
// 场景1：只允许从公司 IP 访问
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*",
  "Condition": {
    "IpAddress": {
      "aws:SourceIp": ["203.0.113.0/24", "198.51.100.0/24"]
    }
  }
}

// 场景2：必须使用 MFA 才能删除
{
  "Effect": "Deny",
  "Action": "s3:DeleteObject",
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {
      "aws:MultiFactorAuthPresent": "false"
    }
  }
}

// 场景3：只允许在工作时间访问
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "DateGreaterThan": {"aws:CurrentTime": "2024-01-01T09:00:00Z"},
    "DateLessThan": {"aws:CurrentTime": "2024-01-01T18:00:00Z"}
  }
}

// 场景4：只能操作带有特定标签的资源
{
  "Effect": "Allow",
  "Action": "ec2:*",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "ec2:ResourceTag/Environment": "Development"
    }
  }
}

// 场景5：必须使用 HTTPS
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "Bool": {
      "aws:SecureTransport": "false"
    }
  }
}
```

---

## 3. 两种策略类型

### 3.1 Identity-based Policy（基于身份的策略）

```
附加在"身份"上，定义"这个身份能做什么"

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  IAM User: Alice                                       │
│      │                                                  │
│      └── 附加策略：                                     │
│          {                                              │
│            "Effect": "Allow",                          │
│            "Action": "s3:GetObject",                   │
│            "Resource": "arn:aws:s3:::my-bucket/*"      │
│          }                                              │
│                                                         │
│  含义：Alice 可以读取 my-bucket 里的对象               │
│                                                         │
└─────────────────────────────────────────────────────────┘

不需要写 Principal，因为策略附加在谁身上，谁就是 Principal
```

**Identity-based Policy 的三种形式：**

```
1. Inline Policy（内联策略）
   直接嵌入在用户/角色中
   1对1 关系，删除身份时一起删除
   适合：特殊的、不复用的权限

2. Managed Policy - AWS Managed
   AWS 预定义的策略
   如：AdministratorAccess, ReadOnlyAccess
   AWS 会更新维护
   适合：通用场景

3. Managed Policy - Customer Managed
   你自己创建的可复用策略
   可以附加到多个身份
   适合：组织特定的权限模板
```

### 3.2 Resource-based Policy（基于资源的策略）

```
附加在"资源"上，定义"谁能访问这个资源"

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  S3 Bucket: my-bucket                                  │
│      │                                                  │
│      └── Bucket Policy：                               │
│          {                                              │
│            "Effect": "Allow",                          │
│            "Principal": {                              │
│              "AWS": "arn:aws:iam::111122223333:root"   │
│            },                                           │
│            "Action": "s3:GetObject",                   │
│            "Resource": "arn:aws:s3:::my-bucket/*"      │
│          }                                              │
│                                                         │
│  含义：账号 111122223333 可以读取这个 bucket 的对象     │
│                                                         │
└─────────────────────────────────────────────────────────┘

必须写 Principal，因为要指明允许谁
```

**支持 Resource-based Policy 的服务：**

```
S3              → Bucket Policy
SQS             → Queue Policy
SNS             → Topic Policy
Lambda          → Function Policy
KMS             → Key Policy
ECR             → Repository Policy
API Gateway     → Resource Policy
Secrets Manager → Resource Policy
...

注意：不是所有服务都支持 Resource-based Policy
```

### 3.3 两种策略的关系

```
同账号内：Identity-based OR Resource-based 任一允许即可

┌─────────────────────────────────────────────────────────┐
│                      同一账号                           │
│                                                         │
│   Alice ────────────────────────────► S3 Bucket        │
│     │                                      │            │
│     │  Identity Policy                     │  Bucket   │
│     │  (Alice 能访问)                      │  Policy   │
│     │       OR                             │  (允许     │
│     └───────────────── 结果：允许 ─────────│  Alice)   │
│                                                         │
│   只要有一边允许（且没有显式 Deny）就允许               │
│                                                         │
└─────────────────────────────────────────────────────────┘


跨账号：Identity-based AND Resource-based 两边都要允许

┌──────────────────────┐      ┌──────────────────────────┐
│     Account A        │      │       Account B          │
│                      │      │                          │
│  Alice ──────────────│──────│───────► S3 Bucket       │
│    │                 │      │              │           │
│    │ Identity Policy │      │  Bucket Policy           │
│    │ (允许访问       │      │  (允许 Account A/Alice)  │
│    │  B 的资源)      │      │              │           │
│    │                 │      │              │           │
│    └─────── AND ─────│──────│──────────────┘           │
│                      │      │                          │
│   两边都要允许才行    │      │                          │
│                      │      │                          │
└──────────────────────┘      └──────────────────────────┘
```

---

## 4. 策略评估逻辑

### 4.1 评估流程图

```
                     请求到达
                         │
                         ▼
            ┌─────────────────────────┐
            │   是否有显式 Deny？      │
            └─────────────────────────┘
                    │         │
                   Yes        No
                    │         │
                    ▼         ▼
                 ┌─────┐   ┌─────────────────────────┐
                 │拒绝 │   │   是否有显式 Allow？     │
                 └─────┘   └─────────────────────────┘
                                  │         │
                                 Yes        No
                                  │         │
                                  ▼         ▼
                              ┌─────┐   ┌─────┐
                              │允许 │   │拒绝 │
                              └─────┘   └─────┘
                                        (隐式)
```

### 4.2 完整评估顺序

```
AWS 的完整策略评估顺序：

┌─────────────────────────────────────────────────────────┐
│  1. Organization SCPs (服务控制策略)                    │
│     └── 组织层面的最高限制                              │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼ 通过
┌─────────────────────────────────────────────────────────┐
│  2. Resource-based Policy                              │
│     └── 资源是否允许这个 Principal？                    │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼ 通过
┌─────────────────────────────────────────────────────────┐
│  3. Identity-based Policy                              │
│     └── Principal 是否有这个权限？                      │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼ 通过
┌─────────────────────────────────────────────────────────┐
│  4. Permission Boundary                                │
│     └── 是否在权限边界内？                              │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼ 通过
┌─────────────────────────────────────────────────────────┐
│  5. Session Policy (如果是 AssumeRole)                 │
│     └── 会话策略是否允许？                              │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
                      最终结果
```

### 4.3 核心规则总结

```
规则1：默认拒绝 (Implicit Deny)
───────────────────────────────
没有明确的 Allow = 拒绝
系统默认不信任任何请求

规则2：显式拒绝优先 (Explicit Deny Wins)
───────────────────────────────
一条 Allow + 一条 Deny = Deny
Deny 永远赢

规则3：Allow 需要明确 (Explicit Allow Required)
───────────────────────────────
必须有至少一条策略明确 Allow
否则就是隐式 Deny

规则4：权限取交集 (Permission Intersection)
───────────────────────────────
多层策略的最终权限 = 所有层的交集
任何一层限制都会生效
```

---

## 5. 策略设计最佳实践

### 5.1 最小权限模板

```json
// 模板：按功能分离
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    },
    {
      "Sid": "AllowWriteWithCondition",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket/uploads/*"
      ],
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "private"
        }
      }
    }
  ]
}
```

### 5.2 常见反模式

```
反模式1：过度宽泛的权限
──────────────────────
❌ 错误：
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

✅ 正确：只给需要的权限


反模式2：忽略 Resource 限制
──────────────────────
❌ 错误：
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "*"          ← 能读所有 bucket！
}

✅ 正确：指定具体资源


反模式3：缺少 Condition
──────────────────────
❌ 错误：允许从任何地方删除
{
  "Effect": "Allow",
  "Action": "s3:DeleteObject",
  "Resource": "arn:aws:s3:::prod-bucket/*"
}

✅ 正确：要求 MFA + 特定 IP
{
  "Effect": "Allow",
  "Action": "s3:DeleteObject",
  "Resource": "arn:aws:s3:::prod-bucket/*",
  "Condition": {
    "Bool": {"aws:MultiFactorAuthPresent": "true"},
    "IpAddress": {"aws:SourceIp": "10.0.0.0/8"}
  }
}


反模式4：使用 NotAction/NotResource
──────────────────────
❌ 危险：
{
  "Effect": "Allow",
  "NotAction": "iam:*",    ← 除了 IAM 都能做
  "Resource": "*"          ← 包括你没想到的新服务！
}

✅ 正确：明确列出允许的 Action
```

---

## 6. 总结

```
┌─────────────────────────────────────────────────────────┐
│                      IAM 核心模型                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  四要素：                                               │
│    Principal  ─  谁                                     │
│    Action     ─  做什么                                 │
│    Resource   ─  对什么                                 │
│    Condition  ─  什么条件下                             │
│                                                         │
│  两类策略：                                             │
│    Identity-based  ─  附加在身份上                      │
│    Resource-based  ─  附加在资源上                      │
│                                                         │
│  三条规则：                                             │
│    1. 默认拒绝                                          │
│    2. 显式拒绝优先                                      │
│    3. 权限取交集                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

> **核心洞见：IAM 策略不是"给权限"，而是"定义在什么条件下允许什么请求"。理解这四个要素如何组合，就掌握了 IAM 的核心。**

---

## 思考题

1. 为什么 Resource-based Policy 在跨账号场景下必不可少？
2. 设计一个策略：只允许特定 IAM Role 在工作时间从公司 IP 读取 S3 特定前缀的对象
3. 如果 Identity Policy 允许，但 Permission Boundary 拒绝，最终结果是什么？为什么？
