# Demo13 — S3 静态网站托管与 CloudFront 加速

## 实验简介

S3 静态网站托管是部署前端应用、文档站点和活动页的零运维方案，结合 CloudFront 可以实现全球边缘加速、HTTPS 和自定义域名。本实验从创建网站存储桶、部署 HTML 页面开始，再接入 CloudFront 分发，完整走通一个生产可用的静态网站架构。

**实验目标：**
- 掌握 S3 静态网站托管的配置方法（索引页、错误页、重定向规则）
- 理解 OAC（Origin Access Control）的工作原理，防止直接访问 S3 绕过 CDN
- 能够独立搭建 S3 + CloudFront 的静态网站架构

**实验流程：**
1. 创建并配置静态网站存储桶
2. 上传 HTML 页面
3. 创建 CloudFront 分发并配置 OAC
4. 验证通过 CloudFront URL 访问网站

**预计时长：** 35 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x
- **权限**：`AmazonS3FullAccess`、`CloudFrontFullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建网站存储桶

CloudFront + OAC 架构下，存储桶**不需要**开启 Public Access，访问权限通过 Bucket Policy 授予 CloudFront 服务主体。直接用 S3 静态网站端点则需要公开，OAC 是更安全的现代做法。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"
aws s3api create-bucket \
  --bucket "${BUCKET_NAME}" \
  --region us-east-1
echo "Bucket created: ${BUCKET_NAME}"
```

**预期输出**：`Bucket created: s3-workshop-website-<ACCOUNT_ID>`

### 2. 上传网站内容

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"

cat > /tmp/index.html << 'EOF'
<!DOCTYPE html>
<html lang="zh">
<head><meta charset="UTF-8"><title>S3 Workshop</title></head>
<body>
  <h1>欢迎来到 S3 Workshop</h1>
  <p>本页面由 S3 + CloudFront 托管</p>
  <p>时间戳：2026-07-01</p>
</body>
</html>
EOF

cat > /tmp/error.html << 'EOF'
<!DOCTYPE html>
<html lang="zh">
<head><meta charset="UTF-8"><title>404 - 页面未找到</title></head>
<body><h1>404</h1><p>页面不存在</p><a href="/">返回首页</a></body>
</html>
EOF

aws s3 cp /tmp/index.html "s3://${BUCKET_NAME}/index.html" --content-type "text/html"
aws s3 cp /tmp/error.html "s3://${BUCKET_NAME}/error.html" --content-type "text/html"
```

**预期输出**：
```
upload: /tmp/index.html to s3://s3-workshop-website-<ACCOUNT_ID>/index.html
upload: /tmp/error.html to s3://s3-workshop-website-<ACCOUNT_ID>/error.html
```

### 3. 创建 CloudFront OAC

OAC（Origin Access Control）是 OAI 的替代方案，支持对 S3 的 SigV4 签名请求，安全性更高。创建 OAC 后，CloudFront 分发会使用它向 S3 发送带签名的请求，S3 Bucket Policy 只允许来自该 CloudFront 分发的请求。

```bash
OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "s3-workshop-oac",
    "Description": "OAC for S3 Workshop",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }' \
  --query 'OriginAccessControl.Id' \
  --output text)
echo "OAC ID: ${OAC_ID}"
```

**预期输出**：`OAC ID: <uuid>`

### 4. 创建 CloudFront 分发

CloudFront 分发的 `DomainName` 必须使用 S3 的区域域名格式（`<bucket>.s3.<region>.amazonaws.com`），而不是静态网站端点，OAC 才能正常工作。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"
OAC_ID=$(aws cloudfront list-origin-access-controls \
  --query "OriginAccessControlList.Items[?Name=='s3-workshop-oac'].Id" \
  --output text)

DISTRIBUTION_ID=$(aws cloudfront create-distribution \
  --distribution-config "{
    \"CallerReference\": \"s3-workshop-$(date +%s)\",
    \"Comment\": \"S3 Workshop Static Website\",
    \"DefaultCacheBehavior\": {
      \"TargetOriginId\": \"S3Origin\",
      \"ViewerProtocolPolicy\": \"redirect-to-https\",
      \"CachePolicyId\": \"658327ea-f89d-4fab-a63d-7e88639e58f6\",
      \"AllowedMethods\": {
        \"Quantity\": 2,
        \"Items\": [\"GET\", \"HEAD\"],
        \"CachedMethods\": {\"Quantity\": 2, \"Items\": [\"GET\", \"HEAD\"]}
      }
    },
    \"Origins\": {
      \"Quantity\": 1,
      \"Items\": [{
        \"Id\": \"S3Origin\",
        \"DomainName\": \"${BUCKET_NAME}.s3.us-east-1.amazonaws.com\",
        \"S3OriginConfig\": {\"OriginAccessIdentity\": \"\"},
        \"OriginAccessControlId\": \"${OAC_ID}\"
      }]
    },
    \"DefaultRootObject\": \"index.html\",
    \"CustomErrorResponses\": {
      \"Quantity\": 1,
      \"Items\": [{
        \"ErrorCode\": 404,
        \"ResponsePagePath\": \"/error.html\",
        \"ResponseCode\": \"404\",
        \"ErrorCachingMinTTL\": 10
      }]
    },
    \"Enabled\": true,
    \"HttpVersion\": \"http2\"
  }" \
  --query 'Distribution.{Id:Id, DomainName:DomainName}' \
  --output json)

echo "${DISTRIBUTION_ID}"
```

**预期输出**：
```json
{
    "Id": "<distribution-id>",
    "DomainName": "<uuid>.cloudfront.net"
}
```

