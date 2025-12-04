# Atlas 检索流程时序图及架构分析

## 一、概述

本文档详细说明 Apache Atlas 的检索流程，特别是请求如何经过 `repository`、`graphdb/api` 和 `graphdb/janus` 三层架构，以及如何替换 `graphdb/janus` 实现。

## 二、检索类型

Atlas 支持以下几种检索方式：

| 检索类型 | 描述 | 索引名 |
|---------|------|-------|
| **DSL 查询** | Domain Specific Language 查询 | VERTEX_INDEX |
| **全文检索 (FullText)** | 基于 Lucene 语法的全文搜索 | FULLTEXT_INDEX |
| **基础检索 (Basic)** | 基于实体属性的检索 | VERTEX_INDEX |
| **快速检索 (Quick)** | 简化的快速搜索 | VERTEX_INDEX + FULLTEXT_INDEX |
| **关系检索 (Relationship)** | 检索实体之间的关系 | EDGE_INDEX |

## 三、核心类说明

### 3.1 各层核心类职责

| 层级 | 类名 | 中文描述 | 核心职责 |
|------|------|---------|---------|
| **REST层** | `DiscoveryREST` | 检索REST接口 | 接收HTTP请求，参数校验，调用服务层 |
| **服务层** | `EntityDiscoveryService` | 实体检索服务 | 检索业务逻辑，结果组装，权限过滤 |
| **服务层** | `SearchContext` | 检索上下文 | 存储检索参数，管理处理器链 |
| **服务层** | `FullTextSearchProcessor` | 全文检索处理器 | 处理全文检索逻辑 |
| **服务层** | `EntitySearchProcessor` | 实体检索处理器 | 处理基于属性的实体检索 |
| **服务层** | `ClassificationSearchProcessor` | 分类检索处理器 | 处理标签/分类过滤 |
| **服务层** | `EntityGraphRetriever` | 实体图检索器 | 从图顶点读取实体属性 |
| **API层** | `AtlasGraph` | 图数据库接口 | 定义图操作和索引查询的抽象接口 |
| **API层** | `AtlasIndexQuery` | 索引查询接口 | 定义索引查询的抽象接口 |
| **实现层** | `AtlasJanusGraph` | JanusGraph实现 | AtlasGraph的JanusGraph具体实现 |
| **实现层** | `AtlasJanusIndexQuery` | Janus索引查询 | AtlasIndexQuery的JanusGraph实现 |
| **实现层** | `AtlasJanusGraphIndexClient` | Janus索引客户端 | 管理索引，提供聚合和建议词功能 |
| **后端** | `Solr6Index` | Solr索引 | Solr全文索引后端 |
| **后端** | `ElasticSearch7Index` | ES索引 | Elasticsearch全文索引后端 |

## 四、架构层次图

```plaintext
┌─────────────────────────────────────────────────────────────────────┐
│              第1层: REST API 层 (webapp 模块)                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ DiscoveryREST - 检索REST接口，处理HTTP请求                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│              第2层: 检索服务层 (repository 模块)                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ EntityDiscoveryService - 检索业务逻辑实现                     │   │
│  │ SearchContext - 检索上下文，管理处理器链                       │   │
│  │ SearchProcessor - 各类检索处理器(全文/实体/分类)              │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│              第3层: 图数据库API层 (graphdb/api 模块)                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ AtlasGraph - 图操作抽象接口                                   │   │
│  │ AtlasIndexQuery - 索引查询抽象接口                            │   │
│  │ AtlasGraphQuery - 图遍历查询抽象接口                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│              第4层: JanusGraph实现层 (graphdb/janus 模块)            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ AtlasJanusGraph - AtlasGraph的JanusGraph实现                 │   │
│  │ AtlasJanusIndexQuery - 索引查询的JanusGraph实现               │   │
│  │ AtlasJanusGraphIndexClient - 索引管理客户端                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│              第5层: 索引后端 (外部存储)                              │
│  ┌──────────────────────┐    ┌──────────────────────┐              │
│  │ Solr (Solr6Index)    │    │ Elasticsearch (ES7)  │              │
│  │ 全文检索引擎          │    │ 全文检索引擎          │              │
│  └──────────────────────┘    └──────────────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

## 五、PlantUML 时序图

### 5.1 全文检索 (FullText Search) 时序图

```plantuml
@startuml fulltext_search_sequence
!theme plain
skinparam backgroundColor #FEFEFE
skinparam sequenceMessageAlign center
skinparam noteFontSize 11
skinparam participantFontSize 12
skinparam sequenceArrowThickness 1.5
skinparam roundcorner 10
skinparam sequenceLifeLineBorderColor #555555

