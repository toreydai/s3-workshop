# Demo10 — S3 访问点（Access Points）与多租户权限隔离

## 实验简介

当多个团队共用同一个 S3 存储桶时，用单一 Bucket Policy 管理所有人的权限会变得极其复杂。S3 访问点（Access Points）为每个团队或应用提供独立的 ARN 和策略，每个访问点只能访问桶内特定前缀，实现多租户权限的清晰隔离，而无需拆分存储桶。

**实验目标：**
- 掌握 S3 访问点的创建与策略配置
- 理解访问点策略与 Bucket Policy 的关系（两者都需放行才能访问）
- 能够为不同团队创建独立访问点并验证隔离效果

**实验流程：**
1. 创建共享存储桶并上传各团队数据
2. 为两个团队分别创建访问点和 IAM 用户/角色
3. 配置访问点策略限定各团队只能访问自己的前缀
4. 配置 Bucket Policy 委托给访问点
5. 验证隔离效果

**预计时长：** 30 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x
- **权限**：`AmazonS3FullAccess`、`IAMFullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-shared-${ACCOUNT_ID}"
echo "Shared Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建共享存储桶并上传各团队数据

模拟两个团队（finance 和 engineering）共用同一个存储桶，各自将数据放在不同前缀下。访问点策略将限制每个团队只能看到自己的前缀。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-shared-${ACCOUNT_ID}"
aws s3api create-bucket --bucket "${BUCKET_NAME}" --region us-east-1

# 上传 finance 团队数据
echo '{"team": "finance", "data": "Q4-2025 Revenue Report"}' > /tmp/finance-report.json
aws s3 cp /tmp/finance-report.json "s3://${BUCKET_NAME}/finance/reports/q4-2025.json"

# 上传 engineering 团队数据
echo '{"team": "engineering", "data": "Build artifacts v2.1.0"}' > /tmp/eng-artifact.json
aws s3 cp /tmp/eng-artifact.json "s3://${BUCKET_NAME}/engineering/artifacts/v2.1.0.json"

aws s3 ls "s3://${BUCKET_NAME}" --recursive
```

**预期输出**：
```
<date> <time>    <size> engineering/artifacts/v2.1.0.json
<date> <time>    <size> finance/reports/q4-2025.json
```

### 2. 为 Finance 团队创建访问点

访问点的 `--name` 在账号+区域内唯一。`VpcConfiguration` 可以将访问点限制为只能从特定 VPC 访问（此处使用 Internet 访问模式用于演示）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-shared-${ACCOUNT_ID}"

aws s3control create-access-point \
  --account-id "${ACCOUNT_ID}" \
  --name "finance-access-point" \
  --bucket "${BUCKET_NAME}" \
  --public-access-block-configuration '{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }'
echo "Finance access point created"
```

**预期输出**：`Finance access point created`

### 3. 为 Finance 访问点配置策略（只允许访问 finance/ 前缀）

访问点策略的 `Resource` 必须同时包含访问点 ARN 和带 `/object/` 路径的对象 ARN，格式与 Bucket Policy 不同，混淆这两种格式是常见错误。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AP_ARN="arn:aws:s3:us-east-1:${ACCOUNT_ID}:accesspoint/finance-access-point"

aws s3control put-access-point-policy \
  --account-id "${ACCOUNT_ID}" \
  --name "finance-access-point" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Principal\": {\"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\"},
      \"Action\": [\"s3:GetObject\", \"s3:PutObject\", \"s3:ListBucket\"],
      \"Resource\": [
        \"${AP_ARN}\",
        \"${AP_ARN}/object/finance/*\"
      ]
    }]
  }"
echo "Finance access point policy configured"
```

**预期输出**：`Finance access point policy configured`

### 4. 为 Engineering 团队创建访问点与策略

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-shared-${ACCOUNT_ID}"

aws s3control create-access-point \
  --account-id "${ACCOUNT_ID}" \
  --name "engineering-access-point" \
  --bucket "${BUCKET_NAME}" \
  --public-access-block-configuration '{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }'

