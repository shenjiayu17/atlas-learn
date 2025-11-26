# Atlas 向量检索非侵入式扩展方案 v2

## 1. 扩展点分析

### 1.1 核心扩展接口

| 接口 | 用途 | 注册方式 |
|------|------|----------|
| `EntityChangeListenerV2` | 实体变更监听 | Spring Set 自动注入 |
| `TypeDefChangeListener` | 类型定义变更监听 | Spring Set 自动注入 |
| `AtlasDiscoveryService` | 检索服务接口 | @Primary 覆盖 |

### 1.2 非侵入式策略

- 新增独立模块 `addons/vector-search`
- 通过 Spring 自动装配注入监听器
- 新增独立 REST API 端点

---

## 2. 方案一：独立向量搜索模块（ES dense_vector）

### 2.1 模块结构

```
addons/vector-search/
├── pom.xml
└── src/main/java/org/apache/atlas/vector/
    ├── config/VectorSearchConfiguration.java
    ├── service/
    │   ├── EmbeddingService.java
    │   ├── VectorIndexService.java
    │   └── impl/
    │       ├── OpenAIEmbeddingService.java
    │       └── ElasticsearchVectorIndexService.java
    ├── listener/VectorIndexEntityListener.java
    ├── rest/VectorSearchREST.java
    └── model/
        ├── VectorSearchParameters.java
        └── VectorSearchResult.java
```

### 2.2 核心实现

#### 配置类

```java
@Component
public class VectorSearchConfiguration {
    public static final String VECTOR_SEARCH_ENABLED = "atlas.search.vector.enabled";
    public static final String ES_HOSTS = "atlas.search.vector.elasticsearch.hosts";
    public static final String EMBEDDING_API_KEY = "atlas.search.vector.embedding.api-key";
    public static final String VECTOR_ATTRIBUTES = "atlas.search.vector.attributes";
    
    // 格式: hive_table.description,hive_column.comment
    public String[] getVectorAttributes() {
        return configuration.getStringArray(VECTOR_ATTRIBUTES);
    }
}
```

#### 实体变更监听器（核心非侵入式扩展）

```java
@Component
public class VectorIndexEntityListener implements EntityChangeListenerV2 {
    
    @Inject
    public VectorIndexEntityListener(VectorSearchConfiguration config,
                                     VectorIndexService vectorIndexService,
                                     EmbeddingService embeddingService) {
        this.config = config;
        this.vectorIndexService = vectorIndexService;
        this.embeddingService = embeddingService;
    }
    
    @Override
    public void onEntitiesAdded(List<AtlasEntity> entities, boolean isImport) {
        if (!config.isEnabled()) return;
        
        for (AtlasEntity entity : entities) {
            String typeName = entity.getTypeName();
            Set<String> vectorAttrs = getVectorAttributes(typeName);
            
            for (String attrName : vectorAttrs) {
                Object attrValue = entity.getAttribute(attrName);
                if (attrValue != null) {
                    float[] vector = embeddingService.embed(attrValue.toString());
                    vectorIndexService.indexEntity(entity.getGuid(), typeName, attrName, vector);
                }
            }
        }
    }
    
    @Override
    public void onEntitiesDeleted(List<AtlasEntity> entities, boolean isImport) {
        for (AtlasEntity entity : entities) {
            vectorIndexService.deleteEntity(entity.getGuid());
        }
    }
}
```

#### ES 向量索引服务

```java
@Service
public class ElasticsearchVectorIndexService implements VectorIndexService {
    
    @Override
    public void initializeIndex(String typeName, int dimension) {
        // 创建 ES 索引映射
        String mapping = """
            {
                "mappings": {
                    "properties": {
                        "guid": {"type": "keyword"},
                        "typeName": {"type": "keyword"},
                        "vector": {
                            "type": "dense_vector",
                            "dims": %d,
                            "index": true,
                            "similarity": "cosine"
                        }
                    }
                }
            }
            """.formatted(dimension);
        
        client.indices().create(new CreateIndexRequest(indexName).source(mapping));
    }
    
    @Override
    public VectorSearchResult search(VectorSearchParameters params) {
        // 使用 script_score 实现向量相似度搜索
        String script = "cosineSimilarity(params.query_vector, 'vector') + 1.0";
        
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder()
            .query(QueryBuilders.scriptScoreQuery(
                QueryBuilders.matchAllQuery(),
                new Script(ScriptType.INLINE, "painless", script, 
                    Map.of("query_vector", params.getQueryVector()))
            ))
            .size(params.getLimit());
        
        SearchResponse response = client.search(new SearchRequest(indexName).source(sourceBuilder));
        return convertToResult(response);
    }
}
```

#### REST API

