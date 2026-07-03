# Demo15 — S3 事件通知与 EventBridge 集成

## 实验简介

EventBridge 是 AWS 推荐的新一代 S3 事件通知目标，相比直接通知 SNS/Lambda，它支持内容级别的精细过滤（按对象大小、前缀、后缀、标签组合过滤）、跨账号事件路由、和事件归档重放。AWS 官方正逐步将事件通知推荐方向从 SNS 迁移到 EventBridge。本实验配置 S3 → EventBridge → Lambda + SQS 的完整链路。

**实验目标：**
- 掌握在存储桶上启用 EventBridge 通知的方式
- 理解 EventBridge 规则的内容过滤语法（比 S3 原生过滤更强大）
- 能够将 S3 事件路由到多个目标，并验证过滤规则生效

**实验流程：**
1. 创建存储桶并启用 EventBridge 通知
2. 创建 SQS 队列和 Lambda 作为目标
3. 配置 EventBridge 规则（带内容过滤）
4. 上传对象验证过滤和路由

**预计时长：** 30 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x，Python3
- **权限**：`AmazonS3FullAccess`、`AmazonEventBridgeFullAccess`、`AmazonSQSFullAccess`、`AWSLambdaFullAccess`、`IAMFullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-eb-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建存储桶并启用 EventBridge 通知

开启 EventBridge 通知只需一个 API 调用，之后所有 S3 事件都会发送到账号默认的 EventBridge 事件总线（`default` bus）。这与配置 SNS/SQS/Lambda 目标不同——EventBridge 通知是全量发送，过滤和路由由 EventBridge 规则负责。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-eb-${ACCOUNT_ID}"
aws s3api create-bucket --bucket "${BUCKET_NAME}" --region us-east-1

# 启用 EventBridge 通知（核心配置，只需一行）
aws s3api put-bucket-notification-configuration \
  --bucket "${BUCKET_NAME}" \
  --notification-configuration '{"EventBridgeConfiguration": {}}'
echo "EventBridge notification enabled on: ${BUCKET_NAME}"
```

**预期输出**：`EventBridge notification enabled on: s3-workshop-eb-<ACCOUNT_ID>`

验证配置：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-eb-${ACCOUNT_ID}"
aws s3api get-bucket-notification-configuration \
  --bucket "${BUCKET_NAME}"
```

**预期输出**：`{"EventBridgeConfiguration": {}}`（对象存在即表示已启用）

### 2. 创建目标 SQS 队列

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

QUEUE_URL=$(aws sqs create-queue \
  --queue-name "s3-eb-events-queue" \
  --region us-east-1 \
  --query 'QueueUrl' --output text)
QUEUE_ARN="arn:aws:sqs:us-east-1:${ACCOUNT_ID}:s3-eb-events-queue"

# 授权 EventBridge 向队列发送消息
aws sqs set-queue-attributes \
  --queue-url "${QUEUE_URL}" \
  --attributes "{
    \"Policy\": \"{\\\"Version\\\":\\\"2012-10-17\\\",\\\"Statement\\\":[{\\\"Effect\\\":\\\"Allow\\\",\\\"Principal\\\":{\\\"Service\\\":\\\"events.amazonaws.com\\\"},\\\"Action\\\":\\\"sqs:SendMessage\\\",\\\"Resource\\\":\\\"${QUEUE_ARN}\\\"}]}\"
  }" \
  --region us-east-1
echo "SQS queue ready: ${QUEUE_ARN}"
```

**预期输出**：`SQS queue ready: arn:aws:sqs:us-east-1:<ACCOUNT_ID>:s3-eb-events-queue`

### 3. 创建 Lambda 目标函数

```bash
mkdir -p /tmp/eb-lambda
cat > /tmp/eb-lambda/handler.py << 'EOF'
import json

def lambda_handler(event, context):
    detail = event.get("detail", {})
    bucket = detail.get("bucket", {}).get("name", "unknown")
    obj = detail.get("object", {})
    key = obj.get("key", "unknown")
    size = obj.get("size", 0)
    event_type = event.get("detail-type", "unknown")
    print(f"[EventBridge] {event_type} | {bucket}/{key} | {size} bytes")
    return {"statusCode": 200}
EOF

cd /tmp/eb-lambda && zip -q function.zip handler.py

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ROLE_ARN=$(aws iam create-role \
  --role-name "s3-eb-lambda-role" \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  --query 'Role.Arn' --output text)
aws iam attach-role-policy --role-name "s3-eb-lambda-role" \
  --policy-arn "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
sleep 10

FUNC_ARN=$(aws lambda create-function \
  --function-name "s3-eb-processor" \
  --runtime python3.12 \
  --role "${ROLE_ARN}" \
  --handler handler.lambda_handler \
  --zip-file fileb:///tmp/eb-lambda/function.zip \
  --region us-east-1 \
  --query 'FunctionArn' --output text)

# 授权 EventBridge 调用 Lambda
aws lambda add-permission \
  --function-name "s3-eb-processor" \
  --statement-id "eb-invoke" \
  --action "lambda:InvokeFunction" \
  --principal "events.amazonaws.com" \
  --region us-east-1 > /dev/null

echo "Lambda ready: ${FUNC_ARN}"
```

