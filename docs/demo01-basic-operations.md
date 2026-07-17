# Demo01 — S3 基础操作：存储桶创建与对象管理

## 实验简介

Amazon S3 是 AWS 最核心的对象存储服务，几乎所有 AWS 架构都会依赖它存储数据。本实验带你从零开始创建存储桶、上传/下载对象、管理元数据，并通过版本控制体验 S3 的数据保护机制。

**实验目标：**
- 掌握 S3 存储桶的创建、配置与删除操作
- 理解 S3 对象的上传、下载、元数据管理工作原理
- 能够独立开启版本控制并恢复历史版本

**实验流程：**
1. 创建 S3 存储桶
2. 上传与下载对象
3. 管理对象元数据与标签
4. 开启版本控制并验证版本恢复

**预计 AI 执行时长：** 20 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x（`aws --version` 确认）
- **权限**：`AmazonS3FullAccess` 或等效权限
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建 S3 存储桶

在 us-east-1 创建存储桶时，**不需要**传 `--create-bucket-configuration` 参数，这是 AWS API 的特殊约定；其他区域必须传该参数，否则报 `IllegalLocationConstraintException`。存储桶名称在全 AWS 范围内唯一，使用 Account ID 后缀避免冲突。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"
aws s3api create-bucket \
  --bucket "${BUCKET_NAME}" \
  --region us-east-1
```

**预期输出**：
```json
{
    "Location": "/s3-workshop-basic-<ACCOUNT_ID>"
}
```

### 2. 开启存储桶版本控制

版本控制让 S3 保留对象的所有历史版本，是误删恢复和合规审计的基础。启用后无法"完全关闭"，只能暂停（Suspended），且已有版本永久保留。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"
aws s3api put-bucket-versioning \
  --bucket "${BUCKET_NAME}" \
  --versioning-configuration Status=Enabled
```

**预期输出**：（无输出表示成功）

验证版本控制已启用：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"
aws s3api get-bucket-versioning \
  --bucket "${BUCKET_NAME}" \
  --query Status \
  --output text
```

**预期输出**：`Enabled`

### 3. 上传对象

先在本地创建两个测试文件，再上传到存储桶。S3 上传时可以指定 `--content-type` 告知对象的 MIME 类型，否则 S3 会默认设为 `binary/octet-stream`。

```bash
# 创建测试文件
echo "Hello, S3 Workshop! Version 1" > /tmp/hello.txt
echo '{"name": "S3 Workshop", "version": 1}' > /tmp/config.json

# 上传文件
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"

aws s3 cp /tmp/hello.txt "s3://${BUCKET_NAME}/hello.txt" --content-type "text/plain"
aws s3 cp /tmp/config.json "s3://${BUCKET_NAME}/config/config.json" --content-type "application/json"
```

**预期输出**：
```
upload: /tmp/hello.txt to s3://s3-workshop-basic-<ACCOUNT_ID>/hello.txt
upload: /tmp/config.json to s3://s3-workshop-basic-<ACCOUNT_ID>/config/config.json
```

### 4. 列出存储桶内容

`s3 ls` 命令带 `--recursive` 可以列出所有层级的对象，类似 `find` 命令。不带该参数时只列出当前"目录"层级（S3 实际上没有目录，斜杠是 key 的一部分）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"
aws s3 ls "s3://${BUCKET_NAME}" --recursive
```

**预期输出**：
```
<date> <time>         30 config/config.json
<date> <time>         30 hello.txt
```

### 5. 为对象添加标签

标签是 S3 对象元数据管理和成本分配的核心工具。添加标签后可以通过标签筛选对象，也可以在生命周期规则、S3 Inventory 中按标签过滤。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"
aws s3api put-object-tagging \
  --bucket "${BUCKET_NAME}" \
  --key "hello.txt" \
  --tagging '{"TagSet": [{"Key": "Environment", "Value": "Workshop"}, {"Key": "Owner", "Value": "Demo01"}]}'
```

**预期输出**：（无输出表示成功）

验证标签已添加：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"
aws s3api get-object-tagging \
  --bucket "${BUCKET_NAME}" \
  --key "hello.txt"
```

**预期输出**：
```json
{
    "TagSet": [
        {"Key": "Environment", "Value": "Workshop"},
        {"Key": "Owner", "Value": "Demo01"}
    ]
}
```

### 6. 上传新版本并验证版本控制

覆盖上传同名对象时，S3 会创建新版本而不是覆盖原内容。每个版本有唯一的 VersionId，可以通过指定 VersionId 下载或恢复任意历史版本。

> ⚠️ **标签继承问题**：上传新版本后，旧版本上的标签**不会自动迁移**到新版本。新版本的标签默认为空。如果需要新版本也带有标签，需在步骤6上传完成后，重新执行 `put-object-tagging` 对最新版本打标签。验证检查点4在步骤6完成后执行 `get-object-tagging` 时，默认读取的是最新版本（无标签），需要在上传后重新打标签才能通过验证。

