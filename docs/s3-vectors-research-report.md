# Amazon S3 Vectors（向量存储桶）调研报告

更新时间：2026-04-10

## 摘要

Amazon S3 Vectors 是 AWS 在 S3 体系内新增的一类原生向量存储能力。它不是“把向量文件放到对象存储里”这么简单，而是提供了专门的 **vector bucket**、**vector index**、向量写入、近似最近邻（ANN）查询、元数据过滤、IAM 与加密治理，以及与 Amazon Bedrock / Amazon OpenSearch 的集成能力。

从产品定位看，S3 Vectors 的核心价值不在于替代所有向量数据库，而在于解决以下长期存在的矛盾：

- 向量数据规模增长极快，长期保留成本高。
- 普通对象存储便宜但不具备原生向量查询能力。
- 向量数据库性能强，但用于承载海量冷数据时总成本往往不合理。

AWS 的思路是把“对象存储的规模、耐久与成本模型”和“原生语义检索能力”结合起来，形成一个更适合作为 **低成本、大规模、长期保存、查询频率相对较低** 的向量存储层。其最佳实践通常不是独立承担所有检索任务，而是与 Bedrock、OpenSearch 或其他向量数据库形成冷热分层架构。

## 1. 背景介绍

### 1.1 什么是 S3 Vectors

Amazon S3 Vectors 是 S3 的一种新能力，提供面向向量嵌入的原生存储与查询接口。它使用独立于传统对象桶的资源模型，包括：

- `Vector bucket`
- `Vector index`
- `Vector`

每个向量通常由以下部分组成：

- 唯一标识 `key`
- 向量数据 `data`
- 可过滤与回传的 `metadata`

与普通 S3 `PutObject` / `GetObject` 不同，S3 Vectors 使用专门的 API，例如 `CreateVectorBucket`、`CreateIndex`、`PutVectors`、`QueryVectors` 等。

### 1.2 为什么会出现这个功能

#### 问题一：Embedding 的规模已经超出传统“热索引”经济边界

在 RAG、企业知识库、AI Agent memory、多模态检索、推荐系统等场景中，原始数据会被切分并转换成大量 embedding。随着数据规模从百万增长到千万、上亿甚至十亿级，向量不再只是一个“索引附属物”，而成为一类需要单独治理的数据资产。

这里的难点不只是存得下，而是：

- 要保留足够多的历史 embedding，保证召回覆盖率
- 要支持在线相似检索，而不是只做离线归档
- 要在多租户、权限、加密、成本之间取得平衡

#### 问题二：直接放对象存储，只解决了“存储”，没有解决“检索”

如果把 embedding 直接序列化后存进普通对象存储，企业确实能以较低成本保留向量，但仍然要自行搭建：

- 向量索引构建链路
- 向量搜索服务
- 元数据过滤
- 权限控制
- 存储与索引之间的一致性维护
- 扩缩容和高可用

这意味着对象存储本身并没有变成“可直接用来做语义检索的数据底座”，它只是持久化介质。

#### 问题三：向量数据库很强，但未必适合长期承载海量冷数据

向量数据库通常为了低延迟、高并发和复杂检索能力，需要常驻计算、索引结构、内存与高性能磁盘。对于“频繁访问的小部分热向量”这是合理的；但对于“大量低频访问、仍需在线可查”的长尾数据，这种资源模型成本往往偏高。

S3 Vectors 针对的正是这个缺口：把向量检索能力直接下沉到对象存储层，以更低的长期持有成本提供可用的语义检索。

### 1.3 S3 Vectors 具体解决什么问题

S3 Vectors 主要解决以下几类问题：

#### 1.3.1 海量向量的持久化和低成本保留

企业不再需要只保留少量热点 embedding，可以更激进地保留更多历史、更多切片、更细粒度的 embedding 数据。

#### 1.3.2 低运维的原生向量检索

使用者无需自己管理索引服务、ANN 引擎与搜索集群，可以直接通过 AWS 提供的 API 进行向量写入和查询。

#### 1.3.3 面向 AI 应用的统一数据底座

如果原始文件已经在 S3 上，embedding 又需要长期保留，那么把向量也放入 S3 体系内，可以减少跨系统同步和额外中间层。

#### 1.3.4 冷热分层的架构基础

S3 Vectors 可以作为更便宜的向量冷层或温层，高性能查询层则交给 OpenSearch、专用向量数据库或应用侧缓存。

### 1.4 产品能力边界

截至当前公开文档，S3 Vectors 的主要公开规格包括：

