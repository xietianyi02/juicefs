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

## 2. 使用场景，以及与纯对象存储、S3 向量存储、Milvus 的区别

本节不再只做抽象对比，而是把三条路线拆开分析：

- **纯对象存储**：embedding 作为普通文件或对象存储，没有原生向量检索能力。
- **S3 Vectors**：对象存储原生提供向量存储与 ANN 查询。
- **Milvus（代表向量数据库）**：专门面向高性能向量检索的数据库系统，官方架构为存算分离，底层同样可以使用对象存储保存索引和历史数据。

这里把 Milvus 单独拿出来讲，是因为“向量数据库”这个词很宽，而 Milvus 的官方能力、架构和适用边界都比较典型，足以代表这一类系统的主流设计。

### 2.1 三条技术路线的本质差异

#### 2.1.1 纯对象存储：解决“保存”，不解决“在线检索”

纯对象存储的典型做法是：

1. 把原始数据存到对象桶中
2. 生成 embedding 后，把向量序列化成文件或对象再写回对象桶
3. 需要查询时，在应用层或单独的检索系统里加载这些向量并计算相似度

这种方式的本质是把对象存储当作“低成本持久化介质”或“向量数据湖”。它擅长：

- 低成本长期保留
- 离线处理和批量重建
- 训练、评估、回放和归档

但它不擅长：

- 原生在线 ANN 检索
- 实时 metadata 过滤
- 多租户在线权限控制
- 低延迟服务化访问

因此，纯对象存储本质上是 **向量数据仓库**，不是 **向量检索系统**。

#### 2.1.2 S3 Vectors：解决“低成本持久化 + 原生基础检索”

S3 Vectors 的核心变化是把向量从“普通对象”升级成“可被原生理解的数据类型”。它不仅能存，还能直接：

- 建立 vector index
- 写入 vector records
- 执行 ANN 查询
- 叠加 metadata filter
- 接入 IAM、KMS、PrivateLink、bucket policy

因此，S3 Vectors 本质上是 **对象存储原生向量层**。它的价值在于：

- 保留对象存储的规模和成本模型
- 补上原生向量检索能力
- 避免为了“可查”而额外维持一个大型向量数据库集群

#### 2.1.3 Milvus：解决“高性能、复杂检索、在线服务”

Milvus 的目标不是做对象存储，而是做向量数据库。根据官方架构文档，Milvus 采用 **shared-storage / storage-compute disaggregation** 设计，包含访问层、协调层、工作节点和存储层。存储层里有：

- 元数据存储
- WAL 存储
- 对象存储

其中对象存储保存日志快照、向量和标量索引文件以及中间结果；Query Node 从对象存储加载历史数据，Data Node 负责 compaction 和 index building。这说明 Milvus 并不是“内存数据库”，而是 **以对象存储为持久层、以数据库引擎为查询层** 的分布式向量检索系统。

更重要的是，Milvus 的官方能力面远大于“只做 dense vector ANN”：

- 支持多种索引与搜索算法
- 支持标量过滤
- 支持 dense、sparse 和 hybrid retrieval
- 支持多向量字段与 rerank
- 新版本还引入了 tiered storage，用于把热数据缓存和远端冷数据进一步分层

所以，Milvus 本质上是 **向量数据库 / 检索引擎**，适合承担高性能和更复杂的在线搜索任务。

### 2.2 典型业务场景的详细分析

下面按业务场景看三者谁更合适。

#### 2.2.1 企业知识库与 RAG

这是最常见的场景，也是三种路线最容易混淆的地方。

#### 纯对象存储

适合把文档、切片文本、embedding 和离线构建结果全部沉淀下来，便于：

- 重新切片
- 更换 embedding 模型
- 批量回放评估
- 构建训练集

但如果直接用纯对象存储承担在线 RAG 检索，就会遇到几个问题：