title <size:16>**Atlas 全文检索 (FullText Search) 时序图**</size>

actor "客户端\nClient" as client #LightBlue
box "第1层: REST API层 (webapp)" #E8F4FD
    participant "DiscoveryREST\n<size:9>检索REST接口</size>" as rest
end box
box "第2层: 服务层 (repository)" #E8FDE8
    participant "EntityDiscoveryService\n<size:9>实体检索服务</size>" as discovery
end box
box "第3层: API层 (graphdb/api)" #FFF8E8
    participant "AtlasGraph\n<size:9>图数据库接口</size>" as graphApi
end box
box "第4层: 实现层 (graphdb/janus)" #FDE8E8
    participant "AtlasJanusGraph\n<size:9>JanusGraph实现</size>" as janusGraph
    participant "AtlasJanusIndexQuery\n<size:9>Janus索引查询</size>" as janusIndexQuery
end box
box "第5层: 索引后端" #E8E8FD
    participant "Solr/ES\n<size:9>全文索引引擎</size>" as indexBackend
end box

== 步骤1: 发起全文检索请求 ==
client -> rest : <b>1.1</b> GET /api/atlas/v2/search/fulltext?query=xxx
note right: 【作用】客户端发送全文检索HTTP请求
activate rest #LightBlue

== 步骤2: 调用检索服务 ==
rest -> discovery : <b>2.1</b> searchUsingFullTextQuery(query, limit, offset)
note right: 【作用】REST层调用服务层执行检索
activate discovery #LightGreen

== 步骤3: 构建索引查询 ==
discovery -> discovery : <b>3.1</b> toAtlasIndexQuery(fullTextQuery)
note right
【作用】构建全文索引查询字符串
查询格式: v."__entityText":(<用户查询词>)
使用索引: FULLTEXT_INDEX (全文索引)
end note

== 步骤4: 获取索引查询对象 ==
discovery -> graphApi : <b>4.1</b> indexQuery(FULLTEXT_INDEX, graphQuery)
note right: 【作用】调用图API创建索引查询对象
activate graphApi #Yellow

graphApi -> janusGraph : <b>4.2</b> indexQuery(indexName, graphQuery)
note right: 【作用】委托给JanusGraph实现创建查询
activate janusGraph #Pink

janusGraph -> janusIndexQuery : <b>4.3</b> new AtlasJanusIndexQuery(graph, query)
note right: 【作用】创建Janus索引查询实例
activate janusIndexQuery #Pink

janusGraph --> graphApi : <b>4.4</b> 返回 AtlasIndexQuery
deactivate janusGraph

graphApi --> discovery : <b>4.5</b> 返回 AtlasIndexQuery
deactivate graphApi

== 步骤5: 执行索引查询 ==
discovery -> janusIndexQuery : <b>5.1</b> vertices()
note right: 【作用】执行顶点检索，获取匹配的顶点迭代器

janusIndexQuery -> indexBackend : <b>5.2</b> 执行Lucene/Solr/ES查询
note right: 【作用】向索引后端发送查询请求
activate indexBackend #LightBlue

indexBackend --> janusIndexQuery : <b>5.3</b> 返回匹配的顶点ID列表及得分
note right: 【作用】返回检索结果(顶点ID+相关性得分)
deactivate indexBackend

janusIndexQuery --> discovery : <b>5.4</b> Iterator<Result> 结果迭代器
note right: 【作用】返回包装后的结果迭代器
deactivate janusIndexQuery

== 步骤6: 处理检索结果 ==
discovery -> discovery : <b>6.1</b> getIndexQueryResults(idxQuery, params)
note right
【作用】处理检索结果
1. 遍历结果迭代器
2. 过滤已删除的实体
3. 读取顶点属性转换为AtlasEntityHeader
4. 记录每个结果的相关性得分
end note

== 步骤7: 返回检索结果 ==
discovery --> rest : <b>7.1</b> AtlasSearchResult (包含fullTextResult)
note right: 【作用】返回封装好的检索结果对象
deactivate discovery