**预期输出**：`Lambda ready: arn:aws:lambda:us-east-1:<ACCOUNT_ID>:function:s3-eb-processor`

### 4. 创建 EventBridge 规则：过滤大于 1KB 的图片上传

EventBridge 的内容过滤比 S3 原生过滤强大得多：可以按对象大小范围、key 前缀/后缀、甚至组合条件过滤。S3 原生通知只支持简单的前缀/后缀过滤。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-eb-${ACCOUNT_ID}"
QUEUE_ARN="arn:aws:sqs:us-east-1:${ACCOUNT_ID}:s3-eb-events-queue"
FUNC_ARN="arn:aws:lambda:us-east-1:${ACCOUNT_ID}:function:s3-eb-processor"

# 规则1：过滤 images/ 前缀下 .jpg 文件的上传事件
RULE_ARN=$(aws events put-rule \
  --name "s3-image-upload-rule" \
  --event-pattern "{
    \"source\": [\"aws.s3\"],
    \"detail-type\": [\"Object Created\"],
    \"detail\": {
      \"bucket\": {\"name\": [\"${BUCKET_NAME}\"]},
      \"object\": {
        \"key\": [{\"suffix\": \".jpg\"}],
        \"size\": [{\"numeric\": [\">\", 1024]}]
      }
    }
  }" \
  --state ENABLED \
  --region us-east-1 \
  --query 'RuleArn' --output text)
echo "Rule created: ${RULE_ARN}"
```

**预期输出**：`Rule created: arn:aws:events:us-east-1:<ACCOUNT_ID>:rule/s3-image-upload-rule`

添加目标（SQS + Lambda 同时接收）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
QUEUE_ARN="arn:aws:sqs:us-east-1:${ACCOUNT_ID}:s3-eb-events-queue"
FUNC_ARN="arn:aws:lambda:us-east-1:${ACCOUNT_ID}:function:s3-eb-processor"

aws events put-targets \
  --rule "s3-image-upload-rule" \
  --targets "[
    {\"Id\": \"sqs-target\", \"Arn\": \"${QUEUE_ARN}\"},
    {\"Id\": \"lambda-target\", \"Arn\": \"${FUNC_ARN}\"}
  ]" \
  --region us-east-1 \
  --query 'FailedEntryCount' \
  --output text
```

**预期输出**：`0`（0 个目标添加失败）

### 5. 创建第二条规则：捕获所有删除事件（用于审计）

EventBridge 可以为同一个存储桶配置多条独立规则，各自有不同的过滤条件和目标，互不干扰。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-eb-${ACCOUNT_ID}"
QUEUE_ARN="arn:aws:sqs:us-east-1:${ACCOUNT_ID}:s3-eb-events-queue"

aws events put-rule \
  --name "s3-delete-audit-rule" \
  --event-pattern "{
    \"source\": [\"aws.s3\"],
    \"detail-type\": [\"Object Deleted\"],
    \"detail\": {
      \"bucket\": {\"name\": [\"${BUCKET_NAME}\"]}
    }
  }" \
  --state ENABLED \
  --region us-east-1 > /dev/null

aws events put-targets \
  --rule "s3-delete-audit-rule" \
  --targets "[{\"Id\": \"audit-queue\", \"Arn\": \"${QUEUE_ARN}\"}]" \
  --region us-east-1 > /dev/null
echo "Delete audit rule created"
```

**预期输出**：`Delete audit rule created`

### 6. 上传文件验证规则生效

上传一个大于 1KB 的 .jpg 文件触发规则1，再上传一个 .txt 文件验证被过滤掉。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-eb-${ACCOUNT_ID}"

# 创建 >1KB 的 jpg 文件（触发规则）
dd if=/dev/urandom of=/tmp/photo.jpg bs=1K count=5 2>/dev/null
aws s3 cp /tmp/photo.jpg "s3://${BUCKET_NAME}/images/photo.jpg"

# 上传 txt 文件（不触发规则，被过滤）
echo "text file" > /tmp/doc.txt
aws s3 cp /tmp/doc.txt "s3://${BUCKET_NAME}/docs/doc.txt"

sleep 5
```

**预期输出**：两次上传成功

检查 SQS 队列收到的消息（只应有 jpg 的事件），用长轮询 + 重试而非固定 `sleep 5` 后单次读取：
```bash
QUEUE_URL=$(aws sqs get-queue-url --queue-name "s3-eb-events-queue" \
  --region us-east-1 --query 'QueueUrl' --output text)

MSG=""
for i in 1 2 3 4 5 6; do
  MSG=$(aws sqs receive-message --queue-url "${QUEUE_URL}" \
    --region us-east-1 --wait-time-seconds 20 --query 'Messages[0].Body' --output text)
  [ "${MSG}" != "None" ] && [ -n "${MSG}" ] && break
  echo "尚未收到消息，重试 ($i/6)..."
done

echo "${MSG}" | python3 -c "
import json, sys
body = json.loads(sys.stdin.read())
detail = body.get('detail', {})
print('Event:', body.get('detail-type'))
print('Key:', detail.get('object', {}).get('key'))
print('Size:', detail.get('object', {}).get('size'), 'bytes')
"
```