- 单个 vector index 最多 20 亿向量
- 单个 vector bucket 最多 10,000 个 index
- 每次 `PutVectors` 最多 500 个向量
- 每个 index 写入上限约 1000 次 Put/Delete 请求每秒，或约 2500 向量每秒
- 查询结果 `topK` 最多 100
- 向量维度范围 1-4096
- 当前数据类型为 `float32`
- 当前距离度量为 `cosine` 和 `euclidean`
- 较频繁查询场景延迟可到约 100ms，低频查询通常为亚秒级

这些能力边界说明，S3 Vectors 的主目标不是极致低延迟或复杂检索，而是成本与规模更优的原生向量存储层。

## 2. 使用场景，以及与对象存储、向量数据库的区别

### 2.1 典型使用场景

#### 2.1.1 企业知识库与 RAG

文档规模大、生命周期长、查询相对分散，希望在可接受延迟下显著降低长期 embedding 成本。

#### 2.1.2 AI Agent 长期记忆

把对话历史、计划步骤、工作记忆摘要等向量化后长期保留。此类数据通常“必须存很久”，但并不都是高频实时访问。

#### 2.1.3 多模态数据归档检索

图像、音频、视频、医疗影像、工业监控等场景天生会生成大量 embedding，适合使用低成本向量持久层。

#### 2.1.4 多租户 SaaS 检索

可按租户拆分 index 进行权限隔离、配额管理与治理，特别适合“每个租户有独立知识空间”的 SaaS 产品。

#### 2.1.5 向量冷热分层

热数据保留在 OpenSearch 或专用向量数据库，温冷数据转入 S3 Vectors，既保留可查性，又控制整体 TCO。

### 2.2 与直接对象存储的区别

| 维度 | 直接对象存储 | S3 Vectors |
|---|---|---|
| 数据形态 | 文件/对象 | 向量、索引、元数据 |
| 原生相似检索 | 无 | 有 |
| 元数据过滤 | 需自建 | 原生支持 |
| API 模型 | `PutObject/GetObject` | `PutVectors/QueryVectors` |
| 使用门槛 | 存储简单，检索复杂 | 存储和检索一体 |
| 运维复杂度 | 需要额外自建检索系统 | 更低 |

直接对象存储的优势是通用、便宜、生态成熟，但它不是向量检索系统。S3 Vectors 的意义在于把“存储”和“基础检索”合并到同一个服务层里。

### 2.3 与向量数据库的区别

| 维度 | S3 Vectors | 向量数据库 / 搜索引擎 |
|---|---|---|
| 产品定位 | 低成本、超大规模向量存储与检索层 | 高性能、复杂检索与在线服务层 |
| 成本模型 | 更偏向长期保留与低频查询优化 | 更偏向实时搜索与高 QPS |
| 延迟特征 | 亚秒级，热点可更低 | 毫秒到几十毫秒 |
| 能力面 | ANN + metadata filter 为主 | 往往支持更丰富的检索能力 |
| 运维 | AWS 托管，较轻 | 托管或自建，复杂度更高 |
| 适用数据温度 | 温 / 冷 | 热 / 温 |

### 2.4 相比直接对象存储的优势与劣势

#### 优势

- 具备原生向量查询，无需自建 ANN 服务
- 支持 metadata filter，适合多租户与业务约束场景
- 继承 AWS 身份、权限、加密、审计与网络治理体系
- 更容易直接接入 Bedrock、OpenSearch 等 AWS AI 与搜索产品

#### 劣势

- 不如普通对象桶通用
- 对向量维度、类型、距离度量和请求配额有明确限制
- 资源模型是新能力，迁移与适配需要专门工作

### 2.5 相比向量数据库的优势与劣势

#### 优势

- 海量低频向量的长期持有成本更有优势
- 更适合作为统一向量底座，减少跨系统同步
- 免运维程度更高
- 与 S3 生态天然一致，适合沉淀为企业向量数据湖

#### 劣势

- 不主打极致低延迟和高并发
- 当前可选数据类型、距离度量与查询能力较少
- 不适合作为所有检索需求的唯一系统
- 在复杂混合检索、全文检索、排序与聚合方面不如搜索型系统

### 2.6 选型判断

可以用一句话概括：

- 如果只是离线保存 embedding，普通对象存储就够了。
- 如果要低成本保留超大规模 embedding 且要支持原生相似检索，S3 Vectors 更合适。
- 如果需要高 QPS、低延迟、关键词加向量混合检索、复杂排序和搜索体验，向量数据库或搜索引擎更合适。

## 3. 使用方式：接口、数据模型与接入流程

### 3.1 资源模型

S3 Vectors 的核心资源包括：

- `Vector bucket`
- `Index`
- `Vector`

