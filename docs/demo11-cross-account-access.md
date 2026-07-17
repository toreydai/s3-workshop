# Demo11 — 跨账号 S3 访问：多账号架构权限模式

## 实验简介

企业多账号架构（AWS Organizations / Landing Zone）中，数据几乎从不只属于一个账号：数据平台账号存储原始数据，应用账号需要读取，安全账号需要审计。S3 的跨账号访问有三种主要模式：Bucket Policy 授权、IAM 角色信任 + Assume Role、以及访问点（Access Point）委托。本实验覆盖最常用的两种，并演示跨账号加密（KMS）的关键授权细节。

**实验目标：**
- 掌握 Bucket Policy 直接授权跨账号访问（适合简单只读场景）
- 掌握 IAM 角色 Assume Role 模式（适合精细权限 + 审计追踪场景）
- 理解跨账号 KMS 加密的双重授权要求（KMS Key Policy + IAM Policy）

**实验流程：**
1. 用 IAM 角色模拟"外部账号"身份
2. 配置 Bucket Policy 授权跨账号读取
3. 配置 IAM 角色的 Assume Role 跨账号访问
4. 演示跨账号 KMS 加密的授权模式

**预计 AI 执行时长：** 30 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x
- **权限**：`AmazonS3FullAccess`、`IAMFullAccess`、`AWSKMSFullAccess`
- **说明**：本实验在单账号内用两个 IAM 角色模拟跨账号场景（账号 A = 数据桶拥有者，账号 B = 访问方），生产中将角色替换为实际的外部账号 ID 即可

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export DATA_BUCKET="s3-workshop-xacct-data-${ACCOUNT_ID}"
echo "Data Bucket (Account A): ${DATA_BUCKET}"
```

---

## 步骤

### 1. 创建数据桶（模拟账号 A 的资源）

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-xacct-data-${ACCOUNT_ID}"
aws s3api create-bucket --bucket "${DATA_BUCKET}" --region us-east-1

echo '{"dataset": "sales-2025", "records": 50000}' > /tmp/sales-data.json
aws s3 cp /tmp/sales-data.json "s3://${DATA_BUCKET}/datasets/sales-2025.json"
echo "Data bucket ready: ${DATA_BUCKET}"
```

**预期输出**：`Data bucket ready: s3-workshop-xacct-data-<ACCOUNT_ID>`

### 2. 创建"外部账号"模拟角色

在生产中，这个角色会属于真实的外部账号（替换 `Principal.AWS` 中的账号 ID）。此处在同一账号内创建独立角色模拟跨账号访问方。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

READER_ROLE_ARN=$(aws iam create-role \
  --role-name "xacct-data-reader-role" \
  --assume-role-policy-document "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Principal\": {\"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\"},
      \"Action\": \"sts:AssumeRole\",
      \"Condition\": {}
    }]
  }" \
  --description "Simulates cross-account data reader (Account B)" \
  --query 'Role.Arn' --output text)
