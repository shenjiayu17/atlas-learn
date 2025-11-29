# GraphBackedSearchIndexer 索引创建机制详解

## 目录

1. [索引创建的完整流程](#索引创建的完整流程)
2. [PropertyKey 的生命周期](#propertykey-的生命周期)
3. [混合索引详解](#混合索引详解)
4. [索引映射机制](#索引映射机制)
5. [常见问题与解决方案](#常见问题与解决方案)

---

## 1 索引创建的完整流程

### 阶段 1：系统初始化

当 Atlas 服务器启动时：

```
1. Spring DI 容器初始化
   ↓
2. @Inject GraphBackedSearchIndexer(AtlasTypeRegistry typeRegistry)
   ↓
3. Constructor → addIndexListener(new SolrIndexHelper(typeRegistry))
   ↓
4. if (!HAConfiguration.isHAEnabled())
       initialize(provider.get());
   ↓
5. notifyInitializationStart();
```

### 阶段 2：GraphBackedSearchIndexer初始化，initialize()详解

```java
private void initialize(AtlasGraph graph) throws RepositoryException, IndexException {
    AtlasGraphManagement management = graph.getManagementSystem();
    
    try {
        // Step 1: 创建三个全局混合索引
        createVertexMixedIndex("atlas_vertex_index");
        createEdgeMixedIndex("atlas_edge_index");
        createFullTextMixedIndex("atlas_fulltext_index");
        
        // Step 2: 为全局属性创建索引 (25+ 个)
        createCommonVertexIndex(GUID_PROPERTY_KEY, ...);
        createCommonVertexIndex(TYPENAME_PROPERTY_KEY, ...);
        // ... 更多
        
        // Step 3: 提交事务
        commit(management);
        
    } catch (Throwable t) {
        rollback(management);
        throw new RepositoryException(t);
    }
}
```

### 具体的索引创建步骤

#### Step 1: 创建混合索引容器

```
Mixed Index: atlas_vertex_index
├── Backing Store: Elasticsearch/Solr/Lucene
├── Field 1: __guid (String, SINGLE)
├── Field 2: __typeName (String, SINGLE)
├── Field 3: __state (String, SINGLE)
├── ...
└── Field N: custom_attribute (String, SET)
```

**代码**：
```java
if (management.getGraphIndex(VERTEX_INDEX) == null) {
    management.createVertexMixedIndex(
        VERTEX_INDEX,          // "atlas_vertex_index"
        BACKING_INDEX,         // "search" (配置文件中定义)
        Collections.emptyList()
    );
    LOG.info("Created index : {}", VERTEX_INDEX);
}
```

#### Step 2: 创建 PropertyKey

每个属性都需要 PropertyKey 定义：

```java
private void createCommonVertexIndex(
    AtlasGraphManagement management,
    String propertyName,        // e.g., "__guid"
    UniqueKind uniqueKind,      // GLOBAL_UNIQUE, NONE, PER_TYPE_UNIQUE
    Class propertyType,         // String.class, Long.class, etc.
    AtlasCardinality cardinality,  // SINGLE, SET, LIST
    boolean indexable,          // true = 可索引
    boolean sortable            // true = 可排序
) {
    // 获取或创建 PropertyKey
    AtlasPropertyKey propertyKey = 
        management.getPropertyKey(propertyName);
    
    if (propertyKey == null) {
        // 不存在，创建新的
        propertyKey = management.makePropertyKey(
            propertyName,
            propertyType,
            cardinality
        );
    }
    
    // 设置唯一约束
    if (uniqueKind == UniqueKind.GLOBAL_UNIQUE) {
        management.setPropertyKeyUnique(propertyKey, true);
        management.setPropertyKeyUnique(propertyKey, 
                                       UniqueKind.GLOBAL_UNIQUE);
    }
    
    // 添加到混合索引
    if (indexable) {
        management.addGraphIndex(
            VERTEX_INDEX,    // 索引名
            propertyKey,     // 要索引的属性
            Direction.BOTH
        );
    }
    
    // 如果需要排序
    if (sortable) {
        management.addGraphIndex(
            VERTEX_INDEX,
            propertyKey,
            Direction.BOTH
        );
        // 配置排序参数
    }
}
```

### 阶段 3：类型定义时的索引创建 onChange

当新建 EntityDef 时：

```java
@Override
public void onChange(ChangedTypeDefs changedTypeDefs) throws AtlasBaseException {
    AtlasGraphManagement management = provider.get().getManagementSystem();
    
    try {
        // 处理新创建的类型
        for (AtlasBaseTypeDef typeDef : changedTypeDefs.getCreatedTypeDefs()) {
            updateIndexForTypeDef(management, typeDef);
        }
        
        // 处理更新的类型
        for (AtlasBaseTypeDef typeDef : changedTypeDefs.getUpdatedTypeDefs()) {
            updateIndexForTypeDef(management, typeDef);
        }
        
        // 处理删除的类型
        for (AtlasBaseTypeDef typeDef : changedTypeDefs.getDeletedTypeDefs()) {
            deleteIndexForType(management, typeDef);
        }
        
        // 解析索引字段名映射
        resolveIndexFieldNames(management, changedTypeDefs);
        
        // 创建关系边标签
        createEdgeLabels(management, changedTypeDefs.getCreatedTypeDefs());
        
        // 提交
        commit(management);
        
        // 通知监听器
        notifyChangeListeners(changedTypeDefs);
        
    } catch (RepositoryException | IndexException e) {
        LOG.error("Failed to update indexes for changed typedefs", e);
        attemptRollback(changedTypeDefs, management);
    }
}
```

---

## 2 PropertyKey 的生命周期

```
┌─────────────────────────────────────────────────────┐
│                 PropertyKey                          │
│                                                      │
│ State 1: Not Registered                            │
│   - 不存在于管理系统                                 │
│   - 可以创建新的                                     │
│                                                      │
│ State 2: Created                                    │
│   - 通过 management.makePropertyKey() 创建          │
│   - 但尚未添加到任何索引                             │
│                                                      │
│ State 3: Indexed                                    │
│   - 通过 management.addGraphIndex() 添加到混合索引  │
│   - 对该属性的查询将使用索引                        │
│                                                      │
│ State 4: Unique (可选)                              │
│   - 通过 setPropertyKeyUnique() 设置                │
│   - 确保属性值全局唯一                              │
└─────────────────────────────────────────────────────┘
```

#### PropertyKey 生命周期

```
┌─ NOT_REGISTERED ─────────────────────────┐
│ 属性还不在管理系统中                       │
│ 可以创建新 PropertyKey                    │
└──────────┬────────────────────────────────┘
           │ management.makePropertyKey()
           ▼
┌─ CREATED ────────────────────────────────┐
│ PropertyKey 已创建但未索引                 │
│ 属性存在但查询不使用索引                   │
└──────────┬────────────────────────────────┘
           │ management.addGraphIndex()
           ▼
┌─ INDEXED ────────────────────────────────┐
│ PropertyKey 已添加到混合索引               │
│ 查询时会自动使用索引加速                   │
└──────────┬────────────────────────────────┘
           │ management.setPropertyKeyUnique()
           ▼
┌─ UNIQUE (可选) ─────────────────────────┐
│ PropertyKey 设置唯一约束                  │
│ 值必须全局唯一（如 __guid）              │
└──────────────────────────────────────────┘
```

#### 创建过程示例

**场景**: 创建新的 EntityDef "DataSet"

```java
public class DataSet {
    String name;        // 要索引
    String description; // 要索引
    Long createdAt;     // 要索引
    Boolean active;     // 不索引 (在 INDEX_EXCLUSION_CLASSES 中)
}
```

**执行流程**：

```
1. onChange() 触发
   ↓
2. updateIndexForTypeDef(management, DataSetDef)
   ↓
3. addIndexForType(management, DataSetDef)
   ├─ 获取 DataSet 类型实例
   ├─ 遍历所有属性 (name, description, createdAt, active)
   └─ 对每个属性调用 createPropertyIndex()
       ↓
       4a. PropertyKey "DataSet.name" 
           ├─ Type: String
           ├─ Cardinality: SINGLE
           ├─ isIndexApplicable?: YES (String, SINGLE)
           ├─ Create PropertyKey
           ├─ Add to VERTEX_INDEX
           └─ Resolve field name → "datasset_name_t"
           
       4b. PropertyKey "DataSet.description"
           ├─ Type: String
           ├─ Cardinality: SINGLE
           ├─ isIndexApplicable?: YES
           ├─ Create PropertyKey
           ├─ Add to VERTEX_INDEX
           └─ Resolve field name → "datasset_description_t"
           
       4c. PropertyKey "DataSet.createdAt"
           ├─ Type: Long
           ├─ Cardinality: SINGLE
           ├─ isIndexApplicable?: YES (Long, SINGLE)
           ├─ Create PropertyKey
           ├─ Add to VERTEX_INDEX
           └─ Resolve field name → "datasset_createdat_l"
           
       4d. PropertyKey "DataSet.active"
           ├─ Type: Boolean
           ├─ Cardinality: SINGLE
           ├─ isIndexApplicable?: NO (Boolean in INDEX_EXCLUSION_CLASSES)
           ├─ Create PropertyKey
           └─ DO NOT Add to VERTEX_INDEX
   
5. resolveIndexFieldNames()
   ├─ For each indexed property
   └─ Query backing store for actual field name
       ("datasset_name_t", "datasset_description_t", ...)
   
6. commit()
   └─ 提交事务，索引生效
```

---

## 3 混合索引详解

```
JanusGraph 的索引分两种：

1. Local Index (本地索引)
   - 存储在 JanusGraph 内
   - 快速，但只支持简单查询
   
2. Mixed Index (混合索引) ← GraphBackedSearchIndexer 使用
   - 存储在外部索引引擎 (Elasticsearch/Solr/Lucene)
   - 支持复杂查询、全文搜索、范围查询
   - 支持排序、分页
```

### Atlas 使用的三个混合索引

#### 1. **atlas_vertex_index** (顶点索引)

```
Purpose: 索引顶点的属性
Backing: Elasticsearch / Solr / Lucene
Type: Mixed Index

Field Mapping:
__guid         → __guid_t (String, unique)
__typeName     → __typename_t (String)
__state        → __state_t (String)
__timestamp    → __timestamp_l (Long)
__classification → __classification_t (String, full-text)
DataSet.name   → dataset_name_t (String)
DataSet.owner  → dataset_owner_t (String)
...

Usage:
SELECT * FROM vertices 
WHERE __typeName = 'DataSet' 
AND DataSet.name = 'my_data'
```

#### 2. **atlas_edge_index** (边索引)

```
Purpose: 索引关系(边)的属性
Backing: Elasticsearch / Solr / Lucene
Type: Mixed Index

Field Mapping:
relationship_guid  → relationship_guid_t (String)
relationship_type  → relationship_type_t (String)
...

Usage:
SELECT * FROM edges 
WHERE relationship_type = 'has_schema' 
AND relationship_guid = 'xxx'
```

#### 3. **atlas_fulltext_index** (全文索引)

```
Purpose: 支持全文搜索
Backing: Elasticsearch / Solr / Lucene
Type: Full Text Mixed Index

Field Mapping:
entity_text → content (analyzed for full-text search)

Usage:
SELECT * FROM vertices 
WHERE entity_text contains 'my table'
```

### 索引的查询流程

```
User: "Find all DataSets with name containing 'sales'"
    ↓
EntityDiscoveryService.searchUsingBasicQuery()
    ↓
GraphBackedSearchIndexer.getVertexIndexKeys()
    → 返回 {__guid, __typeName, __state, ..., dataset_name, ...}
    ↓
构建 Gremlin 查询：
    graph.traversal()
        .V()
        .has("__typeName", "DataSet")  // 使用 __typeName_t 索引
        .has("DataSet.name", Text.textContains("sales"))  // 使用全文索引
        .limit(10)
    ↓
JanusGraph 执行：
    1. 使用 __typeName_t 索引快速定位 DataSet 顶点
    2. 在这些顶点中使用 dataset_name_t 全文索引搜索
    3. 返回最多 10 个结果
    ↓
结果: [DataSet-1, DataSet-2, ...]
```

---

## 4 索引映射机制 resolveIndexFieldName

### 属性名 → 索引字段名的映射

为什么需要映射？

```
Java Property: __classification
    ↓
但在 Elasticsearch 中：
    字段名: __classification_t
    (suffix "_t" 表示 Text 类型，便于全文搜索)
    ↓
Atlas 需要维护这个映射，查询时才知道用哪个字段名
```

### 映射解析流程

```java
private void resolveIndexFieldName(
    AtlasGraphManagement managementSystem,
    AtlasAttribute attribute) {
    
    try {
        // 1. 检查是否已有映射
        if (attribute.getIndexFieldName() != null) {
            return;  // 已解析，跳过
        }
        
        // 2. 检查是否需要索引
        if (!isIndexApplicable(
            getPrimitiveClass(attribute.getTypeName()),
            toAtlasCardinality(attribute.getAttributeDef().getCardinality()))) {
            return;  // 不需要索引，跳过
        }
        
        // 3. 查询 backing store 获取实际字段名
        AtlasPropertyKey propertyKey = 
            managementSystem.getPropertyKey(attribute.getVertexPropertyName());
        
        if (propertyKey != null) {
            // 从 backing store (Elasticsearch) 查询
            String indexFieldName = managementSystem.getIndexFieldName(
                Constants.VERTEX_INDEX,
                propertyKey,
                isStringField(attribute.getIndexType())
            );
            
            // 4. 设置映射
            attribute.setIndexFieldName(indexFieldName);
            
            // 5. 缓存到 TypeRegistry
            typeRegistry.addIndexFieldName(
                attribute.getVertexPropertyName(),
                indexFieldName
            );
            
            LOG.info("Property {} is mapped to index field name {}",
                    attribute.getQualifiedName(),
                    attribute.getIndexFieldName());
        }
    } catch (Exception e) {
        LOG.error("Error resolving index field name", e);
    }
}
```

### 映射缓存结构

```
TypeRegistry
├── Index Field Name Mappings
│   ├── "__guid" → "__guid_t"
│   ├── "__typeName" → "__typename_t"
│   ├── "__classification" → "__classification_t"
│   ├── "DataSet.name" → "dataset_name_t"
│   ├── "DataSet.owner" → "dataset_owner_t"
│   ├── "Table.tableName" → "table_tablename_t"
│   └── ...
└── Type Definitions
```

## 5 索引适用性判断 isIndexApplicable

```java
private boolean isIndexApplicable(Class propertyClass, 
                                  AtlasCardinality cardinality) {
    // 排除条件 1: 属性类型在黑名单中
    if (INDEX_EXCLUSION_CLASSES.contains(propertyClass)) {
        return false;  // Boolean, BigDecimal, BigInteger
    }
    
    // 排除条件 2: 基数为多值 (SET/LIST)
    if (cardinality.isMany()) {
        return false;  // SET, LIST
    }
    
    // 其他情况都可以索引
    return true;
}
```

### 索引适用矩阵

| 属性类型 | 基数 | 可索引？ | 原因 |
|---------|------|--------|------|
| String | SINGLE | ✅ YES | 最常见的索引类型 |
| String | SET | ❌ NO | 多值性能问题 |
| String | LIST | ❌ NO | 多值性能问题 |
| Long | SINGLE | ✅ YES | 支持范围查询 |
| Long | SET | ❌ NO | 多值性能问题 |
| Boolean | SINGLE | ❌ NO | 在黑名单中 |
| BigDecimal | SINGLE | ❌ NO | 在黑名单中 |
| Double | SINGLE | ✅ YES | 浮点数支持 |
| Integer | SINGLE | ✅ YES | 整数支持 |
| Date | SINGLE | ✅ YES | 支持范围查询 |

---

## 6 常见问题与解决方案

### Q1: 为什么 Boolean 属性不索引？

**原因**：

```
Boolean 属性的值只有 true/false
索引的价值不大（高基数性能更好）
节省索引空间、减少索引维护成本
```

**解决方案**：
```
如果确实需要搜索 Boolean：
使用 String ("true"/"false") 代替
或在查询时遍历所有顶点（接受较慢性能）
```

### Q2: 为什么 SET/LIST 类型属性不索引？

**原因**：
```
多值属性在索引中导致：
1. 索引膨胀 (每个值创建一个索引条目)
2. 查询复杂性增加 (需要特殊的集合查询)
3. 性能下降
```

**解决方案**：
```
Option 1: 改用 SINGLE 基数属性
Option 2: 在应用层面处理集合查询
Option 3: 接受全扫描性能
```

### Q3: 新增属性后为什么搜索不到？

**问题流程**：
```
1. 定义新 EntityDef 添加属性
2. 系统创建了 PropertyKey 和索引
3. 但 resolveIndexFieldName() 可能失败
4. 索引字段名映射未建立
5. 查询时找不到字段名
```

**解决方案**：
```
1. 检查日志，确认 resolveIndexFieldName() 是否执行
2. 检查属性是否真的满足 isIndexApplicable() 条件
3. 手动重新初始化索引：
   DELETE /atlas/v2/entity/guid/{guid}
   (重新索引)
```

### Q4: 索引修改后需要重新索引吗？

**答案**：有时需要

```
场景 1: 添加新属性
    → 自动创建索引
    → 新增实体自动被索引
    → ✅ 不需要重新索引

场景 2: 修改属性类型
    → PropertyKey 修改
    → 现有数据的索引可能不匹配
    → ❌ 需要重新索引

场景 3: 删除属性
    → PropertyKey 保留（不删除）
    → 索引项保留
    → ⚠️ 考虑清理索引
```

---

## 性能优化建议

### 1. 优化索引属性选择

```
DON'T:
✗ 为所有属性创建索引
✗ 索引高基数属性 (如ID序列)
✗ 索引几乎不搜索的属性

DO:
✓ 为常搜索的属性索引
✓ 为分类、状态等低基数属性索引
✓ 定期审查索引使用情况
```

### 2. 基数优化

```
可改进的设计：
class DataSet {
    @Cardinality(SET)
    String[] tags;  // ❌ 不索引
}

改进后：
class DataSet {
    String primaryTag;  // ✅ 索引主标签
    @Cardinality(SET)
    String[] secondaryTags;  // 应用层过滤
}
```

### 3. 索引字段监控

```
定期检查：
1. 有多少个索引字段
2. 哪些索引未被使用
3. 索引大小增长趋势
4. 查询性能是否下降

使用 management.getGraphIndex() 查询
```

---

## 总结

### GraphBackedSearchIndexer 的核心职责链

```
System Startup
    ↓
initialize() 创建全局索引 (atlas_vertex_index, atlas_edge_index, atlas_fulltext_index)
    ↓
创建 25+ 个全局 PropertyKey 和索引
    ↓
监听 TypeDefChangeListener.onChange()
    ↓
New Type Created
    ↓
addIndexForType() 为属性创建 PropertyKey
    ↓
isIndexApplicable() 决定是否索引
    ↓
resolveIndexFieldNames() 建立属性 → 字段名映射
    ↓
commit() 提交事务
    ↓
notifyChangeListeners() 通知其他组件
    ↓
查询时使用建立的索引加速搜索
```

---