每个 index 创建时要固定以下关键参数：

- `dataType`
- `dimension`
- `distanceMetric`
- metadata 的过滤配置

其中名称、维度、距离度量和部分 metadata 配置创建后不可修改，因此建模阶段必须提前确定 embedding 模型与维度。

### 3.2 API 体系

S3 Vectors 使用独立的 API 命名空间和 endpoint，而不是复用传统 S3 对象操作接口。常见操作包括：

#### 3.2.1 Bucket 级

- `CreateVectorBucket`
- `GetVectorBucket`
- `ListVectorBuckets`
- `DeleteVectorBucket`

#### 3.2.2 Index 级

- `CreateIndex`
- `GetIndex`
- `ListIndexes`
- `DeleteIndex`

#### 3.2.3 Vector 级

- `PutVectors`
- `GetVectors`
- `DeleteVectors`
- `ListVectors`
- `QueryVectors`

#### 3.2.4 权限与治理

- `PutVectorBucketPolicy`
- `GetVectorBucketPolicy`
- `DeleteVectorBucketPolicy`
- `TagResource`
- `UntagResource`

### 3.3 SDK、CLI 与 REST API

常见接入方式包括：

- AWS SDK，例如 Python `boto3.client("s3vectors")`
- AWS CLI，例如 `aws s3vectors ...`
- S3 Vectors REST API
- CloudFormation 资源编排

这也意味着应用需要显式区分：

- 普通对象桶访问走 `s3`
- 向量桶访问走 `s3vectors`

### 3.4 Metadata 与过滤

S3 Vectors 支持 metadata filter。公开文档中支持的过滤操作符包括：

- `$eq`
- `$ne`
- `$gt`
- `$gte`
- `$lt`
- `$lte`
- `$in`
- `$nin`
- `$exists`
- `$and`
- `$or`

Metadata 使用时需要特别注意：

- metadata 总大小有限制
- filterable metadata 大小有限制
- 非过滤 metadata key 需要在建 index 时声明
- metadata key 总数也有限制

因此建模时要区分哪些字段：

- 用于过滤
- 仅用于返回展示
- 仅用于离线追踪

### 3.5 标准接入流程

#### 3.5.1 选定 embedding 模型

最先要确定的是 embedding 模型、向量维度和距离度量。不同 embedding 模型通常不应混放在同一个 index 中。

#### 3.5.2 创建 vector bucket 和 index

建立 bucket 后，再按业务域或租户创建 index。典型拆分方式包括：

- 按业务域拆分
- 按租户拆分
- 按 embedding 模型拆分
- 按冷热数据拆分

#### 3.5.3 批量写入向量

应用将对象切片后生成 embedding，再以批方式调用 `PutVectors` 写入，配合 metadata 写入业务上下文。

#### 3.5.4 查询向量

查询时必须使用与数据写入时相同 embedding 体系生成 query vector，再调用 `QueryVectors` 获取 topK 相似结果，并可叠加 metadata filter 限定范围。

#### 3.5.5 与下游系统组合

检索返回的 key 或 metadata 可进一步关联：

- 原始对象或文档内容
- 权限系统
- 应用侧重排序逻辑
- LLM prompt 组装

### 3.6 最小示意

```python
import boto3

s3vectors = boto3.client("s3vectors", region_name="us-west-2")

s3vectors.put_vectors(
    vectorBucketName="kb-prod",
    indexName="docs",
    vectors=[
        {
            "key": "doc-1",
            "data": {"float32": embedding},
            "metadata": {
                "tenant": "tenant-a",
                "doc_id": "123",
                "source": "manual"
            }
        }
    ]
)

resp = s3vectors.query_vectors(
    vectorBucketName="kb-prod",
    indexName="docs",
    queryVector={"float32": query_embedding},
    topK=10,
    filter={"tenant": "tenant-a"},
    returnDistance=True,
    returnMetadata=True
)
```

### 3.7 最佳实践要点

- 同一个 index 中保持统一的 embedding 模型与维度
- 使用批量写入，避免过小请求
- metadata 只保留必要字段
- 多租户场景按租户或业务域拆 index
- 把高频低延迟查询交给更热的查询层，S3 Vectors 承接长期向量保留

## 4. 业界实现方案与架构路线

当前行业里，与“向量存储桶”相关的实现并不是一条路线，而是多种不同的系统设计哲学。

### 4.1 路线一：对象存储原生提供向量能力

代表：Amazon S3 Vectors

核心思想：

- 向量能力直接下沉到对象存储产品中
- bucket 和 index 成为一等资源
- 向量存储与基础查询能力内建于存储层

架构特点：

