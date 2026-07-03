# Demo12 — S3 CORS 配置：跨域资源共享

## 实验简介

CORS（Cross-Origin Resource Sharing）是浏览器的安全机制，默认阻止网页向不同源（协议+域名+端口）发送请求。当前端 JavaScript 直接调用 S3（上传文件、读取图片）时，S3 必须在响应头中返回正确的 CORS 头，浏览器才会放行。CORS 配置错误是 S3 最高频的踩坑场景，本实验覆盖预检请求（Preflight）、PUT 直传、GET 读取的完整验证流程。

**实验目标：**
- 理解浏览器 CORS 预检（OPTIONS 请求）的工作机制
- 掌握 S3 CORS 规则中 AllowedOrigins、AllowedMethods、AllowedHeaders、ExposeHeaders 的含义
- 能够诊断和修复"No 'Access-Control-Allow-Origin' header"类报错

**实验流程：**
1. 创建存储桶并上传测试资源
2. 验证默认无 CORS 配置时的拒绝行为
3. 配置 CORS 规则（精确 Origin 版 + 宽松版）
4. 模拟浏览器预检请求验证配置生效

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
export BUCKET_NAME="s3-workshop-cors-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建存储桶并上传测试资源

配置存储桶可公开读取（模拟静态资源桶），上传一个 JSON 文件作为 API 数据，一个图片文件作为媒体资源。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-cors-${ACCOUNT_ID}"
aws s3api create-bucket --bucket "${BUCKET_NAME}" --region us-east-1

# 关闭 Block Public Access（演示公开 CORS 场景）
aws s3api put-public-access-block \
  --bucket "${BUCKET_NAME}" \
  --public-access-block-configuration \
  'BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false'

# 设置公读 Bucket Policy
aws s3api put-bucket-policy \
  --bucket "${BUCKET_NAME}" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Principal\": \"*\",
      \"Action\": \"s3:GetObject\",
      \"Resource\": \"arn:aws:s3:::${BUCKET_NAME}/*\"
    }]
  }"

# 上传测试文件
echo '{"status": "ok", "data": [1, 2, 3]}' > /tmp/api-data.json
aws s3 cp /tmp/api-data.json "s3://${BUCKET_NAME}/api/data.json" \
  --content-type "application/json"
echo "Setup complete: ${BUCKET_NAME}"
```

**预期输出**：`Setup complete: s3-workshop-cors-<ACCOUNT_ID>`

### 2. 验证无 CORS 配置时的拒绝行为

用 curl 模拟浏览器预检请求（OPTIONS）。没有 CORS 配置时，S3 不返回 `Access-Control-Allow-Origin` 头，浏览器会报 CORS 错误并阻止请求。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-cors-${ACCOUNT_ID}"

# 模拟浏览器 Preflight（OPTIONS 请求）
curl -s -D - -o /dev/null \
  -X OPTIONS \
  -H "Origin: https://myapp.example.com" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: Content-Type" \
  "https://${BUCKET_NAME}.s3.us-east-1.amazonaws.com/api/data.json" \
  | grep -E "HTTP|Access-Control"
```

**预期输出**：只有 `HTTP/1.1 403 Forbidden`，无任何 `Access-Control-*` 响应头，浏览器会报错

### 3. 配置 CORS 规则（生产推荐：精确 Origin）

生产环境应将 `AllowedOrigins` 设为具体域名，不要用 `*`（通配符会导致浏览器不发送 Cookie 和认证头）。`ExposeHeaders` 中的字段才能被前端 JavaScript 通过 `response.headers.get()` 读取。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-cors-${ACCOUNT_ID}"

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
BUCKET_NAME="s3-workshop-cors-${ACCOUNT_ID}"
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

### 4. 验证 Preflight 请求成功