echo "Reader role (Account B simulation): ${READER_ROLE_ARN}"
sleep 10
```

**预期输出**：`Reader role (Account B simulation): arn:aws:iam::<ACCOUNT_ID>:role/xacct-data-reader-role`

> ⚠️ **IAM 角色传播延迟影响 Bucket Policy 校验**：新建 IAM 角色后立即在 Bucket Policy 的 `Principal.AWS` 中引用该角色 ARN，可能报 `MalformedPolicy: Invalid principal in policy`——S3 会校验 Principal 角色是否已在 IAM 中可查，和 Lambda 执行角色的传播延迟是同类问题。创建角色后等待约 10 秒再执行下一步的 `put-bucket-policy`，如仍报错间隔几秒重试即可。

### 3. 模式一：Bucket Policy 直接授权（适合只读、无需 Assume Role）

最简单的跨账号访问：在 Bucket Policy 中直接列出外部账号的角色 ARN。外部账号持有者用自己的凭证即可访问，不需要 Assume Role。适合给合作伙伴提供只读数据访问。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-xacct-data-${ACCOUNT_ID}"
READER_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/xacct-data-reader-role"

aws s3api put-bucket-policy \
  --bucket "${DATA_BUCKET}" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
        \"Sid\": \"AllowOwnerFullAccess\",
        \"Effect\": \"Allow\",
        \"Principal\": {\"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\"},
        \"Action\": \"s3:*\",
        \"Resource\": [
          \"arn:aws:s3:::${DATA_BUCKET}\",
          \"arn:aws:s3:::${DATA_BUCKET}/*\"
        ]
      },
      {
        \"Sid\": \"AllowCrossAccountRead\",
        \"Effect\": \"Allow\",
        \"Principal\": {\"AWS\": \"${READER_ROLE_ARN}\"},
        \"Action\": [\"s3:GetObject\", \"s3:ListBucket\"],
        \"Resource\": [
          \"arn:aws:s3:::${DATA_BUCKET}\",
          \"arn:aws:s3:::${DATA_BUCKET}/datasets/*\"
        ]
      }
    ]
  }"
echo "Bucket Policy: cross-account read granted to ${READER_ROLE_ARN}"
```

**预期输出**：`Bucket Policy: cross-account read granted to arn:aws:iam::<ACCOUNT_ID>:role/xacct-data-reader-role`

为 Reader 角色添加最小 S3 权限（生产中外部账号自己管理，此处需手动添加）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-xacct-data-${ACCOUNT_ID}"

aws iam put-role-policy \
  --role-name "xacct-data-reader-role" \
  --policy-name "s3-read-policy" \
  --policy-document "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Action\": [\"s3:GetObject\", \"s3:ListBucket\"],
      \"Resource\": [
        \"arn:aws:s3:::${DATA_BUCKET}\",
        \"arn:aws:s3:::${DATA_BUCKET}/*\"
      ]
    }]
  }"
echo "Reader role IAM policy attached"
```

**预期输出**：`Reader role IAM policy attached`

验证跨账号访问（Assume Role 后读取数据）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-xacct-data-${ACCOUNT_ID}"
READER_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/xacct-data-reader-role"

# 模拟 Account B 的操作：先 AssumeRole
CREDS=$(aws sts assume-role \
  --role-arn "${READER_ROLE_ARN}" \
  --role-session-name "cross-account-session" \
  --query 'Credentials' --output json)

export AWS_ACCESS_KEY_ID=$(echo "${CREDS}" | python3 -c "import json,sys; print(json.load(sys.stdin)['AccessKeyId'])")
export AWS_SECRET_ACCESS_KEY=$(echo "${CREDS}" | python3 -c "import json,sys; print(json.load(sys.stdin)['SecretAccessKey'])")
export AWS_SESSION_TOKEN=$(echo "${CREDS}" | python3 -c "import json,sys; print(json.load(sys.stdin)['SessionToken'])")

# 用 Reader 身份列出存储桶
echo "=== 以 Reader Role 身份列出对象 ==="
aws s3 ls "s3://${DATA_BUCKET}/datasets/"

# 读取数据
echo "=== 读取数据内容 ==="
aws s3 cp "s3://${DATA_BUCKET}/datasets/sales-2025.json" -

# 恢复原始凭证
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
echo "=== 恢复原始身份 ==="
```

**预期输出**：
```
=== 以 Reader Role 身份列出对象 ===
<date> <time>    <size> sales-2025.json
=== 读取数据内容 ===
{"dataset": "sales-2025", "records": 50000}
=== 恢复原始身份 ===
```

### 4. 验证 Reader 角色无法写入（权限边界生效）

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-xacct-data-${ACCOUNT_ID}"
READER_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/xacct-data-reader-role"

CREDS=$(aws sts assume-role --role-arn "${READER_ROLE_ARN}" \
  --role-session-name "write-test" --query 'Credentials' --output json)
