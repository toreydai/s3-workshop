# Demo04 — S3 Express One Zone：高性能单可用区存储

## 实验简介

S3 Express One Zone 是 2023 年推出、持续扩展到更多区域的高性能存储类型：数据只存放在你指定的单个可用区（而非 Standard 默认跨 3 个 AZ 复制），换来个位数毫秒的访问延迟——比 S3 Standard 快 10 倍，请求成本低 50%。它使用一种全新的桶架构——**Directory Bucket**（目录桶），命名规则和权限模型都与普通桶不同，专为放在同一 AZ 的计算实例（EC2/EKS/ECS）做高频小对象读写而设计，典型场景是机器学习训练数据集缓存、实时数据处理临时文件。

**实验目标：**
- 理解 Directory Bucket 与普通（General Purpose）桶的架构差异：单 AZ、专属命名规则、`s3express` IAM 权限命名空间
- 掌握 Directory Bucket 的创建、上传、下载操作
- 通过实测对比 Directory Bucket 与普通桶的延迟差异
- 理解 Directory Bucket 只支持 `EXPRESS_ONEZONE` 一种存储类型，不支持生命周期规则迁移到其他存储类型

**实验流程：**
1. 查询候选可用区并创建 Directory Bucket
2. 创建一个普通桶作为对照组
3. 分别上传/下载对象，实测延迟对比
4. 验证 IAM 权限模型（`s3express:CreateSession`）
5. 验证 Directory Bucket 的存储类型限制

**预计时长：** 25 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x（建议 2.23 及以上）
- **权限**：`AmazonS3FullAccess`（额外需要 `s3express:CreateSession`，通常包含在 `AmazonS3FullAccess` 中）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account: ${ACCOUNT_ID}"
```

> ⚠️ S3 Express One Zone 只在部分区域和部分可用区开放，且哪些可用区支持会随时间扩展。Directory Bucket 命名格式固定为 `bucket-base-name--zone-id--x-s3`（用的是可用区 **Zone ID**，如 `use1-az4`，不是可用区名称如 `us-east-1a`——Zone ID 在所有 AWS 账号间是同一个物理可用区的稳定标识，Zone 名称则是账号级随机映射）。下一步用循环尝试几个候选 Zone ID，遇到 `create-bucket` 报错就换下一个。

---

## 步骤

### 1. 查询可用区并创建 Directory Bucket

```bash
aws ec2 describe-availability-zones \
  --region us-east-1 \
  --query 'AvailabilityZones[].{ZoneName:ZoneName, ZoneId:ZoneId}' \
  --output table
```

**预期输出**：一张 ZoneName/ZoneId 对照表（如 `us-east-1a` ↔ `use1-az4`）

依次尝试候选 Zone ID 创建 Directory Bucket，成功后记录下实际生效的 Zone ID：

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_BASE="s3wkshp${ACCOUNT_ID: -8}"

for ZONE_ID in use1-az4 use1-az5 use1-az6 use1-az1 use1-az2; do
  EXPRESS_BUCKET="${BUCKET_BASE}--${ZONE_ID}--x-s3"
  if aws s3api create-bucket \
      --bucket "${EXPRESS_BUCKET}" \
      --create-bucket-configuration "Location={Type=AvailabilityZone,Name=${ZONE_ID}},Bucket={DataRedundancy=SingleAvailabilityZone,Type=Directory}" \
      --region us-east-1 2>/tmp/express-create-err.txt; then
    echo "Created directory bucket: ${EXPRESS_BUCKET} (Zone: ${ZONE_ID})"
    echo "${EXPRESS_BUCKET}" > /tmp/express-bucket-name.txt
    break
  else
    echo "Zone ${ZONE_ID} not available, trying next..."
    cat /tmp/express-create-err.txt
  fi
done
```

**预期输出**：`Created directory bucket: s3wkshp<后8位账号ID>--use1-azN--x-s3 (Zone: use1-azN)`

> ⚠️ Directory Bucket 名称长度、字符集要求比普通桶更严格，建议用不含连字符歧义的短前缀（本实验用账号 ID 后 8 位保证唯一性，避免与他人 Directory Bucket 命名冲突）。

### 2. 创建普通桶作为对照组

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
STANDARD_BUCKET="s3-workshop-express-compare-${ACCOUNT_ID}"
aws s3api create-bucket --bucket "${STANDARD_BUCKET}" --region us-east-1
echo "Standard bucket created: ${STANDARD_BUCKET}"
```

**预期输出**：`Standard bucket created: s3-workshop-express-compare-<ACCOUNT_ID>`

### 3. 上传对象并实测延迟对比

Directory Bucket 只支持 `EXPRESS_ONEZONE` 存储类型，且是隐式的——上传时无需（也不能）指定其他 `--storage-class`。

```bash
EXPRESS_BUCKET=$(cat /tmp/express-bucket-name.txt)
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
STANDARD_BUCKET="s3-workshop-express-compare-${ACCOUNT_ID}"

echo '{"test": "express-one-zone-latency"}' > /tmp/latency-test.json

echo "=== Directory Bucket (Express One Zone) 10 次 PUT 耗时 ==="
time ( for i in $(seq 1 10); do
  aws s3api put-object --bucket "${EXPRESS_BUCKET}" --key "obj-${i}.json" --body /tmp/latency-test.json > /dev/null
done )

