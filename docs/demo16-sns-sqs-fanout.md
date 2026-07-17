# Demo16 — S3 事件通知扇出：SNS + SQS + Lambda 解耦架构

## 实验简介

Demo14 实现了 S3 → Lambda 的直连架构，但当需要多个系统同时响应同一 S3 事件时，直连会造成耦合——新增消费者就要修改 S3 通知配置。扇出（Fan-out）架构解决这个问题：S3 → SNS（广播），SNS 再分发给多个 SQS 队列和 Lambda，各消费者独立扩展、独立故障，互不影响。

**实验目标：**
- 掌握 S3 → SNS → [SQS + Lambda] 扇出架构的配置流程
- 理解 SNS Topic Policy 允许 S3 发布消息的授权机制
- 能够验证单次 S3 事件触发多个下游消费者

**实验流程：**
1. 创建 SNS Topic 和两个 SQS 队列
2. 配置 SNS Topic 订阅（SQS 和 Lambda）
3. 配置 S3 事件通知发布到 SNS
4. 上传对象验证扇出

**预计 AI 执行时长：** 35 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x，Python3
- **权限**：`AmazonS3FullAccess`、`AmazonSNSFullAccess`、`AmazonSQSFullAccess`、`AWSLambdaFullAccess`、`IAMFullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-fanout-${ACCOUNT_ID}"
export TOPIC_NAME="s3-events-fanout-topic"
echo "Bucket: ${BUCKET_NAME} | Topic: ${TOPIC_NAME}"
```

---

## 步骤

### 1. 创建存储桶

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-fanout-${ACCOUNT_ID}"
aws s3api create-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Bucket created: ${BUCKET_NAME}"
```

**预期输出**：`Bucket created: s3-workshop-fanout-<ACCOUNT_ID>`

### 2. 创建 SNS Topic

SNS Topic 是事件广播的中心节点，S3 向它发布消息，所有订阅者（SQS、Lambda、Email 等）都会收到副本。

```bash
TOPIC_ARN=$(aws sns create-topic \
  --name "s3-events-fanout-topic" \
  --region us-east-1 \
  --query 'TopicArn' \
  --output text)
echo "SNS Topic ARN: ${TOPIC_ARN}"
```

**预期输出**：`SNS Topic ARN: arn:aws:sns:us-east-1:<ACCOUNT_ID>:s3-events-fanout-topic`

### 3. 配置 SNS Topic Policy 允许 S3 发布

SNS Topic 默认只允许 Topic 拥有者发布消息，S3 服务必须通过 Topic Policy 获得 `sns:Publish` 权限。`ArnLike` 条件将发布权限限定在特定 S3 存储桶，防止其他桶也能发布到此 Topic。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-fanout-${ACCOUNT_ID}"
TOPIC_ARN="arn:aws:sns:us-east-1:${ACCOUNT_ID}:s3-events-fanout-topic"

aws sns set-topic-attributes \
  --topic-arn "${TOPIC_ARN}" \
  --attribute-name Policy \
  --attribute-value "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Sid\": \"AllowS3Publish\",
      \"Effect\": \"Allow\",
      \"Principal\": {\"Service\": \"s3.amazonaws.com\"},
      \"Action\": \"sns:Publish\",
      \"Resource\": \"${TOPIC_ARN}\",
      \"Condition\": {
        \"ArnLike\": {\"aws:SourceArn\": \"arn:aws:s3:::${BUCKET_NAME}\"},
        \"StringEquals\": {\"aws:SourceAccount\": \"${ACCOUNT_ID}\"}
      }
    }]
  }" \
  --region us-east-1
echo "SNS Topic policy configured"
```

**预期输出**：`SNS Topic policy configured`

### 4. 创建两个 SQS 队列并订阅 SNS

两个 SQS 队列模拟不同的下游处理系统（如一个负责归档日志，一个负责触发数据处理）。SQS 订阅 SNS 需要 SQS 队列策略允许 SNS 发送消息。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
TOPIC_ARN="arn:aws:sns:us-east-1:${ACCOUNT_ID}:s3-events-fanout-topic"

# 创建两个队列
QUEUE1_URL=$(aws sqs create-queue \
  --queue-name "s3-events-archive-queue" \
  --region us-east-1 \
  --query 'QueueUrl' --output text)
QUEUE2_URL=$(aws sqs create-queue \
  --queue-name "s3-events-processing-queue" \
  --region us-east-1 \
  --query 'QueueUrl' --output text)

QUEUE1_ARN="arn:aws:sqs:us-east-1:${ACCOUNT_ID}:s3-events-archive-queue"
QUEUE2_ARN="arn:aws:sqs:us-east-1:${ACCOUNT_ID}:s3-events-processing-queue"