rest --> client : <b>7.2</b> HTTP 200 + JSON响应
note right: 【作用】返回JSON格式的检索结果给客户端
deactivate rest

@enduml
```

### 5.2 基础检索 (Basic Search) 时序图

```plantuml
@startuml basic_search_sequence
!theme plain
skinparam backgroundColor #FEFEFE
skinparam sequenceMessageAlign center
skinparam noteFontSize 11
skinparam participantFontSize 12
skinparam sequenceArrowThickness 1.5
skinparam roundcorner 10
skinparam sequenceLifeLineBorderColor #555555

title <size:16>**Atlas 基础检索 (Basic Search) 时序图**</size>

actor "客户端\nClient" as client #LightBlue
box "第1层: REST API层 (webapp)" #E8F4FD
    participant "DiscoveryREST\n<size:9>检索REST接口</size>" as rest
end box
box "第2层: 服务层 (repository)" #E8FDE8
    participant "EntityDiscoveryService\n<size:9>实体检索服务</size>" as discovery
    participant "SearchContext\n<size:9>检索上下文</size>" as context
    participant "EntitySearchProcessor\n<size:9>实体检索处理器</size>" as entityProcessor
    participant "ClassificationSearchProcessor\n<size:9>分类检索处理器</size>" as classProcessor
end box
box "第3层: API层 (graphdb/api)" #FFF8E8
    participant "AtlasGraph\n<size:9>图数据库接口</size>" as graphApi
end box
box "第4层: 实现层 (graphdb/janus)" #FDE8E8
    participant "AtlasJanusGraph\n<size:9>JanusGraph实现</size>" as janusGraph
    participant "AtlasJanusIndexQuery\n<size:9>Janus索引查询</size>" as janusIndexQuery
end box
box "第5层: 索引后端" #E8E8FD
    participant "Solr/ES\n<size:9>全文索引引擎</size>" as indexBackend
end box

== 步骤1: 发起基础检索请求 ==
client -> rest : <b>1.1</b> POST /api/atlas/v2/search/basic (SearchParameters)
note right: 【作用】客户端发送基础检索请求，包含类型、属性过滤条件
activate rest #LightBlue

== 步骤2: 调用检索服务 ==
rest -> discovery : <b>2.1</b> searchWithParameters(searchParameters)
note right: 【作用】REST层调用服务层执行检索
activate discovery #LightGreen

== 步骤3: 创建检索上下文 ==
discovery -> context : <b>3.1</b> new SearchContext(typeRegistry, graph, searchParameters)
note right: 【作用】创建检索上下文，解析检索参数
activate context #LightGreen

context -> context : <b>3.2</b> 初始化 entityTypes, classificationTypes
note right: 【作用】解析要检索的实体类型和分类类型

context -> context : <b>3.3</b> 确定搜索处理器链
note right
【作用】根据检索条件构建处理器链
处理器执行顺序:
1. FullTextSearchProcessor (有query关键词时)
2. FreeTextSearchProcessor (自由文本检索)
3. EntitySearchProcessor (实体属性检索)
4. ClassificationSearchProcessor (分类过滤)
end note

context --> discovery : <b>3.4</b> 返回 searchContext
deactivate context

== 步骤4: 执行实体检索处理器 ==
discovery -> entityProcessor : <b>4.1</b> execute()
note right: 【作用】执行实体检索处理器
activate entityProcessor #LightGreen

entityProcessor -> entityProcessor : <b>4.2</b> 构建 indexQuery (VERTEX_INDEX)
note right
【作用】构建索引查询条件
查询格式示例:
- 类型过滤: __typeName:(hive_table OR hive_db)
- 属性过滤: qualifiedName:(mydb*)
- 状态过滤: __state:ACTIVE
end note

== 步骤5: 获取索引查询对象 ==
entityProcessor -> graphApi : <b>5.1</b> indexQuery(VERTEX_INDEX, indexQueryString)
note right: 【作用】调用图API创建顶点索引查询
activate graphApi #Yellow

graphApi -> janusGraph : <b>5.2</b> indexQuery(indexName, graphQuery, offset)
note right: 【作用】委托给JanusGraph实现
activate janusGraph #Pink

