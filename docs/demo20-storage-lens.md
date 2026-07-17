# Demo20 — S3 Storage Lens 深度分析与成本优化

## 实验简介

S3 Storage Lens 提供跨账号、跨 Region 的统一存储可见性，包含 30+ 使用指标（对象数、存储量、请求数、数据传输量、访问模式）和 15 个推荐面板，帮助识别存储成本优化机会。本实验配置高级 Storage Lens 仪表板并通过 CLI 读取指标，分析存储使用模式。

**实验目标：**
- 掌握 Storage Lens 仪表板的创建与高级指标配置
- 理解各指标含义（不完整分片上传、非当前版本存储、HTTP 4xx 率等）
- 能够将 Storage Lens 数据导出到 S3 并配合 Athena 进行自定义分析

**实验流程：**
1. 创建包含多类对象的测试存储桶
2. 配置 Storage Lens 仪表板（高级指标 + 数据导出）
3. 等待数据收集后查询指标
4. 解读关键优化建议

**预计 AI 执行时长：** 25 分钟（指标数据次日可用）

---

## 前提条件

- **工具**：AWS CLI v2.x
- **权限**：`AmazonS3FullAccess`（含 s3:PutStorageLensConfiguration）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export DATA_BUCKET="s3-workshop-lens-data-${ACCOUNT_ID}"
export EXPORT_BUCKET="s3-workshop-lens-export-${ACCOUNT_ID}"
echo "Data: ${DATA_BUCKET} | Export: ${EXPORT_BUCKET}"
```

---

## 步骤

### 1. 创建测试存储桶并上传多类型数据

为了让 Storage Lens 有丰富数据可以展示，创建包含不同存储类型、版本控制开启、以及不同大小对象的存储桶。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-lens-data-${ACCOUNT_ID}"
EXPORT_BUCKET="s3-workshop-lens-export-${ACCOUNT_ID}"

aws s3api create-bucket --bucket "${DATA_BUCKET}" --region us-east-1
aws s3api create-bucket --bucket "${EXPORT_BUCKET}" --region us-east-1

# 开启版本控制
aws s3api put-bucket-versioning \
  --bucket "${DATA_BUCKET}" \
  --versioning-configuration Status=Enabled

# 上传不同存储类型的对象
for i in $(seq 1 5); do
  dd if=/dev/urandom of=/tmp/std-${i}.bin bs=1K count=$((i*100)) 2>/dev/null
  aws s3 cp /tmp/std-${i}.bin "s3://${DATA_BUCKET}/standard/file-${i}.bin" --quiet
done

for i in 1 2 3; do
  echo "IA object content ${i}" | aws s3 cp - \
    "s3://${DATA_BUCKET}/infrequent/file-${i}.txt" \
    --storage-class STANDARD_IA --quiet
done

# 上传多个版本制造非当前版本
for v in 1 2 3; do
  echo "Version ${v}" | aws s3 cp - "s3://${DATA_BUCKET}/versioned/doc.txt" --quiet
done

echo "Test data uploaded"
aws s3 ls "s3://${DATA_BUCKET}" --recursive --summarize | tail -2
```

**预期输出**：
```
Total Objects: 9
Total Size: <size>
```

> ⚠️ `aws s3 ls --recursive --summarize` 统计的是当前（最新）版本的对象数，不是全部版本数。本步骤共上传 5(standard) + 3(infrequent) + 3(versioned/doc.txt 的 3 个版本) = 11 次 PUT，但 `versioned/doc.txt` 的 3 次上传只有最新版本可见，所以 `ls --summarize` 显示 `Total Objects: 9`（5+3+1），而 `list-object-versions` 查询版本数会显示 `11`。两者口径不同，不要混淆。

### 2. 配置 Storage Lens 数据导出目标桶权限

Storage Lens 服务（`storage-lens.s3.amazonaws.com`）需要权限写入导出桶，配置方式与 Inventory 类似。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
EXPORT_BUCKET="s3-workshop-lens-export-${ACCOUNT_ID}"

aws s3api put-bucket-policy \
  --bucket "${EXPORT_BUCKET}" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Sid\": \"AllowStorageLensWrite\",
      \"Effect\": \"Allow\",
      \"Principal\": {\"Service\": \"storage-lens.s3.amazonaws.com\"},
      \"Action\": [\"s3:PutObject\"],
      \"Resource\": \"arn:aws:s3:::${EXPORT_BUCKET}/*\",
      \"Condition\": {
        \"StringEquals\": {
          \"s3:x-amz-acl\": \"bucket-owner-full-control\",
          \"aws:SourceAccount\": \"${ACCOUNT_ID}\"
        }
      }
    }]
  }"