echo "Queue1: ${QUEUE1_URL}"
echo "Queue2: ${QUEUE2_URL}"

# 为两个队列设置策略允许 SNS 发送消息
for queue_arn in "${QUEUE1_ARN}" "${QUEUE2_ARN}"; do
  queue_url=$([ "${queue_arn}" = "${QUEUE1_ARN}" ] && echo "${QUEUE1_URL}" || echo "${QUEUE2_URL}")
  aws sqs set-queue-attributes \
    --queue-url "${queue_url}" \
    --attributes "{
      \"Policy\": \"{\\\"Version\\\":\\\"2012-10-17\\\",\\\"Statement\\\":[{\\\"Effect\\\":\\\"Allow\\\",\\\"Principal\\\":{\\\"Service\\\":\\\"sns.amazonaws.com\\\"},\\\"Action\\\":\\\"sqs:SendMessage\\\",\\\"Resource\\\":\\\"${queue_arn}\\\",\\\"Condition\\\":{\\\"ArnEquals\\\":{\\\"aws:SourceArn\\\":\\\"${TOPIC_ARN}\\\"}}}]}\"
    }" \
    --region us-east-1
done
echo "SQS policies configured"
```

**预期输出**：`SQS policies configured`

订阅 SNS：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
TOPIC_ARN="arn:aws:sns:us-east-1:${ACCOUNT_ID}:s3-events-fanout-topic"
QUEUE1_ARN="arn:aws:sqs:us-east-1:${ACCOUNT_ID}:s3-events-archive-queue"
QUEUE2_ARN="arn:aws:sqs:us-east-1:${ACCOUNT_ID}:s3-events-processing-queue"

SUB1=$(aws sns subscribe --topic-arn "${TOPIC_ARN}" --protocol sqs \
  --notification-endpoint "${QUEUE1_ARN}" --region us-east-1 \
  --query 'SubscriptionArn' --output text)
SUB2=$(aws sns subscribe --topic-arn "${TOPIC_ARN}" --protocol sqs \
  --notification-endpoint "${QUEUE2_ARN}" --region us-east-1 \
  --query 'SubscriptionArn' --output text)

echo "Subscription 1 (archive): ${SUB1}"
echo "Subscription 2 (processing): ${SUB2}"
```

**预期输出**：两个 Subscription ARN

### 5. 创建 Lambda 也订阅 SNS

第三个订阅者是 Lambda 函数，实时处理事件并打印日志。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
TOPIC_ARN="arn:aws:sns:us-east-1:${ACCOUNT_ID}:s3-events-fanout-topic"

mkdir -p /tmp/fanout-lambda
cat > /tmp/fanout-lambda/handler.py << 'EOF'
import json

def lambda_handler(event, context):
    for record in event.get('Records', []):
        sns_msg = json.loads(record['Sns']['Message'])
        for s3_record in sns_msg.get('Records', []):
            bucket = s3_record['s3']['bucket']['name']
            key = s3_record['s3']['object']['key']
            size = s3_record['s3']['object'].get('size', 0)
            print(f"[Lambda-Fanout] Bucket: {bucket} | Key: {key} | Size: {size}")
    return {"statusCode": 200}
EOF

cd /tmp/fanout-lambda && zip -q function.zip handler.py

ROLE_ARN=$(aws iam create-role \
  --role-name "s3-fanout-lambda-role" \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  --query 'Role.Arn' --output text)
aws iam attach-role-policy --role-name "s3-fanout-lambda-role" \
  --policy-arn "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
sleep 10

FUNC_ARN=$(aws lambda create-function \
  --function-name "s3-fanout-processor" \
  --runtime python3.12 \
  --role "${ROLE_ARN}" \
  --handler handler.lambda_handler \
  --zip-file fileb:///tmp/fanout-lambda/function.zip \
  --region us-east-1 \
  --query 'FunctionArn' --output text)

aws lambda add-permission \
  --function-name "s3-fanout-processor" \
  --statement-id "sns-invoke" \
  --action "lambda:InvokeFunction" \
  --principal "sns.amazonaws.com" \
  --source-arn "${TOPIC_ARN}" \
  --region us-east-1 > /dev/null

aws sns subscribe --topic-arn "${TOPIC_ARN}" --protocol lambda \
  --notification-endpoint "${FUNC_ARN}" --region us-east-1 \
  --query 'SubscriptionArn' --output text