janusGraph -> janusIndexQuery : <b>5.3</b> new AtlasJanusIndexQuery(graph, query)
note right: 【作用】创建Janus索引查询实例
janusGraph --> graphApi : <b>5.4</b> 返回 AtlasIndexQuery
deactivate janusGraph

graphApi --> entityProcessor : <b>5.5</b> 返回 AtlasIndexQuery
deactivate graphApi

== 步骤6: 执行索引查询 ==
entityProcessor -> janusIndexQuery : <b>6.1</b> vertices(offset, limit, sortBy, sortOrder)
note right: 【作用】执行带排序的分页索引查询
activate janusIndexQuery #Pink

janusIndexQuery -> indexBackend : <b>6.2</b> 执行排序索引查询
note right: 【作用】向索引后端发送带排序的查询
activate indexBackend #LightBlue

indexBackend --> janusIndexQuery : <b>6.3</b> 返回匹配的顶点结果
note right: 【作用】返回按指定字段排序的结果
deactivate indexBackend

janusIndexQuery --> entityProcessor : <b>6.4</b> Iterator<Result> 结果迭代器
note right: 【作用】返回查询结果迭代器
deactivate janusIndexQuery

== 步骤7: 内存过滤 ==
entityProcessor -> entityProcessor : <b>7.1</b> 内存过滤 (inMemoryPredicate)
note right: 【作用】对结果进行内存中的二次过滤

entityProcessor -> entityProcessor : <b>7.2</b> 收集结果顶点
note right: 【作用】将过滤后的顶点收集到结果列表

== 步骤8: 分类过滤 (可选) ==
alt 有 Classification 过滤条件
    entityProcessor -> classProcessor : <b>8.1</b> filter(offsetEntityVertexMap)
    note right: 【作用】应用分类标签过滤
    activate classProcessor #LightGreen
    
    classProcessor -> classProcessor : <b>8.2</b> 应用分类过滤条件
    note right: 【作用】根据标签条件过滤实体
    
    classProcessor --> entityProcessor : <b>8.3</b> 返回过滤后的顶点
    deactivate classProcessor
end

entityProcessor --> discovery : <b>8.4</b> List<AtlasVertex> 结果顶点列表
note right: 【作用】返回最终过滤后的顶点列表
deactivate entityProcessor

== 步骤9: 转换结果 ==
discovery -> discovery : <b>9.1</b> EntityGraphRetriever.toAtlasEntityHeader(vertex)
note right
【作用】将图顶点转换为实体头信息
1. 读取顶点的GUID、名称、类型等属性
2. 读取分类标签信息
3. 构建AtlasEntityHeader对象
end note

== 步骤10: 返回检索结果 ==
discovery --> rest : <b>10.1</b> AtlasSearchResult (包含 entities列表)
note right: 【作用】返回封装的检索结果
deactivate discovery

rest --> client : <b>10.2</b> HTTP 200 + JSON响应
note right: 【作用】返回JSON格式结果给客户端
deactivate rest

@enduml
```

### 5.3 DSL 查询时序图

```plantuml
@startuml dsl_search_sequence
!theme plain
skinparam backgroundColor #FEFEFE
skinparam sequenceMessageAlign center
skinparam noteFontSize 11
skinparam participantFontSize 12
skinparam sequenceArrowThickness 1.5
skinparam roundcorner 10
skinparam sequenceLifeLineBorderColor #555555

title <size:16>**Atlas DSL 查询时序图**</size>

actor "客户端\nClient" as client #LightBlue
box "第1层: REST API层 (webapp)" #E8F4FD
    participant "DiscoveryREST\n<size:9>检索REST接口</size>" as rest
end box
box "第2层: 服务层 (repository)" #E8FDE8
    participant "EntityDiscoveryService\n<size:9>实体检索服务</size>" as discovery
    participant "DSLQueryExecutor\n<size:9>DSL查询执行器</size>" as dslExecutor
end box
box "第3层: API层 (graphdb/api)" #FFF8E8
    participant "AtlasGraph\n<size:9>图数据库接口</size>" as graphApi
end box
box "第4层: 实现层 (graphdb/janus)" #FDE8E8
    participant "AtlasJanusGraph\n<size:9>JanusGraph实现</size>" as janusGraph
    participant "GremlinScriptEngine\n<size:9>Gremlin脚本引擎</size>" as gremlin