```java
@Path("v2/search/vector")
@Service
public class VectorSearchREST {
    
    @POST
    @Path("/semantic")
    public VectorSearchResult semanticSearch(VectorSearchParameters params) {
        if (params.getQueryVector() == null) {
            params.setQueryVector(embeddingService.embed(params.getQuery()));
        }
        return vectorIndexService.search(params);
    }
    
    @GET
    @Path("/similar/{guid}")
    public VectorSearchResult findSimilar(@PathParam("guid") String guid,
                                          @QueryParam("attribute") String attribute) {
        // 获取实体属性值，生成向量，搜索相似实体
    }
}
```

### 2.3 配置示例

```properties
atlas.search.vector.enabled=true
atlas.search.vector.elasticsearch.hosts=localhost:9200
atlas.search.vector.embedding.provider=openai
atlas.search.vector.embedding.api-key=${OPENAI_API_KEY}
atlas.search.vector.embedding.dimension=1536
atlas.search.vector.attributes=hive_table.description,hive_column.comment
```

---

## 3. 方案二：List Float 属性替代方式

### 3.1 方案概述

利用 Atlas 现有的 `array<float>` 属性类型存储向量，在内存中计算相似度。

### 3.2 类型定义

```json
{
    "entityDefs": [{
        "name": "VectorEnabledAsset",
        "superTypes": ["Asset"],
        "attributeDefs": [
            {"name": "description", "typeName": "string"},
            {"name": "descriptionVector", "typeName": "array<float>", "isIndexable": false}
        ]
    }]
}
```

### 3.3 向量属性处理器

```java
@Component
public class VectorAttributeProcessor implements EntityChangeListenerV2 {
    
    private static final String VECTOR_SUFFIX = "Vector";
    
    @Override
    public void onEntitiesAdded(List<AtlasEntity> entities, boolean isImport) {
        for (AtlasEntity entity : entities) {
            Map<String, String> attrMapping = getTextToVectorMapping(entity.getTypeName());
            
            for (Map.Entry<String, String> entry : attrMapping.entrySet()) {
                String textAttr = entry.getKey();
                String vectorAttr = entry.getValue();
                
                Object textValue = entity.getAttribute(textAttr);
                if (textValue != null) {
                    float[] vector = embeddingService.embed(textValue.toString());
                    List<Float> vectorList = toFloatList(vector);
                    entity.setAttribute(vectorAttr, vectorList);
                }
            }
        }
    }
}
```

### 3.4 内存向量搜索

```java
@Service
public class ListFloatVectorSearchService {
    
    public VectorSearchResult search(VectorSearchParameters params) {
        float[] queryVector = params.getQueryVector();
        String typeName = params.getTypeName();
        String vectorAttr = getVectorAttributeName(typeName);
        
        List<ScoredEntity> results = new ArrayList<>();
        
        // 遍历所有实体计算相似度
        Iterator<AtlasVertex> vertices = graph.query()
            .has("__typeName", typeName)
            .has("__state", "ACTIVE")
            .vertices().iterator();
        
        while (vertices.hasNext()) {
            AtlasVertex vertex = vertices.next();
            List<Float> storedVector = vertex.getProperty(vectorAttr, List.class);
            
            if (storedVector != null) {
                float score = cosineSimilarity(queryVector, toFloatArray(storedVector));
                results.add(new ScoredEntity(vertex.getProperty("__guid"), score));
            }
        }
        
        // 排序并返回 Top-K
        return results.stream()
            .sorted(Comparator.comparing(ScoredEntity::getScore).reversed())
            .limit(params.getLimit())
            .collect(toVectorSearchResult());
    }
    
    private float cosineSimilarity(float[] a, float[] b) {
        float dotProduct = 0, normA = 0, normB = 0;
        for (int i = 0; i < a.length; i++) {
            dotProduct += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        return dotProduct / (float)(Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

### 3.5 方案对比

| 特性 | 方案一 (ES dense_vector) | 方案二 (List Float) |
|------|--------------------------|---------------------|
| 性能 | 高（ES 原生 KNN） | 低（内存遍历） |
| 扩展性 | 好（分布式） | 差（单机） |
| 依赖 | ES 7.3+ | 无额外依赖 |
| 适用场景 | 大规模数据 | 小规模数据 (<10万) |
| 实现复杂度 | 中 | 低 |

---

## 4. 方案三：非侵入式替换全部检索

### 4.1 可行性分析

**可以非侵入式替换**，通过以下方式：

1. **@Primary 注解覆盖默认实现**
2. **装饰器模式包装原有服务**
3. **配置开关控制检索路由**

### 4.2 实现方式

#### 4.2.1 装饰器模式

```java
@Service
@Primary  // 覆盖默认的 EntityDiscoveryService
public class VectorEnhancedDiscoveryService implements AtlasDiscoveryService {
    
    @Inject
    @Qualifier("entityDiscoveryService")  // 注入原始实现
    private AtlasDiscoveryService delegate;
    
