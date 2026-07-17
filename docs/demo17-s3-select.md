# Demo17 — S3 Select：对象内容 SQL 查询

## 实验简介

S3 Select 允许直接在 S3 服务端对 CSV、JSON、Parquet 对象执行 SQL 查询，只返回匹配的行和列，而非下载整个对象再在客户端过滤。对于 1GB 的 CSV 文件，如果只需要 10 行数据，S3 Select 可以节省约 99% 的数据传输量和处理时间，是数据湖查询的重要优化手段。

**实验目标：**
- 掌握 S3 Select 对 CSV 和 JSON 格式的查询语法
- 理解 `InputSerialization` 和 `OutputSerialization` 的配置格式
- 能够用 SQL WHERE/SELECT 语句在 S3 对象内部过滤数据

**实验流程：**
1. 创建存储桶并上传 CSV 和 JSON 测试数据
2. 对 CSV 文件执行 S3 Select 查询
3. 对 JSON 文件执行 S3 Select 查询
4. 比较 Select 与全量下载的数据量差异

**预计 AI 执行时长：** 20 分钟

---

## 前提条件

- **工具**：AWS CLI v2.x，Python3
- **权限**：`AmazonS3FullAccess`
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-workshop-select-${ACCOUNT_ID}"
echo "Bucket: ${BUCKET_NAME}"
```

---

## 步骤

### 1. 创建存储桶并生成测试数据

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-select-${ACCOUNT_ID}"
aws s3api create-bucket --bucket "${BUCKET_NAME}" --region us-east-1
```

**预期输出**：`{"Location": "/s3-workshop-select-<ACCOUNT_ID>"}`

生成 1000 行 CSV 测试数据：
```bash
python3 << 'EOF'
import csv, random, io
from datetime import datetime, timedelta

rows = []
departments = ["Engineering", "Finance", "Marketing", "HR", "Operations"]
for i in range(1, 1001):
    rows.append({
        "employee_id": f"EMP{i:04d}",
        "name": f"Employee {i}",
        "department": random.choice(departments),
        "salary": random.randint(40000, 200000),
        "join_date": (datetime(2015, 1, 1) + timedelta(days=random.randint(0, 3000))).strftime("%Y-%m-%d"),
        "active": random.choice(["true", "false"])
    })

with open("/tmp/employees.csv", "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=rows[0].keys(), lineterminator="\n")
    w.writeheader()
    w.writerows(rows)

print(f"Generated {len(rows)} rows, file size: {len(open('/tmp/employees.csv').read())} bytes")
EOF
```

**预期输出**：`Generated 1000 rows, file size: <size> bytes`

> ⚠️ `csv.DictWriter` 默认使用 `\r\n` 作为行终止符。若不显式传 `lineterminator="\n"`，最后一列（本例中的 `active`）的值会带有尾随的 `\r`（例如实际值是 `"true\r"` 而非 `"true"`），导致 S3 Select 对最后一列做 `WHERE active = 'true'` 精确匹配时始终返回 0 行（因为默认 `RecordDelimiter` 是 `\n`，`\r` 被当作字段内容保留）。必须加 `lineterminator="\n"` 避免此坑。

生成 JSON Lines 格式测试数据：
```bash
python3 << 'EOF'
import json, random
products = ["Laptop", "Phone", "Tablet", "Monitor", "Keyboard", "Mouse", "Headphone", "Webcam"]
with open("/tmp/orders.jsonl", "w") as f:
    for i in range(1, 201):
        order = {
            "order_id": f"ORD-{i:05d}",
            "product": random.choice(products),
            "quantity": random.randint(1, 10),
            "unit_price": round(random.uniform(10, 2000), 2),
            "status": random.choice(["PENDING", "SHIPPED", "DELIVERED", "CANCELLED"]),
            "region": random.choice(["us-east-1", "us-west-2", "eu-west-1", "ap-northeast-1"])
        }
        f.write(json.dumps(order) + "\n")
print(f"Generated 200 orders, file size: {len(open('/tmp/orders.jsonl').read())} bytes")
EOF
```

**预期输出**：`Generated 200 orders, file size: <size> bytes`

上传到 S3：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-select-${ACCOUNT_ID}"
aws s3 cp /tmp/employees.csv "s3://${BUCKET_NAME}/data/employees.csv" --content-type "text/csv"
aws s3 cp /tmp/orders.jsonl "s3://${BUCKET_NAME}/data/orders.jsonl" --content-type "application/x-ndjson"
echo "Test data uploaded"
```

**预期输出**：`Test data uploaded`

### 2. 对 CSV 查询：筛选 Engineering 部门薪资 > 150000 的员工

S3 Select 的 SQL 中 `s3Object` 代表 S3 对象本身，`FileHeaderInfo: USE` 表示用第一行作为列名。这个查询只返回匹配行，大幅减少数据传输量。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-select-${ACCOUNT_ID}"

aws s3api select-object-content \
  --bucket "${BUCKET_NAME}" \
  --key "data/employees.csv" \
  --expression "SELECT employee_id, name, salary FROM s3Object WHERE department = 'Engineering' AND CAST(salary AS INT) > 150000" \
  --expression-type SQL \
  --input-serialization '{"CSV": {"FileHeaderInfo": "USE", "RecordDelimiter": "\n", "FieldDelimiter": ","}}' \
  --output-serialization '{"CSV": {"RecordDelimiter": "\n", "FieldDelimiter": ","}}' \
  /tmp/high-salary-engineers.csv \
  --region us-east-1

echo "Results:"
cat /tmp/high-salary-engineers.csv
echo "Total matching rows: $(wc -l < /tmp/high-salary-engineers.csv)"
```