- 没有原生向量查询能力
- 需要自己维护索引文件和加载流程
- metadata filter 需要应用自己实现
- 随着数据增大，在线检索体验会迅速变差

因此，纯对象存储更适合作为 **RAG 的离线底座**，不适合作为唯一在线检索层。

#### S3 Vectors

如果知识库数据量很大、文档生命周期长、查询频率不是特别高，S3 Vectors 很适合：

- embedding 规模达到千万甚至更高
- 希望显著降低长期持有成本
- 希望减少运维和集群管理
- 已经在 AWS 体系内使用 Bedrock Knowledge Bases、S3、OpenSearch

它尤其适合“企业知识库很大，但用户问答流量并不是搜索引擎级别”的场景。问题在于，如果你要做更复杂的检索链路，例如：

- dense + sparse 混合召回
- 多阶段 rerank
- 更复杂的过滤与排序
- 高 QPS、低延迟前台问答

仅靠 S3 Vectors 通常不够，需要再叠加 OpenSearch 或其他检索层。

#### Milvus

Milvus 更适合以下 RAG 场景：

- 检索质量需要持续调优
- 需要 dense + sparse hybrid retrieval
- 需要多向量字段联合召回
- 需要更高并发和更低延迟
- 希望把 RAG 做成稳定对外服务

Milvus 的优势是可以把语义检索、标量过滤、混合召回和数据库级调优放在一个系统里完成。代价是：

- 你需要接受数据库级部署和运维复杂度
- 需要规划 Query Node、Data Node、对象存储、WAL、缓存与扩容策略

**结论**：

- 小规模、偏离线：纯对象存储够用
- 大规模、成本敏感、查询不算太热：S3 Vectors 更优
- 面向线上检索服务、效果与延迟要求高：Milvus 更优

#### 2.2.2 AI Agent 长期记忆

Agent memory 和普通 RAG 的差异是：写入频繁、内容碎片化、保留时间长，但读流量往往高度不均匀。

#### 纯对象存储

适合把所有历史记忆完整沉淀下来，用于：

- 全量回放
- 行为审计
- 模型评估
- 长周期分析

但它本身不提供“可直接用于 Agent 在线检索”的能力，因此只能作为归档层。

#### S3 Vectors

这类场景是 S3 Vectors 的强项之一。原因在于：

- Agent memory 总量增长非常快
- 许多记忆是“必须保存，但很少查询”
- 访问模式天然带有冷热分布

把长期记忆放在 S3 Vectors，可以在保证在线可查的同时控制成本。对于“近期上下文”或“高频记忆”，再额外保留在更热的查询层会更合理。

#### Milvus

如果 Agent 系统要求：

- 高频写入和高频读
- 多租户隔离明显
- 复杂过滤和更强召回质量
- 低延迟工具调用

那么 Milvus 更适合作为在线记忆库。但如果把所有历史记忆都长期保留在 Milvus 中，成本可能会逐步走高，因此更适合和冷层联用。

**结论**：

- 归档与全量沉淀：纯对象存储
- 长期在线可查但成本敏感：S3 Vectors
- 高频活跃记忆、在线工作集：Milvus

#### 2.2.3 多模态归档检索

图像、音频、视频、工业视觉和医疗影像场景中，embedding 数量通常远大于文本知识库。

#### 纯对象存储

如果主要诉求是保存原始媒体、特征文件和批处理结果，那么纯对象存储仍然是必须的，因为原始素材本来就要放在对象存储里。

但如果还要支持在线“相似图片查找”“相似视频片段检索”，光有对象存储不够。

#### S3 Vectors

对于多模态归档检索，S3 Vectors 的优势很明显：

- 与原始媒体对象处于同一存储体系
- 适合超大规模 embedding 长期保留
- 对“偶尔查、但必须查得到”的媒体资产管理尤其合适

它特别适合企业内部资产库、视频素材库、工业归档库，而不是以超低延迟交互搜索为核心的前台应用。

#### Milvus

