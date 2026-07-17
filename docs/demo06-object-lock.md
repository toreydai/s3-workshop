# Demo06 — S3 Object Lock：WORM 合规存储

## 实验简介

金融、医疗、法律等行业要求数据在保留期内不可删改（WORM，Write-Once-Read-Many）。S3 Object Lock 提供两种锁定模式：Governance（有权限的管理员可覆盖）和 Compliance（任何人包括 root 都无法删除，到期前不可关闭）。本实验配置 Compliance 模式保留策略，模拟合规归档场景。

**实验目标：**
- 掌握 Object Lock 的两种模式（Governance vs Compliance）及其区别
- 理解保留期（Retention Period）和合规保留（Legal Hold）的使用场景
- 能够独立验证锁定对象的不可删改特性

**实验流程：**
1. 创建开启 Object Lock 的存储桶（必须在创建时开启）
2. 配置存储桶默认保留策略
3. 上传对象并验证锁定
4. 尝试删除锁定对象验证保护效果

**预计 AI 执行时长：** 25 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x
- **权限**：`AmazonS3FullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-worm-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建开启 Object Lock 的存储桶

Object Lock **必须在存储桶创建时开启**，无法在已有存储桶上启用。开启 Object Lock 会自动开启版本控制（这是 Object Lock 的前提）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-worm-${ACCOUNT_ID}"
aws s3api create-bucket \
  --bucket "${BUCKET_NAME}" \
  --region us-east-1 \
  --object-lock-enabled-for-bucket
echo "Bucket with Object Lock created: ${BUCKET_NAME}"
```

**预期输出**：`Bucket with Object Lock created: s3-workshop-worm-<ACCOUNT_ID>`

验证 Object Lock 已开启且版本控制自动启用：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-worm-${ACCOUNT_ID}"
aws s3api get-object-lock-configuration \
  --bucket "${BUCKET_NAME}" \
  --query 'ObjectLockConfiguration.ObjectLockEnabled' \
  --output text
```

**预期输出**：`Enabled`

### 2. 配置存储桶默认保留策略（Governance 模式，1 天）

设置默认保留策略后，所有上传的对象自动继承此保留配置，无需每次上传时手动指定。此处使用 `GOVERNANCE` 模式便于演示结束后清理（Compliance 模式到期前连 root 也无法删除）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-worm-${ACCOUNT_ID}"
aws s3api put-object-lock-configuration \
  --bucket "${BUCKET_NAME}" \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "GOVERNANCE",
        "Days": 1
      }
    }
  }'
echo "Default retention policy: GOVERNANCE, 1 day"
```

**预期输出**：`Default retention policy: GOVERNANCE, 1 day`

### 3. 上传对象（自动继承保留策略）

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-worm-${ACCOUNT_ID}"

echo '{"record_id": "TXN-2025-001", "amount": 50000, "status": "COMPLETED"}' > /tmp/transaction.json
aws s3 cp /tmp/transaction.json "s3://${BUCKET_NAME}/transactions/TXN-2025-001.json"
```

**预期输出**：`upload: /tmp/transaction.json to s3://s3-workshop-worm-<ACCOUNT_ID>/transactions/TXN-2025-001.json`

查看对象保留信息：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-worm-${ACCOUNT_ID}"
VERSION_ID=$(aws s3api list-object-versions \
  --bucket "${BUCKET_NAME}" \
  --prefix "transactions/TXN-2025-001.json" \
  --query 'Versions[0].VersionId' \
  --output text)

aws s3api get-object-retention \
  --bucket "${BUCKET_NAME}" \
  --key "transactions/TXN-2025-001.json" \
  --version-id "${VERSION_ID}"
```

**预期输出**：
```json
{
    "Retention": {
        "Mode": "GOVERNANCE",
        "RetainUntilDate": "<tomorrow>"
    }
}
```

### 4. 尝试删除锁定对象（预期失败）

在保留期内删除受 Object Lock 保护的对象版本会报错，这验证了 WORM 的保护效果。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-worm-${ACCOUNT_ID}"
VERSION_ID=$(aws s3api list-object-versions \
  --bucket "${BUCKET_NAME}" \
  --prefix "transactions/TXN-2025-001.json" \
  --query 'Versions[0].VersionId' \
  --output text)

aws s3api delete-object \
  --bucket "${BUCKET_NAME}" \
  --key "transactions/TXN-2025-001.json" \
  --version-id "${VERSION_ID}" 2>&1 || echo "Delete blocked as expected (Object Lock)"
```

