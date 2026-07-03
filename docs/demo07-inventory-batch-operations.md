# Demo07 — S3 Inventory 与 Batch Operations：清单驱动的批量对象操作

## 实验简介

对拥有数亿对象的存储桶做治理时，逐页调用 `ListObjectsV2` 既慢又有 API 调用成本。S3 Inventory 定期（每天/每周）自动生成全量对象清单（CSV/ORC/Parquet），S3 Batch Operations 则以清单为输入，用托管任务对清单中的对象批量执行操作（标签更新、加密迁移、Lambda 调用等），支持断点续传、失败重试和完成报告。两者是天然搭档：**Inventory 确定操作范围，Batch Operations 批量执行**——真实生产环境里，Batch Operations 的 Manifest 直接引用 Inventory 产出的 `manifest.json`（`Manifest.Spec.Format = S3InventoryReport_CSV_20211130`），省去重新生成清单的步骤。

**实验目标：**
- 掌握 S3 Inventory 配置（频率、字段、格式、目标桶）
- 理解 Inventory 输出结构，并用 S3 Select 直接对清单文件做过滤查询
- 掌握 Batch Operations 的任务配置：清单来源、操作类型、IAM 角色
- 理解任务状态流转：New → Preparing → Ready → Active → Complete

**实验流程：**
1. 创建源桶和目标桶，上传不同存储类型的测试对象
2. 授予 S3 服务写入目标桶权限，配置两条 Inventory 规则（CSV 全量 + Parquet 增量）
3. 用模拟样本清单演示 Inventory 输出结构，并用 S3 Select 直接查询
4. 生成 Batch Operations 清单，创建 IAM 角色
5. 创建并执行批量标签更新任务
6. 验证结果

**预计时长：** 45 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x，Python3
- **权限**：`AmazonS3FullAccess`、`IAMFullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SOURCE_BUCKET="s3-workshop-invbatch-src-${ACCOUNT_ID}"
export DEST_BUCKET="s3-workshop-invbatch-dest-${ACCOUNT_ID}"
echo "Source: ${SOURCE_BUCKET} | Dest: ${DEST_BUCKET}"
```

---

## 步骤

### 1. 创建源桶和目标桶，上传测试对象

目标桶存放 Inventory 清单和 Batch Operations 报告，与源桶分离是最佳实践，避免权限混淆。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-invbatch-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-invbatch-dest-${ACCOUNT_ID}"

aws s3api create-bucket --bucket "${SOURCE_BUCKET}" --region us-east-1
aws s3api create-bucket --bucket "${DEST_BUCKET}" --region us-east-1
echo "Both buckets created"
```

**预期输出**：`Both buckets created`

上传不同存储类型的测试对象：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-invbatch-src-${ACCOUNT_ID}"

for i in $(seq 1 10); do
  echo "Standard object ${i}" > /tmp/std-${i}.txt
  aws s3 cp /tmp/std-${i}.txt "s3://${SOURCE_BUCKET}/standard/file-${i}.txt" --quiet
done

for i in 1 2 3; do
  echo "IA object ${i}" > /tmp/ia-${i}.txt
  aws s3 cp /tmp/ia-${i}.txt "s3://${SOURCE_BUCKET}/archive/file-${i}.txt" \
    --storage-class STANDARD_IA --quiet
done

echo "Total objects uploaded: $(aws s3 ls s3://${SOURCE_BUCKET} --recursive --summarize | grep 'Total Objects' | awk '{print $3}')"
```

**预期输出**：`Total objects uploaded: 13`

### 2. 授予 S3 服务写入目标桶权限，配置 Inventory 规则

S3 Inventory 服务（`s3.amazonaws.com`）需要权限将清单文件写入目标桶。通过 Bucket Policy 的 `ArnLike` 条件将权限限定在来自特定源桶的 Inventory 请求，防止任意 S3 桶向目标桶写入。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-invbatch-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-invbatch-dest-${ACCOUNT_ID}"

aws s3api put-bucket-policy \
  --bucket "${DEST_BUCKET}" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Sid\": \"AllowS3InventoryWrite\",
      \"Effect\": \"Allow\",
      \"Principal\": {\"Service\": \"s3.amazonaws.com\"},
      \"Action\": \"s3:PutObject\",
      \"Resource\": \"arn:aws:s3:::${DEST_BUCKET}/*\",
      \"Condition\": {
        \"StringEquals\": {
          \"s3:x-amz-acl\": \"bucket-owner-full-control\"
        },
        \"ArnLike\": {
          \"aws:SourceArn\": \"arn:aws:s3:::${SOURCE_BUCKET}\"
        }
      }
    }]
  }"
