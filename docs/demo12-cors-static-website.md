# Demo12 — S3 静态网站托管、CloudFront 加速与 CORS 配置

## 实验简介

S3 静态网站托管是部署前端应用、文档站点和活动页的零运维方案，结合 CloudFront 可以实现全球边缘加速、HTTPS 和自定义域名。但网站上线后，前端 JavaScript 往往还需要绕开 CDN 直接访问 S3——比如用预签名 URL 直传文件、或跨源读取 S3 上的 JSON 数据——这时浏览器的同源策略会介入，S3 必须返回正确的 CORS 头才会放行。本实验先搭建 S3 + CloudFront 静态网站架构，再在同一个桶上补齐 CORS 配置，覆盖预检请求（Preflight）、PUT 直传、GET 读取的完整验证流程。

**实验目标：**
- 掌握 S3 静态网站托管的配置方法，理解 OAC（Origin Access Control）如何防止绕过 CDN 直接访问
- 能够独立搭建 S3 + CloudFront 的静态网站架构
- 理解浏览器 CORS 预检（OPTIONS 请求）的工作机制
- 掌握 CORS 规则中 AllowedOrigins、AllowedMethods、AllowedHeaders、ExposeHeaders 的含义
- 能够诊断和修复"No 'Access-Control-Allow-Origin' header"类报错

**实验流程：**
1. 创建网站存储桶，上传 HTML 页面和一份 JSON 数据文件
2. 创建 CloudFront 分发并配置 OAC，验证网站可访问
3. 验证默认无 CORS 配置时浏览器直连 S3 被拒绝
4. 配置 CORS 规则，验证 Preflight、GET、PUT 直传全部放行
5. 验证不匹配 Origin 仍被拒绝

**预计 AI 执行时长：** 50 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x，curl
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

### 1. 创建网站存储桶并上传内容

CloudFront + OAC 架构下，网站主体内容（`index.html`/`error.html`）**不需要**开启 Public Access，访问权限通过 Bucket Policy 授予 CloudFront 服务主体。直接用 S3 静态网站端点则需要公开，OAC 是更安全的现代做法。本实验第 4 步会额外为 `api/*` 前缀单独开一个窄范围的公开直读策略，用于后续 CORS 演示，网站主体本身不受影响。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"
aws s3api create-bucket \
  --bucket "${BUCKET_NAME}" \
  --region us-east-1
echo "Bucket created: ${BUCKET_NAME}"
```

**预期输出**：`create-bucket` 返回 `Location`/`BucketArn` 的 JSON，随后一行 `Bucket created: s3-workshop-website-<ACCOUNT_ID>`

上传网站页面和一份用于后续 CORS 演示的 JSON 数据（模拟前端跨源读取的 API 数据）：
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

echo '{"status": "ok", "data": [1, 2, 3]}' > /tmp/api-data.json

aws s3 cp /tmp/index.html "s3://${BUCKET_NAME}/index.html" --content-type "text/html"
aws s3 cp /tmp/error.html "s3://${BUCKET_NAME}/error.html" --content-type "text/html"
aws s3 cp /tmp/api-data.json "s3://${BUCKET_NAME}/api/data.json" --content-type "application/json"
```

**预期输出**：三条 `upload:` 消息

### 2. 创建 CloudFront OAC

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

### 3. 创建 CloudFront 分发

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
      \"Quantity\": 2,
      \"Items\": [
        {
          \"ErrorCode\": 404,
          \"ResponsePagePath\": \"/error.html\",
          \"ResponseCode\": \"404\",
          \"ErrorCachingMinTTL\": 10
        },
        {
          \"ErrorCode\": 403,
          \"ResponsePagePath\": \"/error.html\",
          \"ResponseCode\": \"404\",
          \"ErrorCachingMinTTL\": 10
        }
      ]
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
> ⚠️ `CustomErrorResponses` 必须同时映射 404 **和** 403：使用 OAC 时，CloudFront 调用者没有 `s3:ListBucket` 权限，S3 对不存在的对象会返回 `403 AccessDenied` 而不是 `404 NoSuchKey`（避免暴露桶内对象列表）。只配置 404 的话，访问不存在的路径会直接看到 S3 返回的原始 XML 错误，自定义 404 页面永远不会命中。

