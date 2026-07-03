# Demo07 — S3 Inventory：存储清单与大规模分析

## 实验简介

S3 Inventory 定期（每天或每周）自动生成存储桶中所有对象的清单报告（CSV 或 Parquet 格式），包含对象大小、存储类型、加密状态、复制状态等元数据。清单是进行大规模合规审计、成本优化分析和 Batch Operations 输入数据的标准数据源，比逐页调用 ListObjectsV2 更高效。

**实验目标：**
- 掌握 S3 Inventory 配置（频率、字段、格式、目标桶）
- 理解 Inventory 与 ListObjects 的性能差异（亿级对象场景）
- 能够用 Inventory 输出的 Parquet 配合 Athena 做存储分析

**实验流程：**
1. 创建源桶（上传测试数据）和清单目标桶
2. 配置 Inventory 规则
3. 授予 S3 服务写入目标桶的权限
4. 验证配置，了解清单文件结构

**预计时长：** 20 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x
- **权限**：`AmazonS3FullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export SOURCE_BUCKET="s3-workshop-inv-src-${ACCOUNT_ID}"
export DEST_BUCKET="s3-workshop-inv-dest-${ACCOUNT_ID}"
echo "Source: ${SOURCE_BUCKET} | Dest: ${DEST_BUCKET}"
```

---

## 步骤

### 1. 创建源桶和目标桶

目标桶和源桶可以在同一账号，也可以跨账号。目标桶中的清单文件会存放在 `<prefix>/<source-bucket>/<inventory-id>/` 路径下。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-inv-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-inv-dest-${ACCOUNT_ID}"

aws s3api create-bucket --bucket "${SOURCE_BUCKET}" --region us-east-1
aws s3api create-bucket --bucket "${DEST_BUCKET}" --region us-east-1
echo "Both buckets created"
```

**预期输出**：`Both buckets created`

上传不同存储类型的测试对象：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-inv-src-${ACCOUNT_ID}"

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

### 2. 授予 S3 服务写入目标桶的权限

S3 Inventory 服务（`s3.amazonaws.com`）需要权限将清单文件写入目标桶。通过 Bucket Policy 的 `ArnLike` 条件将权限限定在来自特定源桶的 Inventory 请求，防止任意 S3 桶向目标桶写入。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-inv-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-inv-dest-${ACCOUNT_ID}"

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

### 3. 配置 Inventory 规则

清单规则指定：频率（每天/每周）、目标桶和前缀、格式（CSV/ORC/Parquet）、要包含的元数据字段。Parquet 格式与 Athena 和 Redshift Spectrum 直接兼容，是数据分析场景的首选。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-inv-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-inv-dest-${ACCOUNT_ID}"

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
      \"Size\",
      \"StorageClass\",
      \"LastModifiedDate\",
      \"ETag\",
      \"ReplicationStatus\",
      \"EncryptionStatus\",
      \"ObjectLockMode\",
      \"ObjectLockRetainUntilDate\"
    ]
  }"
echo "Inventory rule configured: daily-full-inventory"
```

**预期输出**：`Inventory rule configured: daily-full-inventory`

验证规则配置：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-inv-src-${ACCOUNT_ID}"
aws s3api get-bucket-inventory-configuration \
  --bucket "${SOURCE_BUCKET}" \
  --id "daily-full-inventory" \
  --query 'InventoryConfiguration.{Enabled:IsEnabled, Frequency:Schedule.Frequency, Format:Destination.S3BucketDestination.Format}'
```

**预期输出**：
```json
{
    "Enabled": true,
    "Frequency": "Daily",
    "Format": "CSV"
}
```

> ⚠️ 实测 `"Filter": {"Prefix": ""}`（空字符串前缀，意图表示"匹配所有对象"）会导致 `PutBucketInventoryConfiguration` 报 `MalformedXML`。若不需要前缀过滤，直接省略整个 `Filter` 字段即可，不要传空字符串 Prefix。

### 4. 了解 Inventory 输出结构

Inventory 首次运行（通常在配置后 24 小时内）会在目标桶中生成以下结构：

```
s3-workshop-inv-dest-<ACCOUNT_ID>/
└── inventory-reports/
    └── s3-workshop-inv-src-<ACCOUNT_ID>/
        └── daily-full-inventory/
            ├── 2026-07-02T00-00Z/
            │   ├── manifest.json          # 清单元信息
            │   ├── manifest.checksum      # 完整性校验
            │   └── data/
            │       └── xxxxxx.csv.gz      # 实际清单数据
            └── hive/                      # Athena 分区格式（如选 Parquet）
                └── dt=2026-07-02-00-00/
```

查看目标桶中的清单文件（如果已有历史清单）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DEST_BUCKET="s3-workshop-inv-dest-${ACCOUNT_ID}"
aws s3 ls "s3://${DEST_BUCKET}/inventory-reports/" --recursive 2>/dev/null || \
  echo "清单文件尚未生成（每日运行一次，通常在 UTC 午夜后 24 小时内首次生成）"
```

