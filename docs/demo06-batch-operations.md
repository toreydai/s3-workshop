# Demo06 — S3 Batch Operations：批量对象操作

## 实验简介

当需要对存储桶中数百万个对象执行统一操作时（批量复制、加密迁移、标签更新、调用 Lambda 处理），逐对象循环既耗时又不可靠。S3 Batch Operations 通过 S3 Inventory 清单或自定义 CSV 清单，以托管任务的方式批量处理对象，支持断点续传、失败重试和完成报告，无需管理任何计算资源。

**实验目标：**
- 掌握 Batch Operations 的任务配置：清单来源、操作类型、IAM 角色
- 理解任务状态流转：New → Preparing → Ready → Active → Complete
- 能够配置批量标签更新和批量复制任务并验证结果

**实验流程：**
1. 创建源桶并上传测试对象
2. 准备 CSV 清单（指定要操作的对象列表）
3. 创建 IAM 角色授权 Batch Operations
4. 创建并执行批量标签更新任务
5. 验证结果

**预计时长：** 35 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x，Python3
- **权限**：`AmazonS3FullAccess`、`IAMFullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SOURCE_BUCKET="s3-workshop-batch-src-${ACCOUNT_ID}"
export MANIFEST_BUCKET="s3-workshop-batch-manifest-${ACCOUNT_ID}"
echo "Source: ${SOURCE_BUCKET}"
```

---

## 步骤

### 1. 创建存储桶并上传测试对象

清单桶与源桶分离是最佳实践，避免权限混淆。报告也写回清单桶（减少桶数量）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-batch-src-${ACCOUNT_ID}"
MANIFEST_BUCKET="s3-workshop-batch-manifest-${ACCOUNT_ID}"

for bucket in "${SOURCE_BUCKET}" "${MANIFEST_BUCKET}"; do
  aws s3api create-bucket --bucket "${bucket}" --region us-east-1
  echo "Created: ${bucket}"
done
```

**预期输出**：两行 `Created: <bucket>` 消息

上传 10 个测试对象（可快速完成，清楚演示批处理流程）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-batch-src-${ACCOUNT_ID}"

for i in $(seq 1 10); do
  echo "{\"id\": ${i}, \"status\": \"untagged\", \"category\": \"batch-demo\"}" > /tmp/obj-${i}.json
  aws s3 cp /tmp/obj-${i}.json "s3://${SOURCE_BUCKET}/objects/item-$(printf '%03d' ${i}).json" --quiet
done
echo "Uploaded 10 objects"
aws s3 ls "s3://${SOURCE_BUCKET}/objects/" --summarize | grep "Total Objects"
```

**预期输出**：`Total Objects: 10`

### 2. 生成 CSV 清单文件

Batch Operations 的自定义清单是 CSV 文件，每行格式为 `bucket,key`（或 `bucket,key,version-id`），第一行为头部（`bucket,key`）。

纯 CLI 方式生成 manifest CSV，无需 Python boto3：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-batch-src-${ACCOUNT_ID}"
MANIFEST_BUCKET="s3-workshop-batch-manifest-${ACCOUNT_ID}"

# list-objects-v2 取 key 列表，awk 拼上桶名生成 CSV 每行
aws s3api list-objects-v2 \
  --bucket "${SOURCE_BUCKET}" \
  --prefix "objects/" \
  --query 'Contents[].Key' \
  --output text | \
  tr '\t' '\n' | \
  awk -v b="${SOURCE_BUCKET}" 'NF{print b","$0}' > /tmp/manifest.csv

wc -l < /tmp/manifest.csv
head -3 /tmp/manifest.csv
```

**预期输出**：
```
10
s3-workshop-batch-src-<ACCOUNT_ID>,objects/item-001.json
s3-workshop-batch-src-<ACCOUNT_ID>,objects/item-002.json
s3-workshop-batch-src-<ACCOUNT_ID>,objects/item-003.json
```

上传清单文件并获取 ETag：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
MANIFEST_BUCKET="s3-workshop-batch-manifest-${ACCOUNT_ID}"
aws s3 cp /tmp/manifest.csv "s3://${MANIFEST_BUCKET}/manifests/batch-manifest.csv"
ETAG=$(aws s3api head-object \
  --bucket "${MANIFEST_BUCKET}" \
  --key "manifests/batch-manifest.csv" \
  --query 'ETag' --output text | tr -d '"')
echo "Manifest ETag: ${ETAG}"
```

