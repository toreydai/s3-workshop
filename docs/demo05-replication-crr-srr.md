# Demo05 — 跨区域复制（CRR）与同区域复制（SRR）

## 实验简介

S3 复制功能自动将对象从源存储桶异步复制到目标存储桶，分为跨区域复制（CRR，Cross-Region Replication）和同区域复制（SRR，Same-Region Replication）。CRR 用于灾难恢复和合规数据驻留，SRR 用于日志聚合和开发/测试数据同步。本实验配置 CRR 将 us-east-1 的数据自动复制到 us-west-2。

**实验目标：**
- 掌握 S3 复制规则的配置（复制条件、目标存储类、加密传播）
- 理解复制的前提条件：两端都需开启版本控制，需要专属 IAM 角色
- 能够独立验证复制状态并分析延迟

**实验流程：**
1. 在两个区域创建源桶和目标桶，均开启版本控制
2. 创建复制 IAM 角色
3. 配置 CRR 规则
4. 上传对象验证复制

**预计 AI 执行时长：** 30 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x
- **权限**：`AmazonS3FullAccess`、`IAMFullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SOURCE_BUCKET="s3-workshop-crr-source-${ACCOUNT_ID}"
export DEST_BUCKET="s3-workshop-crr-dest-${ACCOUNT_ID}"
export DEST_REGION="us-west-2"
echo "Source: ${SOURCE_BUCKET} (us-east-1) -> Dest: ${DEST_BUCKET} (${DEST_REGION})"
```

---

## 步骤

### 1. 创建源存储桶并开启版本控制

CRR 要求源桶和目标桶**都必须**开启版本控制，这是 S3 复制的硬前提条件，缺少任意一端就会报 `InvalidRequest`。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-crr-source-${ACCOUNT_ID}"

aws s3api create-bucket \
  --bucket "${SOURCE_BUCKET}" \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket "${SOURCE_BUCKET}" \
  --versioning-configuration Status=Enabled

echo "Source bucket ready: ${SOURCE_BUCKET}"
```

**预期输出**：`Source bucket ready: s3-workshop-crr-source-<ACCOUNT_ID>`

### 2. 创建目标存储桶（us-west-2）并开启版本控制

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DEST_BUCKET="s3-workshop-crr-dest-${ACCOUNT_ID}"

aws s3api create-bucket \
  --bucket "${DEST_BUCKET}" \
  --region us-west-2 \
  --create-bucket-configuration LocationConstraint=us-west-2

aws s3api put-bucket-versioning \
  --bucket "${DEST_BUCKET}" \
  --versioning-configuration Status=Enabled \
  --region us-west-2

echo "Destination bucket ready: ${DEST_BUCKET} (us-west-2)"
```

**预期输出**：`Destination bucket ready: s3-workshop-crr-dest-<ACCOUNT_ID> (us-west-2)`

### 3. 创建 S3 复制 IAM 角色

S3 复制服务需要一个专属 IAM 角色，信任主体为 `s3.amazonaws.com`，权限包括读取源桶对象和写入目标桶对象。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-crr-source-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-crr-dest-${ACCOUNT_ID}"
ROLE_NAME="s3-workshop-crr-role"

aws iam create-role \
  --role-name "${ROLE_NAME}" \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "s3.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }' \
  --query 'Role.Arn' \
  --output text
```

**预期输出**：`arn:aws:iam::<ACCOUNT_ID>:role/s3-workshop-crr-role`

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-crr-source-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-crr-dest-${ACCOUNT_ID}"
ROLE_NAME="s3-workshop-crr-role"

aws iam put-role-policy \
  --role-name "${ROLE_NAME}" \
  --policy-name "s3-replication-policy" \
  --policy-document "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
        \"Effect\": \"Allow\",
        \"Action\": [\"s3:GetReplicationConfiguration\", \"s3:ListBucket\"],
        \"Resource\": \"arn:aws:s3:::${SOURCE_BUCKET}\"
      },
      {
        \"Effect\": \"Allow\",
        \"Action\": [\"s3:GetObjectVersionForReplication\", \"s3:GetObjectVersionAcl\", \"s3:GetObjectVersionTagging\"],
        \"Resource\": \"arn:aws:s3:::${SOURCE_BUCKET}/*\"
      },
      {
        \"Effect\": \"Allow\",
        \"Action\": [\"s3:ReplicateObject\", \"s3:ReplicateDelete\", \"s3:ReplicateTags\"],
        \"Resource\": \"arn:aws:s3:::${DEST_BUCKET}/*\"
      }
    ]
  }"
echo "IAM role policy attached"
```

**预期输出**：`IAM role policy attached`

### 4. 配置 CRR 复制规则

复制规则可以通过 `Filter.Prefix` 限定只复制特定前缀的对象。`DeleteMarkerReplication` 控制是否同步删除操作（此处开启）。`StorageClass` 可以让目标桶使用更便宜的存储类型以节省成本。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-crr-source-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-crr-dest-${ACCOUNT_ID}"
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/s3-workshop-crr-role"

