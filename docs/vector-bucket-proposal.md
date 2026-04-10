# JuiceFS 向量存储桶 (Vector Storage Bucket) 调研与方案设计

## 一、向量存储桶 (Vector Storage Bucket) 详解

### 1.1 什么是向量存储桶

向量存储桶是 AWS 于 2025 年推出的 S3 新型桶类型 (Amazon S3 Vectors)，专为存储和查询**向量嵌入 (Vector Embeddings)** 而设计。它是第一个在云对象存储中原生支持向量操作的服务。

**向量嵌入**是将非结构化数据（文本、图片、音频等）通过 AI 模型转换成的高维浮点数组（如 OpenAI 的 text-embedding-3-small 输出 1536 维 float32 数组）。语义相似的内容在向量空间中距离更近。

### 1.2 核心概念

| 概念 | 说明 |
|------|------|
| **Vector Bucket** | 一种新的桶类型（与通用桶分开），专为向量数据优化 |
| **Vector Index** | 桶内的向量索引，指定固定维度和距离度量。每桶最多 10,000 个索引 |
| **Vector** | 每个向量有：key（字符串 ID）、data（float32 数组）、metadata（用于过滤的键值对） |
| **距离度量** | 余弦相似度 (cosine) 或 欧氏距离 (euclidean) |
| **ANN 查询** | 近似最近邻搜索，返回与查询向量最相似的 top-K 结果 |

### 1.3 API 操作

S3 Vectors 提供独立于常规 S3 的 REST API（JSON 请求/响应体）：

- **桶管理**: `CreateVectorBucket`, `DeleteVectorBucket`, `ListVectorBuckets`, `GetVectorBucket`
- **索引管理**: `CreateIndex`, `DeleteIndex`, `GetIndex`, `ListIndexes`
- **向量操作**: `PutVectors`（批量最多 500 条）, `GetVectors`, `DeleteVectors`, `ListVectors`
- **查询**: `QueryVectors` — 近似最近邻搜索，支持元数据过滤
- **策略/标签**: `PutVectorBucketPolicy`, `GetVectorBucketPolicy`, `TagResource`, `UntagResource`

### 1.4 关键能力

- 每个索引最多 **20 亿**向量，每桶最多 10,000 个索引
- 写入后**强一致性**
- 低频查询亚秒延迟，高频查询低至 **100ms**
- 自动优化存储，无需手动管理基础设施
- 对比专用向量数据库，存储成本**降低 90%**
- Go SDK: `github.com/aws/aws-sdk-go-v2/service/s3vectors`

### 1.5 典型使用场景

- **RAG (检索增强生成)**: 将文档切片后生成向量，查询时用用户问题的向量找到最相关文档片段
- **语义搜索**: 基于含义而非关键词匹配搜索
- **推荐系统**: 找到与用户行为向量最相似的商品/内容
- **AI Agent 记忆**: 存储对话历史的向量表示，检索相关记忆
- **图像/音频相似搜索**: 多模态嵌入的相似度检索

---

## 二、JuiceFS 支持向量存储桶的方案

### 当前架构关键点

- S3 网关 (`pkg/gateway/gateway.go`) 实现了 MinIO 的 `ObjectLayer` 接口，MinIO 控制 HTTP 路由
- 元数据引擎 (`pkg/meta/`) 支持 Redis、SQL (MySQL/PG/SQLite)、TiKV/BadgerDB/etcd/FoundationDB
- 对象存储 (`pkg/object/`) 采用 `Register(name, creator)` 的插件模式，60+ 后端
- 元数据以 xattr 存储 S3 标签/ETag 等扩展信息

---

### 方案一：外部向量数据库代理（轻量级）

**思路**: JuiceFS 作为 S3 Vectors API 的翻译层，将向量操作转发给外部向量数据库（Milvus、Qdrant、pgvector 等）。

```
客户端 (AWS S3 Vectors SDK)
    │
    ▼
[新 HTTP Handler: pkg/gateway/vectors.go]
    │  解析 S3 Vectors REST API → 调用向量数据库客户端
    ▼
[外部向量数据库: Milvus / Qdrant / pgvector]
```

**需要修改的文件**:

- 新增 `pkg/gateway/vectors.go` — S3 Vectors REST API handler
- 新增 `pkg/gateway/vectors_backend.go` — `VectorBackend` 接口 + Milvus/Qdrant/pgvector 实现
- 修改 `cmd/gateway.go` — 添加 `--vector-backend`, `--vector-backend-addr` 等参数
- 轻微修改 `pkg/gateway/gateway.go` — 在 MinIO HTTP server 上挂载向量路由

**映射关系**: Vector Bucket → 向量数据库中的 collection/namespace，Vector Index → 向量数据库中的 index

**优点**:

- 实现复杂度最低
- 从第一天就有生产级 ANN 搜索（HNSW、IVF 等成熟算法）
- 向量数据库可独立扩展
- 不影响 JuiceFS 现有数据通路

**缺点**:

