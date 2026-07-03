# Demo19 — S3 Vectors：原生向量存储与语义检索

## 实验简介

S3 Vectors（2025 年发布）是 S3 第一次原生支持向量数据：一种新的桶类型——**Vector Bucket**，专门存储和查询向量嵌入（Embedding），官方数据显示存储和查询成本比自建向量数据库低到 90%。它跟专用向量数据库（如 OpenSearch、Pinecone）的定位差异是：**牺牲部分极致查询延迟，换取近乎白送的存储成本和零运维**，适合语义检索、RAG 知识库这类对成本敏感、查询量不是极高频的场景，也是 Bedrock Knowledge Bases 的原生存储选项之一。

**实验目标：**
- 理解 Vector Bucket / Vector Index 的核心参数：维度（Dimension）、距离度量（Distance Metric）
- 掌握用 Bedrock Titan Embeddings 生成向量并写入 S3 Vectors 的流程
- 能够执行相似度检索（含 Metadata 过滤）验证语义匹配效果

**实验流程：**
1. 创建 Vector Bucket 和 Vector Index
2. 用 Bedrock Titan Text Embeddings V2 生成向量并写入索引
3. 执行相似度检索，验证语义匹配
4. 结合 Metadata 过滤缩小检索范围

**预计时长：** 30 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x（较新版本，`aws s3vectors` 子命令需要 2025 年后发布的 CLI），Python3，`boto3`
- **权限**：`AmazonS3FullAccess`（含 `s3vectors:*`），`AmazonBedrockFullAccess`
- **模型访问**：需要在 Bedrock 控制台为账号开通 `Amazon Titan Text Embeddings V2`（`amazon.titan-embed-text-v2:0`）模型访问权限——首次使用需手动申请，通常几分钟内自动批准
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export VECTOR_BUCKET="s3-workshop-vectors-${ACCOUNT_ID}"
export INDEX_NAME="support-tickets"
pip3 install --quiet boto3 --upgrade
python3 -c "import boto3; print('boto3', boto3.__version__)"
```

**预期输出**：`boto3 <version>`（确认较新版本，包含 `s3vectors` 客户端支持）

---

## 步骤

### 1. 创建 Vector Bucket

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
VECTOR_BUCKET="s3-workshop-vectors-${ACCOUNT_ID}"

aws s3vectors create-vector-bucket --vector-bucket-name "${VECTOR_BUCKET}"
echo "Vector bucket created: ${VECTOR_BUCKET}"
```

**预期输出**：`Vector bucket created: s3-workshop-vectors-<ACCOUNT_ID>`

### 2. 创建 Vector Index

维度必须与后续使用的 Embedding 模型输出维度一致——Titan Text Embeddings V2 默认输出 1024 维。**创建后维度、距离度量、名称都不可修改**，选错了只能删了重建。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
VECTOR_BUCKET="s3-workshop-vectors-${ACCOUNT_ID}"

aws s3vectors create-index \
  --vector-bucket-name "${VECTOR_BUCKET}" \
  --index-name "support-tickets" \
  --data-type "float32" \
  --dimension 1024 \
  --distance-metric "cosine"

aws s3vectors list-indexes --vector-bucket-name "${VECTOR_BUCKET}" \
  --query 'indexes[].{Name:indexName, Dim:dimension, Metric:distanceMetric}'
```

**预期输出**：
```json
[{"Name": "support-tickets", "Dim": 1024, "Metric": "cosine"}]
```

### 3. 生成向量并写入索引

用 Bedrock Titan Text Embeddings V2 把客服工单文本转成向量，连同原文和分类标签一起写入索引。原文本身作为**不可过滤的元数据**（`source_text`）存储，方便检索后直接展示；`category` 作为**可过滤元数据**，用于第 5 步的范围缩小检索。

```bash
cat > /tmp/put_vectors.py << 'PYEOF'
import boto3, json, os

region = os.environ.get("AWS_REGION", "us-east-1")
bedrock = boto3.client("bedrock-runtime", region_name=region)
s3vectors = boto3.client("s3vectors", region_name=region)

vector_bucket = os.environ["VECTOR_BUCKET"]
index_name = os.environ["INDEX_NAME"]

tickets = [
    {"id": "ticket-001", "text": "包裹签收后发现商品外壳有明显裂痕，申请退货退款", "category": "退换货"},
    {"id": "ticket-002", "text": "下单三天了物流状态一直没更新，想知道包裹到哪了", "category": "物流查询"},
    {"id": "ticket-003", "text": "收到的商品和页面描述的颜色不一致，能换货吗", "category": "退换货"},
    {"id": "ticket-004", "text": "支付时扣款成功但订单显示未支付，钱是不是被重复扣了", "category": "支付问题"},
    {"id": "ticket-005", "text": "快递员说已经派送但我一直没收到包裹，签收记录是别人签的", "category": "物流查询"},
]

vectors = []
for t in tickets:
    resp = bedrock.invoke_model(
        modelId="amazon.titan-embed-text-v2:0",
        body=json.dumps({"inputText": t["text"]}),
    )
    embedding = json.loads(resp["body"].read())["embedding"]
    vectors.append({
        "key": t["id"],
        "data": {"float32": embedding},
        "metadata": {"source_text": t["text"], "category": t["category"]},
    })

