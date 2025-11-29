# Repository 模块

## 1 核心组件

### GraphBackedSearchIndexer
```
位置: repository/graph/GraphBackedSearchIndexer.java
职责: 图数据库的索引创建和维护
核心方法:
  • initialize() - 创建全局索引
  • onChange(ChangedTypeDefs) - 监听类型变化
  • getVertexIndexKeys() - 获取可用索引
```

### EntityDiscoveryService
```
位置: discovery/EntityDiscoveryService.java (1196 lines)
职责: 实体查询和发现
核心方法:
  • searchUsingBasicQuery() - 基本查询
  • searchUsingFullTextQuery() - 全文搜索
  • searchWithParameters() - 参数化搜索
```

### AtlasEntityStoreV2
```
位置: store/graph/v2/AtlasEntityStoreV2.java (1749 lines)
职责: 实体 CRUD 操作
核心方法:
  • create() - 创建实体
  • update() - 更新实体
  • deleteById() - 删除实体
```

### ElasticsearchVectorSearchService
```
位置: discovery/ElasticsearchVectorSearchService.java
职责: 向量相似度搜索 (新增)
核心方法:
  • vectorSearch() - 执行向量搜索
  • buildAndExecuteSearch() - 构建ES查询
```

## 2 架构分层

```
┌─ Discovery Layer ─────────────────────────┐
│ 查询和发现: EntityDiscoveryService         │
│ 向量搜索: ElasticsearchVectorSearchService │
└──────────┬───────────────────────────────┘
           │
┌─ Store Layer ──────────────────────────────┐
│ 实体存储: AtlasEntityStoreV2               │
│ 类型管理: AtlasTypeDefGraphStoreV2        │
│ 关系管理: AtlasRelationshipStoreV2        │
└──────────┬───────────────────────────────┘
           │
┌─ Graph Layer ──────────────────────────────┐
│ 索引管理: GraphBackedSearchIndexer ◄──🔑   │
│ 事务管理: GraphTransactionAdvisor         │
│ 辅助工具: GraphHelper                      │
└──────────┬───────────────────────────────┘
           │
┌─ Database Layer ───────────────────────────┐
│ JanusGraph + Cassandra/HBase/BerkeleyDB   │
└────────────────────────────────────────────┘
```

### 

## 2 package结构详解

```
repository/
├── discovery/               // 发现和搜索服务
│   ├── EntityDiscoveryService          (查询服务)
│   ├── AtlasDiscoveryService           (接口)
│   ├── ElasticsearchVectorSearchService (向量搜索)
│   ├── ElasticsearchClientFactory      (ES客户端)
│   ├── SearchIndexer                   (索引接口)
│   └── SearchProcessor.*               (搜索处理器)
│
├── graph/                  // 图数据库索引管理
│   ├── GraphBackedSearchIndexer        (🔑 核心索引管理)
│   ├── GraphHelper                     (图工具)
│   ├── GraphTransactionAdvisor         (AOP 事务)
│   ├── SolrIndexHelper                 (Solr同步)
│   └── FullTextMapperV2                (全文索引)
│
├── store/graph/
│   ├── AtlasEntityStore.java           (接口)
│   ├── AtlasTypeDefGraphStore.java     (接口)
│   └── v2/                             (V2 实现)
│       ├── AtlasEntityStoreV2          (实体存储)
│       ├── AtlasTypeDefGraphStoreV2    (类型定义)
│       ├── EntityGraphMapper           (Entity↔Vertex映射)
│       ├── EntityGraphRetriever        (检索实体)
│       └── AtlasGraphUtilsV2           (工具)
│
├── impexp/                 // 导入/导出
│   ├── ExportService
│   ├── ImportService
│   ├── ZipSource / ZipSink
│   └── VertexExtractor
│
├── audit/                  // 审计日志
│   ├── EntityAuditListener
│   └── CassandraBasedAuditRepository
│
├── query/                  // 查询处理
│   ├── DSLQueryExecutor
│   ├── GremlinQueryComposer
│   ├── TraversalBasedExecutor
│   └── ScriptEngineBasedExecutor
│
└── ...
```

---

## 3 索引类型

| 索引名 | 类型 | 用途 | 创建时机 |
|--------|------|------|---------|
| `atlas_vertex_index` | Mixed | 顶点属性搜索 | 系统启动 |
| `atlas_edge_index` | Mixed | 边属性搜索 | 系统启动 |
| `atlas_fulltext_index` | FullText | 全文搜索 | 系统启动 |
| 类型特定索引 | Mixed | 实体属性搜索 | 定义类型时 |

---

## 4 全局索引属性 (initialize() 创建)

```
系统级属性 (所有实体都有)
├── __guid              → String, GLOBAL_UNIQUE
├── __typeName          → String, GLOBAL_UNIQUE
├── __state             → String (ACTIVE/DELETED)
├── __timestamp         → Long (创建时间)
├── __modificationTimestamp → Long (修改时间)
├── __createdBy         → String
├── __modifiedBy        → String
├── __classification    → String (分类名)
├── __traitNames        → String SET
├── __labels            → String
├── __customAttributes  → String (JSON)
└── ... (25+ 个)

类型特定属性 (定义类型时添加)
├── DataSet.name        → String
├── Table.tableName     → String
├── Column.columnName   → String
└── ... (用户自定义)
```

---

## 5 关键数据流

### 创建实体流程
```
REST POST /entities
  ↓ AtlasEntityStoreV2.create()
  ↓ EntityGraphMapper.mapEntity()
  ↓ 自动索引 (GraphBackedSearchIndexer)
  ↓ 审计记录 (EntityAuditListener)
  ↓ 返回 EntityMutationResponse
```

### 搜索实体流程
```
REST POST /discovery/search
  ↓ EntityDiscoveryService.searchUsingBasicQuery()
  ↓ GraphBackedSearchIndexer.getVertexIndexKeys()
  ↓ 构建 Gremlin 遍历
  ↓ 使用索引执行查询
  ↓ EntityGraphRetriever.toAtlasEntity()
  ↓ 返回 AtlasSearchResult
```

### 向量搜索流程
```
REST POST /discovery/search/vector
  ↓ ElasticsearchVectorSearchService.vectorSearch()
  ↓ ElasticsearchClient (ES 8.x)
  ↓ kNN 向量搜索
  ↓ 返回相似度结果
```

---



## 6 常用查询模式

### Basic Query
```java
searchUsingBasicQuery(
    String query,           // 搜索关键词
    String type,            // Entity type
    String classification,  // 分类名
    String attrName,        // 属性名
    String attrValuePrefix, // 属性值前缀
    boolean excludeDeleted, // 排除已删除
    int limit,
    int offset
)
```

### DSL Query
```java
searchUsingDslQuery(
    String query,  // DSL 查询字符串
    int limit,
    int offset
)
// 例: "DataSet where name='test'"
```

### Full Text Query
```java
searchUsingFullTextQuery(
    String query,  // 全文关键词
    boolean excludeDeleted,
    int limit,
    int offset
)
// 例: "test" → 搜索所有包含 "test" 的实体
```

### Vector Search (新增)
```java
vectorSearch(
    String keyword,        // 搜索词
    String indexName,      // ES index
    int limit,
    int offset
)
// 执行向量相似度搜索
```



