# Demo08 — S3 安全与访问控制：Bucket Policy、加密与 Block Public Access

## 实验简介

在企业生产环境中，S3 存储桶安全配置不到位是最常见的数据泄露来源之一。本实验覆盖 S3 的三层防御体系：Block Public Access（全局开关）、Bucket Policy（细粒度访问控制）、以及服务端加密（SSE-S3/SSE-KMS），帮助你建立符合合规要求的存储安全基线。

**实验目标：**
- 掌握 Block Public Access 的作用范围与开启方式
- 理解 Bucket Policy 的条件判断（`Condition`）与最小权限原则
- 能够独立配置 SSE-KMS 加密并验证对象加密状态

**实验流程：**
1. 创建存储桶并开启 Block Public Access
2. 配置强制 HTTPS 的 Bucket Policy
3. 配置 SSE-KMS 默认加密
4. 验证加密状态与访问控制效果

**预计时长：** 25 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x
- **权限**：`AmazonS3FullAccess`、`AWSKMSFullAccess`（创建 KMS Key）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-security-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建存储桶

创建实验用存储桶，后续在此桶上逐层叠加安全配置。此步骤不带任何安全参数，基线状态由下面的步骤逐步加固。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-security-${ACCOUNT_ID}"
aws s3api create-bucket \
  --bucket "${BUCKET_NAME}" \
  --region us-east-1
```

**预期输出**：
```json
{
    "Location": "/s3-workshop-security-<ACCOUNT_ID>"
}
```

### 2. 开启 Block Public Access

Block Public Access 是 S3 的全局安全开关，优先级高于 Bucket Policy 和 ACL。即便 Bucket Policy 允许公开访问，只要 `BlockPublicPolicy: true`，策略也不会生效。新账号的默认设置已为 `true`，但显式配置可以防止被人意外修改。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-security-${ACCOUNT_ID}"
aws s3api put-public-access-block \
  --bucket "${BUCKET_NAME}" \
  --public-access-block-configuration '{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }'
```

**预期输出**：（无输出表示成功）

验证配置：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-security-${ACCOUNT_ID}"
aws s3api get-public-access-block \
  --bucket "${BUCKET_NAME}" \
  --query 'PublicAccessBlockConfiguration'
```

**预期输出**：
```json
{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
}
```

### 3. 配置强制 HTTPS 的 Bucket Policy

强制 HTTPS 策略通过 `aws:SecureTransport: false` 条件拒绝所有 HTTP 请求，即使请求方有合法凭证也会被拒绝。这是 CIS AWS Foundations Benchmark 和 PCI-DSS 要求的基线控制项。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-security-${ACCOUNT_ID}"

aws s3api put-bucket-policy \
  --bucket "${BUCKET_NAME}" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
        \"Sid\": \"DenyNonHTTPS\",
        \"Effect\": \"Deny\",
        \"Principal\": \"*\",
        \"Action\": \"s3:*\",
        \"Resource\": [
          \"arn:aws:s3:::${BUCKET_NAME}\",
          \"arn:aws:s3:::${BUCKET_NAME}/*\"
        ],
        \"Condition\": {
          \"Bool\": {
            \"aws:SecureTransport\": \"false\"
          }
        }
      },
      {
        \"Sid\": \"AllowAccountAccess\",
        \"Effect\": \"Allow\",
        \"Principal\": {\"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\"},
        \"Action\": \"s3:*\",
        \"Resource\": [
          \"arn:aws:s3:::${BUCKET_NAME}\",
          \"arn:aws:s3:::${BUCKET_NAME}/*\"
        ]
      }
    ]
  }"
```

**预期输出**：（无输出表示成功）

验证策略已设置：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-security-${ACCOUNT_ID}"
aws s3api get-bucket-policy-status \
  --bucket "${BUCKET_NAME}" \
  --query 'PolicyStatus.IsPublic' \
  --output text
```

**预期输出**：`False`（策略不公开，符合预期）

### 4. 创建 KMS 密钥用于存储桶加密

SSE-KMS 使用客户管理的 KMS Key（CMK）加密对象，相比 SSE-S3 提供更细粒度的密钥访问控制、使用审计和轮换能力。`enable-key-rotation` 让 AWS 每年自动轮换密钥材料。

```bash
KEY_ID=$(aws kms create-key \
  --description "S3 Workshop Encryption Key" \
  --region us-east-1 \
  --query 'KeyMetadata.KeyId' \
  --output text)
echo "KMS Key ID: ${KEY_ID}"

# 创建别名便于识别
aws kms create-alias \
  --alias-name "alias/s3-workshop-key" \
  --target-key-id "${KEY_ID}" \
  --region us-east-1

# 开启自动轮换
aws kms enable-key-rotation \
  --key-id "${KEY_ID}" \
  --region us-east-1

