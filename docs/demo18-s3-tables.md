# Demo18 — S3 Tables：原生 Apache Iceberg 表存储

## 实验简介

S3 Tables（2024 年 re:Invent 发布）是 S3 第一次原生理解"表"语义：一种新的桶类型——**Table Bucket**，专门存储 Apache Iceberg 格式的表数据，后台自动做小文件压缩、快照过期清理和存储优化，官方给出的对比数据是查询性能提升 3 倍、事务吞吐（TPS）提升 10 倍。它跳过了"S3 + Glue Crawler + 手动维护 Iceberg 元数据"的传统组合，把表的创建、Schema 管理都收进了 S3 自身的 API。

**实验目标：**
- 理解 Table Bucket 与普通桶的区别：以表/命名空间为一等公民，而非对象/前缀
- 掌握 Table Bucket 创建、与 Glue Data Catalog 的集成方式
- 能够创建 Namespace 和 Iceberg 表，并用 Athena 查询和写入

**实验流程：**
1. 创建 Table Bucket
2. 集成 AWS Glue Data Catalog（`s3tablescatalog` 联邦目录）
3. 创建 Namespace 和 Iceberg 表
4. 用 Athena 查询表结构、写入数据、验证结果

**预计时长：** 30 分钟

---

## 前提条件

- **工具**：AWS CLI v2.23.10 及以上（`aws s3tables` 子命令在此版本后可用）
- **权限**：托管策略 `AmazonS3TablesFullAccess`，另需 Athena 查询权限（`AmazonAthenaFullAccess`）和 Glue Catalog 权限（`AWSGlueConsoleFullAccess`）
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export TABLE_BUCKET_NAME="s3-workshop-tables-${ACCOUNT_ID}"
export ATHENA_RESULTS_BUCKET="s3-workshop-tables-athena-results-${ACCOUNT_ID}"
aws --version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1
```

**预期输出**：CLI 版本号（确认 ≥ 2.23.10）

---

## 步骤

### 1. 创建 Table Bucket

Table Bucket 名称规则与普通桶类似（3-63 字符，小写字母/数字/连字符），但底层是完全不同的资源类型，不能像普通桶一样直接用 `aws s3api` 操作。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
TABLE_BUCKET_NAME="s3-workshop-tables-${ACCOUNT_ID}"

TABLE_BUCKET_ARN=$(aws s3tables create-table-bucket \
  --region us-east-1 \
  --name "${TABLE_BUCKET_NAME}" \
  --query 'arn' --output text)
echo "Table Bucket ARN: ${TABLE_BUCKET_ARN}"
```

**预期输出**：`Table Bucket ARN: arn:aws:s3tables:us-east-1:<ACCOUNT_ID>:bucket/s3-workshop-tables-<ACCOUNT_ID>`

### 2. 集成 AWS Glue Data Catalog

通过控制台创建 Table Bucket 时会自动完成这一步；用 CLI 创建则需要手动执行一次 `glue create-catalog`，为账号 + Region 建立名为 `s3tablescatalog` 的联邦目录，之后该 Region 内所有 Table Bucket 都会自动挂载为它的子目录，Athena/Redshift/EMR 等分析服务才能发现这些表。**这一步每个账号每个 Region 只需做一次**——如果之前已执行过（比如做过 Lake Formation 相关实验），直接跳到下一步。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws glue create-catalog \
  --name "s3tablescatalog" \
  --catalog-input "{
    \"Description\": \"Federated catalog for S3 Tables\",
    \"FederatedCatalog\": {
      \"Identifier\": \"arn:aws:s3tables:us-east-1:${ACCOUNT_ID}:bucket/*\",
      \"ConnectionName\": \"aws:s3tables\"
    },
    \"CreateDatabaseDefaultPermissions\": [{
      \"Principal\": {\"DataLakePrincipalIdentifier\": \"IAM_ALLOWED_PRINCIPALS\"},
      \"Permissions\": [\"ALL\"]
    }],
    \"CreateTableDefaultPermissions\": [{
      \"Principal\": {\"DataLakePrincipalIdentifier\": \"IAM_ALLOWED_PRINCIPALS\"},
      \"Permissions\": [\"ALL\"]
    }]
  }" 2>&1 | grep -v "AlreadyExistsException" || echo "s3tablescatalog 已存在，跳过"