如果是面向用户的高频多模态搜索，例如：

- 图搜图前台检索
- 视频推荐召回
- 广告素材在线匹配
- 电商图片实时相似搜索

Milvus 更合适，因为这类场景通常要求：

- 更低延迟
- 更高吞吐
- 更精细的索引和检索调优
- 更复杂的过滤与排序链路

**结论**：

- 媒体归档本身：纯对象存储
- 海量资产可查底座：S3 Vectors
- 高频在线多模态搜索：Milvus

#### 2.2.4 面向用户的在线语义搜索

比如站内搜索、商品搜索、内容推荐入口、问答搜索、广告召回等。

#### 纯对象存储

基本不适合作为在线检索系统。

#### S3 Vectors

可以承担部分成本敏感、查询频率不高的在线语义检索，但如果业务是核心用户路径，通常会遇到瓶颈：

- 更高 QPS 压力
- 更严格延迟目标
- 更复杂的召回和排序需求
- 更强的混合检索需求

#### Milvus

这是 Milvus 的主场。Milvus 的官方文档明确支持：

- dense retrieval
- sparse retrieval
- hybrid retrieval
- 多向量字段检索
- 标量过滤

这类能力决定了它更适合承载“用户真正感知到的搜索质量和交互性能”。

**结论**：

- 在线语义搜索优先选 Milvus
- S3 Vectors 更像后端向量底座，不是默认前台搜索引擎

#### 2.2.5 多租户 SaaS 检索

多租户 SaaS 面临的不只是查询本身，还包括：

- 租户隔离
- 权限边界
- 配额管理
- 成本归集
- 生命周期治理

#### 纯对象存储

如果只是归档和离线处理，可以按租户拆桶、拆前缀、拆目录。但一旦要在线检索，系统复杂度会迅速上升，因为检索层仍需单独实现。

#### S3 Vectors

S3 Vectors 在多租户隔离上很自然：

- 可以按 bucket 或 index 组织租户
- 可复用 IAM、bucket policy、KMS 和 AWS 网络治理体系
- 比较适合“每个租户有自己的知识空间，但流量并不极高”的 SaaS

#### Milvus

Milvus 同样适合多租户，但更偏向“平台工程”思路：

- 需要设计 collection / partition / namespace 策略
- 需要管理集群资源和 noisy neighbor 问题
- 需要对查询资源和索引成本做更精细治理

如果 SaaS 的搜索是核心竞争力，Milvus 很有价值；如果 SaaS 更关心长期成本和治理一致性，S3 Vectors 会更轻。

#### 2.2.6 向量冷热分层

这是三者最适合一起出现的场景。

#### 纯对象存储

适合做最底层原始归档：

- 原始对象
- embedding 导出文件
- 回放数据
- 索引快照

#### S3 Vectors

适合做温层或冷层：

- 保留大部分可在线查询的历史 embedding
- 允许亚秒级检索
- 控制长期成本

#### Milvus

适合做热层：

- 承接热点数据
- 满足高 QPS 和更低延迟
- 支持更复杂检索策略

**结论**：

在大型 AI 平台里，最合理的设计往往不是三选一，而是：

1. 纯对象存储保存原始数据和离线产物
2. S3 Vectors 保存海量长期可查 embedding
3. Milvus 负责热点和高性能在线查询

### 2.3 三种路线的能力矩阵

| 维度 | 纯对象存储 | S3 Vectors | Milvus |
|---|---|---|---|
| 产品本质 | 持久化介质 / 数据湖 | 对象存储原生向量层 | 向量数据库 / 检索引擎 |
| 原生 ANN 检索 | 无 | 有 | 有 |
| 标量 / 元数据过滤 | 需自建 | 有，能力相对基础 | 强，适合复杂业务过滤 |
| 混合检索 | 需自建 | 非主打 | 强，官方支持 dense / sparse / hybrid |
| 延迟定位 | 偏离线 | 亚秒级到更低 | 更适合毫秒级到低延迟在线查询 |
| 高 QPS | 不适合 | 非主打 | 主打 |
| 长期保存成本 | 最低 | 低 | 相对更高 |
| 运维复杂度 | 低 | 低 | 中到高 |
| 多租户治理 | 存储层容易，检索层难 | 较自然 | 可做，但需要平台治理 |
| 最适合的数据温度 | 冷 | 温 / 冷 | 热 / 温 |