**预期输出**：若干行 `EMP<id>,Employee <n>,<salary>` 格式数据，总行数远少于 1000

### 3. 对 JSON 查询：筛选 DELIVERED 状态且金额 > 5000 的订单

JSON Lines 格式（每行一个 JSON 对象）需要设置 `Type: LINES`。SQL 中用 `s.product`、`s.quantity` 等访问 JSON 字段。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-select-${ACCOUNT_ID}"

aws s3api select-object-content \
  --bucket "${BUCKET_NAME}" \
  --key "data/orders.jsonl" \
  --expression "SELECT s.order_id, s.product, s.quantity, s.unit_price FROM s3Object[*] s WHERE s.status = 'DELIVERED' AND CAST(s.quantity AS FLOAT) * CAST(s.unit_price AS FLOAT) > 5000" \
  --expression-type SQL \
  --input-serialization '{"JSON": {"Type": "LINES"}}' \
  --output-serialization '{"JSON": {"RecordDelimiter": "\n"}}' \
  /tmp/large-orders.jsonl \
  --region us-east-1

echo "High-value delivered orders:"
cat /tmp/large-orders.jsonl | jq . | head -30
echo "Total matching orders: $(wc -l < /tmp/large-orders.jsonl)"
```

**预期输出**：若干行 JSON 对象，每行包含 `order_id`、`product`、`quantity`、`unit_price`

> ⚠️ `python3 -m json.tool` 只能解析单个 JSON 文档，而 `/tmp/large-orders.jsonl` 是多行 JSON Lines（每行一个独立对象），直接整体喂给 `json.tool` 会报 `Extra data` 错误，因为命令带了 `2>/dev/null` 会静默吞掉错误、只打印空输出。改用 `jq .`（默认支持流式解析多个 JSON 值）即可正确逐行美化打印。

### 4. 统计查询：按部门计算平均薪资

S3 Select 支持 `COUNT`、`AVG`、`SUM` 等聚合函数，可以直接在 S3 层做汇总，不需要下载全部数据到客户端。

> ⚠️ **S3 Select 不支持 `GROUP BY`**（实测报错 `UnsupportedSqlOperation: Unsupported SQL operation GROUP BY`）。S3 Select 的 SQL 子集只支持单表扫描 + 聚合函数作用于整个匹配结果集，不支持分组。要按维度分组统计，只能像下面这样对每个维度值分别发起一次查询（用 `WHERE` 过滤代替 `GROUP BY`），或改用 Athena（Athena 基于标准 SQL 引擎，完整支持 GROUP BY）。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-select-${ACCOUNT_ID}"

echo "Department statistics (active employees):"
echo "department,headcount,avg_salary"
for dept in Engineering Finance Marketing HR Operations; do
  aws s3api select-object-content \
    --bucket "${BUCKET_NAME}" \
    --key "data/employees.csv" \
    --expression "SELECT COUNT(*) as headcount, AVG(CAST(salary AS FLOAT)) as avg_salary FROM s3Object WHERE active = 'true' AND department = '${dept}'" \
    --expression-type SQL \
    --input-serialization '{"CSV": {"FileHeaderInfo": "USE"}}' \
    --output-serialization '{"CSV": {}}' \
    "/tmp/dept-stats-${dept}.csv" \
    --region us-east-1
  echo "${dept},$(cat /tmp/dept-stats-${dept}.csv)"
done
```

**预期输出**：5 行数据（每个部门一次查询），每行格式为 `<Department>,<count>,<avg_salary>`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 存储桶包含 `data/employees.csv`（1000 行）和 `data/orders.jsonl`（200 行）
- [ ] CSV 查询成功返回 Engineering 部门薪资 > 150000 的子集
- [ ] JSON 查询成功返回 DELIVERED 状态大额订单
- [ ] 聚合查询成功按部门输出员工数和平均薪资

---

## 验证检查点

| # | 检查命令（自给自足，无 shell 变量依赖） | 期望精确输出 |
|---|----------------------------------------|-------------|
| 1 | `aws s3api head-object --bucket "s3-workshop-select-$(aws sts get-caller-identity --query Account --output text)" --key "data/employees.csv" --query 'ContentType' --output text` | `text/csv` |
| 2 | `aws s3 ls "s3://s3-workshop-select-$(aws sts get-caller-identity --query Account --output text)/data/" --summarize \| grep "Total Objects"` | `Total Objects: 2` |

---

## 实验总结

S3 Select 将查询下推到 S3 服务端，在大数据场景下显著减少数据传输量和客户端处理负担。支持格式：CSV、JSON、Parquet（Parquet 需配合 Glacier Select）。局限：不支持 JOIN、子查询，不支持 Parquet 嵌套结构的复杂访问——这些场景应使用 Athena。

---

## 清理

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="s3-workshop-select-${ACCOUNT_ID}"
aws s3 rm "s3://${BUCKET_NAME}" --recursive
aws s3api delete-bucket --bucket "${BUCKET_NAME}" --region us-east-1
echo "Bucket deleted: ${BUCKET_NAME}"
```