export AWS_ACCESS_KEY_ID=$(echo "${CREDS}" | python3 -c "import json,sys; print(json.load(sys.stdin)['AccessKeyId'])")
export AWS_SECRET_ACCESS_KEY=$(echo "${CREDS}" | python3 -c "import json,sys; print(json.load(sys.stdin)['SecretAccessKey'])")
export AWS_SESSION_TOKEN=$(echo "${CREDS}" | python3 -c "import json,sys; print(json.load(sys.stdin)['SessionToken'])")

echo "test" | aws s3 cp - "s3://${DATA_BUCKET}/datasets/malicious.txt" 2>&1 || \
  echo "Write blocked as expected (AccessDenied)"

unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

**预期输出**：`Write blocked as expected (AccessDenied)`

### 5. 模式二：跨账号 KMS 加密访问

当数据桶使用 KMS 加密时，跨账号访问需要**两层授权**都到位，缺一不可：
- S3 Bucket Policy 允许外部账号读取对象
- KMS Key Policy 允许外部账号使用密钥解密

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
READER_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/xacct-data-reader-role"

# 创建 KMS Key，并在 Key Policy 中授权 Reader 角色解密
KEY_ID=$(aws kms create-key \
  --description "S3 Cross-Account Encryption Key" \
  --region us-east-1 \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
        \"Sid\": \"AllowOwner\",
        \"Effect\": \"Allow\",
        \"Principal\": {\"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\"},
        \"Action\": \"kms:*\",
        \"Resource\": \"*\"
      },
      {
        \"Sid\": \"AllowCrossAccountDecrypt\",
        \"Effect\": \"Allow\",
        \"Principal\": {\"AWS\": \"${READER_ROLE_ARN}\"},
        \"Action\": [\"kms:Decrypt\", \"kms:DescribeKey\"],
        \"Resource\": \"*\"
      }
    ]
  }" \
  --query 'KeyMetadata.KeyId' --output text)

aws kms create-alias --alias-name "alias/s3-xacct-key" \
  --target-key-id "${KEY_ID}" --region us-east-1

echo "KMS Key created with cross-account decrypt permission: ${KEY_ID}"
```

**预期输出**：`KMS Key created with cross-account decrypt permission: <key-id>`

上传 KMS 加密对象：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-xacct-data-${ACCOUNT_ID}"
KEY_ARN=$(aws kms describe-key --key-id "alias/s3-xacct-key" \
  --region us-east-1 --query 'KeyMetadata.Arn' --output text)

echo '{"encrypted": true, "sensitive": "cross-account KMS test"}' > /tmp/encrypted-data.json
aws s3 cp /tmp/encrypted-data.json \
  "s3://${DATA_BUCKET}/encrypted/data.json" \
  --sse "aws:kms" \
  --sse-kms-key-id "${KEY_ARN}"
echo "KMS-encrypted object uploaded"
```

**预期输出**：`KMS-encrypted object uploaded`

用 Reader 角色解密读取：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-xacct-data-${ACCOUNT_ID}"
READER_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/xacct-data-reader-role"

# 为 Reader 角色追加 KMS 权限（IAM 层）
aws iam put-role-policy \
  --role-name "xacct-data-reader-role" \
  --policy-name "kms-decrypt-policy" \
  --policy-document "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Action\": [\"kms:Decrypt\", \"kms:DescribeKey\"],
      \"Resource\": \"$(aws kms describe-key --key-id alias/s3-xacct-key --region us-east-1 --query 'KeyMetadata.Arn' --output text)\"
    }]
  }"

CREDS=$(aws sts assume-role --role-arn "${READER_ROLE_ARN}" \
  --role-session-name "kms-test" --query 'Credentials' --output json)
export AWS_ACCESS_KEY_ID=$(echo "${CREDS}" | python3 -c "import json,sys; print(json.load(sys.stdin)['AccessKeyId'])")
export AWS_SECRET_ACCESS_KEY=$(echo "${CREDS}" | python3 -c "import json,sys; print(json.load(sys.stdin)['SecretAccessKey'])")
export AWS_SESSION_TOKEN=$(echo "${CREDS}" | python3 -c "import json,sys; print(json.load(sys.stdin)['SessionToken'])")

