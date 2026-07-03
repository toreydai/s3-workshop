# Demo02 — 多部分上传（Multipart Upload）大文件处理

## 实验简介

单次 PUT 请求最大只能上传 5GB 对象，超过 5MB 的对象推荐使用多部分上传（Multipart Upload）。多部分上传将大文件分割成多个分片并行上传，支持断点续传、提升吞吐量，并在部分分片失败时只重传失败的片段。本实验手动演示多部分上传的完整流程，并配置生命周期规则自动清理未完成的多部分上传。

**实验目标：**
- 掌握多部分上传的三步流程：Initiate → Upload Parts → Complete
- 理解分片最小 5MB 的限制（最后一片除外）
- 能够配置生命周期规则自动中止未完成的多部分上传，避免产生隐性费用

**实验流程：**
1. 创建存储桶
2. 启动多部分上传并获取 UploadId
3. 分片上传并收集 ETag
4. 完成多部分上传
5. 配置自动清理规则

**预计时长：** 25 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x，Python3，dd 命令
- **权限**：`AmazonS3FullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-multipart-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建存储桶

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-multipart-${ACCOUNT_ID}"
aws s3api create-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Bucket created: ${BUCKET_NAME}"
```

**预期输出**：`Bucket created: s3-workshop-multipart-<ACCOUNT_ID>`

### 2. 创建测试大文件（三个分片，每片 6MB）

最小分片大小为 5MB（最后一片不限），实验使用 6MB 分片满足限制。生产场景通常使用 50-100MB 的分片以获得更好的并发效率。

```bash
# 创建 3 个 6MB 测试文件作为分片
for i in 1 2 3; do
  dd if=/dev/urandom of="/tmp/part-${i}.bin" bs=1M count=6 2>/dev/null
  echo "Part ${i}: $(ls -lh /tmp/part-${i}.bin | awk '{print $5}')"
done
```

**预期输出**：
```
Part 1: 6.0M
Part 2: 6.0M
Part 3: 6.0M
```

### 3. 启动多部分上传，获取 UploadId

`create-multipart-upload` 仅预留一个 UploadId，不传输任何数据。UploadId 是整个上传会话的凭证，每一个 `upload-part` 请求都需要带上它。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-multipart-${ACCOUNT_ID}"

UPLOAD_ID=$(aws s3api create-multipart-upload \
  --bucket "${BUCKET_NAME}" \
  --key "large-files/18mb-test-file.bin" \
  --content-type "application/octet-stream" \
  --region us-east-1 \
  --query 'UploadId' \
  --output text)
echo "UploadId: ${UPLOAD_ID}"
```

**预期输出**：`UploadId: <long-alphanumeric-string>`

### 4. 并行上传各分片

每个分片上传成功后返回 ETag，Complete 阶段需要提交所有分片的 ETag 列表。分片编号（PartNumber）从 1 开始，最大 10000。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-multipart-${ACCOUNT_ID}"
UPLOAD_ID=$(aws s3api list-multipart-uploads \
  --bucket "${BUCKET_NAME}" \
  --region us-east-1 \
  --query 'Uploads[0].UploadId' --output text)

ETAGS=""
for i in 1 2 3; do
  ETAG=$(aws s3api upload-part \
    --bucket "${BUCKET_NAME}" \
    --key "large-files/18mb-test-file.bin" \
    --part-number "${i}" \
    --upload-id "${UPLOAD_ID}" \
    --body "/tmp/part-${i}.bin" \
    --region us-east-1 \
    --query 'ETag' \
    --output text)
  echo "Part ${i} uploaded, ETag: ${ETAG}"
  ETAGS="${ETAGS}{\"ETag\": ${ETAG}, \"PartNumber\": ${i}},"
done
echo "All parts uploaded"
```

**预期输出**：
```
Part 1 uploaded, ETag: "xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
Part 2 uploaded, ETag: "xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
Part 3 uploaded, ETag: "xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
All parts uploaded
```

列出已上传的分片确认完整性：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-multipart-${ACCOUNT_ID}"
UPLOAD_ID=$(aws s3api list-multipart-uploads \
  --bucket "${BUCKET_NAME}" --region us-east-1 \
  --query 'Uploads[0].UploadId' --output text)

aws s3api list-parts \
  --bucket "${BUCKET_NAME}" \
  --key "large-files/18mb-test-file.bin" \
  --upload-id "${UPLOAD_ID}" \
  --region us-east-1 \
  --query 'Parts[].{Part:PartNumber, Size:Size}'