AP_ARN="arn:aws:s3:us-east-1:${ACCOUNT_ID}:accesspoint/engineering-access-point"
aws s3control put-access-point-policy \
  --account-id "${ACCOUNT_ID}" \
  --name "engineering-access-point" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Principal\": {\"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\"},
      \"Action\": [\"s3:GetObject\", \"s3:PutObject\", \"s3:ListBucket\"],
      \"Resource\": [
        \"${AP_ARN}\",
        \"${AP_ARN}/object/engineering/*\"
      ]
    }]
  }"
echo "Engineering access point created and policy configured"
```

**预期输出**：`Engineering access point created and policy configured`

### 5. 配置 Bucket Policy 将权限委托给访问点

Bucket Policy 必须显式允许来自访问点的请求，否则即使访问点策略放行，S3 也会在存储桶层面拒绝。委托给访问点的 Bucket Policy 格式是固定写法，`Principal` 为 `*`，通过 `Condition` 限定访问点 ARN 前缀。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-shared-${ACCOUNT_ID}"

aws s3api put-bucket-policy \
  --bucket "${BUCKET_NAME}" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Principal\": \"*\",
      \"Action\": \"s3:*\",
      \"Resource\": [
        \"arn:aws:s3:::${BUCKET_NAME}\",
        \"arn:aws:s3:::${BUCKET_NAME}/*\"
      ],
      \"Condition\": {
        \"StringLike\": {
          \"s3:DataAccessPointArn\": \"arn:aws:s3:us-east-1:${ACCOUNT_ID}:accesspoint/*\"
        }
      }
    }]
  }"
echo "Bucket policy delegates to access points"
```

**预期输出**：`Bucket policy delegates to access points`

### 6. 通过访问点 ARN 访问对象

通过访问点访问对象时，使用访问点 ARN 代替存储桶名称，格式为 `arn:aws:s3:region:account:accesspoint/name`。CLI 中用 `--bucket` 传入 ARN 即可。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
FINANCE_AP_ARN="arn:aws:s3:us-east-1:${ACCOUNT_ID}:accesspoint/finance-access-point"

# 通过 Finance 访问点读取 finance 数据
aws s3api get-object \
  --bucket "${FINANCE_AP_ARN}" \
  --key "finance/reports/q4-2025.json" \
  /tmp/finance-downloaded.json \
  --region us-east-1
cat /tmp/finance-downloaded.json
```

**预期输出**：`{"team": "finance", "data": "Q4-2025 Revenue Report"}`

验证访问点数量：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws s3control list-access-points \
  --account-id "${ACCOUNT_ID}" \
  --query 'AccessPointList[].Name' \
  --output text
```

**预期输出**：`engineering-access-point	finance-access-point`（顺序可能不同）

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 共享存储桶中包含 `finance/` 和 `engineering/` 两个前缀的对象
- [ ] 两个访问点（`finance-access-point`、`engineering-access-point`）均已创建
- [ ] 通过 Finance 访问点 ARN 成功读取 `finance/reports/q4-2025.json`
- [ ] Bucket Policy 已配置访问点委托

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3control list-access-points --account-id "$(aws sts get-caller-identity --query Account --output text)" --query 'length(AccessPointList)' --output text` | `2` |
| 2 | `aws s3control get-access-point --account-id "$(aws sts get-caller-identity --query Account --output text)" --name "finance-access-point" --query NetworkOrigin --output text` | `Internet` |
| 3 | `aws s3 ls "s3://s3-workshop-shared-$(aws sts get-caller-identity --query Account --output text)" --recursive --summarize \| grep "Total Objects"` | `Total Objects: 2` |

---

## 实验总结

本实验用 S3 访问点解决了多租户权限隔离的经典难题：不拆分存储桶、不复杂化 Bucket Policy，而是给每个团队独立的访问点 ARN 和策略，前缀级别的权限隔离清晰且可扩展。关键记忆点：访问点策略和 Bucket Policy 是"双重门锁"，两者都必须放行请求才能通过。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-shared-${ACCOUNT_ID}"

# 删除访问点
aws s3control delete-access-point --account-id "${ACCOUNT_ID}" --name "finance-access-point"
aws s3control delete-access-point --account-id "${ACCOUNT_ID}" --name "engineering-access-point"

# 删除存储桶
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Cleanup complete"
```