aws s3api put-bucket-replication \
  --bucket "${SOURCE_BUCKET}" \
  --replication-configuration "{
    \"Role\": \"${ROLE_ARN}\",
    \"Rules\": [{
      \"ID\": \"crr-all-objects\",
      \"Priority\": 1,
      \"Status\": \"Enabled\",
      \"Filter\": {\"Prefix\": \"\"},
      \"Destination\": {
        \"Bucket\": \"arn:aws:s3:::${DEST_BUCKET}\",
        \"StorageClass\": \"STANDARD_IA\"
      },
      \"DeleteMarkerReplication\": {\"Status\": \"Enabled\"}
    }]
  }"
echo "CRR replication rule configured"
```

**预期输出**：`CRR replication rule configured`

验证规则已配置：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-crr-source-${ACCOUNT_ID}"
aws s3api get-bucket-replication \
  --bucket "${SOURCE_BUCKET}" \
  --query 'ReplicationConfiguration.Rules[0].Status' \
  --output text
```

**预期输出**：`Enabled`

### 5. 上传对象触发复制并验证

复制是异步的，对象上传后通常在几秒到几分钟内完成复制。可以通过 `head-object` 上的 `ReplicationStatus` 字段追踪复制状态。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-crr-source-${ACCOUNT_ID}"

echo "CRR Test - $(date)" > /tmp/crr-test.txt
aws s3 cp /tmp/crr-test.txt "s3://${SOURCE_BUCKET}/crr-test.txt"
```

**预期输出**：`upload: /tmp/crr-test.txt to s3://s3-workshop-crr-source-<ACCOUNT_ID>/crr-test.txt`

等待并检查复制状态：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-crr-source-${ACCOUNT_ID}"
sleep 10
aws s3api head-object \
  --bucket "${SOURCE_BUCKET}" \
  --key "crr-test.txt" \
  --query 'ReplicationStatus' \
  --output text
```

**预期输出**：`COMPLETED`（若返回 `PENDING`，等 30 秒后重试）

验证目标桶已有对象：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DEST_BUCKET="s3-workshop-crr-dest-${ACCOUNT_ID}"
aws s3 ls "s3://${DEST_BUCKET}" --region us-west-2
```

**预期输出**：包含 `crr-test.txt` 的列表行

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 源桶和目标桶版本控制均为 `Enabled`
- [ ] CRR 规则 `crr-all-objects` 状态为 `Enabled`
- [ ] 上传到源桶的对象 `crr-test.txt`，`ReplicationStatus` 为 `COMPLETED`
- [ ] 目标桶（us-west-2）中可以看到已复制的对象

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3api get-bucket-versioning --bucket "s3-workshop-crr-source-$(aws sts get-caller-identity --query Account --output text)" --query Status --output text` | `Enabled` |
| 2 | `aws s3api get-bucket-replication --bucket "s3-workshop-crr-source-$(aws sts get-caller-identity --query Account --output text)" --query 'ReplicationConfiguration.Rules[0].Status' --output text` | `Enabled` |
| 3 | `aws s3api head-object --bucket "s3-workshop-crr-source-$(aws sts get-caller-identity --query Account --output text)" --key "crr-test.txt" --query ReplicationStatus --output text` | `COMPLETED` |

---

## 实验总结

本实验实现了 S3 跨区域数据复制，覆盖了复制配置的三个关键要素：版本控制前提、专属 IAM 角色授权、以及复制规则的过滤和目标配置。CRR 的异步复制特性意味着它适合 RPO（恢复点目标）可以接受秒级延迟的灾备场景，而不适合作为强一致性的主备切换。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-crr-source-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-crr-dest-${ACCOUNT_ID}"

# 删除复制规则
aws s3api delete-bucket-replication --bucket "${SOURCE_BUCKET}"

# 删除源桶所有版本
for bucket in "${SOURCE_BUCKET}" "${DEST_BUCKET}"; do
  region_flag=""
  [ "${bucket}" = "${DEST_BUCKET}" ] && region_flag="--region us-west-2"
  aws s3api list-object-versions --bucket "${bucket}" ${region_flag} \
    --query '[Versions[].{Key:Key,VersionId:VersionId}, DeleteMarkers[].{Key:Key,VersionId:VersionId}]' \
    --output json 2>/dev/null | \
    python3 -c "
import json,sys,subprocess
for lst in json.load(sys.stdin):
    for obj in (lst or []):
        cmd=['aws','s3api','delete-object','--bucket','${bucket}','--key',obj['Key'],'--version-id',obj['VersionId']]
        if '${DEST_BUCKET}' == '${bucket}': cmd+=['--region','us-west-2']
        subprocess.run(cmd,capture_output=True)
print(f'Cleaned {\"${bucket}\"}')"
  aws s3api delete-bucket --bucket "${bucket}" ${region_flag}
done

# 删除 IAM 角色
aws iam delete-role-policy --role-name "s3-workshop-crr-role" --policy-name "s3-replication-policy"
aws iam delete-role --role-name "s3-workshop-crr-role"
echo "Cleanup complete"
```