- 需要额外部署和运维向量数据库
- JuiceFS 与向量数据库之间无事务一致性
- 未利用 JuiceFS 的分布式存储能力

**复杂度**: ★★☆☆☆

---

### 方案二：基于现有元数据引擎（中等复杂度）

**思路**: 将向量数据直接存储在 JuiceFS 已有的元数据引擎中。利用 pgvector 扩展（PostgreSQL）实现 ANN 搜索，其他引擎提供暴力搜索。

```
客户端 (AWS S3 Vectors SDK)
    │
    ▼
[新 HTTP Handler: pkg/gateway/vectors.go]
    │
    ▼
[新 VectorMeta 接口: pkg/meta/vectors.go]
    │
    ▼
[现有元数据引擎: Redis / PostgreSQL+pgvector / MySQL / TiKV]
```

**需要修改的文件**:

- 新增 `pkg/meta/vectors.go` — `VectorMeta` 接口定义
- 新增 `pkg/meta/sql_vectors.go` — SQL 实现（新增 `vector_bucket`, `vector_index`, `vector_data` 表）
- 新增 `pkg/meta/redis_vectors.go` — Redis 实现（使用 RediSearch 向量搜索或暴力扫描）
- 新增 `pkg/meta/tkv_vectors.go` — TKV 实现
- 新增 `pkg/gateway/vectors.go` — HTTP handler

**ANN 搜索实现**:

| 后端 | 搜索方式 | 适用规模 |
|------|----------|----------|
| PostgreSQL + pgvector | 原生 HNSW/IVFFlat 索引 | 数百万向量 |
| Redis + RediSearch | 原生向量相似搜索 | 数十万向量 |
| MySQL / SQLite / TiKV | 暴力扫描 + Go 计算距离 | < 10 万向量 |

**优点**:

- 零额外依赖，复用用户已有的元数据引擎
- 与 JuiceFS 架构一致
- PostgreSQL + pgvector 路径提供生产级搜索
- 事务一致性有保证

**缺点**:

- 元数据引擎不是为向量负载设计的，大量高维向量会占用大量内存/存储
- 暴力搜索 O(n) 对大数据集不现实
- 不同后端性能差异巨大（pgvector 优秀，MySQL 暴力搜索很差）
- 无法支持 S3 Vectors 规格的 20 亿向量/索引

**复杂度**: ★★★☆☆

---

### 方案三：内嵌向量索引引擎（高性能自包含）

**思路**: 在 JuiceFS 网关进程中嵌入向量索引库（纯 Go HNSW 或 FAISS CGo 绑定）。向量数据持久化到 JuiceFS 文件系统，ANN 索引加载到内存。

```
客户端 (AWS S3 Vectors SDK)
    │
    ▼
[新 HTTP Handler: pkg/gateway/vectors.go]
    │
    ▼
[向量索引管理器: pkg/vector/manager.go]
    │
    ├──→ [内存 ANN 索引: HNSW 图 (pkg/vector/index.go)]
    │         │ 持久化/加载
    │         ▼
    │     [JuiceFS 文件系统: 索引文件]
    │
    └──→ [向量数据存储: pkg/vector/store.go]
              │
              ▼
          [JuiceFS Chunk Store → 对象存储]
```

**存储布局**:

```
/.vectors/{bucket}/{index}/
    config.json          # 索引配置（维度、度量）
    index.bin            # 序列化的 HNSW 图
    vectors/
        page_0000.bin    # 向量数据分页存储
    metadata/
        meta_0000.json   # 向量元数据分页存储
```

**ANN 搜索**: HNSW (Hierarchical Navigable Small World) 图在内存中，查询延迟极低。可选 FAISS CGo 绑定或纯 Go 实现。

**多网关协调**: 选举一个网关为写入者（通过元数据引擎锁），其他网关定期加载最新持久化索引。或使用 WAL（预写日志）方式。

**优点**:

- 高性能 ANN 搜索，低延迟
- 完全自包含，无外部依赖
- 向量数据利用 JuiceFS 的缓存/压缩/分布式存储
- 部署简单

**缺点**:

- 内存密集：HNSW 索引须在内存中（100 万 1536 维向量 ≈ 6-8GB）
- 多网关协调复杂且易出错
- FAISS CGo 依赖使交叉编译复杂化
- 网关重启时索引加载可能很慢
- 难以扩展到 20 亿向量（需要分片）

**复杂度**: ★★★★☆

---

### 方案四：完整 S3 Vectors API 兼容层 + 可插拔后端（推荐）

**思路**: 综合前三个方案的优势，构建完整的 S3 Vectors API 兼容层，后端引擎采用可插拔架构。