echo "Destination bucket policy configured"
```

**预期输出**：`Destination bucket policy configured`

配置两条 Inventory 规则：每日全量 CSV（合规审计用）+ 每周 Parquet（Athena 分析用，`OptionalFields` 精简）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-invbatch-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-invbatch-dest-${ACCOUNT_ID}"

aws s3api put-bucket-inventory-configuration \
  --bucket "${SOURCE_BUCKET}" \
  --id "daily-full-inventory" \
  --inventory-configuration "{
    \"Id\": \"daily-full-inventory\",
    \"IsEnabled\": true,
    \"Destination\": {
      \"S3BucketDestination\": {
        \"AccountId\": \"${ACCOUNT_ID}\",
        \"Bucket\": \"arn:aws:s3:::${DEST_BUCKET}\",
        \"Format\": \"CSV\",
        \"Prefix\": \"inventory-reports\"
      }
    },
    \"Schedule\": {\"Frequency\": \"Daily\"},
    \"IncludedObjectVersions\": \"Current\",
    \"OptionalFields\": [
      \"Size\", \"StorageClass\", \"LastModifiedDate\", \"ETag\",
      \"ReplicationStatus\", \"EncryptionStatus\",
      \"ObjectLockMode\", \"ObjectLockRetainUntilDate\"
    ]
  }"

aws s3api put-bucket-inventory-configuration \
  --bucket "${SOURCE_BUCKET}" \
  --id "weekly-parquet-inventory" \
  --inventory-configuration "{
    \"Id\": \"weekly-parquet-inventory\",
    \"IsEnabled\": true,
    \"Destination\": {
      \"S3BucketDestination\": {
        \"AccountId\": \"${ACCOUNT_ID}\",
        \"Bucket\": \"arn:aws:s3:::${DEST_BUCKET}\",
        \"Format\": \"Parquet\",
        \"Prefix\": \"parquet-inventory\"
      }
    },
    \"Schedule\": {\"Frequency\": \"Weekly\"},
    \"IncludedObjectVersions\": \"All\",
    \"OptionalFields\": [\"Size\", \"StorageClass\", \"EncryptionStatus\"]
  }"

aws s3api list-bucket-inventory-configurations \
  --bucket "${SOURCE_BUCKET}" \
  --query 'InventoryConfigurationList[].{Id:Id, Frequency:Schedule.Frequency, Format:Destination.S3BucketDestination.Format}'
```

**预期输出**：
```json
[
    {"Id": "daily-full-inventory", "Frequency": "Daily", "Format": "CSV"},
    {"Id": "weekly-parquet-inventory", "Frequency": "Weekly", "Format": "Parquet"}
]
```

> ⚠️ 实测 `"Filter": {"Prefix": ""}`（空字符串前缀，意图表示"匹配所有对象"）会导致 `PutBucketInventoryConfiguration` 报 `MalformedXML`。若不需要前缀过滤，直接省略整个 `Filter` 字段即可，不要传空字符串 Prefix。

### 3. 模拟 Inventory 输出并用 S3 Select 查询

Inventory 首次运行通常在配置后 24 小时内，本实验不等待，手动创建一份结构相同的样本 CSV 上传到目标桶——这正是运维人员次日拿到真实清单后的实际操作。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-invbatch-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-invbatch-dest-${ACCOUNT_ID}"

cat > /tmp/sample-inventory.csv << CSVEOF
"Bucket","Key","VersionId","IsLatest","IsDeleteMarker","Size","LastModifiedDate","ETag","StorageClass","IsMultipartUploaded","ReplicationStatus","EncryptionStatus"
"${SOURCE_BUCKET}","standard/report-001.csv","","true","false","1024","2026-07-01T00:00:00.000Z","abc123","STANDARD","false","NONE","NOT-SSE"
"${SOURCE_BUCKET}","standard/report-002.csv","","true","false","2048","2026-07-01T00:05:00.000Z","def456","STANDARD","false","NONE","NOT-SSE"
"${SOURCE_BUCKET}","standard/log-2026-06-30.txt","","true","false","512","2026-06-30T23:59:00.000Z","ghi789","STANDARD","false","NONE","NOT-SSE"
"${SOURCE_BUCKET}","ia/archive-2026-05.csv","","true","false","5242880","2026-05-01T00:00:00.000Z","jkl012","STANDARD_IA","false","NONE","NOT-SSE"
"${SOURCE_BUCKET}","ia/archive-2026-06.csv","","true","false","8388608","2026-06-01T00:00:00.000Z","mno345","STANDARD_IA","false","NONE","NOT-SSE"
CSVEOF