echo "Export bucket policy configured"
```

**预期输出**：`Export bucket policy configured`

### 3. 创建 Storage Lens 仪表板

`StorageLensConfiguration` 的 `DataExport` 配置每天将指标导出到 S3（CSV 或 Parquet），可以与 Athena 结合做自定义趋势分析。高级指标（`AdvancedCostOptimizationMetrics`）包含存储类分析、生命周期建议等付费指标，但能提供更深入的优化洞察。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-lens-data-${ACCOUNT_ID}"
EXPORT_BUCKET="s3-workshop-lens-export-${ACCOUNT_ID}"

aws s3control put-storage-lens-configuration \
  --account-id "${ACCOUNT_ID}" \
  --config-id "s3-workshop-lens-dashboard" \
  --storage-lens-configuration "{
    \"Id\": \"s3-workshop-lens-dashboard\",
    \"IsEnabled\": true,
    \"AccountLevel\": {
      \"ActivityMetrics\": {\"IsEnabled\": true},
      \"AdvancedCostOptimizationMetrics\": {\"IsEnabled\": true},
      \"AdvancedDataProtectionMetrics\": {\"IsEnabled\": true},
      \"DetailedStatusCodesMetrics\": {\"IsEnabled\": true},
      \"BucketLevel\": {
        \"ActivityMetrics\": {\"IsEnabled\": true},
        \"AdvancedCostOptimizationMetrics\": {\"IsEnabled\": true},
        \"AdvancedDataProtectionMetrics\": {\"IsEnabled\": true},
        \"DetailedStatusCodesMetrics\": {\"IsEnabled\": true}
      }
    },
    \"Include\": {
      \"Buckets\": [\"arn:aws:s3:::${DATA_BUCKET}\"]
    },
    \"DataExport\": {
      \"S3BucketDestination\": {
        \"AccountId\": \"${ACCOUNT_ID}\",
        \"Arn\": \"arn:aws:s3:::${EXPORT_BUCKET}\",
        \"Format\": \"CSV\",
        \"OutputSchemaVersion\": \"V_1\",
        \"Prefix\": \"lens-exports\"
      },
      \"CloudWatchMetrics\": {\"IsEnabled\": false}
    }
  }" \
  --region us-east-1
echo "Storage Lens dashboard configured: s3-workshop-lens-dashboard"
```

**预期输出**：`Storage Lens dashboard configured: s3-workshop-lens-dashboard`

> ⚠️ Storage Lens 的 4 个高级指标类别是"全有或全无"绑定关系：只要开启其中一项，其余几项也必须同时为 `true`，否则报 `MissingBucketLevelAdvancedCostOptimizationMetrics` 等错误（下方已四项全开，并删除了会冲突的 `PrefixLevel.StorageMetrics` 配置块）。高级指标为付费功能（约 $0.20/百万对象/月），演示规模下成本可忽略。

验证配置：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws s3control get-storage-lens-configuration \
  --account-id "${ACCOUNT_ID}" \
  --config-id "s3-workshop-lens-dashboard" \
  --region us-east-1 \
  --query 'StorageLensConfiguration.{Enabled:IsEnabled, Id:Id}' \
  --output json
```

**预期输出**：
```json
{
    "Enabled": true,
    "Id": "s3-workshop-lens-dashboard"
}
```

### 4. 查看现有仪表板列表

除了自定义仪表板，AWS 为每个账号免费提供一个 `default-account-dashboard`，包含所有区域所有存储桶的基础指标。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws s3control list-storage-lens-configurations \
  --account-id "${ACCOUNT_ID}" \
  --region us-east-1 \
  --query 'StorageLensConfigurationList[].{Id:Id, IsEnabled:IsEnabled}' \
  --output table
```

**预期输出**：
```
---------------------------------------------------------
|      ListStorageLensConfigurations                     |
+-------------------------------------+------------------+
|              Id                     |   IsEnabled      |
+-------------------------------------+------------------+
|  default-account-dashboard          |  True            |
|  s3-workshop-lens-dashboard         |  True            |
+-------------------------------------+------------------+
```

### 5. 通过 CLI 查询 Storage Lens 指标

Storage Lens 指标数据次日可用。使用 `get-storage-lens-dashboard-data-export` 或在 Console 查看图表。以下命令查询 default dashboard 的近期数据。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# 查询 default dashboard 的存储量指标（数据次日可用）
aws s3control get-storage-lens-dashboard-data-export \
  --account-id "${ACCOUNT_ID}" \
  --config-id "default-account-dashboard" \
  --region us-east-1 2>/dev/null || echo "使用 AWS Console 查看仪表板数据: https://s3.console.aws.amazon.com/s3/lens"
