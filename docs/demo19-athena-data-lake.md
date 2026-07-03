# Demo19 — S3 + Athena 数据湖查询

## 实验简介

Athena 是 AWS 的 Serverless 交互式查询服务，直接对 S3 中存储的数据运行标准 SQL，无需搬移数据、无需管理集群，按扫描量计费（5 美元/TB）。结合 Glue Data Catalog 管理表结构，S3 + Athena 构成了最轻量的数据湖查询架构，适合日志分析、报表查询和 ad-hoc 探索。

**实验目标：**
- 掌握 Glue Data Catalog 的数据库和表创建
- 理解 Athena 分区投影（Partition Projection）减少扫描量
- 能够对 S3 中的 CSV/JSON 数据运行 SQL 并将结果写入 S3

**实验流程：**
1. 创建 S3 数据桶并上传 CSV 数据
2. 创建 Glue 数据库和表
3. 在 Athena 执行 SQL 查询
4. 配置查询结果存储桶

**预计时长：** 30 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x，Python3
- **权限**：`AmazonS3FullAccess`、`AmazonAthenaFullAccess`、`AWSGlueConsoleFullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export DATA_BUCKET="s3-workshop-datalake-${ACCOUNT_ID}"
export RESULT_BUCKET="s3-workshop-athena-results-${ACCOUNT_ID}"
echo "Data: ${DATA_BUCKET} | Results: ${RESULT_BUCKET}"
```

---

## 步骤

### 1. 创建数据桶和查询结果桶

Athena 要求指定一个 S3 路径存储查询结果，这个桶与数据桶分开，便于设置不同的生命周期策略（查询结果通常只保留 30 天）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-datalake-${ACCOUNT_ID}"
RESULT_BUCKET="s3-workshop-athena-results-${ACCOUNT_ID}"

aws s3api create-bucket --bucket "${DATA_BUCKET}" --region us-east-1
aws s3api create-bucket --bucket "${RESULT_BUCKET}" --region us-east-1
echo "Buckets created"
```

**预期输出**：`Buckets created`

### 2. 生成带分区结构的测试数据

Athena 利用 Hive 分区目录结构（`year=2025/month=01/`）进行分区裁剪，只扫描匹配分区的数据，大幅减少扫描量和成本。

> ⚠️ 直接执行下面这个唯一的代码块即可（纯 bash + Python 单行脚本，`${DATA_BUCKET}` 已通过 shell 变量正确展开）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-datalake-${ACCOUNT_ID}"

for ym in "2025/10" "2025/11" "2025/12" "2026/01"; do
  year=$(echo $ym | cut -d'/' -f1)
  month=$(echo $ym | cut -d'/' -f2)
  python3 -c "
import csv, random
rows = [['EMP{:04d}'.format(i), 'Employee {}'.format(i), random.choice(['Engineering','Finance','Marketing','HR']),
         random.randint(40000,200000), random.choice(['ACTIVE','INACTIVE','ON_LEAVE']),
         '{}-{}-{:02d}'.format('${year}','${month}',random.randint(1,28))] for i in range(1,51)]
print('employee_id,name,department,salary,status,record_date')
for r in rows: print(','.join(str(v) for v in r))
" > /tmp/hr-${year}-${month}.csv
  aws s3 cp /tmp/hr-${year}-${month}.csv \
    "s3://${DATA_BUCKET}/hr-data/year=${year}/month=${month}/data.csv" --quiet
  echo "Uploaded: year=${year}/month=${month}"
done

aws s3 ls "s3://${DATA_BUCKET}/hr-data/" --recursive --summarize | grep "Total Objects"
```

**预期输出**：
```
Uploaded: year=2025/month=10
Uploaded: year=2025/month=11
Uploaded: year=2025/month=12
Uploaded: year=2026/month=01
Total Objects: 4
```

> ⚠️ 原命令用 `grep "Total"` 会同时匹配 `--summarize` 输出的 `Total Objects` 和 `Total Size` 两行，与预期输出（只有一行）不符，已改为 `grep "Total Objects"` 精确匹配。

### 3. 创建 Glue 数据库和表

Glue Data Catalog 是 Athena 的元数据存储。表定义中 `LOCATION` 指向 S3 前缀，`PARTITIONED BY` 声明分区键，`TBLPROPERTIES` 中的 `projection.*` 配置分区投影（自动推断分区，无需手动 MSCK REPAIR TABLE）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-datalake-${ACCOUNT_ID}"

# 创建 Glue 数据库
aws glue create-database \
  --database-input '{"Name": "s3_workshop_db", "Description": "S3 Workshop Data Lake"}' \
  --region us-east-1