REPORT_DATE="2026-07-01-00-00"
aws s3 cp /tmp/sample-inventory.csv \
  "s3://${DEST_BUCKET}/inventory-reports/${SOURCE_BUCKET}/daily-full-inventory/${REPORT_DATE}/data/sample.csv"
echo "Sample inventory uploaded"
```

**预期输出**：`Sample inventory uploaded`

用 S3 Select 直接在 S3 端运行 SQL，查询 STANDARD_IA 对象及其大小（无需下载整个清单文件）：

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-invbatch-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-invbatch-dest-${ACCOUNT_ID}"
REPORT_DATE="2026-07-01-00-00"

aws s3api select-object-content \
  --bucket "${DEST_BUCKET}" \
  --key "inventory-reports/${SOURCE_BUCKET}/daily-full-inventory/${REPORT_DATE}/data/sample.csv" \
  --expression "SELECT s.\"Key\", s.StorageClass, s.\"Size\" FROM S3Object s WHERE s.StorageClass = 'STANDARD_IA'" \
  --expression-type SQL \
  --input-serialization '{"CSV": {"FileHeaderInfo": "USE"}}' \
  --output-serialization '{"CSV": {}}' \
  /tmp/inventory-query-result.txt
cat /tmp/inventory-query-result.txt
```

**预期输出**：
```
ia/archive-2026-05.csv,STANDARD_IA,5242880
ia/archive-2026-06.csv,STANDARD_IA,8388608
```

> ⚠️ `Key` 和 `Size` 是 S3 Select SQL 的保留关键字，直接写 `s.Key`/`s.Size` 会报 `ParseInvalidPathComponent: ... got: KEYWORD`，必须用双引号转义为标识符：`s."Key"`、`s."Size"`。

### 4. 生成 Batch Operations 清单

真实场景中 Batch Operations 可以直接引用 Inventory 产出的 `manifest.json`（`Manifest.Spec.Format` 设为 `S3InventoryReport_CSV_20211130`）。本实验不等待 Inventory 的 24 小时生成延迟，改用 `list-objects-v2` 现取对象列表拼出等价的自定义 CSV 清单（`S3BatchOperations_CSV_20180820` 格式，每行 `bucket,key`），效果一致：

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-invbatch-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-invbatch-dest-${ACCOUNT_ID}"

aws s3api list-objects-v2 \
  --bucket "${SOURCE_BUCKET}" \
  --query 'Contents[].Key' \
  --output text | \
  tr '\t' '\n' | \
  awk -v b="${SOURCE_BUCKET}" 'NF{print b","$0}' > /tmp/manifest.csv

wc -l < /tmp/manifest.csv
head -3 /tmp/manifest.csv
```

**预期输出**：
```
13
s3-workshop-invbatch-src-<ACCOUNT_ID>,archive/file-1.txt
s3-workshop-invbatch-src-<ACCOUNT_ID>,archive/file-2.txt
s3-workshop-invbatch-src-<ACCOUNT_ID>,archive/file-3.txt
```

上传清单文件并获取 ETag（Batch Operations 创建任务时需要清单文件的 ETag 做完整性校验）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DEST_BUCKET="s3-workshop-invbatch-dest-${ACCOUNT_ID}"
aws s3 cp /tmp/manifest.csv "s3://${DEST_BUCKET}/manifests/batch-manifest.csv"
ETAG=$(aws s3api head-object \
  --bucket "${DEST_BUCKET}" \
  --key "manifests/batch-manifest.csv" \
  --query 'ETag' --output text | tr -d '"')
echo "Manifest ETag: ${ETAG}"
```

**预期输出**：`Manifest ETag: <md5-hash>`

### 5. 创建 Batch Operations IAM 角色

Batch Operations 需要权限读取清单桶、对源桶执行操作、写回报告。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-invbatch-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-invbatch-dest-${ACCOUNT_ID}"

ROLE_ARN=$(aws iam create-role \
  --role-name "s3-invbatch-ops-role" \
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
  --role-name "s3-invbatch-ops-role" \
  --policy-name "s3-invbatch-ops-policy" \
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
        \"Resource\": \"arn:aws:s3:::${DEST_BUCKET}/*\"
      }
    ]
  }"