- 存储产品本身承担持久化和基础查询职责
- 资源治理沿用对象存储体系
- 更适合大规模、长期持有、低频或中频语义检索

优点：

- 运维最轻
- 存储成本模型友好
- 容易成为企业统一数据底座的一部分

缺点：

- 能力面通常不如搜索引擎或专用数据库丰富
- 不以极致实时为核心目标

### 4.2 路线二：对象存储后端的 Serverless 向量数据库

代表：Pinecone Serverless

官方公开信息表明，Pinecone Serverless 采用对象存储作为持久层，通过 WAL、LSM 风格 slab、后台 compaction 和无状态 query executors 实现 serverless 检索。

核心思想：

- 数据持久化放在对象存储
- 计算层按需拉起并行执行
- 用数据库方式封装底层对象存储

架构特点：

- 仍然是数据库产品，不是对象存储产品
- 强调数据库体验与 serverless 成本优化并存

优点：

- 在低成本基础上保留较强数据库能力
- 查询层可按需扩展

缺点：

- 仍是专用数据库体系
- 与对象存储相比，系统复杂度和抽象层次更高

### 4.3 路线三：存算分离的开源向量数据库

代表：Milvus / Zilliz

Milvus 的官方架构由访问层、协调层、流式节点、查询节点、数据节点、元数据组件、对象存储和 WAL 等组成。

核心思想：

- 对象存储作为持久层
- 向量数据库负责索引构建、查询执行和分布式调度
- 计算与存储解耦

优点：

- 灵活、开放、可自建
- 能支撑更复杂的数据流与大规模集群部署

缺点：

- 部署和运维复杂
- 需要团队具备数据库平台化能力

### 4.4 路线四：搜索引擎集成向量检索

代表：Azure AI Search、Amazon OpenSearch Service

这条路线的特征不是“最低存储成本”，而是“把向量能力融入搜索引擎”。

核心思想：

- 向量检索、关键词检索、过滤、排序、混合召回在同一搜索系统中协同工作

优点：

- 混合检索能力强
- 更适合用户直接可感知的搜索场景
- 往往具备更丰富的表达式、排序、聚合和运营能力

缺点：

- 长期承载海量冷向量时成本不一定最优
- 底层仍以搜索集群资源为核心

值得注意的是，AWS 自身也在 OpenSearch 中增加了 `s3vector` engine，使 OpenSearch 可以与 S3 Vectors 形成联动。这说明 AWS 自己的产品路线也不是“二选一”，而是“分层协同”。

### 4.5 路线五：关系数据库集成向量能力

代表：pgvector

核心思想：

- 向量作为数据库字段直接存储
- 用 HNSW、IVFFlat 等索引方式提供相似搜索
- 通过 SQL 与事务体系统一访问

优点：

- 与结构化业务数据高度一致
- 支持事务、JOIN、备份恢复
- 对中小规模业务特别友好

缺点：

- 不适合充当超大规模低成本向量冷湖
- 向量规模继续增长后，资源模型往往不如对象存储体系经济

### 4.6 行业趋势分析

从这些路线可以看到几个清晰趋势：

#### 趋势一：向量系统普遍走向存算分离

无论是对象存储原生化，还是 serverless 向量数据库，抑或开源数据库架构，都在把“低成本持久层”和“高性能计算层”逐步解耦。

#### 趋势二：冷热分层会成为长期常态

所有向量都放在高性能实时系统里并不经济。随着 embedding 数量暴涨，企业最终都会走向：

- 热层：高性能搜索引擎或向量数据库
- 温冷层：对象存储或原生向量存储桶

#### 趋势三：对象存储正在成为 AI 数据基础设施

S3 Vectors 的战略意义不只是新 API，而是说明对象存储正在从“静态字节仓库”进化为“面向 AI 的统一数据底座”。

## 5. 这个功能的价值

### 5.1 业务价值

#### 5.1.1 降低 AI 应用长期成本

RAG、Agent、推荐、多模态检索的难点之一是“保留多少向量”。S3 Vectors 降低了保留大规模历史 embedding 的门槛。

#### 5.1.2 提高数据保留策略的上限

企业不必因为向量数据库成本而只保留少量热点 embedding，可以保留更多时间跨度、更细粒度的 embedding，改善召回覆盖率。

#### 5.1.3 简化数据链路

原始数据已经在 S3，向量又放在 S3 Vectors，能够减少跨系统同步与双写复杂度。

### 5.2 技术价值

#### 5.2.1 把语义检索下沉到基础存储层

这使“存储”和“检索”之间的边界发生了变化。对象存储不再只是被动存放文件，而是可以直接支撑语义查询。