end box
box "第5层: 图存储" #E8E8FD
    participant "JanusGraph\n<size:9>图数据库引擎</size>" as jg
end box

== 步骤1: 发起DSL查询请求 ==
client -> rest : <b>1.1</b> GET /api/atlas/v2/search/dsl?query=from hive_table
note right: 【作用】客户端发送DSL查询请求
activate rest #LightBlue

== 步骤2: 调用检索服务 ==
rest -> discovery : <b>2.1</b> searchUsingDslQuery(dslQuery, limit, offset)
note right: 【作用】REST层调用服务层执行DSL查询
activate discovery #LightGreen

== 步骤3: 执行DSL查询 ==
discovery -> dslExecutor : <b>3.1</b> execute(dslQuery, limit, offset)
note right: 【作用】调用DSL查询执行器
activate dslExecutor #LightGreen

dslExecutor -> dslExecutor : <b>3.2</b> DSL解析 & 转换为Gremlin
note right
【作用】将DSL语法转换为Gremlin图查询语言
DSL输入: from hive_table where name='test'
Gremlin输出: g.V().has('__typeName','hive_table')
              .has('name','test')
end note

== 步骤4: 获取Gremlin脚本引擎 ==
dslExecutor -> graphApi : <b>4.1</b> getGremlinScriptEngine()
note right: 【作用】获取Gremlin脚本执行引擎
activate graphApi #Yellow

graphApi -> janusGraph : <b>4.2</b> getGremlinScriptEngine()
note right: 【作用】从JanusGraph获取脚本引擎
activate janusGraph #Pink

janusGraph --> graphApi : <b>4.3</b> 返回 GremlinGroovyScriptEngine
note right: 【作用】返回Groovy脚本引擎实例
deactivate janusGraph

graphApi --> dslExecutor : <b>4.4</b> 返回 ScriptEngine
deactivate graphApi

== 步骤5: 执行Gremlin查询 ==
dslExecutor -> gremlin : <b>5.1</b> eval(gremlinQuery, bindings)
note right: 【作用】执行Gremlin查询脚本
activate gremlin #Pink

gremlin -> jg : <b>5.2</b> traversal().V()...
note right: 【作用】在JanusGraph上执行图遍历
activate jg #LightBlue

jg --> gremlin : <b>5.3</b> 返回图遍历结果
note right: 【作用】返回匹配的顶点/边/属性
deactivate jg

gremlin --> dslExecutor : <b>5.4</b> Object (顶点/边/值 集合)
note right: 【作用】返回Gremlin查询结果对象
deactivate gremlin

== 步骤6: 处理查询结果 ==
dslExecutor -> dslExecutor : <b>6.1</b> 转换结果为AtlasSearchResult
note right
【作用】将Gremlin结果转换为Atlas格式
1. 遍历结果顶点
2. 提取实体属性
3. 构建AtlasEntityHeader列表
end note

dslExecutor --> discovery : <b>6.2</b> AtlasSearchResult
note right: 【作用】返回DSL查询结果
deactivate dslExecutor

== 步骤7: 返回检索结果 ==
discovery --> rest : <b>7.1</b> AtlasSearchResult
note right: 【作用】返回封装的检索结果
deactivate discovery

rest --> client : <b>7.2</b> HTTP 200 + JSON响应
note right: 【作用】返回JSON格式结果给客户端
deactivate rest

@enduml
```

### 5.4 完整检索架构组件图

```plantuml
@startuml complete_search_overview
!theme plain
skinparam backgroundColor #FEFEFE
skinparam componentFontSize 11
skinparam packageFontSize 12
skinparam noteFontSize 10
skinparam roundcorner 10

title <size:16>**Atlas 检索架构组件图**</size>

package "第1层: REST API层\n(webapp模块)" as layer1 #E8F4FD {
    component "DiscoveryREST\n<size:9>检索REST接口</size>" as rest #LightBlue
    note right of rest
    【职责】提供HTTP检索端点
    端点列表:
    - /v2/search/dsl (DSL查询)
    - /v2/search/fulltext (全文检索)
    - /v2/search/basic (基础检索)
    - /v2/search/quick (快速检索)
    - /v2/search/relationship (关系检索)
    end note
}