### 2.4 为什么 Milvus 底层也是对象存储，但 S3 Vectors 仍然更低成本

这是一个非常关键的理解点。很多人第一次看到 Milvus 架构时会觉得：

> Milvus 底层也把数据放在对象存储里，那为什么 S3 Vectors 会更便宜？

答案是：

> **底层落在对象存储，并不等于整个系统的成本结构接近对象存储。**

两者的差异不在“最终数据写到哪里”，而在“为了完成查询，系统还要长期维持什么样的计算、缓存、索引和调度能力”。

#### 2.4.1 Milvus 的对象存储是持久层，不是全部成本

Milvus 的官方架构采用存算分离。对象存储在其中承担的是：

- 日志快照和历史数据持久化
- 向量索引与标量索引文件保存
- 中间结果和历史 segment 保存

但为了提供数据库级检索能力，Milvus 还需要：

- Query Node 承担在线查询
- Data Node 承担 compaction、flush、index build 等任务
- 协调组件管理调度与元数据
- 缓存层和内存层承接热点 segment 与索引
- WAL、元数据服务和集群资源保障高可用

换句话说，**对象存储只是 Milvus 的低成本持久层，真正让它“像数据库一样工作”的，是额外的引擎层和常驻资源层。**

#### 2.4.2 Milvus 的成本来自数据库能力，而不只是存储容量

Milvus 更贵，通常不是因为“对象存储更贵”，而是因为它提供了更强的能力，而这些能力需要更多资源：

- 更低延迟
- 更高 QPS
- 更复杂的标量过滤
- dense + sparse hybrid retrieval
- 多向量字段搜索
- 更细的索引和查询调优

这些能力的代价是：

- 查询节点常驻计算成本
- 热点数据与索引的内存占用
- 索引构建与 compaction 成本
- 集群扩缩容和高可用治理成本

也就是说，你付费买到的是 **“数据库能力”**，而不仅仅是“向量最终存在哪”。

#### 2.4.3 S3 Vectors 低成本的核心是少维护一整层数据库引擎

S3 Vectors 的产品定位更克制。它并不追求把自己做成一个高性能向量数据库，而是：

- 把向量存储和基础 ANN 查询下沉到 S3 服务层
- 优先优化海量向量的长期保留
- 优先服务低频或中频的语义查询
- 避免用户单独维护一套数据库级向量检索集群

因此，S3 Vectors 的低成本来自几个方面：

- 不需要用户单独维护常驻的查询节点和数据节点
- 不需要用户自己承担数据库级高可用和扩缩容复杂度
- 不以高 QPS、极低延迟和复杂混合检索为主要目标
- 能力面更聚焦，因此资源模型也更轻

这也是为什么 AWS 官方会明确把 S3 Vectors 定位在“低频查询优化”，并建议把更高 QPS、更复杂检索交给 OpenSearch 这类热层系统。

#### 2.4.4 本质区别：对象存储做持久层，和对象存储原生提供向量查询，不是一回事

可以把两者理解成两种完全不同的系统边界：

- **Milvus**：对象存储是数据库后端的一部分，数据库系统负责把对象存储上的数据变成低延迟可查询服务。
- **S3 Vectors**：对象存储服务本身直接提供向量存储与基础查询能力。

这两种方式都可以“把数据放在对象存储上”，但它们的成本结构完全不同：

- 前者为数据库性能和复杂能力买单
- 后者为存储原生能力和基础查询买单