aws glue get-catalogs --parent-catalog-id s3tablescatalog \
  --query 'CatalogList[].Name' --output text
```

**预期输出**：子目录列表中包含 `s3-workshop-tables-<ACCOUNT_ID>`

> ⚠️ 用 CLI 创建 Table Bucket 后如果跳过这一步，Table Bucket 本身能正常创建表，但 Athena 会查不到——Athena 依赖 Glue Data Catalog 发现表，而联邦目录集成是让 S3 Tables 出现在 Glue Data Catalog 里的唯一途径。

### 3. 创建 Namespace 和 Iceberg 表

Namespace 相当于数据库，Table 定义里必须全部使用小写字段名——大写字段名会导致 Lake Formation/Glue Data Catalog 无法识别该表，Athena 查询时报 `Unsupported Federation Resource`。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
TABLE_BUCKET_ARN="arn:aws:s3tables:us-east-1:${ACCOUNT_ID}:bucket/s3-workshop-tables-${ACCOUNT_ID}"

aws s3tables create-namespace \
  --table-bucket-arn "${TABLE_BUCKET_ARN}" \
  --namespace analytics_workshop

aws s3tables list-namespaces --table-bucket-arn "${TABLE_BUCKET_ARN}" \
  --query 'namespaces[].namespace' --output text
```

**预期输出**：`analytics_workshop`

创建 `orders` 表（Iceberg 格式）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
TABLE_BUCKET_ARN="arn:aws:s3tables:us-east-1:${ACCOUNT_ID}:bucket/s3-workshop-tables-${ACCOUNT_ID}"

cat > /tmp/orders-table-def.json << EOF
{
    "tableBucketARN": "${TABLE_BUCKET_ARN}",
    "namespace": "analytics_workshop",
    "name": "orders",
    "format": "ICEBERG",
    "metadata": {
        "iceberg": {
            "schema": {
                "fields": [
                    {"name": "order_id", "type": "int", "required": true},
                    {"name": "customer_id", "type": "string"},
                    {"name": "amount", "type": "double"},
                    {"name": "order_date", "type": "string"}
                ]
            }
        }
    }
}
EOF

aws s3tables create-table --cli-input-json file:///tmp/orders-table-def.json
echo "Table created: analytics_workshop.orders"
```

**预期输出**：`Table created: analytics_workshop.orders`

### 4. 用 Athena 查询表结构

Table Bucket 在 Athena 里通过 `s3tablescatalog/<table-bucket-name>` 作为 Catalog 名称访问，需要一个 Athena 查询结果桶（如果已有可复用）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ATHENA_RESULTS_BUCKET="s3-workshop-tables-athena-results-${ACCOUNT_ID}"
TABLE_BUCKET_NAME="s3-workshop-tables-${ACCOUNT_ID}"

aws s3api create-bucket --bucket "${ATHENA_RESULTS_BUCKET}" --region us-east-1

QUERY_ID=$(aws athena start-query-execution \
  --query-string "SELECT * FROM \"s3tablescatalog/${TABLE_BUCKET_NAME}\".\"analytics_workshop\".\"orders\" LIMIT 10" \
  --result-configuration "OutputLocation=s3://${ATHENA_RESULTS_BUCKET}/query-results/" \
  --query 'QueryExecutionId' --output text)

sleep 5
aws athena get-query-execution --query-execution-id "${QUERY_ID}" \
  --query 'QueryExecution.Status.State' --output text
```

**预期输出**：`SUCCEEDED`（空表，返回 0 行，但 Schema 已可被 Athena 识别）

### 5. 用 Athena 写入数据并验证

Iceberg 表支持标准 SQL `INSERT INTO`，S3 Tables 会在后台自动做小文件合并，不需要手动管理 Iceberg 快照/Compaction。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ATHENA_RESULTS_BUCKET="s3-workshop-tables-athena-results-${ACCOUNT_ID}"
TABLE_BUCKET_NAME="s3-workshop-tables-${ACCOUNT_ID}"

INSERT_QUERY_ID=$(aws athena start-query-execution \
  --query-string "INSERT INTO \"s3tablescatalog/${TABLE_BUCKET_NAME}\".\"analytics_workshop\".\"orders\" VALUES (1, 'cust-001', 199.99, '2026-07-01'), (2, 'cust-002', 89.50, '2026-07-02'), (3, 'cust-001', 45.00, '2026-07-03')" \
  --result-configuration "OutputLocation=s3://${ATHENA_RESULTS_BUCKET}/query-results/" \
  --query 'QueryExecutionId' --output text)