package "第2层: 服务层\n(repository模块)" as layer2 #E8FDE8 {
    interface "AtlasDiscoveryService\n<size:9>检索服务接口</size>" as discoveryInterface
    component "EntityDiscoveryService\n<size:9>实体检索服务实现</size>" as discovery #LightGreen
    
    package "检索处理器组" as processors {
        component "FullTextSearchProcessor\n<size:9>全文检索处理器</size>" as ftProcessor #90EE90
        component "EntitySearchProcessor\n<size:9>实体属性检索处理器</size>" as entityProcessor #90EE90
        component "ClassificationSearchProcessor\n<size:9>分类标签检索处理器</size>" as classProcessor #90EE90
        component "FreeTextSearchProcessor\n<size:9>自由文本检索处理器</size>" as freeTextProcessor #90EE90
        component "RelationshipSearchProcessor\n<size:9>关系检索处理器</size>" as relProcessor #90EE90
    }
    
    component "SearchContext\n<size:9>检索上下文管理</size>" as context #98FB98
    component "GraphIndexQueryBuilder\n<size:9>图索引查询构建器</size>" as queryBuilder #98FB98
    component "EntityGraphRetriever\n<size:9>实体图数据读取器</size>" as retriever #98FB98
}

package "第3层: 图数据库API层\n(graphdb/api模块)" as layer3 #FFF8E8 {
    interface "AtlasGraph<V,E>\n<size:9>图操作抽象接口</size>" as atlasGraph #Yellow
    interface "AtlasIndexQuery<V,E>\n<size:9>索引查询抽象接口</size>" as atlasIndexQuery #Yellow
    interface "AtlasGraphQuery<V,E>\n<size:9>图遍历查询抽象接口</size>" as atlasGraphQuery #Yellow
    interface "AtlasGraphIndexClient\n<size:9>索引管理客户端接口</size>" as indexClient #Yellow
    
    note right of atlasGraph
    【职责】定义图数据库抽象接口
    核心方法:
    - indexQuery() 创建索引查询
    - query() 创建图遍历查询
    - getGraphIndexClient() 获取索引客户端
    end note
}

package "第4层: JanusGraph实现层\n(graphdb/janus模块)" as layer4 #FDE8E8 {
    component "AtlasJanusGraph\n<size:9>AtlasGraph的JanusGraph实现</size>" as janusGraph #Pink
    component "AtlasJanusIndexQuery\n<size:9>索引查询的JanusGraph实现</size>" as janusIndexQuery #Pink
    component "AtlasJanusGraphQuery\n<size:9>图遍历的JanusGraph实现</size>" as janusGraphQuery #Pink
    component "AtlasJanusGraphIndexClient\n<size:9>索引客户端的JanusGraph实现</size>" as janusIndexClient #Pink
    
    note right of janusGraph
    【职责】JanusGraph具体实现
    - 封装JanusGraph原生API
    - 管理图连接和事务
    - 委托索引查询给JanusGraphIndexQuery
    end note
}

package "第5层: 索引后端\n(外部存储)" as layer5 #E8E8FD {
    database "Solr\n(Solr6Index)\n<size:9>全文检索引擎</size>" as solr #LightBlue
    database "Elasticsearch\n(ES7Index)\n<size:9>全文检索引擎</size>" as es #LightBlue
    
    note bottom of solr
    【职责】提供全文检索能力
    支持功能:
    - 全文检索 (FullText)
    - 聚合统计 (Aggregation)
    - 自动建议 (Suggestion)
    end note
}

' ===== 连接关系 =====
' 第1层 -> 第2层
rest --> discoveryInterface : 调用

' 第2层内部
discoveryInterface <|.. discovery : 实现
discovery --> context : 创建上下文
discovery --> retriever : 读取实体

context --> ftProcessor : 构建处理器链
context --> entityProcessor
context --> classProcessor
context --> freeTextProcessor
context --> relProcessor

ftProcessor --> queryBuilder : 构建查询
entityProcessor --> queryBuilder

' 第2层 -> 第3层
queryBuilder --> atlasGraph : 调用接口
retriever --> atlasGraph : 调用接口

' 第3层 -> 第4层 (接口实现)
atlasGraph <|.. janusGraph : 实现
atlasIndexQuery <|.. janusIndexQuery : 实现
atlasGraphQuery <|.. janusGraphQuery : 实现
indexClient <|.. janusIndexClient : 实现

