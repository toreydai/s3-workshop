# Demo13 — VPC Endpoint for S3 私网访问

## 实验简介

默认情况下，EC2 访问 S3 的流量走公网，既有数据泄露风险，又会产生 NAT 网关费用。VPC Gateway Endpoint for S3 将 S3 流量限制在 AWS 私有网络内，EC2 无需 NAT 网关或公网 IP 即可访问 S3，同时可以通过 Endpoint Policy 限定只能访问特定存储桶。

**实验目标：**
- 掌握 S3 VPC Gateway Endpoint 的创建与路由表关联
- 理解 Endpoint Policy 限制 VPC 内只能访问指定存储桶
- 能够通过 Bucket Policy 验证请求来自特定 VPC Endpoint

**实验流程：**
1. 创建 VPC 和私有子网
2. 创建 S3 Gateway Endpoint 并关联路由表
3. 配置 Endpoint Policy 和 Bucket Policy
4. 验证私网访问路径

**预计时长：** 45 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x
- **权限**：`AmazonS3FullAccess`、`AmazonVPCFullAccess`、`AmazonEC2FullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-vpc-ep-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建 VPC 和私有子网

创建一个不关联 Internet Gateway 的私有 VPC，模拟没有公网出口的隔离环境。在这个 VPC 中的 EC2 不经过 VPC Endpoint 无法访问 S3。

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "10.100.0.0/16" \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=s3-workshop-vpc}]' \
  --region us-east-1 \
  --query 'Vpc.VpcId' \
  --output text)
echo "VPC: ${VPC_ID}"

SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "${VPC_ID}" \
  --cidr-block "10.100.1.0/24" \
  --availability-zone "us-east-1a" \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=s3-workshop-private-subnet}]' \
  --region us-east-1 \
  --query 'Subnet.SubnetId' \
  --output text)
echo "Subnet: ${SUBNET_ID}"
```

**预期输出**：
```
VPC: vpc-xxxxxxxx
Subnet: subnet-xxxxxxxx
```

获取路由表：
```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=s3-workshop-vpc" \
  --region us-east-1 \
  --query 'Vpcs[0].VpcId' --output text)
RTB_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --region us-east-1 \
  --query 'RouteTables[0].RouteTableId' \
  --output text)
echo "Route Table: ${RTB_ID}"
```

**预期输出**：`Route Table: rtb-xxxxxxxx`

### 2. 创建 S3 存储桶

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-vpc-ep-${ACCOUNT_ID}"
aws s3api create-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "test content from VPC Endpoint demo" > /tmp/vpc-test.txt
aws s3 cp /tmp/vpc-test.txt "s3://${BUCKET_NAME}/vpc-test.txt"
echo "Bucket and test object created"
```

**预期输出**：`Bucket and test object created`

### 3. 创建 S3 VPC Gateway Endpoint

Gateway Endpoint 是免费的（不像 Interface Endpoint 按小时计费），路由到 S3 的流量自动走 AWS 私网。关联路由表后，VPC 内的流量访问 S3 IP 范围时会走 Endpoint 路由而非默认网关。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-vpc-ep-${ACCOUNT_ID}"
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=s3-workshop-vpc" \
  --region us-east-1 \
  --query 'Vpcs[0].VpcId' --output text)
RTB_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --region us-east-1 \
  --query 'RouteTables[0].RouteTableId' --output text)

ENDPOINT_ID=$(aws ec2 create-vpc-endpoint \
  --vpc-id "${VPC_ID}" \
  --service-name "com.amazonaws.us-east-1.s3" \
  --route-table-ids "${RTB_ID}" \
  --policy-document "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Principal\": \"*\",
      \"Action\": [\"s3:GetObject\", \"s3:PutObject\", \"s3:ListBucket\"],
      \"Resource\": [
        \"arn:aws:s3:::${BUCKET_NAME}\",
        \"arn:aws:s3:::${BUCKET_NAME}/*\"
      ]
    }]
  }" \
  --region us-east-1 \
  --query 'VpcEndpoint.VpcEndpointId' \
  --output text)
echo "VPC Endpoint created: ${ENDPOINT_ID}"
```

**预期输出**：`VPC Endpoint created: vpce-xxxxxxxxxxxxxxxx`

