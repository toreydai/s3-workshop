# S3 Workshop
## Amazon S3 Hands-on Lab Collection

覆盖基础操作到企业级安全合规、成本优化、数据湖架构 · 全球区（us-east-1）· Prompt 驱动执行

---

## Demo 列表

按技术主题分组（同一主题的 Demo 归在一起），组内按难度从低到高排列；编号已按此顺序重新连续排列为 01-20，跟着编号顺序做就是从易到难的完整路径：

### 1. 存储与生命周期
- [Demo01 — S3 基础操作：存储桶创建与对象管理](docs/demo01-basic-operations.md)
- [Demo02 — 多部分上传（Multipart Upload）大文件处理](docs/demo02-multipart-upload.md)
- [Demo03 — S3 存储类型与生命周期管理](docs/demo03-lifecycle-storage-classes.md)
- [Demo04 — 跨区域复制（CRR）与同区域复制（SRR）](docs/demo04-replication-crr-srr.md)
- [Demo05 — S3 Object Lock：WORM 合规存储](docs/demo05-object-lock.md)
- [Demo06 — S3 Batch Operations：批量对象操作](docs/demo06-batch-operations.md)
- [Demo07 — S3 Inventory：存储清单与大规模分析](docs/demo07-inventory.md)

### 2. 安全与权限控制
- [Demo08 — S3 安全与访问控制：Bucket Policy、加密与 Block Public Access](docs/demo08-security-access-control.md)
- [Demo09 — 预签名 URL 与临时授权访问](docs/demo09-presigned-urls.md)
- [Demo10 — S3 访问点（Access Points）与多租户权限隔离](docs/demo10-access-points.md)
- [Demo11 — 跨账号 S3 访问：多账号架构权限模式](docs/demo11-cross-account-access.md)

### 3. 网络与分发
- [Demo12 — S3 CORS 配置：跨域资源共享](docs/demo12-cors.md)
- [Demo13 — S3 静态网站托管与 CloudFront 加速](docs/demo13-static-website-cloudfront.md)
- [Demo14 — VPC Endpoint for S3 私网访问](docs/demo14-vpc-endpoint.md)

### 4. 事件驱动集成
- [Demo15 — S3 事件通知与 Lambda 集成](docs/demo15-event-notifications.md)
- [Demo16 — S3 事件通知与 EventBridge 集成](docs/demo16-eventbridge.md)
- [Demo17 — S3 事件通知扇出：SNS + SQS + Lambda 解耦架构](docs/demo17-sns-sqs-fanout.md)

### 5. 数据分析与可观测性
- [Demo18 — S3 Select：对象内容 SQL 查询](docs/demo18-s3-select.md)
- [Demo19 — S3 + Athena 数据湖查询](docs/demo19-athena-data-lake.md)
- [Demo20 — S3 Storage Lens 深度分析与成本优化](docs/demo20-storage-lens.md)

---

## 使用方式

1. 在此目录下打开 Claude Code
2. 将对应 [`docs/`](docs/) 目录中的 Demo 文件内容粘贴到对话框，由 AI 自主执行
3. 每个 Demo 末尾均有**清理**步骤，实验结束后执行以避免持续计费

---

## 环境要求

| 工具 | 版本 |
|------|------|
| AWS CLI | 2.x |
| Python3 | 3.8+ |
| curl | latest |

操作机建议使用 Amazon Linux 2023 EC2，绑定具备 `AmazonS3FullAccess` 的 IAM Role；部分 Demo 需要额外权限：

| Demo | 所需额外权限（除 `AmazonS3FullAccess` 外）|
|------|----------------------------------------|
| Demo04 | `IAMFullAccess` |
| Demo06 | `IAMFullAccess` |
| Demo08 | `AWSKMSFullAccess` |
| Demo10 | （仅 S3 Control，包含在 S3FullAccess 中）|
| Demo11 | `IAMFullAccess`、`AWSKMSFullAccess` |
| Demo12 | （无额外权限，需开启存储桶公读） |
| Demo13 | `CloudFrontFullAccess` |
| Demo14 | `AmazonVPCFullAccess`、`AmazonEC2FullAccess` |
| Demo15、Demo17 | `AWSLambdaFullAccess`、`IAMFullAccess` |
| Demo16 | `AmazonEventBridgeFullAccess`、`AmazonSQSFullAccess`、`AWSLambdaFullAccess`、`IAMFullAccess` |
| Demo17 | `AmazonSNSFullAccess`、`AmazonSQSFullAccess`、`AWSLambdaFullAccess` |
| Demo19 | `AmazonAthenaFullAccess`、`AWSGlueConsoleFullAccess` |

## License

MIT - see the [LICENSE](LICENSE) file for details.

## 免责声明

本项目仅供学习与技术参考，不构成生产部署方案。运行过程中会创建 AWS 资源并产生费用，请在实验结束后及时清理。作者不对因使用本项目产生的任何费用或损失承担责任。本项目与 Amazon Web Services 无官方关联，相关服务的可用性与定价以 AWS 官方文档为准。生产环境使用前请根据实际需求进行安全评估与调整。
