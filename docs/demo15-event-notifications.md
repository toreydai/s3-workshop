# Demo15 — S3 事件通知与 Lambda 集成

## 实验简介

S3 事件通知是构建事件驱动架构的关键能力，允许存储桶在对象创建、删除等操作发生时自动触发下游处理。本实验创建一个完整的 S3 → Lambda 处理管道：每当有文件上传到存储桶，Lambda 函数自动被调用并记录文件元信息。

**实验目标：**
- 掌握 S3 事件通知的配置方式（触发器类型、前缀/后缀过滤）
- 理解 Lambda 资源策略与 S3 触发器的授权关系
- 能够独立搭建 S3 → Lambda 的事件驱动处理链路

**实验流程：**
1. 创建 Lambda 执行角色与函数
2. 授予 S3 调用 Lambda 的权限
3. 配置 S3 存储桶事件通知
4. 上传文件触发 Lambda，验证 CloudWatch Logs

**预计时长：** 25 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x，Python3（本地打包 Lambda）
- **权限**：`AmazonS3FullAccess`、`AWSLambdaFullAccess`、`IAMFullAccess`（创建角色）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-events-${ACCOUNT_ID}"
export FUNCTION_NAME="s3-event-processor"
export ROLE_NAME="s3-workshop-lambda-role"
echo "Account: <ACCOUNT_ID> | Bucket: ${BUCKET_NAME} | Function: ${FUNCTION_NAME}"
```

---

## 步骤

### 1. 创建 Lambda 执行角色

Lambda 需要一个 IAM 角色才能被服务调用和写入 CloudWatch Logs。角色包含两部分：信任策略（允许 Lambda 服务担任此角色）和权限策略（允许写日志）。

```bash
ROLE_NAME="s3-workshop-lambda-role"
aws iam create-role \
  --role-name "${ROLE_NAME}" \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }' \
  --query 'Role.Arn' \
  --output text
```

**预期输出**：`arn:aws:iam::<ACCOUNT_ID>:role/s3-workshop-lambda-role`

附加日志权限：
```bash
ROLE_NAME="s3-workshop-lambda-role"
aws iam attach-role-policy \
  --role-name "${ROLE_NAME}" \
  --policy-arn "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
```

**预期输出**：（无输出表示成功）

> ⚠️ 角色创建后需等待约 10 秒才能在 Lambda 中使用，否则报 `InvalidParameterValueException: The role defined for the function cannot be assumed by Lambda`。

### 2. 打包并创建 Lambda 函数

Lambda 代码记录 S3 事件中的存储桶名称、对象 Key 和大小，并将结果写入 CloudWatch Logs。Lambda 部署包要求 zip 格式，函数入口为 `handler` 文件中的 `lambda_handler` 方法。

```bash
# 创建 Lambda 函数代码
mkdir -p /tmp/lambda-s3-event
cat > /tmp/lambda-s3-event/handler.py << 'EOF'
import json
import urllib.parse