等待 Endpoint 就绪：
```bash
ENDPOINT_ID=$(aws ec2 describe-vpc-endpoints \
  --filters "Name=service-name,Values=com.amazonaws.us-east-1.s3" \
           "Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters 'Name=tag:Name,Values=s3-workshop-vpc' --region us-east-1 --query 'Vpcs[0].VpcId' --output text)" \
  --region us-east-1 \
  --query 'VpcEndpoints[0].VpcEndpointId' --output text)

aws ec2 describe-vpc-endpoints \
  --vpc-endpoint-ids "${ENDPOINT_ID}" \
  --region us-east-1 \
  --query 'VpcEndpoints[0].State' \
  --output text
```

**预期输出**：`available`（若返回 `pending`，等 30 秒后重试）

### 4. 配置 Bucket Policy 要求请求来自 VPC Endpoint

通过 `aws:sourceVpce` 条件限制只有来自指定 VPC Endpoint 的请求才能进行对象读写，实现网络层面的数据访问控制。

> **设计说明**：Deny 只限制对象操作（`s3:GetObject`、`s3:PutObject`、`s3:DeleteObject`、`s3:ListBucket`），**不限制管理操作**（DeleteBucketPolicy、DeleteBucket 等）。这样实验结束后 CLI 仍可正常清理资源，不会因为 Deny 策略把自己锁死。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-vpc-ep-${ACCOUNT_ID}"
ENDPOINT_ID=$(aws ec2 describe-vpc-endpoints \
  --filters "Name=service-name,Values=com.amazonaws.us-east-1.s3" \
           "Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters 'Name=tag:Name,Values=s3-workshop-vpc' --region us-east-1 --query 'Vpcs[0].VpcId' --output text)" \
  --region us-east-1 \
  --query 'VpcEndpoints[0].VpcEndpointId' --output text)

# Deny 只限制对象读写操作，管理操作不受限（防止锁死 cleanup）
aws s3api put-bucket-policy \
  --bucket "${BUCKET_NAME}" \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Sid\": \"DenyObjectAccessOutsideVPCE\",
      \"Effect\": \"Deny\",
      \"Principal\": \"*\",
      \"Action\": [
        \"s3:GetObject\",
        \"s3:PutObject\",
        \"s3:DeleteObject\",
        \"s3:ListBucket\"
      ],
      \"Resource\": [
        \"arn:aws:s3:::${BUCKET_NAME}\",
        \"arn:aws:s3:::${BUCKET_NAME}/*\"
      ],
      \"Condition\": {
        \"StringNotEquals\": {
          \"aws:sourceVpce\": \"${ENDPOINT_ID}\"
        }
      }
    }]
  }"
echo "Bucket policy configured: object access restricted to VPC Endpoint"
```

**预期输出**：`Bucket policy configured: VPC Endpoint restriction`

### 5. 验证 VPC Endpoint 和路由表关联

检查路由表中是否出现了 `pl-xxxxxxxx`（S3 Prefix List）的路由条目，该条目指向 VPC Endpoint，表明 S3 流量已被路由到私网。

```bash
RTB_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters 'Name=tag:Name,Values=s3-workshop-vpc' --region us-east-1 --query 'Vpcs[0].VpcId' --output text)" \
  --region us-east-1 \
  --query 'RouteTables[0].RouteTableId' --output text)

aws ec2 describe-route-tables \
  --route-table-ids "${RTB_ID}" \
  --region us-east-1 \
  --query 'RouteTables[0].Routes[?GatewayId!=`local`].{Dest:DestinationPrefixListId, GW:GatewayId}'
```

**预期输出**：包含 `GatewayId: vpce-xxxx` 和 `DestinationPrefixListId: pl-xxxxxxxx` 的路由条目

### 6. 从 VPC 内部 EC2 验证私网 S3 访问

路由表验证只能证明"配置存在"，无法证明"流量确实走私网"。本步骤在私有子网内启动 EC2，通过 SSM Session Manager（同样走 Interface Endpoint，无需 NAT 或公网 IP）在 EC2 上运行 `aws s3 ls`，从实例内部证明 S3 访问路径是私网。

**6a. 开启 VPC DNS 主机名，创建安全组**
```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=s3-workshop-vpc" \
  --region us-east-1 --query 'Vpcs[0].VpcId' --output text)

# 开启 DNS Hostname（Interface Endpoint 私有 DNS 解析所需）
aws ec2 modify-vpc-attribute --vpc-id "${VPC_ID}" --enable-dns-hostnames