echo "=== 普通桶（Standard）10 次 PUT 耗时 ==="
time ( for i in $(seq 1 10); do
  aws s3api put-object --bucket "${STANDARD_BUCKET}" --key "obj-${i}.json" --body /tmp/latency-test.json > /dev/null
done )
```

**预期输出**：两组 `real` 耗时对比，Directory Bucket 的总耗时明显更低（具体倍数受本机到 AWS 的网络路径影响，控制台/EC2 同 AZ 内发起请求时差异最明显；从跨区域的客户端发起两组请求都会被公网延迟摊平，倍数差会缩小，但 Directory Bucket 仍应更快）

> 本实验的时延对比在任意客户端都能跑通，但 S3 Express One Zone"个位数毫秒"的官方指标是在**同 AZ 内的 EC2/EKS/ECS 实例**发起请求时测得的——如果操作机就在 `use1-azN` 对应的可用区内，效果会更明显。

### 4. 验证 IAM 权限模型

Directory Bucket 的资源 ARN 使用独立的 `s3express` 命名空间（而不是 `s3`），对应的 IAM Action 也是 `s3express:CreateSession`——这是 AWS CLI/SDK 访问 Directory Bucket 时在底层自动调用的会话建立 API，用来换取短期高性能访问凭证，日常使用 `aws s3api put-object`/`get-object` 时无需手动调用。

```bash
EXPRESS_BUCKET=$(cat /tmp/express-bucket-name.txt)
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "Directory Bucket 资源 ARN 命名空间示例："
echo "arn:aws:s3express:us-east-1:${ACCOUNT_ID}:bucket/${EXPRESS_BUCKET}"

# 显式调用 CreateSession 验证权限（SDK/CLI 高层命令会自动完成这一步）
aws s3api create-session \
  --bucket "${EXPRESS_BUCKET}" \
  --region us-east-1 \
  --query 'Credentials.SessionToken' \
  --output text | head -c 20
echo "... (会话令牌已获取，说明 s3express:CreateSession 权限生效)"
```

**预期输出**：一段会话令牌前缀 + 提示信息

### 5. 验证存储类型限制

```bash
EXPRESS_BUCKET=$(cat /tmp/express-bucket-name.txt)
aws s3api head-object \
  --bucket "${EXPRESS_BUCKET}" \
  --key "obj-1.json" \
  --query '{StorageClass: StorageClass}'
```

**预期输出**：`{"StorageClass": "EXPRESS_ONEZONE"}`（或省略字段直接不返回，因为 `EXPRESS_ONEZONE` 是 Directory Bucket 的唯一/默认存储类型）

> ⚠️ Directory Bucket 不支持生命周期规则把对象迁移到 STANDARD_IA/GLACIER 等其他存储类型——单 AZ、高性能是它唯一的定位，跨存储类型分层需要用生命周期规则先过期删除，再由应用层写入其他类型的普通桶。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 成功创建符合 `base--zone-id--x-s3` 命名规则的 Directory Bucket
- [ ] Directory Bucket 中对象的 `StorageClass` 为 `EXPRESS_ONEZONE`
- [ ] 实测到 Directory Bucket 的 PUT 耗时低于同批次普通桶
- [ ] 理解 `s3express:CreateSession` 权限模型与普通桶 `s3:PutObject` 的区别

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `cat /tmp/express-bucket-name.txt` | 形如 `s3wkshp<8位数字>--use1-azN--x-s3` |
| 2 | `aws s3api list-objects-v2 --bucket "$(cat /tmp/express-bucket-name.txt)" --query 'length(Contents)' --output text` | `10` |

---

## 实验总结

S3 Express One Zone 用"放弃跨 AZ 冗余"换取数量级的延迟提升，是 S3 家族里第一个引入全新桶架构（Directory Bucket）的存储类型——命名规则、权限模型（`s3express` 而非 `s3`）、存储类型限制都与普通桶不同。它不是要替代 Standard，而是给延迟敏感、生命周期短的场景（训练数据缓存、日志暂存、批处理中间文件）一个专门优化的选项；持久化归档需求仍然用普通桶配合生命周期规则（见 Demo03）。

---

## 清理

```bash
EXPRESS_BUCKET=$(cat /tmp/express-bucket-name.txt 2>/dev/null)
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
STANDARD_BUCKET="s3-workshop-express-compare-${ACCOUNT_ID}"

if [ -n "${EXPRESS_BUCKET}" ]; then
  aws s3api list-objects-v2 --bucket "${EXPRESS_BUCKET}" --query 'Contents[].Key' --output text | \
    tr '\t' '\n' | while read -r key; do
      [ -n "${key}" ] && aws s3api delete-object --bucket "${EXPRESS_BUCKET}" --key "${key}"
    done
  aws s3api delete-bucket --bucket "${EXPRESS_BUCKET}" --region us-east-1
  echo "Deleted: ${EXPRESS_BUCKET}"
fi

aws s3 rm "s3://${STANDARD_BUCKET}" --recursive 2>/dev/null
aws s3api delete-bucket --bucket "${STANDARD_BUCKET}" --region us-east-1 2>/dev/null
echo "Deleted: ${STANDARD_BUCKET}"

rm -f /tmp/express-bucket-name.txt /tmp/express-create-err.txt
echo "Cleanup complete"
```