' 第4层内部
janusGraph --> janusIndexQuery : 创建
janusGraph --> janusGraphQuery : 创建
janusGraph --> janusIndexClient : 创建

' 第4层 -> 第5层
janusIndexQuery --> solr : 查询
janusIndexQuery --> es : 查询
janusIndexClient --> solr : 管理
janusIndexClient --> es : 管理

@enduml
```

## 六、关键类与接口说明

### 6.1 GraphDB API 层 (graphdb/api)

| 接口/类 | 说明 |
|--------|------|
| `AtlasGraph<V,E>` | 图数据库核心接口，定义图操作和索引查询方法 |
| `AtlasIndexQuery<V,E>` | 索引查询接口，支持顶点/边查询、分页、排序 |
| `AtlasGraphQuery<V,E>` | 图遍历查询接口 |
| `AtlasGraphIndexClient` | 索引管理客户端接口（聚合、建议词等） |
| `GraphIndexQueryParameters` | 索引查询参数封装类 |

### 6.2 GraphDB Janus 层 (graphdb/janus)

| 类 | 说明 |
|----|------|
| `AtlasJanusGraph` | `AtlasGraph` 的 JanusGraph 实现 |
| `AtlasJanusIndexQuery` | `AtlasIndexQuery` 的 JanusGraph 实现 |
| `AtlasJanusGraphQuery` | `AtlasGraphQuery` 的 JanusGraph 实现 |
| `AtlasJanusGraphIndexClient` | 索引客户端实现，支持 Solr/ES |
| `AtlasJanusGraphDatabase` | JanusGraph 数据库初始化和管理 |

### 6.3 索引类型

| 索引名 | 用途 | 配置 |
|-------|------|------|
| `VERTEX_INDEX` | 顶点属性索引，支持 Basic Search | `vertex_index` |
| `FULLTEXT_INDEX` | 全文索引，支持 FullText Search | `fulltext_index` |
| `EDGE_INDEX` | 边属性索引，支持 Relationship Search | `edge_index` |

## 七、替换 graphdb/janus 的影响分析

### 7.1 检索功能是否需要同步修改？

**答案：需要修改，但修改范围可控。**

由于 Atlas 采用了良好的分层架构设计，`graphdb/api` 定义了标准接口，`graphdb/janus` 只是其中一个实现。如果要替换 JanusGraph：

#### 7.1.1 需要实现的接口

```java
// 必须实现的核心接口
public class AtlasYourGraph implements AtlasGraph<YourVertex, YourEdge> {
    // 索引查询 - 检索核心
    AtlasIndexQuery<V, E> indexQuery(String indexName, String queryString);
    AtlasIndexQuery<V, E> indexQuery(String indexName, String queryString, int offset);
    AtlasIndexQuery<V, E> indexQuery(GraphIndexQueryParameters indexQueryParameters);
    
    // 图查询
    AtlasGraphQuery<V, E> query();
    
    // 索引客户端
    AtlasGraphIndexClient getGraphIndexClient();
    
    // Gremlin 支持
    GremlinGroovyScriptEngine getGremlinScriptEngine();
    Object executeGremlinScript(String query, boolean isPath);
}

// 索引查询实现
public class AtlasYourIndexQuery implements AtlasIndexQuery<YourVertex, YourEdge> {
    Iterator<Result<V, E>> vertices();
    Iterator<Result<V, E>> vertices(int offset, int limit);
    Iterator<Result<V, E>> vertices(int offset, int limit, String sortBy, Order sortOrder);
    Long vertexTotals();
    
    // 边查询
    Iterator<Result<V, E>> edges();
    Iterator<Result<V, E>> edges(int offset, int limit);
}

// 索引客户端实现
public class AtlasYourGraphIndexClient implements AtlasGraphIndexClient {
    // 聚合查询
    Map<String, List<AtlasAggregationEntry>> getAggregatedMetrics(AggregationContext ctx);
    
    // 建议词
    List<String> getSuggestions(String prefixString, String indexFieldName);
    