echo "Glue database created: s3_workshop_db"
```

**预期输出**：`Glue database created: s3_workshop_db`

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-datalake-${ACCOUNT_ID}"

aws glue create-table \
  --database-name "s3_workshop_db" \
  --table-input "{
    \"Name\": \"hr_records\",
    \"Description\": \"HR Employee Records partitioned by year/month\",
    \"StorageDescriptor\": {
      \"Columns\": [
        {\"Name\": \"employee_id\", \"Type\": \"string\"},
        {\"Name\": \"name\", \"Type\": \"string\"},
        {\"Name\": \"department\", \"Type\": \"string\"},
        {\"Name\": \"salary\", \"Type\": \"int\"},
        {\"Name\": \"status\", \"Type\": \"string\"},
        {\"Name\": \"record_date\", \"Type\": \"string\"}
      ],
      \"Location\": \"s3://${DATA_BUCKET}/hr-data/\",
      \"InputFormat\": \"org.apache.hadoop.mapred.TextInputFormat\",
      \"OutputFormat\": \"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat\",
      \"SerdeInfo\": {
        \"SerializationLibrary\": \"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe\",
        \"Parameters\": {
          \"field.delim\": \",\",
          \"skip.header.line.count\": \"1\"
        }
      }
    },
    \"PartitionKeys\": [
      {\"Name\": \"year\", \"Type\": \"string\"},
      {\"Name\": \"month\", \"Type\": \"string\"}
    ],
    \"TableType\": \"EXTERNAL_TABLE\",
    \"Parameters\": {
      \"classification\": \"csv\",
      \"projection.enabled\": \"true\",
      \"projection.year.type\": \"enum\",
      \"projection.year.values\": \"2025,2026\",
      \"projection.month.type\": \"enum\",
      \"projection.month.values\": \"01,02,03,04,05,06,07,08,09,10,11,12\",
      \"storage.location.template\": \"s3://${DATA_BUCKET}/hr-data/year=\${year}/month=\${month}/\"
    }
  }" \
  --region us-east-1
echo "Glue table created: hr_records"
```

**预期输出**：`Glue table created: hr_records`

### 4. 配置 Athena 查询结果存储并执行 SQL

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
RESULT_BUCKET="s3-workshop-athena-results-${ACCOUNT_ID}"

aws athena update-work-group \
  --work-group primary \
  --configuration-updates "ResultConfigurationUpdates={OutputLocation=s3://${RESULT_BUCKET}/query-results/}" \
  --region us-east-1
echo "Athena result location configured: s3://${RESULT_BUCKET}/query-results/"
```

**预期输出**：`Athena result location configured`

执行查询：按部门统计 2025 年 Q4 的平均薪资（利用分区裁剪只扫描 3 个月数据）：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
RESULT_BUCKET="s3-workshop-athena-results-${ACCOUNT_ID}"

QUERY_ID=$(aws athena start-query-execution \
  --query-string "SELECT department, COUNT(*) as headcount, AVG(salary) as avg_salary, MIN(salary) as min_salary, MAX(salary) as max_salary FROM s3_workshop_db.hr_records WHERE year = '2025' AND month IN ('10', '11', '12') AND status = 'ACTIVE' GROUP BY department ORDER BY avg_salary DESC" \
  --query-execution-context Database=s3_workshop_db \
  --result-configuration "OutputLocation=s3://${RESULT_BUCKET}/query-results/" \
  --region us-east-1 \
  --query 'QueryExecutionId' \
  --output text)
echo "Query started: ${QUERY_ID}"
```

**预期输出**：`Query started: <query-id>`

等待查询完成并获取结果：
```bash
QUERY_ID=$(aws athena list-query-executions \
  --region us-east-1 --query 'QueryExecutionIds[0]' --output text)

# 轮询状态（若返回 RUNNING，等 10 秒后重试）
aws athena get-query-execution \
  --query-execution-id "${QUERY_ID}" \
  --region us-east-1 \
  --query 'QueryExecution.Status.State' \
  --output text
```

**预期输出**：`SUCCEEDED`

```bash
QUERY_ID=$(aws athena list-query-executions \
  --region us-east-1 --query 'QueryExecutionIds[0]' --output text)

aws athena get-query-results \
  --query-execution-id "${QUERY_ID}" \
  --region us-east-1 \
  --query 'ResultSet.Rows[*].Data[*].VarCharValue' \
  --output table
```

**预期输出**：部门统计表格，包含 headcount、avg_salary、min_salary、max_salary

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 数据桶包含 4 个分区目录（year=2025/month=10 ~ year=2026/month=01）
- [ ] Glue 数据库 `s3_workshop_db` 和表 `hr_records` 已创建
- [ ] Athena 查询 Q4 2025 数据，状态为 `SUCCEEDED`
- [ ] 查询结果写入到 `s3-workshop-athena-results-<ACCOUNT_ID>/query-results/`

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws glue get-database --name s3_workshop_db --region us-east-1 --query 'Database.Name' --output text` | `s3_workshop_db` |
| 2 | `aws glue get-table --database-name s3_workshop_db --name hr_records --region us-east-1 --query 'Table.StorageDescriptor.Location' --output text` | `s3://s3-workshop-datalake-<ACCOUNT_ID>/hr-data/` |
| 3 | `aws athena list-query-executions --region us-east-1 --query 'length(QueryExecutionIds)' --output text` | ≥ `1` |

---

## 实验总结

本实验构建了最轻量的数据湖架构：S3 存储原始数据、Glue Catalog 管理元数据、Athena 运行 SQL 查询、结果写回 S3。核心优化：分区裁剪（WHERE year/month）让 Athena 只扫描相关分区而非全量数据，直接影响查询速度和费用。生产建议：将常用数据转换为 Parquet + Snappy 压缩，相比 CSV 可减少 80-90% 的扫描量。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
DATA_BUCKET="s3-workshop-datalake-${ACCOUNT_ID}"
RESULT_BUCKET="s3-workshop-athena-results-${ACCOUNT_ID}"

# 删除 Glue 表和数据库
aws glue delete-table --database-name s3_workshop_db --name hr_records --region us-east-1 2>/dev/null
aws glue delete-database --name s3_workshop_db --region us-east-1 2>/dev/null

# 删除存储桶
for bucket in "${DATA_BUCKET}" "${RESULT_BUCKET}"; do
  aws s3 rm "s3://${bucket}" --recursive 2>/dev/null
  aws s3api delete-bucket --bucket "${bucket}" --region us-east-1 2>/dev/null
done
echo "Cleanup complete"
```