aws s3 cp "s3://${DATA_BUCKET}/encrypted/data.json" -

unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
echo "KMS cross-account decrypt successful"
```

**预期输出**：`{"encrypted": true, "sensitive": "cross-account KMS test"}`

---

## 跨账号访问模式速查

| 场景 | 推荐模式 | 关键配置 |
|------|---------|---------|
| 只读静态数据共享 | Bucket Policy 直接授权 | Principal 填外部账号/角色 ARN |
| 需要审计追踪、精细权限 | Assume Role | 信任策略 + Bucket Policy 双放行 |
| 多团队共用同一桶 | Access Points（Demo10） | 每团队独立 AP ARN |
| KMS 加密桶跨账号读取 | Bucket Policy + Key Policy | 两层都要授权，缺一报 AccessDenied |

---

## 验收标准

完成本实验后，你应当能够：
- [ ] `xacct-data-reader-role` 通过 Assume Role 成功读取 `datasets/sales-2025.json`
- [ ] 同一角色尝试写入被 AccessDenied 拒绝
- [ ] Reader 角色通过 KMS Key Policy + IAM Policy 双重授权后成功解密 KMS 加密对象

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws iam get-role --role-name xacct-data-reader-role --query 'Role.RoleName' --output text` | `xacct-data-reader-role` |
| 2 | `aws s3api get-bucket-policy --bucket "s3-workshop-xacct-data-$(aws sts get-caller-identity --query Account --output text)" --query 'Policy' --output text \| python3 -c "import json,sys; p=json.loads(sys.stdin.read()); print(len(p['Statement']))"` | `2` |
| 3 | `aws kms describe-key --key-id "alias/s3-xacct-key" --region us-east-1 --query 'KeyMetadata.KeyState' --output text` | `Enabled` |
| 4 | `aws s3 ls "s3://s3-workshop-xacct-data-$(aws sts get-caller-identity --query Account --output text)/" --recursive --summarize \| grep "Total Objects"` | `Total Objects: 2` |

---

## 实验总结

本实验覆盖了企业多账号架构中最常见的 S3 跨账号访问模式。核心规则：**Bucket Policy 控制谁能进桶，IAM Policy 控制角色能做什么，两者取交集**；KMS 加密再加一层 Key Policy 控制谁能解密。生产建议：使用 Assume Role 模式而非直接在 Bucket Policy 里写死凭证，这样可以通过 IAM 和 CloudTrail 追踪每次访问的身份和操作，满足合规审计要求。至此 20 个 Demo 全部完成，覆盖了 S3 从基础到企业级的完整实践路径。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-xacct-data-${ACCOUNT_ID}"

# 删除存储桶
aws s3 rm "s3://${DATA_BUCKET}" --recursive
aws s3api delete-bucket --bucket "${DATA_BUCKET}" --region us-east-1

# 删除 IAM 角色
for policy in "s3-read-policy" "kms-decrypt-policy"; do
  aws iam delete-role-policy --role-name "xacct-data-reader-role" \
    --policy-name "${policy}" 2>/dev/null
done
aws iam delete-role --role-name "xacct-data-reader-role"

# 删除 KMS Key（计划删除，7 天后生效）
KEY_ID=$(aws kms describe-key --key-id "alias/s3-xacct-key" \
  --region us-east-1 --query 'KeyMetadata.KeyId' --output text 2>/dev/null)
if [ -n "${KEY_ID}" ]; then
  aws kms delete-alias --alias-name "alias/s3-xacct-key" --region us-east-1
  aws kms schedule-key-deletion \
    --key-id "${KEY_ID}" --pending-window-in-days 7 --region us-east-1
  echo "KMS Key scheduled for deletion: ${KEY_ID}"
fi

echo "Cleanup complete"
```