**预期输出**：`清单文件尚未生成...`（这是正常的，清单配置后最快次日生成）

### 5. 配置第二个 Parquet 格式清单（用于 Athena 分析）

为同一源桶配置多个 Inventory 规则，每个规则可以有不同格式、频率和目标。Parquet 格式输出可以直接被 Athena 查询，无需 COPY 到其他存储。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-inv-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-inv-dest-${ACCOUNT_ID}"

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
echo "Second inventory rule (Parquet, weekly) configured"
```

**预期输出**：`Second inventory rule (Parquet, weekly) configured`

列出所有 Inventory 规则：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-inv-src-${ACCOUNT_ID}"
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

### 6. 预览 Inventory 输出格式（手动模拟样本）

真实清单文件次日才生成，但可以手动创建一份结构相同的样本 CSV 上传到目标桶，立即用 S3 Select 演示查询——这正是运维人员次日拿到真实清单后的实际操作。

创建样本清单 CSV（格式与 S3 Inventory CSV 输出完全一致）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-inv-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-inv-dest-${ACCOUNT_ID}"

cat > /tmp/sample-inventory.csv << CSVEOF
"Bucket","Key","VersionId","IsLatest","IsDeleteMarker","Size","LastModifiedDate","ETag","StorageClass","IsMultipartUploaded","ReplicationStatus","EncryptionStatus"
"${SOURCE_BUCKET}","standard/report-001.csv","","true","false","1024","2026-07-01T00:00:00.000Z","abc123","STANDARD","false","NONE","NOT-SSE"
"${SOURCE_BUCKET}","standard/report-002.csv","","true","false","2048","2026-07-01T00:05:00.000Z","def456","STANDARD","false","NONE","NOT-SSE"
"${SOURCE_BUCKET}","standard/log-2026-06-30.txt","","true","false","512","2026-06-30T23:59:00.000Z","ghi789","STANDARD","false","NONE","NOT-SSE"
"${SOURCE_BUCKET}","ia/archive-2026-05.csv","","true","false","5242880","2026-05-01T00:00:00.000Z","jkl012","STANDARD_IA","false","NONE","NOT-SSE"
"${SOURCE_BUCKET}","ia/archive-2026-06.csv","","true","false","8388608","2026-06-01T00:00:00.000Z","mno345","STANDARD_IA","false","NONE","NOT-SSE"
CSVEOF

# 上传到目标桶（路径模拟 S3 Inventory 实际输出目录结构）
REPORT_DATE="2026-07-01-00-00"
aws s3 cp /tmp/sample-inventory.csv \
  "s3://${DEST_BUCKET}/inventory-reports/${SOURCE_BUCKET}/daily-full-inventory/${REPORT_DATE}/data/sample.csv"
echo "Sample inventory uploaded"
```

**预期输出**：`Sample inventory uploaded`

用 S3 Select 直接在 S3 上运行 SQL，查询 STANDARD_IA 对象及其大小（无需下载整个文件）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-inv-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-inv-dest-${ACCOUNT_ID}"
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
>
> **说明**：S3 Select 直接在 S3 端过滤数据，只返回匹配行，适合对大型清单文件（数百 MB）做快速汇总，无需先下载。真实清单交付后，用同样的 SQL 替换 key 路径即可。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 源桶包含 13 个对象（Standard 和 Standard-IA 两种存储类）
- [ ] 两个 Inventory 规则均已配置且 `IsEnabled: true`
- [ ] 目标桶 Bucket Policy 允许 S3 服务写入清单文件
- [ ] `daily-full-inventory` 规则配置了 8 个 OptionalFields

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3api list-bucket-inventory-configurations --bucket "s3-workshop-inv-src-$(aws sts get-caller-identity --query Account --output text)" --query 'length(InventoryConfigurationList)' --output text` | `2` |
| 2 | `aws s3api get-bucket-inventory-configuration --bucket "s3-workshop-inv-src-$(aws sts get-caller-identity --query Account --output text)" --id "daily-full-inventory" --query 'InventoryConfiguration.IsEnabled' --output text` | `True` |
| 3 | `aws s3 ls "s3://s3-workshop-inv-src-$(aws sts get-caller-identity --query Account --output text)" --recursive --summarize \| grep "Total Objects"` | `Total Objects: 13` |

---

## 实验总结

S3 Inventory 是大规模存储治理的基础设施：对于拥有数亿对象的存储桶，每日生成一次清单比持续调用 ListObjects（有 API 调用限制和成本）更高效和经济。清单数据与 Demo06 的 Batch Operations 是天然搭档——用 Inventory 确定操作范围，用 Batch Operations 批量执行。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="s3-workshop-inv-src-${ACCOUNT_ID}"
DEST_BUCKET="s3-workshop-inv-dest-${ACCOUNT_ID}"

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
echo "Cleanup complete"
```