再次发送 OPTIONS 请求，这次 Origin 匹配规则，S3 应返回 `Access-Control-Allow-Origin` 等头部，浏览器放行正式请求。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-cors-${ACCOUNT_ID}"

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
Access-Control-Max-Age: 3600
Vary: Origin, Access-Control-Request-Headers, Access-Control-Request-Method
```

> ⚠️ `Access-Control-Allow-Headers` 响应头返回的是请求中实际包含的 Header 名称（小写），而非规则中配置的通配符 `*`。这是 S3 的标准 CORS 行为：S3 根据请求的 `Access-Control-Request-Headers` 匹配规则，返回实际被允许的 Header 列表，而非原始配置值。

### 5. 验证正式 GET 请求携带 CORS 头

Preflight 通过后，浏览器发出实际 GET 请求，响应中必须包含 `Access-Control-Allow-Origin`，否则浏览器仍会阻止 JavaScript 读取响应体。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-cors-${ACCOUNT_ID}"

curl -s -D - \
  -H "Origin: https://myapp.example.com" \
  "https://${BUCKET_NAME}.s3.us-east-1.amazonaws.com/api/data.json" \
  | grep -E "Access-Control|HTTP|^{"
```

**预期输出**：
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://myapp.example.com
Access-Control-Expose-Headers: ETag, x-amz-server-side-encryption, x-amz-request-id, x-amz-id-2

{"status": "ok", "data": [1, 2, 3]}
```

### 6. 验证不匹配 Origin 被拒绝

来自 `https://attacker.com` 的请求 Origin 不在允许列表中，S3 不返回 CORS 头，浏览器拦截。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-cors-${ACCOUNT_ID}"

curl -s -D - -o /dev/null \
  -X OPTIONS \
  -H "Origin: https://attacker.com" \
  -H "Access-Control-Request-Method: GET" \
  "https://${BUCKET_NAME}.s3.us-east-1.amazonaws.com/api/data.json" \
  | grep -E "HTTP|Access-Control"
```

**预期输出**：`HTTP/1.1 403 Forbidden`，无 `Access-Control-Allow-Origin` 头

### 7. 模拟前端 PUT 直传预检（上传场景）

前端使用预签名 URL 上传文件时，浏览器先发 OPTIONS 预检确认 PUT 是否被允许。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-cors-${ACCOUNT_ID}"

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
```

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
- [ ] `https://myapp.example.com` 的 OPTIONS 请求返回 `Access-Control-Allow-Origin`
- [ ] `https://attacker.com` 的 OPTIONS 请求返回 403，无 CORS 头
- [ ] GET 请求响应包含 `Access-Control-Expose-Headers: ETag`
- [ ] PUT 的 Preflight 请求从 `http://localhost:3000` 发出时返回 200

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3api get-bucket-cors --bucket "s3-workshop-cors-$(aws sts get-caller-identity --query Account --output text)" --query 'CORSRules[0].MaxAgeSeconds' --output text` | `3600` |
| 2 | `aws s3api get-bucket-cors --bucket "s3-workshop-cors-$(aws sts get-caller-identity --query Account --output text)" --query 'length(CORSRules[0].AllowedMethods)' --output text` | `5` |
| 3 | `curl -s -o /dev/null -w "%{http_code}" -X OPTIONS -H "Origin: https://myapp.example.com" -H "Access-Control-Request-Method: GET" "https://s3-workshop-cors-$(aws sts get-caller-identity --query Account --output text).s3.us-east-1.amazonaws.com/api/data.json"` | `200` |
| 4 | `curl -s -o /dev/null -w "%{http_code}" -X OPTIONS -H "Origin: https://attacker.com" -H "Access-Control-Request-Method: GET" "https://s3-workshop-cors-$(aws sts get-caller-identity --query Account --output text).s3.us-east-1.amazonaws.com/api/data.json"` | `403` |

---

## 实验总结

本实验覆盖了 S3 CORS 的核心机制：浏览器先发 OPTIONS 预检，S3 根据请求中的 `Origin` 和 `Access-Control-Request-Method` 匹配 CORS 规则，匹配成功才在响应头中放行。关键决策点：生产环境用精确 Origin（不用 `*`）；`AllowedHeaders: ["*"]` 方便但建议收紧；`ExposeHeaders` 决定前端能读哪些响应头（ETag 是断点续传必需的）。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-cors-${ACCOUNT_ID}"
aws s3api delete-bucket-cors --bucket "${BUCKET_NAME}"
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Bucket deleted: ${BUCKET_NAME}"
```