    @Inject
    private VectorIndexService vectorIndexService;
    
    @Inject
    private EmbeddingService embeddingService;
    
    @Value("${atlas.search.vector.replace-all:false}")
    private boolean replaceAllSearch;
    
    @Override
    public AtlasSearchResult searchWithParameters(SearchParameters params) throws AtlasBaseException {
        if (replaceAllSearch && params.getQuery() != null) {
            // 使用向量搜索
            return vectorSearch(params);
        }
        // 降级到原始搜索
        return delegate.searchWithParameters(params);
    }
    
    private AtlasSearchResult vectorSearch(SearchParameters params) {
        float[] queryVector = embeddingService.embed(params.getQuery());
        
        VectorSearchParameters vectorParams = new VectorSearchParameters();
        vectorParams.setQueryVector(queryVector);
        vectorParams.setTypeName(params.getTypeName());
        vectorParams.setLimit(params.getLimit());
        
        VectorSearchResult vectorResult = vectorIndexService.search(vectorParams);
        
        // 转换为 AtlasSearchResult
        return convertToAtlasSearchResult(vectorResult);
    }
    
    // 其他方法委托给原始实现
    @Override
    public AtlasSearchResult searchUsingDslQuery(String query, int limit, int offset) {
        return delegate.searchUsingDslQuery(query, limit, offset);
    }
    
    @Override
    public AtlasSearchResult searchUsingFullTextQuery(String query, boolean excludeDeleted, int limit, int offset) {
        if (replaceAllSearch) {
            return vectorSearch(createSearchParams(query, limit, offset));
        }
        return delegate.searchUsingFullTextQuery(query, excludeDeleted, limit, offset);
    }
}
```

#### 4.2.2 混合搜索策略

```java
@Service
@Primary
public class HybridDiscoveryService implements AtlasDiscoveryService {
    
    @Value("${atlas.search.hybrid.vector-weight:0.5}")
    private float vectorWeight;
    
    @Override
    public AtlasSearchResult searchWithParameters(SearchParameters params) {
        // 并行执行关键词搜索和向量搜索
        CompletableFuture<AtlasSearchResult> keywordFuture = 
            CompletableFuture.supplyAsync(() -> delegate.searchWithParameters(params));
        
        CompletableFuture<VectorSearchResult> vectorFuture = 
            CompletableFuture.supplyAsync(() -> vectorSearch(params));
        
        // 融合结果 (RRF - Reciprocal Rank Fusion)
        AtlasSearchResult keywordResult = keywordFuture.join();
        VectorSearchResult vectorResult = vectorFuture.join();
        
        return fuseResults(keywordResult, vectorResult, vectorWeight);
    }
    
    private AtlasSearchResult fuseResults(AtlasSearchResult keyword, 
                                          VectorSearchResult vector, 
                                          float vectorWeight) {
        Map<String, Float> scores = new HashMap<>();
        
        // 关键词搜索得分
        int rank = 1;
        for (AtlasEntityHeader entity : keyword.getEntities()) {
            float score = (1 - vectorWeight) / rank++;
            scores.merge(entity.getGuid(), score, Float::sum);
        }
        
        // 向量搜索得分
        rank = 1;
        for (VectorSearchResult.VectorHit hit : vector.getHits()) {
            float score = vectorWeight / rank++;
            scores.merge(hit.getGuid(), score, Float::sum);
        }
        
        // 按融合得分排序
        List<String> sortedGuids = scores.entrySet().stream()
            .sorted(Map.Entry.<String, Float>comparingByValue().reversed())
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
        
        return buildResult(sortedGuids);
    }
}
```

### 4.3 配置

```properties
# 完全替换检索
atlas.search.vector.replace-all=true

# 或使用混合搜索
atlas.search.hybrid.enabled=true
atlas.search.hybrid.vector-weight=0.5
```

---

## 5. 方案对比与建议

| 维度 | 方案一 (独立ES模块) | 方案二 (List Float) | 方案三 (替换全部) |
|------|---------------------|---------------------|-------------------|
| 侵入性 | 无 | 无 | 低 |
| 性能 | 高 | 低 | 高 |
| 实现复杂度 | 中 | 低 | 高 |
| 适用规模 | 大 | 小 | 大 |
| 推荐场景 | 生产环境 | POC/测试 | 全面语义化 |

### 5.1 推荐路径

1. **POC 阶段**: 方案二（快速验证）
2. **生产阶段**: 方案一（独立模块）
3. **全面升级**: 方案三（替换全部检索）

### 5.2 关键文件清单

| 文件 | 说明 |
|------|------|
| `EntityChangeListenerV2.java` | 实体变更监听接口 |
| `AtlasEntityChangeNotifier.java` | 监听器注册中心 |
| `AtlasDiscoveryService.java` | 检索服务接口 |
| `EntityDiscoveryService.java` | 默认检索实现 |
