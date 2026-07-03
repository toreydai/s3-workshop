# Demo09 — 预签名 URL 与临时授权访问

## 实验简介

预签名 URL（Presigned URL）是 S3 实现"私有对象临时共享"的核心机制，广泛用于下载中心、文件分享链接、前端直传等场景。URL 携带了请求方的凭证签名和过期时间，持有 URL 的任何人（无需 AWS 账号）都可以在有效期内访问指定对象。本实验覆盖下载预签名和上传预签名两种用法。

**实验目标：**
- 掌握 GET 预签名 URL 的生成方式与有效期控制
- 理解 PUT 预签名 URL 实现前端直传 S3 的工作原理
- 能够通过预签名 URL 实现私有对象的安全临时授权

**实验流程：**
1. 创建私有存储桶并上传测试对象
2. 生成 GET 预签名 URL 并验证访问
3. 生成 PUT 预签名 URL 并模拟前端直传
4. 验证过期机制

**预计时长：** 20 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x，curl
- **权限**：`AmazonS3FullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-presign-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建私有存储桶并上传对象

存储桶保持默认的私有状态（Block Public Access 全开），后续通过预签名 URL 演示在不修改权限的情况下临时授权访问。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-presign-${ACCOUNT_ID}"
aws s3api create-bucket --bucket "${BUCKET_NAME}" --region us-east-1

echo "这是一份机密报告，只有持有链接的人才能在有效期内下载" > /tmp/report.txt
aws s3 cp /tmp/report.txt "s3://${BUCKET_NAME}/reports/confidential-report.txt"
```

**预期输出**：`upload: /tmp/report.txt to s3://s3-workshop-presign-<ACCOUNT_ID>/reports/confidential-report.txt`

验证对象已上传且存储桶为私有：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-presign-${ACCOUNT_ID}"
aws s3api get-public-access-block \
  --bucket "${BUCKET_NAME}" \
  --query 'PublicAccessBlockConfiguration.BlockPublicPolicy' \
  --output text
```

**预期输出**：`True`

### 2. 生成 GET 预签名 URL（有效期 5 分钟）

`presign` 命令用当前凭证对请求进行签名，生成包含时效信息的 URL。注意：生成 URL 的凭证被删除或 URL 过期后，链接立即失效；URL 本身无法撤销，所以过期时间要设置合理。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-presign-${ACCOUNT_ID}"
PRESIGN_URL=$(aws s3 presign \
  "s3://${BUCKET_NAME}/reports/confidential-report.txt" \
  --expires-in 300 \
  --region us-east-1)
echo "预签名 URL（5 分钟有效）:"
echo "${PRESIGN_URL}" | cut -c1-100
echo "..."
```

**预期输出**：以 `https://s3-workshop-presign-` 开头的长 URL（含 `X-Amz-Expires=300`）

### 3. 用预签名 URL 下载对象

任何人（包括没有 AWS 凭证的人）都可以用 curl 下载这个 URL，HTTP 响应头中可以看到 S3 的签名验证信息。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-presign-${ACCOUNT_ID}"
PRESIGN_URL=$(aws s3 presign \
  "s3://${BUCKET_NAME}/reports/confidential-report.txt" \
  --expires-in 300 \
  --region us-east-1)

curl -s "${PRESIGN_URL}"
```

**预期输出**：`这是一份机密报告，只有持有链接的人才能在有效期内下载`

### 4. 生成 PUT 预签名 URL（前端直传）

PUT 预签名 URL 让客户端直接上传文件到 S3，无需经过你的服务器，减少带宽消耗。服务端生成 URL 时可以限定 `Content-Type`，客户端上传时必须传相同 Content-Type，否则签名校验失败报 403。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-presign-${ACCOUNT_ID}"

# AWS CLI v2 不支持 --http-method PUT，使用 boto3 生成 PUT 预签名 URL
PUT_URL=$(python3 -c "
import boto3
s3 = boto3.client('s3', region_name='us-east-1')
url = s3.generate_presigned_url(
    'put_object',
    Params={'Bucket': '${BUCKET_NAME}', 'Key': 'uploads/user-upload.txt', 'ContentType': 'text/plain'},
    ExpiresIn=600
)
print(url)
")
echo "PUT 预签名 URL（10 分钟有效）:"
echo "${PUT_URL}" | cut -c1-100
echo "..."
```