**预期输出**：包含 `AccessDenied` 或 `Object Lock` 的错误信息，以及 `Delete blocked as expected (Object Lock)`

### 5. 为对象设置 Legal Hold（法律留存）

Legal Hold 是独立于保留期的另一层保护，用于诉讼保全等场景。开启后即使保留期到期，对象也无法删除；必须显式关闭 Legal Hold 才能清理。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-worm-${ACCOUNT_ID}"

echo '{"record_id": "TXN-2025-002", "status": "UNDER_INVESTIGATION"}' > /tmp/transaction2.json
aws s3 cp /tmp/transaction2.json "s3://${BUCKET_NAME}/transactions/TXN-2025-002.json"

VERSION_ID=$(aws s3api list-object-versions \
  --bucket "${BUCKET_NAME}" \
  --prefix "transactions/TXN-2025-002.json" \
  --query 'Versions[0].VersionId' \
  --output text)

aws s3api put-object-legal-hold \
  --bucket "${BUCKET_NAME}" \
  --key "transactions/TXN-2025-002.json" \
  --version-id "${VERSION_ID}" \
  --legal-hold '{"Status": "ON"}'
echo "Legal Hold enabled on TXN-2025-002.json"
```

**预期输出**：`Legal Hold enabled on TXN-2025-002.json`

验证 Legal Hold 状态：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-worm-${ACCOUNT_ID}"
VERSION_ID=$(aws s3api list-object-versions \
  --bucket "${BUCKET_NAME}" \
  --prefix "transactions/TXN-2025-002.json" \
  --query 'Versions[0].VersionId' \
  --output text)

aws s3api get-object-legal-hold \
  --bucket "${BUCKET_NAME}" \
  --key "transactions/TXN-2025-002.json" \
  --version-id "${VERSION_ID}" \
  --query 'LegalHold.Status' \
  --output text
```

**预期输出**：`ON`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 存储桶 Object Lock 状态为 `Enabled`，默认保留模式为 `GOVERNANCE`，保留期 1 天
- [ ] `TXN-2025-001.json` 有保留策略，尝试删除返回 AccessDenied
- [ ] `TXN-2025-002.json` 的 Legal Hold 状态为 `ON`

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3api get-object-lock-configuration --bucket "s3-workshop-worm-$(aws sts get-caller-identity --query Account --output text)" --query 'ObjectLockConfiguration.Rule.DefaultRetention.Mode' --output text` | `GOVERNANCE` |
| 2 | `aws s3api get-bucket-versioning --bucket "s3-workshop-worm-$(aws sts get-caller-identity --query Account --output text)" --query Status --output text` | `Enabled` |
| 3 | `aws s3api list-object-versions --bucket "s3-workshop-worm-$(aws sts get-caller-identity --query Account --output text)" --query 'length(Versions)' --output text` | `2` |

---

## 实验总结

本实验构建了 S3 WORM 合规存储：Object Lock Governance 模式适用于内部数据治理（管理员可覆盖），Compliance 模式适用于强合规场景（任何人都无法删除，包括 AWS 支持）。Legal Hold 提供了叠加保护，用于法律程序中的证据保全。两者结合可以满足 SEC Rule 17a-4、FINRA 等金融监管要求。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-worm-${ACCOUNT_ID}"

# 先关闭 Legal Hold
VERSION_ID=$(aws s3api list-object-versions \
  --bucket "${BUCKET_NAME}" \
  --prefix "transactions/TXN-2025-002.json" \
  --query 'Versions[0].VersionId' --output text 2>/dev/null)
[ -n "${VERSION_ID}" ] && aws s3api put-object-legal-hold \
  --bucket "${BUCKET_NAME}" \
  --key "transactions/TXN-2025-002.json" \
  --version-id "${VERSION_ID}" \
  --legal-hold '{"Status": "OFF"}'

# Governance 模式：使用 bypass 删除（需要 s3:BypassGovernanceRetention 权限）
for key in "transactions/TXN-2025-001.json" "transactions/TXN-2025-002.json"; do
  VID=$(aws s3api list-object-versions \
    --bucket "${BUCKET_NAME}" --prefix "${key}" \
    --query 'Versions[0].VersionId' --output text 2>/dev/null)
  [ -n "${VID}" ] && aws s3api delete-object \
    --bucket "${BUCKET_NAME}" --key "${key}" --version-id "${VID}" \
    --bypass-governance-retention 2>/dev/null
done

aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Cleanup complete"
```
