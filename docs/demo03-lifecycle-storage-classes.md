# Demo03 — S3 存储类型与生命周期管理

## 实验简介

S3 提供多种存储类型，访问频率越低、存储成本越低，但检索延迟和费用越高。生命周期规则让你无需手动干预，自动将对象从高频访问类型（Standard）迁移到低频（IA）再到归档（Glacier），并定期清理旧版本，实现存储成本的自动化优化。

**实验目标：**
- 掌握 S3 Standard、Intelligent-Tiering、Standard-IA、Glacier Instant Retrieval 的适用场景
- 理解生命周期规则的转换条件（天数）与过期删除机制
- 能够独立配置多层生命周期规则并结合版本控制清理旧版本

**实验流程：**
1. 创建存储桶并开启版本控制
2. 上传不同类型的测试对象
3. 配置多层存储类型转换的生命周期规则
4. 配置版本历史自动清理规则
5. 验证规则配置并观察生效状态

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
export BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建存储桶并开启版本控制

生命周期规则中的"非当前版本"过期清理（`NoncurrentVersionExpiration`）依赖版本控制，必须先启用。没有版本控制时，该配置项会被忽略。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"

aws s3api create-bucket \
  --bucket "${BUCKET_NAME}" \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket "${BUCKET_NAME}" \
  --versioning-configuration Status=Enabled

echo "Bucket created with versioning: ${BUCKET_NAME}"
```

**预期输出**：`Bucket created with versioning: s3-workshop-lifecycle-<ACCOUNT_ID>`

### 2. 上传不同类型的测试对象

模拟实际业务场景：`logs/` 前缀存放访问日志（典型冷数据）、`data/` 前缀存放业务数据（温数据），后续为这两类数据配置不同的生命周期策略。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"

# 上传日志类数据（冷数据，适合快速归档）
for i in 1 2 3; do
  echo "Log entry ${i} - $(date)" > /tmp/access-${i}.log
  aws s3 cp /tmp/access-${i}.log "s3://${BUCKET_NAME}/logs/2024/access-${i}.log" --quiet
done

# 上传业务数据（温数据，中等访问频率）
for i in 1 2; do
  echo '{"record": '${i}', "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' > /tmp/record-${i}.json
  aws s3 cp /tmp/record-${i}.json "s3://${BUCKET_NAME}/data/record-${i}.json" --quiet
done

# 验证上传结果
aws s3 ls "s3://${BUCKET_NAME}" --recursive --summarize | grep "Total"
```

**预期输出**：
```
Total Objects: 5
Total Size: <size>
```

### 3. 直接上传 Intelligent-Tiering 类型的对象

对于访问模式不可预测的数据，可以在上传时直接指定 `INTELLIGENT_TIERING` 存储类型。S3 会自动监控访问频率，将 30 天未访问的数据移到低频层，120 天未访问的数据移到归档层，无需手动配置规则。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"

echo '{"type": "unpredictable-access", "data": "intelligent-tiering-demo"}' > /tmp/variable-data.json
aws s3 cp /tmp/variable-data.json \
  "s3://${BUCKET_NAME}/archive/variable-data.json" \
  --storage-class INTELLIGENT_TIERING
```

**预期输出**：`upload: /tmp/variable-data.json to s3://s3-workshop-lifecycle-<ACCOUNT_ID>/archive/variable-data.json`

验证存储类型：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"
aws s3api head-object \
  --bucket "${BUCKET_NAME}" \
  --key "archive/variable-data.json" \
  --query 'StorageClass' \
  --output text
```

**预期输出**：`INTELLIGENT_TIERING`

### 4. 配置多层生命周期规则（logs 前缀）

日志数据通常有清晰的"热-温-冷-删除"生命周期：上传后短期内可能需要查询，30 天后基本不用，90 天后归档，1 年后删除。这个规则仅作用于 `logs/` 前缀，不影响其他对象。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"

aws s3api put-bucket-lifecycle-configuration \
  --bucket "${BUCKET_NAME}" \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "logs-tiering-rule",
        "Status": "Enabled",
        "Filter": {"Prefix": "logs/"},
        "Transitions": [
          {
            "Days": 30,
            "StorageClass": "STANDARD_IA"
          },
          {
            "Days": 90,
            "StorageClass": "GLACIER_IR"
          }
        ],
        "Expiration": {
          "Days": 365
        },
        "NoncurrentVersionExpiration": {
          "NoncurrentDays": 30
        }
      },
      {
        "ID": "data-ia-rule",
        "Status": "Enabled",
        "Filter": {"Prefix": "data/"},
        "Transitions": [
          {
            "Days": 60,
            "StorageClass": "STANDARD_IA"
          }
        ],
        "NoncurrentVersionExpiration": {
          "NoncurrentDays": 60,
          "NewerNoncurrentVersions": 3
        }
      },
      {
        "ID": "cleanup-delete-markers",
        "Status": "Enabled",
        "Filter": {"Prefix": ""},
        "Expiration": {
          "ExpiredObjectDeleteMarker": true
        }
      }
    ]
  }'
echo "Lifecycle rules configured"
```