> ⚠️ CloudFront 分发创建后需要约 5-10 分钟才能部署到所有边缘节点，状态从 `InProgress` 变为 `Deployed` 才可访问。

### 5. 为 S3 配置 Bucket Policy 授权 CloudFront

只有添加了 Bucket Policy，CloudFront 才能访问私有存储桶中的对象。`AWS:SourceArn` 条件将权限锁定到特定分发，防止其他 CloudFront 分发也能读取此桶。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"
DISTRIBUTION_ID=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Comment=='S3 Workshop Static Website'].Id" \
  --output text)

aws s3api put-bucket-policy \
  --bucket "${BUCKET_NAME}" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Sid\": \"AllowCloudFrontOAC\",
      \"Effect\": \"Allow\",
      \"Principal\": {\"Service\": \"cloudfront.amazonaws.com\"},
      \"Action\": \"s3:GetObject\",
      \"Resource\": \"arn:aws:s3:::${BUCKET_NAME}/*\",
      \"Condition\": {
        \"StringEquals\": {
          \"AWS:SourceArn\": \"arn:aws:cloudfront::${ACCOUNT_ID}:distribution/${DISTRIBUTION_ID}\"
        }
      }
    }]
  }"
echo "Bucket policy updated for CloudFront distribution: ${DISTRIBUTION_ID}"
```

**预期输出**：`Bucket policy updated for CloudFront distribution: <distribution-id>`

### 6. 等待分发就绪并验证访问

CloudFront 分发状态必须为 `Deployed` 才能正常响应请求。等待期间可以继续其他操作，不需要轮询等待。

```bash
DISTRIBUTION_ID=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Comment=='S3 Workshop Static Website'].Id" \
  --output text)

aws cloudfront get-distribution \
  --id "${DISTRIBUTION_ID}" \
  --query 'Distribution.Status' \
  --output text
```

**预期输出**：`Deployed`（若返回 `InProgress`，等 5 分钟后重试）

获取访问 URL 并验证：
```bash
CF_DOMAIN=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Comment=='S3 Workshop Static Website'].DomainName" \
  --output text)
echo "Website URL: https://${CF_DOMAIN}"
curl -s "https://${CF_DOMAIN}" | grep -o "<h1>.*</h1>"
```

**预期输出**：`<h1>欢迎来到 S3 Workshop</h1>`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] S3 存储桶 `s3-workshop-website-<ACCOUNT_ID>` 中包含 `index.html` 和 `error.html`
- [ ] CloudFront 分发状态为 `Deployed`，`DefaultRootObject` 为 `index.html`
- [ ] 通过 `https://<uuid>.cloudfront.net` 访问返回 H1 标题
- [ ] 访问不存在的路径返回自定义 404 页面

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3 ls "s3://s3-workshop-website-$(aws sts get-caller-identity --query Account --output text)/" --summarize \| grep "Total Objects"` | `Total Objects: 2` |
| 2 | `aws cloudfront list-distributions --query "DistributionList.Items[?Comment=='S3 Workshop Static Website'].Status" --output text` | `Deployed` |
| 3 | `aws cloudfront list-distributions --query "DistributionList.Items[?Comment=='S3 Workshop Static Website'].DefaultCacheBehavior.ViewerProtocolPolicy" --output text` | `redirect-to-https` |
| 4 | `aws cloudfront list-origin-access-controls --query "OriginAccessControlList.Items[?Name=='s3-workshop-oac'].SigningProtocol" --output text` | `sigv4` |

---

## 实验总结

本实验构建了生产级静态网站架构：S3 存储内容、CloudFront 负责全球分发和 HTTPS 终结、OAC 确保 S3 不对公网暴露。核心概念是 OAC 的 SigV4 签名机制——CloudFront 以自己的身份访问 S3，S3 Bucket Policy 通过 `AWS:SourceArn` 锁定特定分发，形成最小权限闭环。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"

# 禁用并删除 CloudFront 分发（必须先禁用）
DISTRIBUTION_ID=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Comment=='S3 Workshop Static Website'].Id" \
  --output text)
if [ -n "${DISTRIBUTION_ID}" ]; then
  ETAG=$(aws cloudfront get-distribution-config \
    --id "${DISTRIBUTION_ID}" --query 'ETag' --output text)
  CONFIG=$(aws cloudfront get-distribution-config \
    --id "${DISTRIBUTION_ID}" --query 'DistributionConfig' --output json | \
    python3 -c "import json,sys; c=json.load(sys.stdin); c['Enabled']=False; print(json.dumps(c))")
  aws cloudfront update-distribution \
    --id "${DISTRIBUTION_ID}" --if-match "${ETAG}" \
    --distribution-config "${CONFIG}" > /dev/null
  echo "Distribution disabled. Wait ~5 min then run delete manually:"
  echo "aws cloudfront delete-distribution --id ${DISTRIBUTION_ID} --if-match \$(aws cloudfront get-distribution --id ${DISTRIBUTION_ID} --query ETag --output text)"
fi

# 删除 OAC
OAC_ID=$(aws cloudfront list-origin-access-controls \
  --query "OriginAccessControlList.Items[?Name=='s3-workshop-oac'].Id" --output text)
OAC_ETAG=$(aws cloudfront get-origin-access-control \
  --id "${OAC_ID}" --query 'ETag' --output text 2>/dev/null)
[ -n "${OAC_ID}" ] && aws cloudfront delete-origin-access-control \
  --id "${OAC_ID}" --if-match "${OAC_ETAG}" 2>/dev/null

# 删除存储桶
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Bucket deleted: ${BUCKET_NAME}"
```
