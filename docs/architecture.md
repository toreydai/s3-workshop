# 架构文档

本仓库包含 20 个 Demo，这里不做全量架构图汇总，只对其中组件交互较复杂、值得可视化的 Demo 提供架构图；其余 Demo 请直接看对应的 `docs/demoXX-*.md`。

以下 6 个 Demo 涉及多组件事件链路、跨账号/跨区域架构、治理数据流或生成式 AI 集成，复杂度明显高于其余以"配置单个桶属性 + 验证"为主的 Demo，因此单独画图：

- **Demo05 — CRR/SRR 跨区域复制**：双区域双桶 + 专属 IAM 角色的异步复制链路
- **Demo07 — Inventory + Batch Operations**：清单生成与批量执行两阶段治理闭环
- **Demo11 — 跨账号访问**：Bucket Policy / Assume Role 两种模式 + KMS 双重授权
- **Demo16 — SNS+SQS 扇出架构**：S3 事件广播给 SNS，再并行分发给多个 SQS 队列与 Lambda，多消费者解耦链路
- **Demo18 — S3 Tables（Apache Iceberg）**：Table Bucket + Glue 联邦目录集成 + Athena 查询/写入的数据湖仓链路
- **Demo19 — S3 Vectors**：Bedrock Titan Embeddings 生成向量 + S3 Vectors 存储与相似度检索的语义检索链路

---

## Demo05 — 跨区域复制（CRR）与同区域复制（SRR）

源桶（us-east-1）与目标桶（us-west-2）都必须开启版本控制，这是复制的硬前提。专属 IAM 角色（信任 `s3.amazonaws.com`）负责读取源桶对象版本并写入目标桶，复制规则可按前缀过滤、指定目标存储类（此处降级为 STANDARD_IA 省成本）、并同步删除标记。复制是异步的，通过 `ReplicationStatus` 字段追踪进度。

```mermaid
flowchart LR
  Source["S3 Source Bucket\ns3-workshop-crr-source-<acct>\n(us-east-1, 版本控制 Enabled)"]
  Role["IAM Role: s3-workshop-crr-role\n(Trust: s3.amazonaws.com)\nGetObjectVersionForReplication / ReplicateObject"]
  Dest["S3 Dest Bucket\ns3-workshop-crr-dest-<acct>\n(us-west-2, 版本控制 Enabled\n目标存储类 STANDARD_IA)"]

  Source -->|"put-bucket-replication\nRule: crr-all-objects, Priority 1\nDeleteMarkerReplication: Enabled"| Role
  Role -->|异步复制对象版本| Dest
  Source -.->|"head-object\nReplicationStatus: PENDING→COMPLETED"| Source
```

---

## Demo07 — S3 Inventory 与 Batch Operations：清单驱动的批量对象操作

Inventory 确定操作范围，Batch Operations 批量执行——两者是天然搭档。源桶配置两条 Inventory 规则（每日全量 CSV + 每周 Parquet），清单写入目标桶（Bucket Policy 用 `ArnLike SourceArn` 限定只信任特定源桶）。真实生产中 Batch Operations 的 Manifest 直接引用 Inventory 产出的 `manifest.json`；Batch Operations 专属 IAM 角色（信任 `batchoperations.s3.amazonaws.com`）读取清单、对源桶对象执行标签更新等操作，任务状态经历 `New → Preparing → Ready(需人工确认) → Active → Complete`，完成报告写回目标桶。