**预期输出**：`Manifest ETag: <md5-hash>`

### 3. 创建 Batch Operations IAM 角色

Batch Operations 需要权限读取清单桶、对源桶执行操作、写回报告（写到同一清单桶的 `reports/` 前缀下）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-batch-src-${ACCOUNT_ID}"
MANIFEST_BUCKET="s3-workshop-batch-manifest-${ACCOUNT_ID}"

ROLE_ARN=$(aws iam create-role \
  --role-name "s3-batch-ops-role" \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "batchoperations.s3.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }' \
  --query 'Role.Arn' --output text)

aws iam put-role-policy \
  --role-name "s3-batch-ops-role" \
  --policy-name "s3-batch-ops-policy" \
  --policy-document "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
        \"Effect\": \"Allow\",
        \"Action\": [\"s3:GetObject\", \"s3:GetObjectTagging\", \"s3:PutObjectTagging\"],
        \"Resource\": \"arn:aws:s3:::${SOURCE_BUCKET}/*\"
      },
      {
        \"Effect\": \"Allow\",
        \"Action\": [\"s3:GetObject\", \"s3:PutObject\"],
        \"Resource\": \"arn:aws:s3:::${MANIFEST_BUCKET}/*\"
      }
    ]
  }"
sleep 10
echo "Role ARN: ${ROLE_ARN}"
```

**预期输出**：`Role ARN: arn:aws:iam::<ACCOUNT_ID>:role/s3-batch-ops-role`

### 4. 创建批量标签更新任务

任务操作类型为 `S3PutObjectTagging`，将为清单中所有对象添加 `Environment=Workshop` 和 `ProcessedBy=BatchOps` 两个标签。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-batch-src-${ACCOUNT_ID}"
MANIFEST_BUCKET="s3-workshop-batch-manifest-${ACCOUNT_ID}"
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/s3-batch-ops-role"
ETAG=$(aws s3api head-object \
  --bucket "${MANIFEST_BUCKET}" \
  --key "manifests/batch-manifest.csv" \
  --query 'ETag' --output text | tr -d '"')

JOB_ID=$(aws s3control create-job \
  --account-id "${ACCOUNT_ID}" \
  --operation '{
    "S3PutObjectTagging": {
      "TagSet": [
        {"Key": "Environment", "Value": "Workshop"},
        {"Key": "ProcessedBy", "Value": "BatchOps"},
        {"Key": "ProcessedDate", "Value": "2026-07-01"}
      ]
    }
  }' \
  --manifest "{
    \"Spec\": {
      \"Format\": \"S3BatchOperations_CSV_20180820\",
      \"Fields\": [\"Bucket\", \"Key\"]
    },
    \"Location\": {
      \"ObjectArn\": \"arn:aws:s3:::${MANIFEST_BUCKET}/manifests/batch-manifest.csv\",
      \"ETag\": \"${ETAG}\"
    }
  }" \
  --report "{
    \"Bucket\": \"arn:aws:s3:::${MANIFEST_BUCKET}\",
    \"Format\": \"Report_CSV_20180820\",
    \"Enabled\": true,
    \"Prefix\": \"batch-reports\",
    \"ReportScope\": \"AllTasks\"
  }" \
  --role-arn "${ROLE_ARN}" \
  --priority 10 \
  --confirmation-required \
  --region us-east-1 \
  --query 'JobId' \
  --output text)
echo "Job created: ${JOB_ID}"
```

**预期输出**：`Job created: <job-id>`

### 5. 确认并执行任务

Batch Operations 任务默认需要手动确认才能执行（`--confirmation-required`）。确认后任务进入 `Active` 状态开始处理。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
JOB_ID=$(aws s3control list-jobs \
  --account-id "${ACCOUNT_ID}" \
  --job-statuses Suspended \
  --region us-east-1 \
  --query 'Jobs[0].JobId' --output text)