    // 搜索权重
    void applySearchWeight(String collectionName, Map<String, Integer> weights);
}
```

### 7.2 修改步骤

#### 步骤 1：创建新的 graphdb 模块

```plaintext
graphdb/
├── api/                          # 保持不变
├── janus/                        # JanusGraph 实现（可作为参考）
└── your-impl/                    # 新的图数据库实现
    ├── pom.xml
    └── src/main/java/org/apache/atlas/repository/graphdb/yourimpl/
        ├── AtlasYourGraph.java
        ├── AtlasYourGraphDatabase.java
        ├── AtlasYourIndexQuery.java
        ├── AtlasYourGraphQuery.java
        ├── AtlasYourGraphIndexClient.java
        ├── AtlasYourVertex.java
        ├── AtlasYourEdge.java
        └── GraphDbObjectFactory.java
```

#### 步骤 2：实现索引查询 (核心)

索引查询是检索功能的核心，需要特别注意：

```java
public class AtlasYourIndexQuery implements AtlasIndexQuery<AtlasYourVertex, AtlasYourEdge> {
    private final AtlasYourGraph graph;
    private final String indexName;
    private final String queryString;
    
    @Override
    public Iterator<Result<AtlasYourVertex, AtlasYourEdge>> vertices(int offset, int limit) {
        // 1. 将 queryString 转换为您的图数据库的查询格式
        // 2. 执行索引查询
        // 3. 将结果转换为 AtlasIndexQuery.Result
        
        // 示例：如果使用 Elasticsearch
        SearchRequest request = buildSearchRequest(indexName, queryString, offset, limit);
        SearchResponse response = esClient.search(request);
        
        return transformToResults(response.getHits());
    }
    
    @Override
    public Long vertexTotals() {
        // 返回总匹配数（用于分页）
        return executeCountQuery();
    }
}
```

#### 步骤 3：配置依赖注入

修改 `atlas-application.properties`：

```properties
# 图数据库实现配置
atlas.graph.storage.backend=your-backend
atlas.graph.index.search.backend=elasticsearch  # 或其他索引后端

# 类加载配置
atlas.graphdb.impl=org.apache.atlas.repository.graphdb.yourimpl.AtlasYourGraphDatabase
```

#### 步骤 4：确保索引兼容

Atlas 使用三种索引，您的实现需要支持：

| 索引 | 查询语法示例 |
|-----|-------------|
| VERTEX_INDEX | `v."__typeName":(hive_table) AND v."__state":(ACTIVE)` |
| FULLTEXT_INDEX | `v."__entityText":(search keywords)` |
| EDGE_INDEX | `e."__typeName":(relationship_type)` |

### 7.3 repository 层是否需要修改？

**答案：通常不需要修改。**

`repository` 层（包括 `EntityDiscoveryService`、`SearchProcessor` 等）只依赖 `graphdb/api` 接口，不直接依赖 `graphdb/janus`。只要新的实现正确实现了 API 接口，repository 层可以无缝使用。

```plaintext
┌─────────────────┐
│   repository    │  ──── 依赖 ────→  ┌──────────────┐
│ (不需要修改)     │                   │  graphdb/api │
└─────────────────┘                   └──────────────┘
                                              ↑
                                              │ 实现
                                    ┌─────────┴─────────┐
                                    │                   │
                              ┌─────────────┐    ┌─────────────┐
                              │graphdb/janus│    │graphdb/your │
                              │  (旧实现)    │    │  (新实现)    │
                              └─────────────┘    └─────────────┘
```

### 7.4 注意事项

1. **查询语法兼容性**：Atlas 使用类似 Lucene 的查询语法，新的图数据库需要能够解析或转换这种语法

2. **事务支持**：Atlas 使用 `@GraphTransaction` 注解管理事务，新实现需要支持

3. **索引管理**：需要实现 `AtlasGraphManagement` 接口来管理索引的创建和更新

4. **Gremlin 兼容**：DSL 查询依赖 Gremlin，如果新的图数据库不支持 Gremlin，需要修改 DSL 执行器

## 八、总结

| 组件 | 替换影响 | 修改复杂度 |
|------|---------|-----------|
| `graphdb/api` | 无需修改 | - |
| `repository` | 无需修改（如果 API 实现正确） | 低 |
| `graphdb/janus` → 新实现 | 需要完全重新实现 | 高 |
| 索引后端 | 可以复用或更换 | 中 |

Atlas 的分层设计使得替换底层图数据库成为可能，但需要完整实现 `graphdb/api` 中定义的所有接口，特别是与检索相关的 `AtlasIndexQuery` 和 `AtlasGraphIndexClient`。