**预期输出**：`Lifecycle rules configured`

> ⚠️ 生命周期规则配置后最多 24 小时才会开始执行对象转换，这是正常的 S3 行为。验证阶段只需确认规则存在且配置正确，不需要等待对象实际转换。

### 5. 验证生命周期规则配置

列出所有生命周期规则，确认三条规则均已启用，转换天数和存储类型与预期一致。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"
aws s3api get-bucket-lifecycle-configuration \
  --bucket "${BUCKET_NAME}" \
  --query 'Rules[].{ID:ID, Status:Status, Prefix:Filter.Prefix}'
```

**预期输出**：
```json
[
    {"ID": "logs-tiering-rule", "Status": "Enabled", "Prefix": "logs/"},
    {"ID": "data-ia-rule", "Status": "Enabled", "Prefix": "data/"},
    {"ID": "cleanup-delete-markers", "Status": "Enabled", "Prefix": ""}
]
```

查看 logs 规则的转换配置：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"
aws s3api get-bucket-lifecycle-configuration \
  --bucket "${BUCKET_NAME}" \
  --query "Rules[?ID=='logs-tiering-rule'].Transitions"
```

**预期输出**：
```json
[
    [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER_IR"}
    ]
]
```

### 6. 配置同区域复制（SRR）

同区域复制（Same-Region Replication, SRR）将对象自动复制到同一区域内的另一个存储桶，常用于数据隔离、日志聚合和合规备份。SRR 与 CRR（跨区域）相比延迟更低且无跨区域传输费用。本步骤把 `data/` 前缀下的对象自动复制到独立的目标桶。

**前提**：S3 复制要求源和目标存储桶都开启版本控制（源桶已在 Step 1 完成）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"
SRR_DEST="s3-workshop-srr-dest-${ACCOUNT_ID}"

# 创建目标桶并开启版本控制
aws s3api create-bucket --bucket "${SRR_DEST}" --region us-east-1
aws s3api put-bucket-versioning \
  --bucket "${SRR_DEST}" \
  --versioning-configuration Status=Enabled

# 创建复制所需的 IAM 角色
aws iam create-role \
  --role-name "s3-srr-role-${ACCOUNT_ID}" \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "s3.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }' > /dev/null

# 附加复制权限：读源桶 + 写目标桶
aws iam put-role-policy \
  --role-name "s3-srr-role-${ACCOUNT_ID}" \
  --policy-name "srr-policy" \
  --policy-document "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
        \"Effect\": \"Allow\",
        \"Action\": [\"s3:GetObjectVersionForReplication\", \"s3:GetObjectVersionAcl\", \"s3:ListBucket\"],
        \"Resource\": [\"arn:aws:s3:::${BUCKET_NAME}\", \"arn:aws:s3:::${BUCKET_NAME}/*\"]
      },
      {
        \"Effect\": \"Allow\",
        \"Action\": [\"s3:ReplicateObject\", \"s3:ReplicateDelete\"],
        \"Resource\": \"arn:aws:s3:::${SRR_DEST}/*\"
      }
    ]
  }"

# 在源桶上启用复制规则（仅复制 data/ 前缀）
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/s3-srr-role-${ACCOUNT_ID}"
aws s3api put-bucket-replication \
  --bucket "${BUCKET_NAME}" \
  --replication-configuration "{
    \"Role\": \"${ROLE_ARN}\",
    \"Rules\": [{
      \"ID\": \"srr-data-prefix\",
      \"Priority\": 1,
      \"Status\": \"Enabled\",
      \"Filter\": {\"Prefix\": \"data/\"},
      \"Destination\": {\"Bucket\": \"arn:aws:s3:::${SRR_DEST}\"},
      \"DeleteMarkerReplication\": {\"Status\": \"Disabled\"}
    }]
  }"
echo "SRR configured: ${BUCKET_NAME}/data/ -> ${SRR_DEST}"
```

**预期输出**：`SRR configured: s3-workshop-lifecycle-<ACCOUNT_ID>/data/ -> s3-workshop-srr-dest-<ACCOUNT_ID>`

验证复制规则已启用：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"
aws s3api get-bucket-replication \
  --bucket "${BUCKET_NAME}" \
  --query 'ReplicationConfiguration.Rules[0].Status' \
  --output text
```

**预期输出**：`Enabled`

上传新对象，触发 SRR 复制（约 5-15 秒后在目标桶出现）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"
SRR_DEST="s3-workshop-srr-dest-${ACCOUNT_ID}"

echo '{"type": "srr-test", "ts": "now"}' > /tmp/srr-test.json
aws s3 cp /tmp/srr-test.json "s3://${BUCKET_NAME}/data/srr-test.json"