#### 5.2.2 形成更清晰的分层架构

S3 Vectors 非常适合承担：

- 向量冷层
- 长期记忆层
- 向量归档层
- 大规模多租户底座

而高性能在线搜索仍可由专用检索系统承担。

#### 5.2.3 更自然地接入 AWS 治理体系

IAM、bucket policy、KMS、PrivateLink、区域端点、ARN 资源模型都可以延续 AWS 原生治理方式。

### 5.3 架构价值

S3 Vectors 的真正架构价值，不是单点性能，而是把企业向量架构从“高性能数据库承载一切”改造成“冷温热分层”的更经济模型。

典型架构可以是：

1. 原始文档保存在 S3 普通对象桶
2. embedding 保存在 S3 vector bucket
3. 热点向量同步到 OpenSearch 或专用向量数据库
4. 应用按查询类型选择查询层

这种分层设计可以同时获得：

- 更低长期存储成本
- 较好的在线检索能力
- 更强的数据治理一致性

### 5.4 战略价值

从 AWS 的产品布局看，S3 已不再只是单一对象桶，而是在演化为多种专门化数据容器：

- 普通对象桶
- 目录桶
- 表桶
- 向量桶

这意味着 S3 正在向“统一数据底座”扩展。对 AWS 而言，S3 Vectors 的价值不仅是一个 AI 功能点，更是把 AI 数据生命周期继续绑定在 S3 之上的战略步骤。

## 6. 结论与建议

### 6.1 结论

S3 Vectors 最适合的定位是：

- 海量向量的长期保存层
- 低成本原生语义检索层
- AI 应用的向量冷层或温层
- AWS 内部 AI 生态的基础存储层

它不应被理解为“传统向量数据库的全面替代品”，而应被理解为：

> 一个把对象存储成本模型和基础向量检索能力结合起来的新基础设施层。

### 6.2 建议

如果业务目标是以下方向，S3 Vectors 值得优先评估：

- embedding 数量预计达到千万到十亿级
- 查询频率不高，但必须在线可查
- 成本敏感，希望减少长期向量存储账单
- 已经大量使用 S3、Bedrock、OpenSearch 等 AWS 服务

如果业务目标主要是以下方向，则仍应优先考虑搜索引擎或专用向量数据库：

- 高频低延迟在线搜索
- 混合检索、复杂排序与搜索运营能力
- 更强的查询表达式与结果控制
- 面向交互式搜索产品的前台体验

## 参考资料

1. AWS What’s New: Amazon S3 Vectors Preview  
   https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-s3-vectors-preview-native-support-storing-querying-vectors/

2. AWS What’s New: Amazon S3 Vectors GA  
   https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-s3-vectors-generally-available

3. Amazon S3 Vectors 产品页  
   https://aws.amazon.com/s3/features/vectors/

4. Amazon S3 Vectors 用户指南  
   https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors.html

5. Amazon S3 Vectors 限制与配额  
   https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-limitations.html

6. Amazon S3 Vectors API Operations  
   https://docs.aws.amazon.com/AmazonS3/latest/API/API_Operations_Amazon_S3_Vectors.html

7. Amazon S3 Vectors Getting Started  
   https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-getting-started.html

8. Amazon S3 Vectors 与 Bedrock Knowledge Bases 集成  
   https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-bedrock-kb.html

9. Amazon S3 Vectors 与 OpenSearch 集成  
   https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-opensearch.html

10. OpenSearch `s3vector` engine 文档  
    https://docs.aws.amazon.com/opensearch-service/latest/developerguide/s3-vector-opensearch-integration-engine.html

11. Amazon S3 Pricing  
    https://aws.amazon.com/s3/pricing/

12. Pinecone Architecture: How Pinecone Works  
    https://www.pinecone.io/how-pinecone-works/

13. Milvus 架构概览  
    https://milvus.io/docs/zh/architecture_overview.md

14. pgvector 官方仓库  
    https://github.com/pgvector/pgvector

15. Google Vertex AI Storage-Optimized Vector Search  
    https://docs.cloud.google.com/vertex-ai/docs/vector-search/storage-optimized-vector-search

16. Azure AI Search Vector Search Overview  
    https://learn.microsoft.com/en-us/azure/search/vector-search-overview

17. Amazon S3 Vectors 区域与端点  
    https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-regions-quotas.html

18. Amazon S3 Vectors 访问控制  
    https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-access-management.html

19. Amazon S3 Vectors PrivateLink  
    https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-privatelink.html

20. Amazon S3 Vectors 加密与数据保护  
    https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-data-encryption.html

21. Amazon S3 Vectors Best Practices  
    https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-best-practices.html