#### 2.4.5 什么时候这种低成本优势最明显

S3 Vectors 的低成本优势并不是在所有场景下都绝对成立，而是在以下前提下最明显：

- 向量数量极大
- 需要长期保留
- 查询存在，但不是高频高并发
- 可以接受亚秒级而非极致毫秒级体验
- 业务更关注总拥有成本，而不是最强搜索功能

如果业务目标换成以下方向：

- 高频低延迟检索
- 复杂 hybrid retrieval
- 高并发在线搜索
- 搜索效果调优是核心竞争力

那么 Milvus 更高的成本往往是合理的，因为你买到的是更强的在线检索能力，而不是单纯存储。

### 2.5 这里提到的 OpenSearch 到底是什么

在本报告的语境中，提到的 OpenSearch 主要指的是：

- **OpenSearch**：开源搜索与分析引擎
- **Amazon OpenSearch Service**：AWS 托管版 OpenSearch
- **OpenSearch Serverless**：AWS 提供的无服务器形态，其中包含专门的 vector search collection

也就是说，这里并不是泛指“任意搜索系统”，而是在讨论 AWS 官方推荐的、与 S3 Vectors 配合使用的 **搜索热层 / 检索引擎层**。

#### 2.5.1 OpenSearch 的本质是搜索引擎，不只是向量库

OpenSearch 最初是典型的搜索与分析引擎，擅长：

- 全文检索
- 倒排索引
- 过滤
- 聚合
- 排序
- 实时搜索 API

后来 OpenSearch 增加了向量搜索能力，因此它现在既能做：

- 关键词搜索
- 向量相似搜索
- 关键词 + 向量混合搜索

AWS 官方文档中，OpenSearch Service 的向量搜索支持 `knn_vector` 字段、k-NN / approximate k-NN 查询以及多种距离度量。OpenSearch Serverless 的 vector search collection 还明确支持：

- full-text search
- advanced filtering
- aggregations
- geospatial queries
- nested queries

这说明 OpenSearch 的定位不是“只会搜向量”，而是 **把向量检索纳入搜索引擎能力体系**。

#### 2.5.2 为什么 S3 Vectors 讨论里会出现 OpenSearch

原因很简单：AWS 官方并不把 S3 Vectors 定位成一个全面的高性能搜索引擎。

S3 Vectors 更偏向：

- 低成本
- 超大规模
- 长期保存
- 基础向量查询
- 低频或中频检索

而 OpenSearch 更偏向：

- 更高吞吐
- 更低延迟
- 更复杂过滤
- 混合搜索
- 聚合和分析
- 更接近用户前台体验的搜索服务

因此在 AWS 推荐架构里，这两者通常不是替代关系，而是分层关系：

- **S3 Vectors** 承担低成本向量底座
- **OpenSearch** 承担高性能、复杂查询和搜索前台能力

#### 2.5.3 OpenSearch 在架构里一般扮演什么角色

在 S3 Vectors 相关架构中，OpenSearch 通常扮演 **热层搜索引擎** 或 **在线检索层**。

一种典型分层方式是：

1. 原始文件放在普通 S3
2. embedding 长期保存在 S3 Vectors
3. 热点数据或高价值数据导入 OpenSearch
4. 前台搜索、RAG 在线召回、复杂过滤和混合检索都走 OpenSearch

这种做法的好处是：

- 长期存储成本由 S3 Vectors 控制
- 前台搜索性能和能力由 OpenSearch 提供
- 形成冷温热分层，而不是让一个系统承担所有目标

#### 2.5.4 AWS 官方是如何把两者连起来的

AWS 官方文档提供了从 **Amazon S3 Vectors 导入 OpenSearch Serverless** 的方案，底层通过 **OpenSearch Ingestion** 管道把向量从 S3 vector index 导入到 OpenSearch Serverless 的 vector collection 中。

需要注意的一点是：