# 创建安全组：允许 VPC 内 HTTPS（SSM Endpoint 通信）
SG_ENDPOINT_ID=$(aws ec2 create-security-group \
  --group-name "ssm-endpoint-sg" \
  --description "Allow HTTPS from VPC for SSM endpoints" \
  --vpc-id "${VPC_ID}" \
  --region us-east-1 \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress \
  --group-id "${SG_ENDPOINT_ID}" \
  --protocol tcp --port 443 --cidr "10.100.0.0/16" \
  --region us-east-1
echo "Endpoint SG: ${SG_ENDPOINT_ID}"
```

**预期输出**：`Endpoint SG: sg-xxxxxxxxxx`

**6b. 创建 3 个 SSM Interface Endpoints（无需 NAT 即可在私网管理 EC2）**
```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=s3-workshop-vpc" \
  --region us-east-1 --query 'Vpcs[0].VpcId' --output text)
SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --region us-east-1 --query 'Subnets[0].SubnetId' --output text)
SG_ENDPOINT_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=ssm-endpoint-sg" "Name=vpc-id,Values=${VPC_ID}" \
  --region us-east-1 --query 'SecurityGroups[0].GroupId' --output text)

for svc in ssm ssmmessages ec2messages; do
  aws ec2 create-vpc-endpoint \
    --vpc-id "${VPC_ID}" \
    --vpc-endpoint-type Interface \
    --service-name "com.amazonaws.us-east-1.${svc}" \
    --subnet-ids "${SUBNET_ID}" \
    --security-group-ids "${SG_ENDPOINT_ID}" \
    --private-dns-enabled \
    --region us-east-1 \
    --query 'VpcEndpoint.VpcEndpointId' --output text
  echo "Created Interface Endpoint: ${svc}"
done
```

**预期输出**：三行 `Created Interface Endpoint: <service>` 消息

**6b-2. 等待 3 个 Interface Endpoint 全部 available 且私有 DNS 生效**

根因：Interface Endpoint 创建后需要 1~3 分钟才会变为 `available` 并完成私有 DNS 传播。若在此之前就启动 EC2，SSM Agent 首次握手会失败并进入指数退避重试，导致注册延迟被放大到 20 分钟以上（而不是单纯等更久就能解决）。因此必须先确认 3 个 Endpoint 都 `available` 再启动 EC2。

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=s3-workshop-vpc" \
  --region us-east-1 --query 'Vpcs[0].VpcId' --output text)

echo "等待 3 个 SSM Interface Endpoints 变为 available..."
for i in {1..24}; do
  STATES=$(aws ec2 describe-vpc-endpoints \
    --filters "Name=vpc-id,Values=${VPC_ID}" "Name=vpc-endpoint-type,Values=Interface" \
    --region us-east-1 \
    --query 'VpcEndpoints[].State' --output text)
  READY_COUNT=$(echo "${STATES}" | tr '\t' '\n' | grep -c '^available$')
  echo "  已就绪: ${READY_COUNT}/3 (${STATES})"
  [ "${READY_COUNT}" -eq 3 ] && echo "3 个 Interface Endpoints 均已 available" && break
  sleep 15
done
```

**预期输出**：`3 个 Interface Endpoints 均已 available`（若循环结束仍未达到 3/3，检查安全组入站规则和子网选择，不要继续下一步）

**6c. 创建 EC2 IAM 实例角色**
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-vpc-ep-${ACCOUNT_ID}"

aws iam create-role \
  --role-name "s3-workshop-ec2-ssm-role" \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}' > /dev/null
aws iam attach-role-policy \
  --role-name "s3-workshop-ec2-ssm-role" \
  --policy-arn "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
aws iam put-role-policy \
  --role-name "s3-workshop-ec2-ssm-role" \
  --policy-name "s3-read" \
  --policy-document "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Action\":[\"s3:ListBucket\",\"s3:GetObject\"],\"Resource\":[\"arn:aws:s3:::${BUCKET_NAME}\",\"arn:aws:s3:::${BUCKET_NAME}/*\"]}]}"
aws iam create-instance-profile --instance-profile-name "s3-workshop-ec2-profile" > /dev/null
aws iam add-role-to-instance-profile \
  --instance-profile-name "s3-workshop-ec2-profile" \
  --role-name "s3-workshop-ec2-ssm-role"