> ⚠️ AWS CLI v2 的 `aws s3 presign` 命令不支持 `--http-method PUT` 参数，只能生成 GET 预签名 URL。生成 PUT 预签名 URL 需要使用 AWS SDK（如 Python boto3 的 `generate_presigned_url('put_object', ...)`）。

**预期输出**：含 `X-Amz-Expires=600` 的 PUT 预签名 URL

用 curl 模拟客户端直传：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-presign-${ACCOUNT_ID}"
PUT_URL=$(python3 -c "
import boto3
s3 = boto3.client('s3', region_name='us-east-1')
url = s3.generate_presigned_url(
    'put_object',
    Params={'Bucket': '${BUCKET_NAME}', 'Key': 'uploads/user-upload.txt', 'ContentType': 'text/plain'},
    ExpiresIn=600
)
print(url)
")

curl -s -X PUT \
  -H "Content-Type: text/plain" \
  -d "这是用户通过预签名 URL 直传的文件内容" \
  "${PUT_URL}"
echo "Upload via PUT presigned URL: done"
```

**预期输出**：（无输出或空响应表示上传成功，HTTP 200）

验证文件已上传：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-presign-${ACCOUNT_ID}"
aws s3 cp "s3://${BUCKET_NAME}/uploads/user-upload.txt" - 2>/dev/null
```

**预期输出**：`这是用户通过预签名 URL 直传的文件内容`

### 5. 验证过期机制

生成一个 5 秒有效期的极短链接，等待过期后再次访问，确认收到 403 拒绝响应。这验证了预签名 URL 的时效性保护机制。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-presign-${ACCOUNT_ID}"
SHORT_URL=$(aws s3 presign \
  "s3://${BUCKET_NAME}/reports/confidential-report.txt" \
  --expires-in 5 \
  --region us-east-1)

echo "等待 10 秒让链接过期..."
sleep 10

HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "${SHORT_URL}")
echo "过期后 HTTP 状态码: ${HTTP_CODE}"
```

**预期输出**：`过期后 HTTP 状态码: 403`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 私有存储桶中包含 `reports/confidential-report.txt` 和 `uploads/user-upload.txt`
- [ ] GET 预签名 URL 在有效期内返回文件内容
- [ ] PUT 预签名 URL 成功将内容上传到 `uploads/user-upload.txt`
- [ ] 过期的预签名 URL 返回 HTTP 403

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3 ls "s3://s3-workshop-presign-$(aws sts get-caller-identity --query Account --output text)" --recursive --summarize \| grep "Total Objects"` | `Total Objects: 2` |
| 2 | `aws s3api get-public-access-block --bucket "s3-workshop-presign-$(aws sts get-caller-identity --query Account --output text)" --query 'PublicAccessBlockConfiguration.BlockPublicAcls' --output text` | `True` |
| 3 | `aws s3 cp "s3://s3-workshop-presign-$(aws sts get-caller-identity --query Account --output text)/uploads/user-upload.txt" - 2>/dev/null \| grep -c "预签名"` | `1` |

---

## 实验总结

本实验掌握了预签名 URL 的两种核心用法：GET 用于私有内容的临时分享，PUT 用于前端绕过服务器直传 S3（降低服务端带宽压力）。关键安全概念：URL 的有效性与签发凭证绑定，凭证吊销即失效；URL 一旦生成无法撤销，有效期设置要精准。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-presign-${ACCOUNT_ID}"
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Bucket deleted: ${BUCKET_NAME}"
```