```mermaid
flowchart TB
  Source["S3 Source Bucket\n13 个对象（Standard + Standard-IA）"]
  Dest["S3 Dest Bucket\n(Bucket Policy: 仅信任 Source 的 Inventory 写入)"]
  InvCSV["Inventory 规则: daily-full-inventory\n(CSV, 全量, 每日)"]
  InvParquet["Inventory 规则: weekly-parquet-inventory\n(Parquet, 每周)"]
  Select["S3 Select\n直接对清单 SQL 查询\n（无需下载整份清单）"]
  Manifest["Batch Manifest\n(S3BatchOperations_CSV_20180820\n生产环境可直接引用 Inventory 的 manifest.json)"]
  Role["IAM Role: s3-invbatch-ops-role\n(Trust: batchoperations.s3.amazonaws.com)"]
  Job["S3 Batch Operations Job\nS3PutObjectTagging\n状态: New→Preparing→Ready→Active→Complete"]
  Report["完成报告\nbatch-reports/"]

  Source -->|定期生成| InvCSV --> Dest
  Source -->|定期生成| InvParquet --> Dest
  Dest --> Select
  Source -->|list-objects-v2 拼装等价清单| Manifest --> Dest
  Manifest --> Role --> Job
  Job -->|批量 PutObjectTagging| Source
  Job --> Report --> Dest
```

---

## Demo11 — 跨账号 S3 访问：多账号架构权限模式

企业多账号架构中最常见的两种跨账号访问模式：**Bucket Policy 直接授权**（适合只读、无需 Assume Role）和 **Assume Role**（适合精细权限 + 审计追踪）。当数据桶启用 KMS 加密时，跨账号访问需要 Bucket Policy 与 KMS Key Policy **两层授权同时到位**，缺一即报 `AccessDenied`。

```mermaid
flowchart TB
  subgraph AcctA["账号 A（数据拥有者）"]
    Bucket["S3 Data Bucket\ns3-workshop-xacct-data-<acct>"]
    BucketPolicy["Bucket Policy\nPrincipal: Reader Role ARN\nAction: GetObject/ListBucket"]
    KMSKey["KMS Key: alias/s3-xacct-key\nKey Policy: 允许 Reader Role Decrypt"]
  end

  subgraph AcctB["账号 B（模拟：Reader Role）"]
    ReaderRole["IAM Role: xacct-data-reader-role\n(生产中 Principal 为真实外部账号 ARN)"]
  end

  ReaderRole -->|"sts:assume-role"| ReaderRole
  ReaderRole -->|"模式一：GetObject（无需 Assume Role 亦可，此处统一走角色）"| BucketPolicy
  BucketPolicy --> Bucket
  ReaderRole -->|"PutObject（写入）"| BucketPolicy
  BucketPolicy -.->|"AccessDenied（策略未授予写权限）"| ReaderRole

  Bucket -->|"SSE-KMS 加密对象"| KMSKey
  ReaderRole -->|"kms:Decrypt（IAM Policy + Key Policy 双重放行）"| KMSKey
  KMSKey --> Bucket
```

```mermaid
sequenceDiagram
  participant Ops as 操作者（模拟账号B）
  participant STS as AWS STS
  participant Role as Reader Role
  participant S3 as Data Bucket（账号A）
  participant KMS as KMS Key（账号A）

  Ops->>STS: assume-role(xacct-data-reader-role)
  STS-->>Ops: 临时凭证
  Ops->>S3: GetObject datasets/sales-2025.json（以 Role 身份）
  S3-->>Ops: 200 数据内容（Bucket Policy 放行）
  Ops->>S3: PutObject malicious.txt
  S3-->>Ops: 403 AccessDenied（策略未授予写权限）
  Ops->>S3: GetObject encrypted/data.json（SSE-KMS）
  S3->>KMS: Decrypt（校验 IAM Policy + Key Policy）
  KMS-->>S3: 明文数据密钥
  S3-->>Ops: 200 解密后内容
```

---

## Demo16 — S3 事件通知扇出：SNS + SQS + Lambda 解耦架构

Demo14 是 S3 → Lambda 的直连架构，新增消费者需要改 S3 通知配置；本 Demo 把 S3 事件先广播到 SNS Topic，再由 SNS 并行分发给两个 SQS 队列（模拟归档、处理两个下游系统）和一个 Lambda，三个消费者各自独立订阅、独立扩展、互不影响。