sleep 10
echo "IAM instance profile ready"
```

**预期输出**：`IAM instance profile ready`

**6d. 启动 EC2 并等待 SSM 注册**

user-data 中加入 `systemctl restart amazon-ssm-agent` 作为兜底：AL2023 AMI 预装的 SSM Agent 有时会在网络接口就绪之前完成首次握手尝试并进入较长的退避周期，实例启动后主动重启一次 Agent 可以立即触发重新握手，避免空等退避计时器。

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=s3-workshop-vpc" \
  --region us-east-1 --query 'Vpcs[0].VpcId' --output text)
SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --region us-east-1 --query 'Subnets[0].SubnetId' --output text)
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" "Name=state,Values=available" \
  --region us-east-1 \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' --output text)

USER_DATA=$(base64 -w0 <<'EOF'
#!/bin/bash
sleep 30
systemctl restart amazon-ssm-agent
EOF
)

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "${AMI_ID}" \
  --instance-type t3.micro \
  --subnet-id "${SUBNET_ID}" \
  --iam-instance-profile Name="s3-workshop-ec2-profile" \
  --no-associate-public-ip-address \
  --user-data "${USER_DATA}" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=s3-vpc-ep-test}]' \
  --region us-east-1 \
  --query 'Instances[0].InstanceId' --output text)
echo "Launched: ${INSTANCE_ID}"

echo "等待 SSM 注册（最多 5 分钟）..."
for i in {1..30}; do
  PING=$(aws ssm describe-instance-information \
    --filters "Key=InstanceIds,Values=${INSTANCE_ID}" \
    --region us-east-1 \
    --query 'InstanceInformationList[0].PingStatus' --output text 2>/dev/null)
  [ "${PING}" = "Online" ] && echo "SSM: Online" && break
  sleep 10
done
```

**预期输出**：`SSM: Online`（若 5 分钟后仍未注册：1) 确认 6b-2 的 3/3 available 检查已通过，未跳过；2) 检查 `ssm-endpoint-sg` 是否放行了子网 CIDR 的 443 入站；3) 仍未恢复可执行 `aws ssm send-command` 前先用 `aws ec2 reboot-instances` 重启实例，重新触发 Agent 握手）

**6e. 通过 SSM 在 EC2 内运行 aws s3 ls，证明私网访问**
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-vpc-ep-${ACCOUNT_ID}"
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=s3-vpc-ep-test" "Name=instance-state-name,Values=running" \
  --region us-east-1 \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)

CMD_ID=$(aws ssm send-command \
  --instance-ids "${INSTANCE_ID}" \
  --document-name "AWS-RunShellScript" \
  --parameters "commands=[\"aws s3 ls s3://${BUCKET_NAME}/ --region us-east-1\"]" \
  --region us-east-1 \
  --query 'Command.CommandId' --output text)
echo "SSM Command ID: ${CMD_ID}"

sleep 15
aws ssm get-command-invocation \
  --command-id "${CMD_ID}" \
  --instance-id "${INSTANCE_ID}" \
  --region us-east-1 \
  --query '{Status:Status, Output:StandardOutputContent}' \
  --output json
```

**预期输出**：
```json
{
    "Status": "Success",
    "Output": "2026-07-01 00:00:00         36 vpc-test.txt\n"
}
```

> EC2 在私有子网、无公网 IP、无 NAT 网关的情况下成功列出 S3 对象——证明流量通过 VPC Gateway Endpoint 走私网路由。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] VPC `s3-workshop-vpc` 和 Gateway Endpoint 均已创建，状态为 `available`
- [ ] 路由表中出现了指向 VPC Endpoint 的 S3 Prefix List 路由
- [ ] Bucket Policy 配置了 `DenyObjectAccessOutsideVPCE` 语句（`aws:sourceVpce` 条件仅限制对象读写操作，不限制管理操作，避免锁死自己）
- [ ] 3 个 SSM Interface Endpoints 均创建并启用私有 DNS
- [ ] 私有子网内的 EC2 通过 SSM 成功执行 `aws s3 ls`，输出包含 `vpc-test.txt`

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws ec2 describe-vpc-endpoints --filters "Name=service-name,Values=com.amazonaws.us-east-1.s3" --region us-east-1 --query 'VpcEndpoints[0].State' --output text` | `available` |
| 2 | `aws ec2 describe-vpc-endpoints --filters "Name=service-name,Values=com.amazonaws.us-east-1.s3" --region us-east-1 --query 'VpcEndpoints[0].VpcEndpointType' --output text` | `Gateway` |
| 3 | `aws s3 ls "s3://s3-workshop-vpc-ep-$(aws sts get-caller-identity --query Account --output text)/" --summarize \| grep "Total Objects"` | `Total Objects: 1` |
| 4 | `aws ec2 describe-vpc-endpoints --filters "Name=service-name,Values=com.amazonaws.us-east-1.ssm" --region us-east-1 --query 'VpcEndpoints[0].State' --output text` | `available` |
| 5 | `aws ssm describe-instance-information --filters "Key=tag:Name,Values=s3-vpc-ep-test" --region us-east-1 --query 'InstanceInformationList[0].PingStatus' --output text` | `Online` |