```

**预期输出**：
```json
[
    {"Part": 1, "Size": 6291456},
    {"Part": 2, "Size": 6291456},
    {"Part": 3, "Size": 6291456}
]
```

### 5. 完成多部分上传

Complete 请求将所有分片合并为一个完整对象。`multipart.json` 列出所有分片的 ETag，顺序必须与 PartNumber 对应，否则合并后内容错乱。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-multipart-${ACCOUNT_ID}"
UPLOAD_ID=$(aws s3api list-multipart-uploads \
  --bucket "${BUCKET_NAME}" --region us-east-1 \
  --query 'Uploads[0].UploadId' --output text)

PARTS_JSON=$(aws s3api list-parts \
  --bucket "${BUCKET_NAME}" \
  --key "large-files/18mb-test-file.bin" \
  --upload-id "${UPLOAD_ID}" \
  --region us-east-1 \
  --query 'Parts[].{ETag:ETag, PartNumber:PartNumber}' \
  --output json)

aws s3api complete-multipart-upload \
  --bucket "${BUCKET_NAME}" \
  --key "large-files/18mb-test-file.bin" \
  --upload-id "${UPLOAD_ID}" \
  --multipart-upload "{\"Parts\": ${PARTS_JSON}}" \
  --region us-east-1 \
  --query '{Location:Location, ETag:ETag}' \
  --output json
```

**预期输出**：
```json
{
    "Location": "https://s3-workshop-multipart-<ACCOUNT_ID>.s3.us-east-1.amazonaws.com/large-files%2F18mb-test-file.bin",
    "ETag": "\"xxxxxxxxxxxxxxxxxxxxxxxxxxxx-3\""
}
```

> ⚠️ 合并后对象的 ETag 格式为 `"<md5>-<分片数>"`，与普通上传的 ETag（纯 MD5）不同，不要用 ETag 来校验多部分上传对象的内容完整性。
>
> ⚠️ 实测 `Location` 字段包含区域域名（`s3.us-east-1.amazonaws.com` 而非 `s3.amazonaws.com`），且 Key 中的 `/` 被 URL 编码为 `%2F`，与部分旧文档示例不同，属正常行为。

### 6. 配置生命周期规则自动中止未完成的多部分上传

未完成的多部分上传会产生存储费用但对象不可见，这是 S3 成本的常见"隐性泄漏"。配置 `AbortIncompleteMultipartUpload` 规则，超过 7 天的未完成上传自动清理。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-multipart-${ACCOUNT_ID}"

aws s3api put-bucket-lifecycle-configuration \
  --bucket "${BUCKET_NAME}" \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "abort-incomplete-multipart",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }]
  }'
echo "Lifecycle rule configured: abort incomplete multipart after 7 days"
```

**预期输出**：`Lifecycle rule configured: abort incomplete multipart after 7 days`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 存储桶 `s3-workshop-multipart-<ACCOUNT_ID>` 中包含已合并的 `large-files/18mb-test-file.bin`
- [ ] 对象大小为 18MB（3 × 6MB）
- [ ] 对象 ETag 后缀为 `-3`（表示 3 个分片）
- [ ] 生命周期规则 `abort-incomplete-multipart` 状态为 `Enabled`

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3api head-object --bucket "s3-workshop-multipart-$(aws sts get-caller-identity --query Account --output text)" --key "large-files/18mb-test-file.bin" --query 'ContentLength' --output text` | `18874368` |
| 2 | `aws s3api list-multipart-uploads --bucket "s3-workshop-multipart-$(aws sts get-caller-identity --query Account --output text)" --region us-east-1 --query "length(Uploads || \`[]\`)" --output text` | `0` |
| 3 | `aws s3api get-bucket-lifecycle-configuration --bucket "s3-workshop-multipart-$(aws sts get-caller-identity --query Account --output text)" --query 'Rules[0].AbortIncompleteMultipartUpload.DaysAfterInitiation' --output text` | `7` |

---

## 实验总结

本实验完整演示了多部分上传的三步流程，并揭示了一个重要的成本陷阱：未完成的分片会持续计费，必须通过生命周期规则兜底清理。实际生产中推荐使用 AWS SDK 的高级传输管理器（如 Python boto3 的 `S3Transfer`），它会自动处理分片大小计算、并发控制和断点续传。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-multipart-${ACCOUNT_ID}"
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Bucket deleted: ${BUCKET_NAME}"
```