echo "Key rotation enabled for: ${KEY_ID}"
```

**预期输出**：
```
KMS Key ID: <uuid>
Key rotation enabled for: <uuid>
```

### 5. 配置存储桶默认 SSE-KMS 加密

设置存储桶级别的默认加密后，所有后续上传的对象自动使用指定 KMS Key 加密，无需上传时显式指定。`BucketKeyEnabled: true` 减少对 KMS API 的调用次数，降低费用。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-security-${ACCOUNT_ID}"
KEY_ARN=$(aws kms describe-key \
  --key-id "alias/s3-workshop-key" \
  --region us-east-1 \
  --query 'KeyMetadata.Arn' \
  --output text)

aws s3api put-bucket-encryption \
  --bucket "${BUCKET_NAME}" \
  --server-side-encryption-configuration "{
    \"Rules\": [{
      \"ApplyServerSideEncryptionByDefault\": {
        \"SSEAlgorithm\": \"aws:kms\",
        \"KMSMasterKeyID\": \"${KEY_ARN}\"
      },
      \"BucketKeyEnabled\": true
    }]
  }"
echo "Encryption configured with KMS Key: ${KEY_ARN}"
```

**预期输出**：`Encryption configured with KMS Key: arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/<uuid>`

验证加密配置：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-security-${ACCOUNT_ID}"
aws s3api get-bucket-encryption \
  --bucket "${BUCKET_NAME}" \
  --query 'ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.SSEAlgorithm' \
  --output text
```

**预期输出**：`aws:kms`

### 6. 上传对象并验证加密状态

上传一个测试对象，然后通过 `head-object` 查看对象的加密算法和 KMS Key 信息，确认 SSE-KMS 生效。

```bash
echo "Sensitive data - encrypted with KMS" > /tmp/sensitive.txt
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-security-${ACCOUNT_ID}"
aws s3 cp /tmp/sensitive.txt "s3://${BUCKET_NAME}/sensitive.txt"
```

**预期输出**：`upload: /tmp/sensitive.txt to s3://s3-workshop-security-<ACCOUNT_ID>/sensitive.txt`

查看对象加密信息：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-security-${ACCOUNT_ID}"
aws s3api head-object \
  --bucket "${BUCKET_NAME}" \
  --key "sensitive.txt" \
  --query '{Encryption: ServerSideEncryption, KMSKeyId: SSEKMSKeyId}'
```

**预期输出**：
```json
{
    "Encryption": "aws:kms",
    "KMSKeyId": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/<uuid>"
}
```

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 存储桶 `s3-workshop-security-<ACCOUNT_ID>` Block Public Access 全部为 `true`
- [ ] Bucket Policy 包含 `DenyNonHTTPS` 语句，强制 HTTPS 访问
- [ ] 存储桶默认加密算法为 `aws:kms`，使用 `alias/s3-workshop-key`
- [ ] 上传的 `sensitive.txt` 对象加密算法显示 `aws:kms`

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3api get-public-access-block --bucket "s3-workshop-security-$(aws sts get-caller-identity --query Account --output text)" --query 'PublicAccessBlockConfiguration.BlockPublicPolicy' --output text` | `True` |
| 2 | `aws s3api get-bucket-encryption --bucket "s3-workshop-security-$(aws sts get-caller-identity --query Account --output text)" --query 'ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.SSEAlgorithm' --output text` | `aws:kms` |
| 3 | `aws s3api head-object --bucket "s3-workshop-security-$(aws sts get-caller-identity --query Account --output text)" --key "sensitive.txt" --query 'ServerSideEncryption' --output text` | `aws:kms` |
| 4 | `aws s3api get-bucket-policy-status --bucket "s3-workshop-security-$(aws sts get-caller-identity --query Account --output text)" --query 'PolicyStatus.IsPublic' --output text` | `False` |
| 5 | `aws kms describe-key --key-id "alias/s3-workshop-key" --region us-east-1 --query 'KeyMetadata.KeyState' --output text` | `Enabled` |

---

## 实验总结

本实验构建了 S3 的三层安全防护体系：Block Public Access 作为最高优先级的全局开关防止意外公开，Bucket Policy 的 `DenyNonHTTPS` 条件强制传输加密，SSE-KMS 保障静态数据加密并提供完整的密钥审计轨迹。这三者共同覆盖了 CIS AWS Foundations Benchmark 对 S3 的核心要求。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-security-${ACCOUNT_ID}"

# 删除主存储桶
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket-policy --bucket "${BUCKET_NAME}"
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1

# 删除 KMS Key（计划删除，最少等待 7 天）
KEY_ID=$(aws kms describe-key \
  --key-id "alias/s3-workshop-key" \
  --region us-east-1 \
  --query 'KeyMetadata.KeyId' \
  --output text 2>/dev/null)
if [ -n "${KEY_ID}" ]; then
  aws kms delete-alias --alias-name "alias/s3-workshop-key" --region us-east-1
  aws kms schedule-key-deletion \
    --key-id "${KEY_ID}" \
    --pending-window-in-days 7 \
    --region us-east-1
  echo "KMS Key scheduled for deletion in 7 days: ${KEY_ID}"
fi

echo "Cleanup complete"
```