- 导入后，数据仍然保留在 S3 vector index 中
- 也就是说，OpenSearch 热层和 S3 Vectors 底层通常会同时存在

这恰恰反映了官方推荐的分工：

- S3 Vectors 负责长期、低成本、可查的向量底座
- OpenSearch 负责高性能和复杂检索

#### 2.5.5 OpenSearch 和 Milvus 的区别

虽然 OpenSearch 和 Milvus 都可以承担“热检索层”，但它们属于不同路线：

- **Milvus**：更纯粹的向量数据库路线，重点在向量检索能力本身
- **OpenSearch**：搜索引擎路线，重点是把向量检索和全文检索、过滤、聚合、搜索运营能力放在一起

如果业务更像：

- 站内搜索
- 商品搜索
- 内容搜索
- 关键词 + 向量混合检索
- 搜索分析和运营

那么 OpenSearch 往往更自然。

如果业务更像：

- 纯向量召回
- 多向量字段 ANN 检索
- 向量数据库能力优先
- 检索链路围绕 embedding 本身设计

那么 Milvus 更典型。

### 2.6 三者的优势与劣势

#### 2.6.1 纯对象存储

#### 优势

- 成本最低，弹性最好
- 最适合作为原始数据和 embedding 的长期归档层
- 适合离线处理、重建索引和模型回放
- 不绑定特定向量检索实现

#### 劣势

- 没有原生向量检索能力
- 在线搜索能力几乎全部要自建
- 元数据过滤、权限、召回质量都需要额外系统配合

#### 2.6.2 S3 Vectors

#### 优势

- 直接把向量检索能力做进对象存储
- 海量向量长期保留成本更友好
- AWS 治理和安全能力可直接复用
- 很适合作为 AI 平台的统一向量底座

#### 劣势

- 能力面比专业向量数据库窄
- 不以高 QPS、极低延迟在线检索为主要目标
- 更复杂的 hybrid retrieval 和搜索前台能力要靠外部系统补齐

#### 2.6.3 Milvus

#### 优势

- 具备数据库级向量检索能力
- 适合复杂过滤、dense + sparse 检索、多向量字段检索
- 官方架构支持存算分离和水平扩展
- 更适合作为线上搜索和召回系统的核心引擎

#### 劣势

- 部署、扩容、索引管理和稳定性治理复杂度更高
- 总体资源和长期持有成本通常高于对象存储型方案
- 如果只是承载大量低频冷向量，性价比不一定高

### 2.7 选型建议

如果只看一句话：

- **只想低成本保存 embedding**：选纯对象存储。
- **想让海量 embedding 低成本且原生可查**：选 S3 Vectors。
- **想把向量检索做成高性能在线服务**：选 Milvus。

如果从架构演进看，更常见的真实路径是：

1. 初期先用纯对象存储保存原始数据和 embedding
2. 随着在线语义检索需求增加，引入 S3 Vectors 作为统一向量层
3. 当流量、延迟、混合检索和搜索质量要求进一步提高，再引入 Milvus 作为热层或在线检索层

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

22. Milvus Hybrid Search with Milvus  
    https://milvus.io/docs/hybrid_search_with_milvus.md

23. Milvus Filtered Search  
    https://blog.milvus.io/docs/filtered-search.md

24. Milvus Tiered Storage Overview  
    https://milvus.io/docs/tiered-storage-overview.md

25. What is Amazon OpenSearch Service  
    https://docs.aws.amazon.com/opensearch-service/latest/developerguide/what-is.html

26. Vector search in Amazon OpenSearch Service  
    https://docs.aws.amazon.com/opensearch-service/latest/developerguide/vector-search.html

27. Working with vector search collections in OpenSearch Serverless  
    https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless-vector-search.html

28. Import from Amazon S3 Vectors to OpenSearch Serverless  
    https://docs.aws.amazon.com/opensearch-service/latest/developerguide/s3-opensearch-vector-bucket-integration.html