```bash
echo "Hello, S3 Workshop! Version 2 - Updated" > /tmp/hello.txt
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"
aws s3 cp /tmp/hello.txt "s3://${BUCKET_NAME}/hello.txt" --content-type "text/plain"
```

列出所有版本：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"
aws s3api list-object-versions \
  --bucket "${BUCKET_NAME}" \
  --prefix "hello.txt" \
  --query 'Versions[].{VersionId:VersionId, LastModified:LastModified, IsLatest:IsLatest}'
```

**预期输出**（两个版本）：
```json
[
    {"VersionId": "xxx", "LastModified": "...", "IsLatest": true},
    {"VersionId": "yyy", "LastModified": "...", "IsLatest": false}
]
```

### 7. 通过 VersionId 恢复历史版本

版本控制的核心价值在于可以按 VersionId 精确下载任意历史版本，实现"误操作回滚"。先取出旧版本的 VersionId，再通过 `--version-id` 下载，验证内容是 "Version 1" 而非最新的 "Version 2"。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"

# 取出非最新版本（即 Version 1）的 VersionId
OLD_VERSION_ID=$(aws s3api list-object-versions \
  --bucket "${BUCKET_NAME}" \
  --prefix "hello.txt" \
  --query 'Versions[?IsLatest==`false`].VersionId' \
  --output text)
echo "旧版本 VersionId: ${OLD_VERSION_ID}"

# 按 VersionId 下载旧版本
aws s3api get-object \
  --bucket "${BUCKET_NAME}" \
  --key "hello.txt" \
  --version-id "${OLD_VERSION_ID}" \
  /tmp/hello-v1.txt > /dev/null
cat /tmp/hello-v1.txt
```

**预期输出**：
```
旧版本 VersionId: <old-version-id>
Hello, S3 Workshop! Version 1
```

### 8. 下载最新版本（对比验证）

不指定 `--version-id` 时默认取最新版本，验证与旧版本内容不同，确认版本隔离正常工作。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"
aws s3 cp "s3://${BUCKET_NAME}/hello.txt" /tmp/hello-latest.txt
cat /tmp/hello-latest.txt
```

**预期输出**：
```
Hello, S3 Workshop! Version 2 - Updated
```

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 成功创建 S3 存储桶，名称格式为 `s3-workshop-basic-<ACCOUNT_ID>`
- [ ] 存储桶版本控制状态为 `Enabled`
- [ ] 存储桶中包含至少两个对象：`hello.txt` 和 `config/config.json`
- [ ] `hello.txt` 有两个版本，最新版本内容包含 "Version 2"
- [ ] `hello.txt` 带有 `Environment=Workshop` 标签

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3api get-bucket-versioning --bucket "s3-workshop-basic-$(aws sts get-caller-identity --query Account --output text)" --query Status --output text` | `Enabled` |
| 2 | `aws s3 ls "s3://s3-workshop-basic-$(aws sts get-caller-identity --query Account --output text)" --recursive --summarize \| grep "Total Objects"` | `Total Objects: 2` |
| 3 | `aws s3api list-object-versions --bucket "s3-workshop-basic-$(aws sts get-caller-identity --query Account --output text)" --prefix "hello.txt" --query 'length(Versions)' --output text` | `2` |
| 4 | `aws s3api get-object-tagging --bucket "s3-workshop-basic-$(aws sts get-caller-identity --query Account --output text)" --key "hello.txt" --query "TagSet[?Key=='Environment'].Value" --output text` | `Workshop` |

---

## 实验总结

本实验构建了 S3 对象存储的完整基础操作链路：从创建存储桶、上传对象、管理标签，到开启版本控制并验证版本历史。核心概念包括：S3 Key 的命名与"目录"模拟机制、版本控制的不可逆开启行为，以及标签在资源治理中的作用。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-basic-${ACCOUNT_ID}"

# 删除所有对象版本（版本控制开启时必须先删除所有版本才能删桶）
aws s3api list-object-versions \
  --bucket "${BUCKET_NAME}" \
  --query '{Objects: Versions[].{Key:Key, VersionId:VersionId}}' \
  --output json | \
  python3 -c "
import json, sys, subprocess
data = json.load(sys.stdin)
if data.get('Objects'):
    for obj in data['Objects']:
        subprocess.run(['aws', 's3api', 'delete-object',
          '--bucket', '${BUCKET_NAME}',
          '--key', obj['Key'],
          '--version-id', obj['VersionId']])
print('All versions deleted')
"

# 删除所有删除标记
aws s3api list-object-versions \
  --bucket "${BUCKET_NAME}" \
  --query '{Objects: DeleteMarkers[].{Key:Key, VersionId:VersionId}}' \
  --output json | \
  python3 -c "
import json, sys, subprocess
data = json.load(sys.stdin)
if data.get('Objects'):
    for obj in data['Objects']:
        subprocess.run(['aws', 's3api', 'delete-object',
          '--bucket', '${BUCKET_NAME}',
          '--key', obj['Key'],
          '--version-id', obj['VersionId']])
print('All delete markers removed')
"

# 删除存储桶
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Bucket ${BUCKET_NAME} deleted"
```