---

## 实验总结

本实验将 S3 访问路径从公网切换到 AWS 私网：VPC Gateway Endpoint（免费）把 S3 流量路由到 AWS 骨干网；3 个 SSM Interface Endpoint 让私有子网内的 EC2 无需 NAT 也能被 Systems Manager 管理；Bucket Policy 用 `aws:sourceVpce` 条件只 Deny 来自 Endpoint 之外的对象读写操作（GetObject/PutObject/DeleteObject/ListBucket），不限制 DeleteBucketPolicy 等管理操作，避免策略把自己锁死、清理资源时又要绕路。最终通过 SSM 在 EC2 内部执行 `aws s3 ls` 的结果，做到了端到端的私网访问闭环验证。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-vpc-ep-${ACCOUNT_ID}"
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=s3-workshop-vpc" \
  --region us-east-1 --query 'Vpcs[0].VpcId' --output text 2>/dev/null)

# 终止 EC2 实例
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=s3-vpc-ep-test" "Name=instance-state-name,Values=running,stopped" \
  --region us-east-1 --query 'Reservations[0].Instances[0].InstanceId' --output text 2>/dev/null)
[ -n "${INSTANCE_ID}" ] && [ "${INSTANCE_ID}" != "None" ] && \
  aws ec2 terminate-instances --instance-ids "${INSTANCE_ID}" --region us-east-1 && \
  echo "Terminating ${INSTANCE_ID}..."

# 删除 IAM 实例角色
aws iam remove-role-from-instance-profile \
  --instance-profile-name "s3-workshop-ec2-profile" \
  --role-name "s3-workshop-ec2-ssm-role" 2>/dev/null || true
aws iam delete-instance-profile --instance-profile-name "s3-workshop-ec2-profile" 2>/dev/null || true
aws iam detach-role-policy --role-name "s3-workshop-ec2-ssm-role" \
  --policy-arn "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore" 2>/dev/null || true
aws iam delete-role-policy --role-name "s3-workshop-ec2-ssm-role" --policy-name "s3-read" 2>/dev/null || true
aws iam delete-role --role-name "s3-workshop-ec2-ssm-role" 2>/dev/null || true

# 等待 EC2 完全终止再删 VPC 资源
[ -n "${INSTANCE_ID}" ] && [ "${INSTANCE_ID}" != "None" ] && \
  aws ec2 wait instance-terminated --instance-ids "${INSTANCE_ID}" --region us-east-1

# 删除所有 VPC Endpoints（包含 Gateway S3 和 3 个 SSM Interface Endpoints）
if [ -n "${VPC_ID}" ] && [ "${VPC_ID}" != "None" ]; then
  EP_IDS=$(aws ec2 describe-vpc-endpoints \
    --filters "Name=vpc-id,Values=${VPC_ID}" \
    --region us-east-1 \
    --query 'VpcEndpoints[].VpcEndpointId' --output text 2>/dev/null)
  [ -n "${EP_IDS}" ] && aws ec2 delete-vpc-endpoints \
    --vpc-endpoint-ids ${EP_IDS} --region us-east-1
  sleep 5

  # 删除安全组
  SG_ID=$(aws ec2 describe-security-groups \
    --filters "Name=group-name,Values=ssm-endpoint-sg" "Name=vpc-id,Values=${VPC_ID}" \
    --region us-east-1 --query 'SecurityGroups[0].GroupId' --output text 2>/dev/null)
  [ -n "${SG_ID}" ] && [ "${SG_ID}" != "None" ] && \
    aws ec2 delete-security-group --group-id "${SG_ID}" --region us-east-1 2>/dev/null || true

  # 删除子网和 VPC
  SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}" \
    --region us-east-1 --query 'Subnets[0].SubnetId' --output text 2>/dev/null)
  [ -n "${SUBNET_ID}" ] && [ "${SUBNET_ID}" != "None" ] && \
    aws ec2 delete-subnet --subnet-id "${SUBNET_ID}" --region us-east-1
  aws ec2 delete-vpc --vpc-id "${VPC_ID}" --region us-east-1
fi

# 先删除桶策略（解锁，否则 IAM 角色无法操作桶）
aws s3api delete-bucket-policy --bucket "${BUCKET_NAME}" --region us-east-1

# 删除存储桶
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Cleanup complete"
```