s3vectors.put_vectors(
    vectorBucketName=vector_bucket,
    indexName=index_name,
    vectors=vectors,
)
print(f"Inserted {len(vectors)} vectors into {index_name}")
PYEOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
VECTOR_BUCKET="s3-workshop-vectors-${ACCOUNT_ID}" INDEX_NAME="support-tickets" python3 /tmp/put_vectors.py
```

**预期输出**：`Inserted 5 vectors into support-tickets`

### 4. 相似度检索

查询"包裹一直没到，物流一直不更新"，语义上应该匹配 `ticket-002`（物流状态未更新）和 `ticket-005`（快递未签收），而不是关键词完全不同的退换货类工单。

```bash
cat > /tmp/query_vectors.py << 'PYEOF'
import boto3, json, os

region = os.environ.get("AWS_REGION", "us-east-1")
bedrock = boto3.client("bedrock-runtime", region_name=region)
s3vectors = boto3.client("s3vectors", region_name=region)

vector_bucket = os.environ["VECTOR_BUCKET"]
index_name = os.environ["INDEX_NAME"]

query_text = "包裹一直没到，物流状态也不更新"
resp = bedrock.invoke_model(
    modelId="amazon.titan-embed-text-v2:0",
    body=json.dumps({"inputText": query_text}),
)
query_embedding = json.loads(resp["body"].read())["embedding"]

result = s3vectors.query_vectors(
    vectorBucketName=vector_bucket,
    indexName=index_name,
    queryVector={"float32": query_embedding},
    topK=3,
    returnDistance=True,
    returnMetadata=True,
)
for v in result["vectors"]:
    print(f"{v['key']}\t distance={v['distance']:.4f}\t category={v['metadata']['category']}\t text={v['metadata']['source_text']}")
PYEOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
VECTOR_BUCKET="s3-workshop-vectors-${ACCOUNT_ID}" INDEX_NAME="support-tickets" python3 /tmp/query_vectors.py
```

**预期输出**（distance 数值会有细微差异，但排名前两位应为物流类工单）：
```
ticket-002	 distance=0.XXXX	 category=物流查询	 text=下单三天了物流状态一直没更新，想知道包裹到哪了
ticket-005	 distance=0.XXXX	 category=物流查询	 text=快递员说已经派送但我一直没收到包裹，签收记录是别人签的
ticket-001	 distance=0.XXXX	 category=退换货	 text=包裹签收后发现商品外壳有明显裂痕，申请退货退款
```

### 5. 结合 Metadata 过滤缩小检索范围

真实场景中经常需要"只在某个类目里做语义检索"（比如客服系统按工单类型分流）。`filter` 参数在语义排序的基础上先做精确过滤。

```bash
cat >> /tmp/query_vectors.py << 'PYEOF'

print("\n--- 加上 category=支付问题 过滤后 ---")
result_filtered = s3vectors.query_vectors(
    vectorBucketName=vector_bucket,
    indexName=index_name,
    queryVector={"float32": query_embedding},
    topK=3,
    filter={"category": "支付问题"},
    returnDistance=True,
    returnMetadata=True,
)
for v in result_filtered["vectors"]:
    print(f"{v['key']}\t category={v['metadata']['category']}\t text={v['metadata']['source_text']}")
PYEOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
VECTOR_BUCKET="s3-workshop-vectors-${ACCOUNT_ID}" INDEX_NAME="support-tickets" python3 /tmp/query_vectors.py | tail -3
```

**预期输出**：只返回 `ticket-004`（唯一 `category=支付问题` 的工单），即使它的语义距离比物流类工单更远。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Vector Index 维度为 1024，距离度量为 cosine
- [ ] 索引中包含 5 条向量记录
- [ ] 语义查询"物流未更新"返回的 Top2 结果都属于"物流查询"类目
- [ ] 加 `category` 过滤后，只返回匹配该类目的结果

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3vectors list-indexes --vector-bucket-name "s3-workshop-vectors-$(aws sts get-caller-identity --query Account --output text)" --query 'indexes[0].dimension' --output text` | `1024` |
| 2 | `aws s3vectors list-vectors --vector-bucket-name "s3-workshop-vectors-$(aws sts get-caller-identity --query Account --output text)" --index-name support-tickets --query 'length(vectors)' --output text` | `5` |

---

## 实验总结

S3 Vectors 把向量检索的成本模型从"专用集群按小时计费"改成了"按存储量 + 查询量计费"，用极低运维成本换取语义检索能力，特别适合 RAG 知识库这类查询量不算极高但数据量可能很大的场景。它跟 Bedrock Knowledge Bases 也是原生集成的——生产环境如果不想手写 `put_vectors`/`query_vectors`，可以直接用 Bedrock Knowledge Bases 的"Quick create vector store"选项自动生成 Vector Bucket 并托管整个 embedding→检索流程。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
VECTOR_BUCKET="s3-workshop-vectors-${ACCOUNT_ID}"

aws s3vectors delete-index --vector-bucket-name "${VECTOR_BUCKET}" --index-name "support-tickets" 2>/dev/null
aws s3vectors delete-vector-bucket --vector-bucket-name "${VECTOR_BUCKET}" 2>/dev/null
rm -f /tmp/put_vectors.py /tmp/query_vectors.py
echo "Cleanup complete"
```