### 4. 为 S3 配置 Bucket Policy 授权 CloudFront，并放开 `api/*` 的公开直读

只有添加了 Bucket Policy，CloudFront 才能访问私有存储桶中的对象。`AWS:SourceArn` 条件将权限锁定到特定分发，防止其他 CloudFront 分发也能读取此桶。

这里同一个桶还要承担后续 CORS 演示的角色：`api/data.json` 需要允许浏览器绕开 CDN 直连 S3 区域端点读取。桶策略里额外加一条 `PublicReadApiData` 语句，只对 `api/*` 前缀开放匿名 `s3:GetObject`，网站主体（`index.html` 等）仍然只能通过 CloudFront+OAC 访问。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"
DISTRIBUTION_ID=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Comment=='S3 Workshop Static Website'].Id" \
  --output text)

# S3 新建桶默认开启 Block Public Access 全部四项，必须先放开 BlockPublicPolicy /
# RestrictPublicBuckets 两项，桶策略中的公开语句才允许写入生效；ACL 相关两项保持开启。
aws s3api put-public-access-block \
  --bucket "${BUCKET_NAME}" \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=false,RestrictPublicBuckets=false

aws s3api put-bucket-policy \
  --bucket "${BUCKET_NAME}" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
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
      },
      {
        \"Sid\": \"PublicReadApiData\",
        \"Effect\": \"Allow\",
        \"Principal\": \"*\",
        \"Action\": \"s3:GetObject\",
        \"Resource\": \"arn:aws:s3:::${BUCKET_NAME}/api/*\"
      }
    ]
  }"