echo "Lambda subscribed to SNS"
```

**预期输出**：Lambda 订阅的 SubscriptionArn

### 6. 配置 S3 事件通知 → SNS

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-fanout-${ACCOUNT_ID}"
TOPIC_ARN="arn:aws:sns:us-east-1:${ACCOUNT_ID}:s3-events-fanout-topic"

aws s3api put-bucket-notification-configuration \
  --bucket "${BUCKET_NAME}" \
  --notification-configuration "{
    \"TopicConfigurations\": [{
      \"TopicArn\": \"${TOPIC_ARN}\",
      \"Events\": [\"s3:ObjectCreated:*\"]
    }]
  }"
echo "S3 -> SNS notification configured"
```

**预期输出**：`S3 -> SNS notification configured`

### 7. 上传对象触发扇出

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-fanout-${ACCOUNT_ID}"

echo "Fan-out test file $(date)" > /tmp/fanout-test.txt
aws s3 cp /tmp/fanout-test.txt "s3://${BUCKET_NAME}/fanout-test.txt"

# 验证 SQS 队列收到消息（用轮询代替固定 sleep，SNS→SQS 投递延迟不稳定）
QUEUE1_URL=$(aws sqs get-queue-url --queue-name "s3-events-archive-queue" \
  --region us-east-1 --query 'QueueUrl' --output text)
for i in $(seq 1 10); do
  MSG=$(aws sqs receive-message --queue-url "${QUEUE1_URL}" \
    --region us-east-1 --query 'Messages[0].Body' --output text)
  [ "${MSG}" != "None" ] && [ -n "${MSG}" ] && break
  sleep 3
done
echo "Archive queue received message:"
echo "${MSG}" | python3 -c "import json,sys; m=json.loads(sys.stdin.read()); print(json.loads(m['Message'])['Records'][0]['s3']['object']['key'])"
```

**预期输出**：`fanout-test.txt`

> ⚠️ 实测 SNS → SQS 的投递延迟不稳定，固定 `sleep 5` 偶尔不够（第一次 `receive-message` 拿到空结果导致后续 `json.loads` 报 `KeyError`/`JSONDecodeError`），已改为轮询最多 30 秒（10 次 × 3 秒）等待消息到达，更稳妥。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] SNS Topic 有 3 个订阅者（2 个 SQS + 1 个 Lambda）
- [ ] 上传对象后，两个 SQS 队列均收到消息，Lambda 日志出现记录
- [ ] S3 存储桶事件通知目标为 SNS Topic ARN

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws sns list-subscriptions-by-topic --topic-arn "arn:aws:sns:us-east-1:$(aws sts get-caller-identity --query Account --output text):s3-events-fanout-topic" --region us-east-1 --query 'length(Subscriptions)' --output text` | `3` |
| 2 | `aws s3api get-bucket-notification-configuration --bucket "s3-workshop-fanout-$(aws sts get-caller-identity --query Account --output text)" --query 'length(TopicConfigurations)' --output text` | `1` |
| 3 | `aws lambda get-function --function-name s3-fanout-processor --region us-east-1 --query 'Configuration.State' --output text` | `Active` |

---

## 实验总结

扇出架构将 S3 事件的生产者（S3）和消费者（SQS、Lambda）解耦：新增消费者只需订阅 SNS，无需修改 S3 配置；某个消费者故障不影响其他消费者；SQS 提供缓冲，消费者可以按自己的速度处理消息。这是构建弹性事件驱动数据管道的标准模式。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-fanout-${ACCOUNT_ID}"
TOPIC_ARN="arn:aws:sns:us-east-1:${ACCOUNT_ID}:s3-events-fanout-topic"

# 删除 SNS 订阅
for sub in $(aws sns list-subscriptions-by-topic --topic-arn "${TOPIC_ARN}" \
  --region us-east-1 --query 'Subscriptions[].SubscriptionArn' --output text); do
  aws sns unsubscribe --subscription-arn "${sub}" --region us-east-1 2>/dev/null
done

# 删除 SNS Topic
aws sns delete-topic --topic-arn "${TOPIC_ARN}" --region us-east-1

# 删除 SQS 队列
for q in "s3-events-archive-queue" "s3-events-processing-queue"; do
  QURL=$(aws sqs get-queue-url --queue-name "${q}" --region us-east-1 \
    --query 'QueueUrl' --output text 2>/dev/null)
  [ -n "${QURL}" ] && aws sqs delete-queue --queue-url "${QURL}" --region us-east-1
done

# 删除 Lambda
aws lambda delete-function --function-name "s3-fanout-processor" --region us-east-1 2>/dev/null
aws iam detach-role-policy --role-name "s3-fanout-lambda-role" \
  --policy-arn "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole" 2>/dev/null
aws iam delete-role --role-name "s3-fanout-lambda-role" 2>/dev/null

# 删除存储桶
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Cleanup complete"
```