aws s3control update-job-status \
  --account-id "${ACCOUNT_ID}" \
  --job-id "${JOB_ID}" \
  --requested-job-status Ready \
  --region us-east-1 \
  --query 'Status' --output text
```

**预期输出**：`Ready`

等待并检查任务状态（可能需要 1-2 分钟）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
JOB_ID=$(aws s3control list-jobs \
  --account-id "${ACCOUNT_ID}" \
  --region us-east-1 \
  --query 'Jobs[0].JobId' --output text)

aws s3control describe-job \
  --account-id "${ACCOUNT_ID}" \
  --job-id "${JOB_ID}" \
  --region us-east-1 \
  --query 'Job.{Status:Status, SucceededTasks:ProgressSummary.NumberOfTasksSucceeded, TotalTasks:ProgressSummary.TotalNumberOfTasks}'
```

**预期输出**（任务完成后）：
```json
{
    "Status": "Complete",
    "SucceededTasks": 10,
    "TotalTasks": 10
}
```

> ⚠️ **Batch Operations 任务历史永久保留，无法删除**：`list-jobs` 返回账号内该 Region 所有历史任务（包括之前跑过本实验留下的 Complete 任务），S3 没有提供删除 Job 记录的 API。重复执行本实验时 `--job-statuses Complete` 的任务数会累加，不会是固定值 `1`；验证检查点已改为按 `CreationTime` 取最新一个任务的 `Status`，不依赖历史任务总数。

### 6. 验证标签已批量添加

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-batch-src-${ACCOUNT_ID}"
aws s3api get-object-tagging \
  --bucket "${SOURCE_BUCKET}" \
  --key "objects/item-001.json" \
  --query 'TagSet'
```

**预期输出**：
```json
[
    {"Key": "Environment", "Value": "Workshop"},
    {"Key": "ProcessedBy", "Value": "BatchOps"},
    {"Key": "ProcessedDate", "Value": "2026-07-01"}
]
```

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 源桶包含 10 个对象
- [ ] Batch Operations 任务状态为 `Complete`，成功处理 10 个对象
- [ ] 随机抽查的对象包含 `ProcessedBy=BatchOps` 标签
- [ ] 清单桶的 `batch-reports/` 前缀下存在任务完成报告

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3control list-jobs --account-id "$(aws sts get-caller-identity --query Account --output text)" --region us-east-1 --query 'Jobs \| sort_by(@, &CreationTime)[-1].Status' --output text` | `Complete` |
| 2 | `aws s3api get-object-tagging --bucket "s3-workshop-batch-src-$(aws sts get-caller-identity --query Account --output text)" --key "objects/item-005.json" --query "TagSet[?Key=='ProcessedBy'].Value" --output text` | `BatchOps` |
| 3 | `aws s3 ls "s3://s3-workshop-batch-src-$(aws sts get-caller-identity --query Account --output text)/objects/" --summarize \| grep "Total Objects"` | `Total Objects: 10` |

---

## 实验总结

本实验展示了 S3 Batch Operations 的核心工作流程：CSV 清单指定对象范围 → IAM 角色授权 → 任务确认 → 批量执行 → 结果报告。除标签更新外，同样的框架可以用于加密迁移（PutObjectCopy 变更 KMS Key）、版本恢复（RestoreObject）、和自定义 Lambda 处理。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
for bucket in "s3-workshop-batch-src-${ACCOUNT_ID}" \
              "s3-workshop-batch-manifest-${ACCOUNT_ID}"; do
  aws s3 rm "s3://${bucket}" --recursive 2>/dev/null
  aws s3api delete-bucket --bucket "${bucket}" --region us-east-1 2>/dev/null
  echo "Deleted: ${bucket}"
done

aws iam delete-role-policy --role-name "s3-batch-ops-role" --policy-name "s3-batch-ops-policy"
aws iam delete-role --role-name "s3-batch-ops-role"
echo "Cleanup complete"
```