echo "Bucket policy updated for CloudFront distribution: ${DISTRIBUTION_ID}"
```

**预期输出**：`Bucket policy updated for CloudFront distribution: <distribution-id>`

> ⚠️ 本实验把 CORS 演示合并进了 CloudFront 网站同一个桶。如果跳过 `put-public-access-block` 和 `PublicReadApiData` 语句，第 9 步的正式 GET 请求会返回 `403 Forbidden`（而不是带 CORS 头的 JSON）——因为 CORS 预检（OPTIONS）本身不需要 `s3:GetObject` 权限，S3 只按 CORS 规则放行 OPTIONS，但真正的 GET 仍然要走桶策略鉴权，而桶策略此时只信任 CloudFront 服务主体。生产环境中应像这里一样，只对明确要公开跨源读取的前缀（如 `api/*`）开权限，网站资源本身仍然锁定给 CloudFront。

### 5. 等待分发就绪并验证访问

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

```bash
CF_DOMAIN=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Comment=='S3 Workshop Static Website'].DomainName" \
  --output text)
echo "Website URL: https://${CF_DOMAIN}"
curl -s "https://${CF_DOMAIN}" | grep -o "<h1>.*</h1>"
```

**预期输出**：`<h1>欢迎来到 S3 Workshop</h1>`

验证自定义 404 页面（访问不存在的路径）：
```bash
CF_DOMAIN=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Comment=='S3 Workshop Static Website'].DomainName" \
  --output text)
curl -s -o /dev/null -w "%{http_code}\n" "https://${CF_DOMAIN}/nonexistent-page"
curl -s "https://${CF_DOMAIN}/nonexistent-page" | grep -o "<h1>.*</h1>"
```

**预期输出**：`404`，接着 `<h1>404</h1>`

### 6. 验证无 CORS 配置时浏览器直连 S3 被拒绝

网站已经能通过 CloudFront 访问，但前端 JS 若需要绕开 CDN 直接跨源读取 `api/data.json`（例如预签名 URL 直传场景需要直连 S3 区域端点），此时浏览器会先发 OPTIONS 预检。用 curl 模拟：没有 CORS 配置时，S3 不返回 `Access-Control-Allow-Origin` 头，浏览器会报 CORS 错误并阻止请求。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"

curl -s -D - -o /dev/null \
  -X OPTIONS \
  -H "Origin: https://myapp.example.com" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: Content-Type" \
  "https://${BUCKET_NAME}.s3.us-east-1.amazonaws.com/api/data.json" \
  | grep -E "HTTP|Access-Control"
```

**预期输出**：只有 `HTTP/1.1 403 Forbidden`，无任何 `Access-Control-*` 响应头，浏览器会报错

### 7. 配置 CORS 规则（生产推荐：精确 Origin）

生产环境应将 `AllowedOrigins` 设为具体域名，不要用 `*`（通配符会导致浏览器不发送 Cookie 和认证头）。`ExposeHeaders` 中的字段才能被前端 JavaScript 通过 `response.headers.get()` 读取。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"

aws s3api put-bucket-cors \
  --bucket "${BUCKET_NAME}" \
  --cors-configuration '{
    "CORSRules": [
      {
        "ID": "allow-webapp",
        "AllowedOrigins": [
          "https://myapp.example.com",
          "http://localhost:3000",
          "http://localhost:5173"
        ],
        "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
        "AllowedHeaders": ["*"],
        "ExposeHeaders": [
          "ETag",
          "x-amz-server-side-encryption",
          "x-amz-request-id",
          "x-amz-id-2"
        ],
        "MaxAgeSeconds": 3600
      }
    ]
  }'
echo "CORS rules configured"
```

**预期输出**：（无输出表示成功）

验证规则已写入：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"
aws s3api get-bucket-cors \
  --bucket "${BUCKET_NAME}" \
  --query 'CORSRules[0].{Origins:AllowedOrigins, Methods:AllowedMethods, MaxAge:MaxAgeSeconds}'
```

**预期输出**：
```json
{
    "Origins": ["https://myapp.example.com", "http://localhost:3000", "http://localhost:5173"],
    "Methods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
    "MaxAge": 3600
}
```

### 8. 验证 Preflight 请求成功

再次发送 OPTIONS 请求，这次 Origin 匹配规则，S3 应返回 `Access-Control-Allow-Origin` 等头部，浏览器放行正式请求。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"

curl -s -D - -o /dev/null \
  -X OPTIONS \
  -H "Origin: https://myapp.example.com" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: Content-Type" \
  "https://${BUCKET_NAME}.s3.us-east-1.amazonaws.com/api/data.json" \
  | grep -E "HTTP|Access-Control|Vary"
```

**预期输出**：
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://myapp.example.com
Access-Control-Allow-Methods: GET, PUT, POST, DELETE, HEAD
Access-Control-Allow-Headers: content-type
Access-Control-Expose-Headers: ETag, x-amz-server-side-encryption, x-amz-request-id, x-amz-id-2
Access-Control-Max-Age: 3600
Access-Control-Allow-Credentials: true
Vary: Origin, Access-Control-Request-Headers, Access-Control-Request-Method
```

> ⚠️ `Access-Control-Allow-Headers` 响应头返回的是请求中实际包含的 Header 名称（小写），而非规则中配置的通配符 `*`。这是 S3 的标准 CORS 行为：S3 根据请求的 `Access-Control-Request-Headers` 匹配规则，返回实际被允许的 Header 列表，而非原始配置值。
> ⚠️ 实测中 S3 在 Preflight 响应里还会带上 `Access-Control-Expose-Headers` 和 `Access-Control-Allow-Credentials: true`（规则未显式配置凭证相关字段也会返回，因为 `AllowedOrigins` 用了精确域名而非 `*`）。这两项不影响预检放行判断，可忽略。

### 9. 验证正式 GET 请求携带 CORS 头

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"

curl -s -D - \
  -H "Origin: https://myapp.example.com" \
  "https://${BUCKET_NAME}.s3.us-east-1.amazonaws.com/api/data.json" \
  | grep -E "Access-Control|HTTP|^{"
```

**预期输出**：
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://myapp.example.com
Access-Control-Allow-Methods: GET, PUT, POST, DELETE, HEAD
Access-Control-Expose-Headers: ETag, x-amz-server-side-encryption, x-amz-request-id, x-amz-id-2
Access-Control-Max-Age: 3600
Access-Control-Allow-Credentials: true
Vary: Origin, Access-Control-Request-Headers, Access-Control-Request-Method

{"status": "ok", "data": [1, 2, 3]}
```

> ⚠️ 这一步能拿到 200 而不是 403，前提是第 4 步已经给 `api/*` 加了公开直读策略——否则 CORS 头会正常出现在预检里，但真正的 GET 仍会被桶策略拒绝（403 AccessDenied），是本实验合并两个 Demo 后最容易踩的坑。

### 10. 验证不匹配 Origin 被拒绝

来自 `https://attacker.com` 的请求 Origin 不在允许列表中，S3 不返回 CORS 头，浏览器拦截。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"

curl -s -D - -o /dev/null \
  -X OPTIONS \
  -H "Origin: https://attacker.com" \
  -H "Access-Control-Request-Method: GET" \
  "https://${BUCKET_NAME}.s3.us-east-1.amazonaws.com/api/data.json" \
  | grep -E "HTTP|Access-Control"
```

**预期输出**：`HTTP/1.1 403 Forbidden`，无 `Access-Control-Allow-Origin` 头

### 11. 模拟前端 PUT 直传预检（上传场景）

前端使用预签名 URL 上传文件时，浏览器先发 OPTIONS 预检确认 PUT 是否被允许。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"

curl -s -D - -o /dev/null \
  -X OPTIONS \
  -H "Origin: http://localhost:3000" \
  -H "Access-Control-Request-Method: PUT" \
  -H "Access-Control-Request-Headers: Content-Type,x-amz-acl" \
  "https://${BUCKET_NAME}.s3.us-east-1.amazonaws.com/uploads/user-file.jpg" \
  | grep -E "HTTP|Access-Control"
```

**预期输出**：
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET, PUT, POST, DELETE, HEAD
Access-Control-Allow-Headers: content-type, x-amz-acl
Access-Control-Expose-Headers: ETag, x-amz-server-side-encryption, x-amz-request-id, x-amz-id-2
Access-Control-Max-Age: 3600
Access-Control-Allow-Credentials: true
Vary: Origin, Access-Control-Request-Headers, Access-Control-Request-Method
```

> ⚠️ 该 `uploads/user-file.jpg` 对象并不存在，也不在公开的 `api/*` 前缀内——但 OPTIONS 预检不需要 `s3:GetObject` 权限，S3 仅按 CORS 规则判定是否放行，所以预检照常返回 200。这一步只演示"预检通过"，并不代表匿名 PUT 真的会成功；真实上传场景仍需搭配预签名 URL 提供的临时授权。

---

## 常见 CORS 报错与原因

| 报错信息 | 根本原因 | 修复方法 |
|---------|---------|---------|
| `No 'Access-Control-Allow-Origin' header` | Origin 不在 AllowedOrigins 列表 | 添加正确的 Origin 到规则 |
| `Method PUT is not allowed` | AllowedMethods 没有 PUT | 在规则中添加 PUT |
| `Header 'x-amz-acl' is not allowed` | AllowedHeaders 没覆盖该 Header | 设为 `["*"]` 或显式列出 |
| `CORS header missing on GET` | 请求没有携带 Origin 头 | 检查前端代码是否设置 mode: 'cors' |
| Cookie/Authorization 不发送 | AllowedOrigins 使用了 `*` | 改为具体域名，`*` 不支持携带凭证 |

---

## 验收标准

完成本实验后，你应当能够：
- [ ] S3 存储桶中包含 `index.html`、`error.html`、`api/data.json`
- [ ] CloudFront 分发状态为 `Deployed`，`DefaultRootObject` 为 `index.html`
- [ ] 通过 `https://<uuid>.cloudfront.net` 访问返回 H1 标题，访问不存在路径返回自定义 404 页面
- [ ] `https://myapp.example.com` 直连 S3 区域端点的 OPTIONS 请求返回 `Access-Control-Allow-Origin`
- [ ] `https://attacker.com` 的 OPTIONS 请求返回 403，无 CORS 头
- [ ] PUT 的 Preflight 请求从 `http://localhost:3000` 发出时返回 200

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3 ls "s3://s3-workshop-website-$(aws sts get-caller-identity --query Account --output text)/" --recursive --summarize \| grep "Total Objects"` | `Total Objects: 3` |
| 2 | `aws cloudfront list-distributions --query "DistributionList.Items[?Comment=='S3 Workshop Static Website'].Status" --output text` | `Deployed` |
| 3 | `aws s3api get-bucket-cors --bucket "s3-workshop-website-$(aws sts get-caller-identity --query Account --output text)" --query 'CORSRules[0].MaxAgeSeconds' --output text` | `3600` |
| 4 | `curl -s -o /dev/null -w "%{http_code}" -X OPTIONS -H "Origin: https://myapp.example.com" -H "Access-Control-Request-Method: GET" "https://s3-workshop-website-$(aws sts get-caller-identity --query Account --output text).s3.us-east-1.amazonaws.com/api/data.json"` | `200` |
| 5 | `curl -s -o /dev/null -w "%{http_code}" -X OPTIONS -H "Origin: https://attacker.com" -H "Access-Control-Request-Method: GET" "https://s3-workshop-website-$(aws sts get-caller-identity --query Account --output text).s3.us-east-1.amazonaws.com/api/data.json"` | `403` |

---

## 实验总结

本实验构建了生产级静态网站架构，并补齐了它绕不开的 CORS 场景：S3 存储内容、CloudFront 负责全球分发和 HTTPS 终结、OAC 确保 S3 不对公网暴露，这条链路是"同源"的，浏览器不会触发 CORS 检查。但只要前端 JS 需要绕过 CDN 直连 S3 区域端点（预签名 URL 直传、跨源读取数据），浏览器就会先发 OPTIONS 预检，S3 必须按 CORS 规则返回 `Access-Control-*` 头才会放行。生产环境用精确 Origin（不用 `*`）；`AllowedHeaders: ["*"]` 方便但建议收紧；`ExposeHeaders` 决定前端能读哪些响应头（ETag 是断点续传必需的）。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-website-${ACCOUNT_ID}"

# 禁用并删除 CloudFront 分发（必须先禁用，再等分发把"禁用"这次变更部署完才能删除）
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
  echo "Distribution disable requested, waiting for it to finish deploying (usually 2-5 min)..."

  # 轮询直到 Status=Deployed 且 Enabled=False，再执行真正的删除
  for i in $(seq 1 24); do
    STATUS=$(aws cloudfront get-distribution --id "${DISTRIBUTION_ID}" --query 'Distribution.Status' --output text)
    ENABLED=$(aws cloudfront get-distribution --id "${DISTRIBUTION_ID}" --query 'Distribution.DistributionConfig.Enabled' --output text)
    echo "  [$i] Status=${STATUS} Enabled=${ENABLED}"
    [ "${STATUS}" = "Deployed" ] && [ "${ENABLED}" = "False" ] && break
    sleep 30
  done

  FINAL_ETAG=$(aws cloudfront get-distribution --id "${DISTRIBUTION_ID}" --query ETag --output text)
  aws cloudfront delete-distribution --id "${DISTRIBUTION_ID}" --if-match "${FINAL_ETAG}"
  echo "Distribution deleted: ${DISTRIBUTION_ID}"
fi

# 删除 OAC
OAC_ID=$(aws cloudfront list-origin-access-controls \
  --query "OriginAccessControlList.Items[?Name=='s3-workshop-oac'].Id" --output text)
OAC_ETAG=$(aws cloudfront get-origin-access-control \
  --id "${OAC_ID}" --query 'ETag' --output text 2>/dev/null)
[ -n "${OAC_ID}" ] && aws cloudfront delete-origin-access-control \
  --id "${OAC_ID}" --if-match "${OAC_ETAG}" 2>/dev/null

# 删除 CORS 配置和存储桶
aws s3api delete-bucket-cors --bucket "${BUCKET_NAME}" 2>/dev/null
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Bucket deleted: ${BUCKET_NAME}"
```

> ⚠️ "禁用"分发本身也是一次分发配置变更，同样要等 `Status` 从 `InProgress` 变回 `Deployed` 才能删除（实测约 2-5 分钟），不要在禁用后立刻调用 `delete-distribution`，会报 `DistributionNotDisabled` / `PreconditionFailed`。上面的脚本已经内置轮询，全程约 3-8 分钟跑完。
