# Apache Atlas Repository 模块架构分析

## 目录

1. [Repository 模块概览](#repository-模块概览)
2. [核心架构](#核心架构)
3. [主要技术栈](#主要技术栈)
4. [GraphBackedSearchIndexer 深度分析](#graphbackedsearchindexer-深度分析)
5. [关键子模块详解](#关键子模块详解)
6. [数据流与交互](#数据流与交互)

---

## 1 Repository 模块概览

### 模块定位

`atlas-repository` 是 Apache Atlas 的**核心数据持久化和检索模块**，负责：

- **元数据持久化**：将所有元数据对象存储到图数据库（JanusGraph）
- **全文搜索**：提供快速的元数据搜索和发现能力
- **向量搜索**：支持基于 Elasticsearch 8.x 的向量相似度搜索（*POC版本）
- **图索引管理**：创建和维护图数据库的各类索引
- **实体管理**：CRUD 操作、关系管理、分类管理
- **导入导出**：支持 ZIP 格式的大规模数据导入导出
- **审计追踪**：记录所有数据变更历史（基于 Cassandra）

### Maven 信息

```xml
<artifactId>atlas-repository</artifactId>
<packaging>jar</packaging>
<name>Apache Atlas Repository</name>
<description>Apache Atlas Repository Module</description>
```

### 包输出

生成 **JAR 文件**（包含所有元数据持久化逻辑），被 `atlas-webapp` 依赖

---

## 2 核心架构

### 分层架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                      Discovery Layer                             │
│  (EntityDiscoveryService, AtlasDiscoveryService)                 │
│  (全文搜索、DSL 查询、向量搜索)                                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌─────────────────────────┴────────────────────────────────────────┐
│                      Store Layer (v2)                             │
│  (AtlasEntityStoreV2, AtlasTypeDefGraphStoreV2,                  │
│   AtlasRelationshipStoreV2)                                       │
│  (实体存储、类型定义、关系管理)                                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌─────────────────────────┴────────────────────────────────────────┐
│                    Graph Layer                                    │
│  (GraphBackedSearchIndexer, GraphHelper,                          │
│   SearchIndexer, GraphTransactionAdvisor)                         │
│  (图索引、事务管理、搜索索引维护)                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌─────────────────────────┴────────────────────────────────────────┐
│                  Database Layer                                   │
│  (JanusGraph Backend)                                             │
│  (图数据库持久化、Cassandra/HBase/BerkeleyDB)                     │
└─────────────────────────────────────────────────────────────────┘
```

### 关键组件

| 组件 | 包位置 | 职责 |
|------|--------|------|
| **GraphBackedSearchIndexer** | `repository/graph/` | 🔑 核心：管理图的全局索引和类型索引 |
| **EntityDiscoveryService** | `discovery/` | 实体查询和发现 |
| **AtlasEntityStoreV2** | `store/graph/v2/` | 实体 CRUD 操作 |
| **AtlasTypeDefGraphStoreV2** | `store/graph/v2/` | 类型定义管理 |
| **ElasticsearchVectorSearchService** | `discovery/` | 向量搜索实现（新增） |
| **SearchIndexer** | `discovery/` | 搜索索引管理接口 |
| **EntityGraphMapper** | `store/graph/v2/` | 实体到图的映射 |
| **EntityGraphRetriever** | `store/graph/v2/` | 从图检索实体 |

---

## 3 主要技术栈

### 核心依赖

#### 1. 图数据库
```xml
<!-- JanusGraph（通过 graphdb 模块） -->
<!-- 支持 Cassandra/HBase/BerkeleyDB 作为后端存储 -->
<!-- Gremlin 3.x 用于图遍历查询 -->
```

#### 2. 搜索与索引（*POC版本）
```xml
<!-- Elasticsearch Java Client 8.11.0 -->
<groupId>co.elastic.clients</groupId>
<artifactId>elasticsearch-java</artifactId>
<version>8.11.0</version>

<!-- Jakarta JSON API 2.1.0 -->
<groupId>jakarta.json</groupId>
<artifactId>jakarta.json-api</artifactId>
<version>2.1.0</version>

<!-- Parsson 1.1.2 (Jakarta JSON 实现) -->
<groupId>org.eclipse.parsson</groupId>
<artifactId>parsson</artifactId>
<version>1.1.2</version>

<!-- Jackson (JSON 序列化) -->
<artifactId>jackson-databind</artifactId>
<version>${jackson.databind.version}</version>
```

#### 3. 数据存储
```xml
<!-- Cassandra Driver (审计日志存储) -->
<groupId>com.datastax.cassandra</groupId>
<artifactId>cassandra-driver-core</artifactId>

<!-- HBase Client (可选后端) -->
<groupId>org.apache.hbase</groupId>
<artifactId>hbase-client</artifactId>
<artifactId>hbase-server</artifactId>
```

#### 4. 查询处理
```xml
<!-- ANTLR 4.7 (DSL 查询解析) -->
<groupId>org.antlr</groupId>
<artifactId>antlr4-runtime</artifactId>
```

#### 5. 工具库
```xml
<!-- Apache Commons -->
<artifactId>commons-lang3</artifactId>
<artifactId>commons-codec</artifactId>

<!-- Google Guava -->
<!-- JodaTime (日期处理) -->

<!-- Spring Framework (AOP、事务) -->
<groupId>org.springframework</groupId>
<artifactId>spring-aop</artifactId>
```

---

## 4 GraphBackedSearchIndexer 深度分析

### 核心职责

GraphBackedSearchIndexer 是 Repository 模块最**关键的索引管理组件**，负责：

1. **全局索引初始化**：在系统启动时创建图的全局混合索引

2. **类型索引管理**：当新类型定义创建时自动为其属性创建索引

3. **索引维护**：监听类型定义变化，动态更新索引

4. **索引字段映射**：管理属性到索引字段名的映射关系

5. **事务管理**：提交/回滚索引操作

   ```
   职责 1: 全局索引初始化
     ├─ 创建 atlas_vertex_index
     ├─ 创建 atlas_edge_index
     ├─ 创建 atlas_fulltext_index
     └─ 创建 25+ 个全局属性索引
   
   职责 2: 类型定义监听
     ├─ 监听 TypeDefChangeListener
     ├─ 新增类型 → 创建属性索引
     ├─ 修改类型 → 更新索引
     └─ 删除类型 → 清理索引
   
   职责 3: PropertyKey 管理
     ├─ 创建 PropertyKey
     ├─ 设置唯一约束
     ├─ 添加到混合索引
     └─ 配置排序选项
   
   职责 4: 索引字段映射
     ├─ 属性名 → 索引字段名映射
     ├─ 缓存映射关系到 TypeRegistry
     ├─ 支持查询时快速查找
     └─ 支持多个索引后端
   
   职责 5: 高可用支持
     ├─ HA 节点状态处理
     ├─ 主节点初始化索引
     ├─ 从节点无需初始化
     └─ 保证索引一致性
   ```

   

### 类定义

```java
@Component
@Order(1)
public class GraphBackedSearchIndexer implements SearchIndexer, 
                                                 ActiveStateChangeHandler, 
                                                 TypeDefChangeListener
```

**关键接口**：
- `SearchIndexer`：索引管理接口
- `ActiveStateChangeHandler`：HA（高可用）事件处理
- `TypeDefChangeListener`：类型定义变化监听

### 核心成员变量

```java
// 类型注册表
private final AtlasTypeRegistry typeRegistry;

// 索引变化监听器（如 SolrIndexHelper）
private final List<IndexChangeListener> indexChangeListeners = new ArrayList<>();

// 图提供者
private IAtlasGraphProvider provider;

// 索引键缓存
private boolean recomputeIndexedKeys = true;
private boolean recomputeEdgeIndexedKeys = true;
private Set<String> vertexIndexKeys = new HashSet<>();
private Set<String> edgeIndexKeys = new HashSet<>();
```

### 关键方法详解

#### 初始化方法：`initialize()`

**职责**：在系统启动时创建全局索引

```java
private void initialize(AtlasGraph graph) throws RepositoryException, IndexException
```

**创建的索引类型**：

| 索引名 | 类型 | 用途 |
|--------|------|------|
| `atlas_vertex_index` | Vertex Mixed Index | 顶点全局搜索 |
| `atlas_edge_index` | Edge Mixed Index | 边的属性搜索 |
| `atlas_fulltext_index` | Full Text Index | 全文搜索 |

**创建的顶点索引属性** (共 25+ 个)：

| 属性名 | 类型 | 唯一性 | 目的 |
|--------|------|--------|------|
| `__guid` | String | 全局唯一 | 实体全局唯一标识符 |
| `__typeName` | String | 全局唯一 | 实体类型（如 DataSet） |
| `__state` | String | 无 | 实体状态（ACTIVE/DELETED） |
| `__timestamp` | Long | 无 | 创建时间 |
| `__modificationTimestamp` | Long | 无 | 修改时间 |
| `__classification` | String | 无 | 分类名称（全文可搜索） |
| `__propagated_classificationNames` | String | 无 | 继承的分类 |
| `__traitNames` | String (Set) | 无 | 性状名称 |
| `__labels` | String | 无 | 标签集合 |
| `__customAttributes` | String | 无 | 自定义属性（JSON） |
| 以及任何 Entity/BusinessMetadata 的自定义属性 |

**示例：创建 GUID 索引**

```java
createCommonVertexIndex(management, GUID_PROPERTY_KEY,      // 属性名
                       UniqueKind.GLOBAL_UNIQUE,            // 唯一性约束
                       String.class,                         // 属性类型
                       SINGLE,                               // 基数（单值）
                       true,                                 // 可索引
                       false);                               // 不排序
```

#### 类型变化监听：`onChange()`

**职责**：监听类型定义创建/修改/删除，动态更新索引结构和属性

```java
@Override
public void onChange(ChangedTypeDefs changedTypeDefs) throws AtlasBaseException
```

**处理流程**：

```
ChangedTypeDefs (包含 created/updated/deleted)
    ↓
遍历 created 类型 → updateIndexForTypeDef()
    ↓
遍历 updated 类型 → updateIndexForTypeDef()
    ↓
遍历 deleted 类型 → deleteIndexForType()
    ↓
resolveIndexFieldNames() → 解析索引字段名映射
    ↓
createEdgeLabels() → 创建关系边标签
    ↓
commit() → 提交事务
```

**例如**：当定义新类型 `MyDataSet` 时

```java
// 1. 扫描 MyDataSet 的所有属性
// 2. 对每个字符串属性创建索引
// 3. 记录属性 → 索引字段名的映射
// 4. 创建关系边标签（如果有结构体属性）
```

#### HA 事件处理

```java
@Override
public void instanceIsActive() throws AtlasException
    // 实例变为主节点时初始化索引

@Override
public void instanceIsPassive()
    // 实例变为从节点时，无需操作
```

#### 索引字段名解析：`resolveIndexFieldNames()`

**目的**：建立属性 → 实际索引字段名的映射

```
Property Name: "myEntity.myAttribute"
         ↓
Query Index: Which index field name to use?
         ↓
Atlas 维护映射表，供检索时使用
```

### 核心工作流程

#### 流程 1：系统启动

```
Server Startup
    ↓
Spring 初始化 Bean
    ↓
@Inject GraphBackedSearchIndexer()
    ↓
addIndexListener(new SolrIndexHelper()) // 添加 Solr 索引监听器
    ↓
if (!HAConfiguration.isHAEnabled())
    initialize(graph)  // 单节点或主节点初始化
    ↓
notifyInitializationStart()  // 通知监听器初始化开始
```

#### 流程 2：新建类型定义

```
新建 EntityDef (如 "MyTable")
    ↓
TypeDefChangeListener.onChange()
    ↓
updateIndexForTypeDef(MyTable)
    ↓
addIndexForType(management, MyTable)
    ↓
遍历 MyTable 的所有属性
    ↓
对索引适用的属性创建索引
    ↓
resolveIndexFieldNames() 建立映射
    ↓
commit()
    ↓
notifyChangeListeners() 通知 Solr 等
```

#### 流程 3：查询执行

```
User Query: "find all tables with name = 'test'"
    ↓
EntityDiscoveryService 接收查询
    ↓
通过 GraphBackedSearchIndexer.getVertexIndexKeys()
获取可用索引字段
    ↓
构建 Gremlin 遍历查询
    ↓
在 "atlas_vertex_index" 上执行搜索
    ↓
返回匹配结果
```

### 索引类型详解

#### ① Vertex Mixed Index（顶点混合索引）

**特点**：
- 支持数值、字符串、地理位置等多种类型
- 自动支持全文搜索
- 支持范围查询、模糊查询

**应用属性**：
- `__guid`（全局唯一）
- `__typeName`
- 所有实体自定义属性

#### ② Edge Mixed Index（边混合索引）

**特点**：

- 对关系（边）的属性进行索引
- 加速关系的查询

**应用属性**：

- `relationship_guid`
- `relationship_type`
- 自定义关系属性

#### ③ Full Text Index（全文索引）

**特点**：

- 支持关键词搜索
- 支持中文分词（如配置了分词器）
- 用于模糊搜索

**应用属性**：
- `entity_text` (所有实体文本属性的汇总)

#### ④ 索引创建细节：createCommonVertexIndex()

```java
private void createCommonVertexIndex(AtlasGraphManagement management,
                                     String propertyName,
                                     UniqueKind uniqueKind,
                                     Class propertyType,
                                     AtlasCardinality cardinality,
                                     boolean indexable,
                                     boolean sortable) {
    // 1. 创建或获取 PropertyKey
    PropertyKey propertyKey = getOrCreatePropertyKey(management, propertyName, 
                                                      propertyType, cardinality);
    
    // 2. 根据 uniqueKind 设置唯一约束
    if (uniqueKind == UniqueKind.GLOBAL_UNIQUE) {
        management.setPropertyKeyUnique(propertyKey, true);
    }
    
    // 3. 将 PropertyKey 添加到 Vertex 混合索引
    if (indexable) {
        management.addGraphIndex(VERTEX_INDEX, propertyKey, ...);
    }
    
    // 4. 如果需要排序，添加排序配置
    if (sortable) {
        management.addIndexSort(propertyKey, ...);
    }
}
```

### 与其他组件的交互

#### 与 EntityDiscoveryService 的交互（查询）

```
EntityDiscoveryService
    ↓
调用 getVertexIndexKeys() 获取可用的索引属性
    ↓
构建查询时知道哪些属性有索引
    ↓
优化查询性能
```

#### 与 AtlasEntityStoreV2 的交互（数据写入）

```
新增实体时：
    AtlasEntityStoreV2.create()
    ↓
    写入图数据
    ↓
    自动被 Vertex 索引捕获
    ↓
    可立即被全文搜索查到

删除实体时：
    标记为 DELETED 状态
    ↓
    __state 索引自动更新
    ↓
    查询会排除已删除实体
```

#### 与 SolrIndexHelper 的交互

```
GraphBackedSearchIndexer.addIndexListener(new SolrIndexHelper())
    ↓
当类型定义变化时：
    GraphBackedSearchIndexer 通知 SolrIndexHelper
    ↓
    SolrIndexHelper 同步更新 Solr 索引
    ↓
    保持多个搜索后端的索引一致性
```

---

## 5 关键子模块详解

### discovery 包

**职责**：实体发现和搜索

| 类名 | 职责 |
|------|------|
| `EntityDiscoveryService` | 实现 AtlasDiscoveryService 接口，执行各类型查询 |
| `SearchIndexer` | 索引管理接口 |
| `ElasticsearchVectorSearchService` | **新增**：向量搜索实现（*POC版本） |
| `SearchProcessor` | 搜索处理链 |
| `EntitySearchProcessor` | 实体属性搜索 |
| `FullTextSearchProcessor` | 全文搜索 |
| `FreeTextSearchProcessor` | 自由文本搜索 |
| `ClassificationSearchProcessor` | 分类搜索 |

### store/graph/v2 包

**职责**：实体和关系的存储与检索（V2 版本）

| 类名 | 职责 |
|------|------|
| `AtlasEntityStoreV2` | 实体的 CRUD |
| `AtlasTypeDefGraphStoreV2` | 类型定义的存储 |
| `AtlasRelationshipStoreV2` | 关系的 CRUD |
| `EntityGraphMapper` | 实体对象 ↔ 图顶点 映射 |
| `EntityGraphRetriever` | 从图检索实体对象 |
| `AtlasGraphUtilsV2` | 图操作工具类 |

### impexp 包

**职责**：导入/导出功能

| 类名 | 职责 |
|------|------|
| `ExportService` | 导出元数据到 ZIP |
| `ImportService` | 从 ZIP 导入元数据 |
| `ZipSource` | ZIP 源读取 |
| `ZipSink` | ZIP 目标写入 |
| `VertexExtractor` | 提取顶点用于导出 |

### query 包

**职责**：DSL 查询处理

| 类名 | 职责 |
|------|------|
| `DSLQueryExecutor` | DSL 查询执行器 |
| `GremlinQueryComposer` | 组织 Gremlin 查询 |
| `TraversalBasedExecutor` | 基于遍历的执行引擎 |
| `ScriptEngineBasedExecutor` | 基于脚本引擎的执行 |

### audit 包

**职责**：审计日志

| 类名 | 职责 |
|------|------|
| `EntityAuditListener` | 审计事件监听 |
| `CassandraBasedAuditRepository` | Cassandra 存储实现 |

---

## 6 数据流与交互

### 数据流 1：创建实体

```
REST API: POST /entities
    ↓
AtlasEntityStoreV2.create(AtlasEntity)
    ↓
EntityGraphMapper.mapEntity() 
    → 创建顶点 + 边
    ↓
GraphBackedSearchIndexer
    → 自动为顶点创建索引项
    ↓
EntityAuditListener
    → 记录审计日志到 Cassandra
    ↓
通知 EntityChangeListenerV2
    ↓
业务流程处理（分类传播、关系维护等）
```

### 数据流 2：搜索实体

```
REST API: POST /discovery/search
    ↓
EntityDiscoveryService.searchUsingBasicQuery()
    ↓
SearchProcessor 链：
    1. ClassificationSearchProcessor
    2. EntitySearchProcessor
    3. FullTextSearchProcessor
    4. FreeTextSearchProcessor
    ↓
GraphBackedSearchIndexer.getVertexIndexKeys()
    获取可用索引属性
    ↓
构建 Gremlin 遍历查询
    ↓
使用 GraphBackedSearchIndexer 维护的索引执行查询
    ↓
EntityGraphRetriever.toAtlasEntity()
    从图顶点构建 AtlasEntity 对象
    ↓
返回搜索结果
```

### 数据流 3：向量搜索 (新增) (*POC版本)

```
REST API: POST /discovery/search/vector
    ↓
ElasticsearchVectorSearchService.vectorSearch()
    ↓
调用 Elasticsearch 8.x 向量 API
    使用 kNN 算法搜索相似向量
    ↓
返回相似的实体列表
    ↓
可与传统搜索组合
```

### 数据流 4：类型定义更新

```
REST API: POST /types/typedefs
    ↓
AtlasTypeDefGraphStoreV2.create()
    ↓
更新 TypeRegistry
    ↓
触发 TypeDefChangeListener.onChange()
    ↓
GraphBackedSearchIndexer.onChange()
    ↓
updateIndexForTypeDef()
    为新属性创建索引
    ↓
resolveIndexFieldNames()
    ↓
notifyChangeListeners()
    通知 SolrIndexHelper 等同步更新
    ↓
commit()
```

---

## 索引性能特性

### 索引优化

#### 1. 唯一性约束

```
__guid (GLOBAL_UNIQUE)
    ↓
Guarantee: 每个实体的 GUID 全局唯一
    ↓
查询性能: O(1)
```

#### 2. 属性级索引

```
Indexed: __typeName, __classification, __labels, ...
    ↓
Unindexed: Boolean, BigDecimal, BigInteger (性能考虑)
    ↓
cardinality为 SET/LIST 的属性也不会索引（性能原因）
```

#### 3. 全文索引

```
entity_text 索引包含：
    - 所有字符串属性的文本
    - 分类名称
    - 自定义属性值
    ↓
支持: 模糊搜索、关键词搜索
```

### 查询优化

```
查询流程：
1. 检查属性是否有索引 (getVertexIndexKeys())
2. 如有索引 → 使用索引查询 (O(log n))
3. 无索引 → 全扫描 (O(n)) - 应该避免
```

---