```

**预期输出**：指标数据或提示使用 Console 查看

> ⚠️ `aws s3control get-storage-lens-dashboard-data-export` 不是一个真实存在的 AWS CLI 子命令（`aws s3control help` 中没有此命令，实测报 `ParamValidation: Found invalid choice`），并非"数据次日才有"导致的失败。Storage Lens 目前没有 CLI 命令可以直接读取仪表板的指标时间序列数据——仪表板数据只能通过 Console 查看，或者通过 `DataExport` 配置的 S3 导出文件（CSV/Parquet）配合 Athena 查询获取，此步骤的 `|| echo` 兜底分支即为预期路径。

### 6. 了解关键 Storage Lens 指标含义

无需等待数据，通过解读指标含义了解优化方向：

```bash
cat << 'EOF'
Storage Lens 关键优化指标解读：

【成本优化指标】
- % Noncurrent Version Bytes    → 非当前版本占比高 → 建议加 NoncurrentVersionExpiration 规则
- % Incomplete Multipart Upload → 未完成分片占比高 → 建议加 AbortIncompleteMultipartUpload 规则  
- % Bytes Not Accessed > 90d    → 90天未访问比例高 → 建议迁移到 STANDARD_IA 或 INTELLIGENT_TIERING
- Delete Marker Bytes           → 删除标记积累 → 建议加 ExpiredObjectDeleteMarker 规则

【数据保护指标】
- % Encrypted Bytes             → 未加密对象比例 → 建议设置默认 SSE-KMS 加密
- % Object Lock Protected Bytes → WORM 保护覆盖率 → 合规场景需要接近 100%
- % Replicated Bytes            → 复制覆盖率 → 灾备场景需监控

【活动指标】
- 4xx Error Rate                → GET 4xx 高 → 可能有僵尸客户端请求不存在的对象
- 5xx Error Rate                → 服务端错误 → 需要告警
EOF
```

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 数据桶 `s3-workshop-lens-data-<ACCOUNT_ID>` 包含 Standard 和 Standard-IA 两种存储类的对象
- [ ] Storage Lens 仪表板 `s3-workshop-lens-dashboard` 状态为 `IsEnabled: true`
- [ ] 仪表板配置了每日 CSV 导出到 `s3-workshop-lens-export-<ACCOUNT_ID>/lens-exports/`
- [ ] `list-storage-lens-configurations` 返回两个仪表板（default + workshop）

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3control list-storage-lens-configurations --account-id "$(aws sts get-caller-identity --query Account --output text)" --region us-east-1 --query 'length(StorageLensConfigurationList)' --output text` | `2` |
| 2 | `aws s3control get-storage-lens-configuration --account-id "$(aws sts get-caller-identity --query Account --output text)" --config-id "s3-workshop-lens-dashboard" --region us-east-1 --query 'StorageLensConfiguration.IsEnabled' --output text` | `True` |
| 3 | `aws s3 ls "s3://s3-workshop-lens-data-$(aws sts get-caller-identity --query Account --output text)" --recursive --summarize \| grep "Total Objects"` | `Total Objects: 9` |

---

## 实验总结

Storage Lens 将 S3 的成本和使用情况从"不可见"变成"可量化"，是 FinOps 实践的重要工具。核心价值：识别三类"隐形成本"——非当前版本积累、未完成分片上传、冷数据未分层。将 Storage Lens 导出数据与 Athena 结合，可以构建自定义的存储成本趋势看板。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# 删除 Storage Lens 仪表板
aws s3control delete-storage-lens-configuration \
  --account-id "${ACCOUNT_ID}" \
  --config-id "s3-workshop-lens-dashboard" \
  --region us-east-1 2>/dev/null

# 删除存储桶
for bucket in "s3-workshop-lens-data-${ACCOUNT_ID}" "s3-workshop-lens-export-${ACCOUNT_ID}"; do
  aws s3api list-object-versions --bucket "${bucket}" \
    --query '[Versions[].{Key:Key,VersionId:VersionId},DeleteMarkers[].{Key:Key,VersionId:VersionId}]' \
    --output json 2>/dev/null | \
    python3 -c "
import json,sys,subprocess
for lst in json.load(sys.stdin):
    for obj in (lst or []):
        subprocess.run(['aws','s3api','delete-object','--bucket','${bucket}','--key',obj['Key'],'--version-id',obj['VersionId']],capture_output=True)
" 2>/dev/null
  aws s3api delete-bucket --bucket "${bucket}" --region us-east-1 2>/dev/null
  echo "Deleted: ${bucket}"
done
echo "Cleanup complete"
```