```
客户端 (AWS SDK s3vectors / 任何 S3 Vectors 客户端)
    │
    ▼
[S3 Vectors HTTP Server: pkg/s3vectors/server.go]
    │  完整 REST API，复用 MinIO SigV4 认证
    ▼
[S3 Vectors Service: pkg/s3vectors/service.go]
    │
    ├──→ [目录服务: pkg/s3vectors/catalog.go]
    │         │  管理 bucket/index 注册
    │         ▼
    │     [JuiceFS Meta Engine (现有)]
    │
    └──→ [向量引擎接口: pkg/s3vectors/engine.go]
              │
              ├──→ [内嵌 HNSW: engine_hnsw.go]       # 零依赖
              ├──→ [外部 Milvus: engine_milvus.go]    # 生产级
              └──→ [pgvector: engine_pgvector.go]     # 复用 PG 元数据
```

**核心接口**:

```go
type VectorEngine interface {
    CreateIndex(ctx context.Context, config IndexConfig) error
    DeleteIndex(ctx context.Context, bucket, index string) error
    PutVectors(ctx context.Context, bucket, index string, vectors []Vector) error
    GetVectors(ctx context.Context, bucket, index string, keys []string) ([]Vector, error)
    DeleteVectors(ctx context.Context, bucket, index string, keys []string) error
    QueryVectors(ctx context.Context, bucket, index string, query []float32, topK int, filter *FilterExpr) ([]QueryResult, error)
    ListVectors(ctx context.Context, bucket, index string, cursor string, limit int) ([]VectorSummary, string, error)
}
```

**API 路由**:

- `POST /vectorBuckets` — CreateVectorBucket
- `DELETE /vectorBuckets/{name}` — DeleteVectorBucket
- `GET /vectorBuckets` — ListVectorBuckets
- `POST /vectorBuckets/{name}/indexes` — CreateIndex
- `DELETE /vectorBuckets/{name}/indexes/{idx}` — DeleteIndex
- `GET /vectorBuckets/{name}/indexes/{idx}` — GetIndex
- `GET /vectorBuckets/{name}/indexes` — ListIndexes
- `POST /vectorBuckets/{name}/indexes/{idx}/vectors/put` — PutVectors
- `POST /vectorBuckets/{name}/indexes/{idx}/vectors/get` — GetVectors
- `POST /vectorBuckets/{name}/indexes/{idx}/vectors/delete` — DeleteVectors
- `POST /vectorBuckets/{name}/indexes/{idx}/vectors/query` — QueryVectors
- `GET /vectorBuckets/{name}/indexes/{idx}/vectors` — ListVectors

**可选启动方式**:

1. 嵌入 S3 网关: `juicefs gateway --vectors --vectors-engine=hnsw ...`
2. 独立服务: `juicefs vectors --engine=milvus --engine-addr=... META-URL`

**优点**:

- 完全兼容 S3 Vectors API，客户端可直接用 AWS SDK
- 可插拔架构适应不同场景和规模
- 与现有代码隔离（新包 `pkg/s3vectors/`）
- 可渐进实现：先做一个引擎后端，逐步添加
- 遵循 JuiceFS 的插件注册模式（类似 `pkg/object/` 的 Register 模式）

**缺点**:

- 总体实现工作量最大
- 更多代码维护
- 过滤表达式解析增加复杂度

**复杂度**: ★★★★★

---

## 三、方案对比总结

| | 方案一：外部代理 | 方案二：元数据引擎 | 方案三：内嵌索引 | 方案四：完整兼容层 |
|---|---|---|---|---|
| **复杂度** | 低 | 中 | 高 | 最高 |
| **外部依赖** | 需要向量数据库 | 无（pgvector 可选） | 无（FAISS 可选） | 可插拔 |
| **搜索性能** | 最优（专用 DB） | 取决于后端 | 优（HNSW 内存） | 取决于引擎 |
| **可扩展性** | 最优 | 有限 | 受内存限制 | 取决于引擎 |
| **部署简单度** | 中（需运维 DB） | 最简 | 简单 | 灵活 |
| **API 兼容性** | 可实现 | 可实现 | 可实现 | 最完整 |
| **推荐场景** | 快速上线 | 小规模试用 | 中等规模自包含 | 长期产品级 |

## 四、建议的渐进实施路径

1. **Phase 1** — 先实现方案四的 HTTP 骨架 + 类型定义 + 方案一的外部后端（Milvus 或 pgvector），快速获得可工作的 S3 Vectors API
2. **Phase 2** — 添加内嵌 HNSW 引擎（方案三），支持零外部依赖的用户
3. **Phase 3** — 添加元数据引擎原生后端（方案二），满足小规模简单部署需求
4. **Phase 4** — 完善过滤表达式、多网关协调、索引分片等高级功能

## 参考资料

- [Amazon S3 Vectors 文档](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors.html)
- [S3 Vectors API Reference](https://docs.aws.amazon.com/AmazonS3/latest/API/API_Operations_Amazon_S3_Vectors.html)
- [S3 Vectors GA 公告](https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-s3-vectors-generally-available/)
- [Go SDK s3vectors](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/s3vectors)
- [MinIO AIStor + Milvus 向量索引基准测试](https://www.min.io/blog/accelerating-vector-indexing-with-minio-aistor-milvus-and-nvidia-cuvs)