**预期输出**：
```
Event: Object Created
Key: images/photo.jpg
Size: 5120 bytes
```

> ⚠️ **SQS 队列策略更新后首次投递可能静默丢失**：实测 `set-queue-attributes` 设置完 Policy 后立即上传触发的第一条事件，EventBridge 向 SQS 投递会因策略生效传播延迟而丢失（Lambda 目标同一事件正常收到，只有 SQS 目标丢失，且没有任何报错或重试记录），几分钟后再次上传的事件才能正常送达。这与 IAM 角色传播延迟同类，只是发生在 SQS 资源策略上。解决方法是用长轮询（`--wait-time-seconds`）+ 多次重试读取队列，不要用固定 `sleep 5` 后只读一次就判定失败；生产环境建议在配置队列策略与首次依赖投递之间预留至少 1-2 分钟缓冲，或对首次投递失败有告警和重试机制。
>
> ⚠️ `doc.txt` 不在队列中，因为它不匹配 `.jpg` 后缀过滤条件，这验证了 EventBridge 内容过滤生效。

---

## EventBridge vs SNS 直接通知对比

| 维度 | S3 直接通知 SNS/SQS/Lambda | S3 → EventBridge |
|------|---------------------------|-----------------|
| 过滤粒度 | 仅前缀/后缀 | 前缀/后缀 + 对象大小 + 标签 |
| 多目标 | 需要 SNS 扇出 | 规则直接配多目标 |
| 跨账号路由 | 不支持 | 支持（通过 Event Bus） |
| 事件归档重放 | 不支持 | 支持 |
| 调试工具 | 无 | EventBridge 沙盒可测试规则 |
| 延迟 | 极低（毫秒） | 低（秒级） |

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 存储桶 EventBridge 通知已启用（`EventBridgeConfiguration: {}`）
- [ ] EventBridge 规则 `s3-image-upload-rule` 已创建，过滤 `.jpg` + size > 1024
- [ ] 上传 `.jpg` 后 SQS 队列收到事件，`key` 字段为 `images/photo.jpg`
- [ ] 上传 `.txt` 不触发队列消息

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3api get-bucket-notification-configuration --bucket "s3-workshop-eb-$(aws sts get-caller-identity --query Account --output text)" --query 'keys(@)' --output text` | `EventBridgeConfiguration` |
| 2 | `aws events describe-rule --name "s3-image-upload-rule" --region us-east-1 --query State --output text` | `ENABLED` |
| 3 | `aws events list-targets-by-rule --rule "s3-image-upload-rule" --region us-east-1 --query 'length(Targets)' --output text` | `2` |
| 4 | `aws lambda get-function --function-name s3-eb-processor --region us-east-1 --query 'Configuration.State' --output text` | `Active` |

---

## 实验总结

本实验将 S3 事件路由从"硬连接"（直接指定 SNS/Lambda ARN）升级到"规则引擎"（EventBridge 内容过滤 + 多目标路由）。核心优势：一次启用，多条规则独立管理；内容过滤（size > 1KB、.jpg 后缀）在 EventBridge 侧完成，不消耗 Lambda 调用；事件归档让你能在调试时重放历史事件。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-eb-${ACCOUNT_ID}"

# 删除 EventBridge 规则和目标
for rule in "s3-image-upload-rule" "s3-delete-audit-rule"; do
  TARGETS=$(aws events list-targets-by-rule --rule "${rule}" \
    --region us-east-1 --query 'Targets[].Id' --output text 2>/dev/null)
  [ -n "${TARGETS}" ] && aws events remove-targets \
    --rule "${rule}" --ids ${TARGETS} --region us-east-1 > /dev/null 2>&1
  aws events delete-rule --name "${rule}" --region us-east-1 2>/dev/null
done

# 删除 SQS 队列
QUEUE_URL=$(aws sqs get-queue-url --queue-name "s3-eb-events-queue" \
  --region us-east-1 --query 'QueueUrl' --output text 2>/dev/null)
[ -n "${QUEUE_URL}" ] && aws sqs delete-queue --queue-url "${QUEUE_URL}" --region us-east-1

# 删除 Lambda
aws lambda delete-function --function-name "s3-eb-processor" --region us-east-1 2>/dev/null
aws iam detach-role-policy --role-name "s3-eb-lambda-role" \
  --policy-arn "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole" 2>/dev/null
aws iam delete-role --role-name "s3-eb-lambda-role" 2>/dev/null

# 删除存储桶
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Cleanup complete"
```