sleep 8
aws athena get-query-execution --query-execution-id "${INSERT_QUERY_ID}" \
  --query 'QueryExecution.Status.State' --output text
```

**预期输出**：`SUCCEEDED`

验证数据已写入：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ATHENA_RESULTS_BUCKET="s3-workshop-tables-athena-results-${ACCOUNT_ID}"
TABLE_BUCKET_NAME="s3-workshop-tables-${ACCOUNT_ID}"

VERIFY_QUERY_ID=$(aws athena start-query-execution \
  --query-string "SELECT COUNT(*) as cnt, SUM(amount) as total FROM \"s3tablescatalog/${TABLE_BUCKET_NAME}\".\"analytics_workshop\".\"orders\"" \
  --result-configuration "OutputLocation=s3://${ATHENA_RESULTS_BUCKET}/query-results/" \
  --query 'QueryExecutionId' --output text)

sleep 8
aws athena get-query-results --query-execution-id "${VERIFY_QUERY_ID}" \
  --query 'ResultSet.Rows[1].Data[*].VarCharValue' --output text
```

**预期输出**：`3	334.49`（3 行，金额合计 334.49）

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Table Bucket 已创建并挂载到 `s3tablescatalog` 联邦目录下
- [ ] `analytics_workshop` Namespace 下存在 `orders` 表
- [ ] Athena 可以查询到 `orders` 表结构（即使为空表）
- [ ] 通过 Athena `INSERT INTO` 写入 3 行数据，`COUNT(*)` 返回 3

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3tables list-namespaces --table-bucket-arn "arn:aws:s3tables:us-east-1:$(aws sts get-caller-identity --query Account --output text):bucket/s3-workshop-tables-$(aws sts get-caller-identity --query Account --output text)" --query 'namespaces[0].namespace' --output text` | `analytics_workshop` |
| 2 | `aws s3tables list-tables --table-bucket-arn "arn:aws:s3tables:us-east-1:$(aws sts get-caller-identity --query Account --output text):bucket/s3-workshop-tables-$(aws sts get-caller-identity --query Account --output text)" --namespace analytics_workshop --query 'tables[0].name' --output text` | `orders` |

---

## 实验总结

S3 Tables 把"存储"和"表"这两个原本靠 Glue Data Catalog 手动粘合的概念收进了 S3 自身：Table Bucket 天然理解 Iceberg 语义，写入即自动做存储优化，Athena/Redshift/EMR 通过一次性的 `s3tablescatalog` 联邦目录集成即可直接查询。跟传统"普通 S3 桶 + Glue Crawler 爬取 + 手动维护 Iceberg 表"的组合相比，省掉了 Crawler 调度和 Compaction 运维，是新建数据湖仓时的默认优先选项。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
TABLE_BUCKET_ARN="arn:aws:s3tables:us-east-1:${ACCOUNT_ID}:bucket/s3-workshop-tables-${ACCOUNT_ID}"
ATHENA_RESULTS_BUCKET="s3-workshop-tables-athena-results-${ACCOUNT_ID}"

aws s3tables delete-table \
  --table-bucket-arn "${TABLE_BUCKET_ARN}" \
  --namespace analytics_workshop --name orders 2>/dev/null
aws s3tables delete-namespace \
  --table-bucket-arn "${TABLE_BUCKET_ARN}" --namespace analytics_workshop 2>/dev/null
aws s3tables delete-table-bucket --table-bucket-arn "${TABLE_BUCKET_ARN}" 2>/dev/null
echo "Table Bucket deleted"

aws s3 rm "s3://${ATHENA_RESULTS_BUCKET}" --recursive 2>/dev/null
aws s3api delete-bucket --bucket "${ATHENA_RESULTS_BUCKET}" --region us-east-1 2>/dev/null
echo "Cleanup complete"
```

> 注：`s3tablescatalog` 联邦目录是账号 + Region 级共享资源，其他 Table Bucket（如后续实验创建的）可能仍依赖它，清理时不要删除该目录本身，只删除本实验创建的 Table Bucket。