# 等待复制完成（轮询目标桶）
echo "Waiting for replication..."
for i in {1..12}; do
  RESULT=$(aws s3api head-object --bucket "${SRR_DEST}" --key "data/srr-test.json" 2>/dev/null)
  if [ $? -eq 0 ]; then
    echo "Replicated successfully to ${SRR_DEST}/data/srr-test.json"
    break
  fi
  sleep 5
done
```

**预期输出**：`Replicated successfully to s3-workshop-srr-dest-<ACCOUNT_ID>/data/srr-test.json`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 存储桶 `s3-workshop-lifecycle-<ACCOUNT_ID>` 版本控制状态为 `Enabled`
- [ ] 存储桶中包含 6 个对象，分布在 `logs/`、`data/`、`archive/` 三个前缀下
- [ ] `archive/variable-data.json` 的存储类型为 `INTELLIGENT_TIERING`
- [ ] 生命周期规则 `logs-tiering-rule`、`data-ia-rule`、`cleanup-delete-markers` 全部 `Enabled`
- [ ] `logs-tiering-rule` 配置了 30 天转 STANDARD_IA、90 天转 GLACIER_IR、365 天过期
- [ ] 复制规则 `srr-data-prefix` 状态为 `Enabled`，目标桶 `s3-workshop-srr-dest-<ACCOUNT_ID>` 中收到 `data/srr-test.json`

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3api get-bucket-versioning --bucket "s3-workshop-lifecycle-$(aws sts get-caller-identity --query Account --output text)" --query Status --output text` | `Enabled` |
| 2 | `aws s3api head-object --bucket "s3-workshop-lifecycle-$(aws sts get-caller-identity --query Account --output text)" --key "archive/variable-data.json" --query StorageClass --output text` | `INTELLIGENT_TIERING` |
| 3 | `aws s3api get-bucket-lifecycle-configuration --bucket "s3-workshop-lifecycle-$(aws sts get-caller-identity --query Account --output text)" --query 'length(Rules)' --output text` | `3` |
| 4 | `aws s3api get-bucket-lifecycle-configuration --bucket "s3-workshop-lifecycle-$(aws sts get-caller-identity --query Account --output text)" --output json \| python3 -c "import json,sys; rules=[r for r in json.load(sys.stdin)['Rules'] if r['ID']=='logs-tiering-rule']; print(rules[0]['Transitions'][0]['Days'])"` | `30` |
| 5 | `aws s3api get-bucket-replication --bucket "s3-workshop-lifecycle-$(aws sts get-caller-identity --query Account --output text)" --query 'ReplicationConfiguration.Rules[0].Status' --output text` | `Enabled` |

---

## 实验总结

本实验完整覆盖了 S3 的成本优化工具链：通过生命周期规则自动化"热→温→冷→删除"数据分层，通过 Intelligent-Tiering 处理不可预测访问模式，通过 NoncurrentVersionExpiration 清理版本历史避免版本控制的隐性成本，最后用同区域复制（SRR）为关键数据建立同账号、同区域的独立副本，满足数据隔离和合规备份需求。SRR 与 Demo05 的 CRR 合力构成完整的 S3 复制体系。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-lifecycle-${ACCOUNT_ID}"

# 删除生命周期规则
aws s3api delete-bucket-lifecycle --bucket "${BUCKET_NAME}"

# 删除所有对象版本和删除标记
aws s3api list-object-versions \
  --bucket "${BUCKET_NAME}" \
  --query '[Versions, DeleteMarkers]' \
  --output json | \
  python3 -c "
import json, sys, subprocess
lists = json.load(sys.stdin)
for lst in lists:
    if lst:
        for obj in lst:
            subprocess.run(['aws', 's3api', 'delete-object',
              '--bucket', '${BUCKET_NAME}',
              '--key', obj['Key'],
              '--version-id', obj['VersionId']], capture_output=True)
print('All versions and markers deleted')
"

# 删除存储桶
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1

# 删除 SRR 目标桶（先清空版本，再删桶）
SRR_DEST="s3-workshop-srr-dest-${ACCOUNT_ID}"
aws s3api list-object-versions \
  --bucket "${SRR_DEST}" \
  --query '[Versions, DeleteMarkers]' \
  --output json 2>/dev/null | \
  python3 -c "
import json, sys, subprocess
try:
    lists = json.load(sys.stdin)
    for lst in lists:
        if lst:
            for obj in lst:
                subprocess.run(['aws', 's3api', 'delete-object',
                  '--bucket', '${SRR_DEST}',
                  '--key', obj['Key'],
                  '--version-id', obj['VersionId']], capture_output=True)
except: pass
print('SRR dest cleared')
" 2>/dev/null || true
aws s3api delete-bucket --bucket "${SRR_DEST}" --region us-east-1 2>/dev/null || true

# 删除 SRR IAM 角色
aws iam delete-role-policy \
  --role-name "s3-srr-role-${ACCOUNT_ID}" \
  --policy-name "srr-policy" 2>/dev/null || true
aws iam delete-role --role-name "s3-srr-role-${ACCOUNT_ID}" 2>/dev/null || true

echo "Cleanup complete"
```