```mermaid
flowchart TB
  Bucket["S3 Bucket\ns3-workshop-fanout-<acct>"]
  Topic["SNS Topic\ns3-events-fanout-topic\n(Topic Policy 允许 s3.amazonaws.com Publish)"]

  Q1["SQS: s3-events-archive-queue\n(队列策略允许 SNS SendMessage)"]
  Q2["SQS: s3-events-processing-queue\n(队列策略允许 SNS SendMessage)"]
  Lambda["Lambda: s3-fanout-processor\n(Python 3.12, 打印 bucket/key/size)"]

  Bucket -->|"put-bucket-notification-configuration\nEvents: s3:ObjectCreated:*"| Topic
  Topic -->|订阅 1: protocol=sqs| Q1
  Topic -->|订阅 2: protocol=sqs| Q2
  Topic -->|订阅 3: protocol=lambda| Lambda
```

```mermaid
sequenceDiagram
  participant U as 执行者
  participant S3 as S3 Bucket
  participant SNS as SNS Topic
  participant Q1 as SQS(archive)
  participant Q2 as SQS(processing)
  participant L as Lambda

  U->>S3: aws s3 cp fanout-test.txt
  S3->>SNS: ObjectCreated 事件（经 Topic Policy 授权发布）
  par 并行扇出
    SNS->>Q1: 投递消息
    SNS->>Q2: 投递消息
    SNS->>L: 触发调用（经 lambda:InvokeFunction 授权）
  end
  U->>Q1: receive-message 轮询验证（投递延迟不稳定，最多轮询30秒）
  L-->>U: CloudWatch Logs 打印 Bucket/Key/Size
```

---

## Demo18 — S3 Tables：原生 Apache Iceberg 表存储

Table Bucket 是一种新的桶类型，原生理解 Iceberg 表语义。用 CLI 创建 Table Bucket 后需要额外执行一次账号+Region 级的 `glue create-catalog`，把 Table Bucket 挂载到 `s3tablescatalog` 联邦目录下，Athena 才能发现并查询/写入其中的表。

```mermaid
flowchart TB
  TableBucket["S3 Table Bucket\ns3-workshop-tables-<acct>"]
  Catalog["Glue Data Catalog\ns3tablescatalog（联邦目录，账号+Region 级，仅需建一次）"]
  Namespace["Namespace: analytics_workshop"]
  IcebergTable["Iceberg Table: orders\n(order_id, customer_id, amount, order_date)"]
  Athena["Amazon Athena\n(SELECT / INSERT INTO)"]
  ResultsBucket["S3: Athena 查询结果桶"]

  TableBucket -->|"glue create-catalog\n(FederatedCatalog, 一次性)"| Catalog
  Catalog -.自动挂载子目录.-> TableBucket
  TableBucket --> Namespace --> IcebergTable
  Athena -->|"s3tablescatalog/<bucket>.analytics_workshop.orders"| IcebergTable
  Athena --> ResultsBucket
```

---

## Demo19 — S3 Vectors：原生向量存储与语义检索

Vector Bucket 是原生支持向量嵌入存储与查询的新桶类型。用 Bedrock Titan Text Embeddings V2 把客服工单文本转成 1024 维向量并写入 S3 Vectors 索引，之后既可以做纯语义相似度检索，也可以叠加 Metadata 过滤（如按 `category` 精确过滤后再排序）。

```mermaid
flowchart TB
  Tickets["5 条客服工单文本\n(带 category 标签)"]
  Bedrock["Amazon Bedrock Runtime\namazon.titan-embed-text-v2:0\n(1024 维 Embedding)"]
  VectorBucket["S3 Vector Bucket\ns3-workshop-vectors-<acct>"]
  Index["Vector Index: support-tickets\n(dimension=1024, metric=cosine)"]

  Query["查询文本\n(如: 包裹一直没到)"]
  QueryVec["query_vectors()\ntopK=3, returnDistance/Metadata"]
  Filtered["query_vectors(filter={category:...})\n先精确过滤再语义排序"]

  Tickets --> Bedrock -->|put_vectors: key+embedding+metadata| Index
  VectorBucket --> Index
  Query --> Bedrock
  Bedrock -->|embedding| QueryVec --> Index
  Index -->|Top-K 相似结果| QueryVec
  Index --> Filtered
```