sleep 10
echo "Role ARN: ${ROLE_ARN}"
```

**预期输出**：`Role ARN: arn:aws:iam::<ACCOUNT_ID>:role/s3-invbatch-ops-role`

### 6. 创建并执行批量标签更新任务

任务操作类型为 `S3PutObjectTagging`，将为清单中所有对象添加 `Environment=Workshop` 和 `ProcessedBy=BatchOps` 两个标签。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DEST_BUCKET="s3-workshop-invbatch-dest-${ACCOUNT_ID}"
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/s3-invbatch-ops-role"
ETAG=$(aws s3api head-object \
  --bucket "${DEST_BUCKET}" \
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
      \"ObjectArn\": \"arn:aws:s3:::${DEST_BUCKET}/manifests/batch-manifest.csv\",
      \"ETag\": \"${ETAG}\"
    }
  }" \
  --report "{
    \"Bucket\": \"arn:aws:s3:::${DEST_BUCKET}\",
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

> ⚠️ `create-job` 返回 JobId 后，`list-jobs --job-statuses Suspended` 可能有几秒的传播延迟，紧接着查询会返回空列表（`Jobs[0].JobId` 为 `None`）。若下一步 `update-job-status` 报 `Invalid length for parameter JobId`，说明取到的是 `None`，直接重试一次 `list-jobs` 即可。

Batch Operations 任务默认需要手动确认才能执行（`--confirmation-required`）。确认后任务进入 `Active` 状态开始处理：

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
    "SucceededTasks": 13,
    "TotalTasks": 13
}
```

> ⚠️ **Batch Operations 任务历史永久保留，无法删除**：`list-jobs` 返回账号内该 Region 所有历史任务（包括之前跑过本实验留下的 Complete 任务），S3 没有提供删除 Job 记录的 API。重复执行本实验时 `--job-statuses Complete` 的任务数会累加，不会是固定值 `1`；验证检查点已改为按 `CreationTime` 取最新一个任务的 `Status`，不依赖历史任务总数。

### 7. 验证标签已批量添加

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-invbatch-src-${ACCOUNT_ID}"
aws s3api get-object-tagging \
  --bucket "${SOURCE_BUCKET}" \
  --key "standard/file-1.txt" \
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
- [ ] 源桶包含 13 个对象（Standard 和 Standard-IA 两种存储类）
- [ ] 两个 Inventory 规则均已配置且 `IsEnabled: true`
- [ ] 用 S3 Select 从模拟清单中筛选出 2 个 STANDARD_IA 对象
- [ ] Batch Operations 任务状态为 `Complete`，成功处理 13 个对象
- [ ] 随机抽查的对象包含 `ProcessedBy=BatchOps` 标签
- [ ] 目标桶的 `batch-reports/` 前缀下存在任务完成报告

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3api list-bucket-inventory-configurations --bucket "s3-workshop-invbatch-src-$(aws sts get-caller-identity --query Account --output text)" --query 'length(InventoryConfigurationList)' --output text` | `2` |
| 2 | `aws s3 ls "s3://s3-workshop-invbatch-src-$(aws sts get-caller-identity --query Account --output text)" --recursive --summarize \| grep "Total Objects"` | `Total Objects: 13` |
| 3 | `aws s3control list-jobs --account-id "$(aws sts get-caller-identity --query Account --output text)" --region us-east-1 --query 'Jobs \| sort_by(@, &CreationTime)[-1].Status' --output text` | `Complete` |
| 4 | `aws s3api get-object-tagging --bucket "s3-workshop-invbatch-src-$(aws sts get-caller-identity --query Account --output text)" --key "standard/file-5.txt" --query "TagSet[?Key=='ProcessedBy'].Value" --output text` | `BatchOps` |

---

## 实验总结

本实验展示了 Inventory → Batch Operations 的完整治理闭环：Inventory 定期生成全量清单（比持续调用 ListObjects 更高效经济），S3 Select 可以直接对清单文件做过滤查询而无需下载，Batch Operations 则以清单（无论是 Inventory 原生输出还是自定义 CSV）为输入批量执行操作。除标签更新外，同样的框架可以用于加密迁移（PutObjectCopy 变更 KMS Key）、版本恢复（RestoreObject）和自定义 Lambda 处理。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-invbatch-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-invbatch-dest-${ACCOUNT_ID}"

# 删除 Inventory 规则
aws s3api delete-bucket-inventory-configuration \
  --bucket "${SOURCE_BUCKET}" --id "daily-full-inventory"
aws s3api delete-bucket-inventory-configuration \
  --bucket "${SOURCE_BUCKET}" --id "weekly-parquet-inventory"

# 删除存储桶
for bucket in "${SOURCE_BUCKET}" "${DEST_BUCKET}"; do
  aws s3 rm "s3://${bucket}" --recursive 2>/dev/null
  aws s3api delete-bucket --bucket "${bucket}" --region us-east-1 2>/dev/null
done

# 删除 IAM 角色
aws iam delete-role-policy --role-name "s3-invbatch-ops-role" --policy-name "s3-invbatch-ops-policy"
aws iam delete-role --role-name "s3-invbatch-ops-role"
echo "Cleanup complete"
```