def lambda_handler(event, context):
    print("S3 Event received:", json.dumps(event))
    for record in event.get('Records', []):
        bucket = record['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(record['s3']['object']['key'])
        size = record['s3']['object'].get('size', 0)
        event_name = record['eventName']
        print(f"Event: {event_name} | Bucket: {bucket} | Key: {key} | Size: {size} bytes")
    return {"statusCode": 200, "body": "Processed"}
EOF

# 打包为 zip
cd /tmp/lambda-s3-event && zip -q function.zip handler.py && echo "Zip created: $(ls -lh function.zip | awk '{print $5}')"
```

**预期输出**：`Zip created: <size>K`

```bash
# 等待角色传播
sleep 10

# 创建 Lambda 函数
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/s3-workshop-lambda-role"
aws lambda create-function \
  --function-name "s3-event-processor" \
  --runtime python3.12 \
  --role "${ROLE_ARN}" \
  --handler handler.lambda_handler \
  --zip-file fileb:///tmp/lambda-s3-event/function.zip \
  --timeout 30 \
  --region us-east-1 \
  --query 'FunctionArn' \
  --output text
```

**预期输出**：`arn:aws:lambda:us-east-1:<ACCOUNT_ID>:function:s3-event-processor`

等待函数激活：
```bash
aws lambda get-function \
  --function-name "s3-event-processor" \
  --region us-east-1 \
  --query 'Configuration.State' \
  --output text
```

**预期输出**：`Active`（若返回 `Pending`，等几秒后重试）

### 3. 授予 S3 调用 Lambda 的权限

S3 调用 Lambda 不通过 IAM 角色，而通过 Lambda 资源策略（`add-permission`）。缺少这个授权，事件通知会静默失败——存储桶不会报错，但 Lambda 也不会被调用。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-events-${ACCOUNT_ID}"
aws lambda add-permission \
  --function-name "s3-event-processor" \
  --statement-id "s3-invoke-permission" \
  --action "lambda:InvokeFunction" \
  --principal "s3.amazonaws.com" \
  --source-arn "arn:aws:s3:::${BUCKET_NAME}" \
  --source-account "${ACCOUNT_ID}" \
  --region us-east-1 \
  --query 'Statement' \
  --output text
```

**预期输出**：包含 `"Effect":"Allow"` 和 `"Service":"s3.amazonaws.com"` 的策略 JSON 字符串

### 4. 创建 S3 存储桶并配置事件通知

事件通知配置指定：触发哪些事件（`s3:ObjectCreated:*`）、过滤哪些对象（此处过滤后缀 `.jpg`）、以及发送通知到哪个目标（Lambda ARN）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-events-${ACCOUNT_ID}"

# 创建存储桶
aws s3api create-bucket \
  --bucket "${BUCKET_NAME}" \
  --region us-east-1

echo "Bucket created: ${BUCKET_NAME}"
```

**预期输出**：`Bucket created: s3-workshop-events-<ACCOUNT_ID>`

配置事件通知：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-events-${ACCOUNT_ID}"
FUNCTION_ARN="arn:aws:lambda:us-east-1:${ACCOUNT_ID}:function:s3-event-processor"

aws s3api put-bucket-notification-configuration \
  --bucket "${BUCKET_NAME}" \
  --notification-configuration "{
    \"LambdaFunctionConfigurations\": [{
      \"LambdaFunctionArn\": \"${FUNCTION_ARN}\",
      \"Events\": [\"s3:ObjectCreated:*\"],
      \"Filter\": {
        \"Key\": {
          \"FilterRules\": [
            {\"Name\": \"suffix\", \"Value\": \".jpg\"},
            {\"Name\": \"prefix\", \"Value\": \"uploads/\"}
          ]
        }
      }
    }]
  }"
```

**预期输出**：（无输出表示成功）

验证通知配置：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-events-${ACCOUNT_ID}"
aws s3api get-bucket-notification-configuration \
  --bucket "${BUCKET_NAME}" \
  --query 'LambdaFunctionConfigurations[0].Events'
```

**预期输出**：`["s3:ObjectCreated:*"]`

### 5. 触发事件并验证 Lambda 执行

上传一个 `.jpg` 文件到 `uploads/` 前缀路径，触发 Lambda。Lambda 执行后日志会出现在 CloudWatch Logs，日志组名称格式为 `/aws/lambda/<function-name>`。

```bash
# 创建测试图片文件
echo "fake-image-content" > /tmp/test-photo.jpg

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-events-${ACCOUNT_ID}"

aws s3 cp /tmp/test-photo.jpg "s3://${BUCKET_NAME}/uploads/test-photo.jpg"
```

**预期输出**：`upload: /tmp/test-photo.jpg to s3://s3-workshop-events-<ACCOUNT_ID>/uploads/test-photo.jpg`

等待 5 秒后查看 Lambda 日志：
```bash
sleep 5
aws logs filter-log-events \
  --log-group-name "/aws/lambda/s3-event-processor" \
  --filter-pattern '"Key:"' \
  --region us-east-1 \
  --query 'events[-1].message' \
  --output text
```

**预期输出**：包含 `Key: uploads/test-photo.jpg` 的日志行

> ⚠️ `--filter-pattern` 直接传 `"Key:"`（不带内层引号）会报 `InvalidParameterException: Invalid character(s) in term ':'`——CloudWatch Logs 过滤模式语法把裸露的 `:` 当作特殊语法字符解析，必须用内层双引号把词语包起来当作字面量匹配，即 shell 里写成 `--filter-pattern '"Key:"'`。
>
> ⚠️ 上传不匹配前缀/后缀的文件（如 `s3://${BUCKET_NAME}/other.txt`）不会触发 Lambda，这是预期行为，可以用来验证过滤规则生效。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Lambda 函数 `s3-event-processor` 状态为 `Active`
- [ ] S3 存储桶 `s3-workshop-events-<ACCOUNT_ID>` 已配置事件通知，目标为 Lambda ARN
- [ ] 上传 `uploads/test-photo.jpg` 后，CloudWatch Logs 中出现包含该文件 Key 的日志
- [ ] 上传不符合过滤规则的文件不会触发额外 Lambda 调用

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws lambda get-function --function-name s3-event-processor --region us-east-1 --query 'Configuration.State' --output text` | `Active` |
| 2 | `aws s3api get-bucket-notification-configuration --bucket "s3-workshop-events-$(aws sts get-caller-identity --query Account --output text)" --query 'LambdaFunctionConfigurations[0].Events[0]' --output text` | `s3:ObjectCreated:*` |
| 3 | `aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/s3-event-processor" --region us-east-1 --query 'logGroups[0].logGroupName' --output text` | `/aws/lambda/s3-event-processor` |
| 4 | `aws lambda get-policy --function-name s3-event-processor --region us-east-1 --query 'Policy' --output text \| python3 -c "import json,sys; p=json.loads(sys.stdin.read()); print(p['Statement'][0]['Effect'])"` | `Allow` |

---

## 实验总结

本实验搭建了完整的 S3 → Lambda 事件驱动处理链路，覆盖了角色授权、资源策略、事件过滤规则的完整配置流程。核心概念包括：Lambda 资源策略 vs IAM 角色的区别（前者控制"谁能调用"，后者控制"Lambda 能做什么"），以及 S3 事件过滤的前缀/后缀机制。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-events-${ACCOUNT_ID}"

# 清空并删除存储桶
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1

# 删除 Lambda 函数
aws lambda delete-function --function-name "s3-event-processor" --region us-east-1

# 删除 IAM 角色
aws iam detach-role-policy \
  --role-name "s3-workshop-lambda-role" \
  --policy-arn "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
aws iam delete-role --role-name "s3-workshop-lambda-role"

# 删除 CloudWatch 日志组
aws logs delete-log-group \
  --log-group-name "/aws/lambda/s3-event-processor" \
  --region us-east-1

echo "Cleanup complete"
```
