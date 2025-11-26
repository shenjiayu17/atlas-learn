## 当前现状

**目前Atlas不支持Elasticsearch的向量检索功能**。虽然Atlas可以使用Elasticsearch作为索引后端，但现有接口主要针对传统文本搜索和属性查询，没有专门支持向量搜索的功能。

从代码分析中可以看出：

1. Atlas通过JanusGraph使用Elasticsearch作为索引后端，支持的版本为7.x（可能需要升级）
2. 现有的搜索接口（如`AtlasGraph.indexQuery()`）虽然可以接受Elasticsearch查询DSL，但没有专门为向量搜索设计的API
3. 现有的搜索处理器（如`EntitySearchProcessor`、`FullTextSearchProcessor`等）主要处理基于文本和属性的搜索，没有向量相似度计算逻辑



## 方案一  侵入式修改

### 1. 向量/混合索引的创建和维护（新增）

org/apache/atlas/repository/graphdb/janus/AtlasJanusGraphDatabase.java   

封装JanusGraph实例，加载依赖的类到StandardStoreManager（hbase2、rdbms、solr、elasticsearch），提供底层的图存储和索引支持

**org/apache/atlas/repository/graph/GraphBackedSearchIndexer.java**

负责图数据库的索引结构的管理：创建和维护索引结构，包括顶点索引、边索引和全文索引；维护属性键和索引字段之间的映射关系

通过AtlasGraphManagement接口与底层的JanusGraph进行交互。//需添加向量索引的创建/更新逻辑，处理向量字段

1）initialize 创建三种主要索引：

- VERTEX_INDEX：顶点混合索引
- EDGE_INDEX：边混合索引
- FULLTEXT_INDEX：全文索引

然后为各种属性创建索引，如GUID、类型名称、时间戳等。

2）onChange 负责更新索引结构

`ChangedTypeDefs` 包含了 **类型系统（Type System）的变更**，包括：

- 新增/删除/修改 `EntityDef`

- 新增/删除/修改 `ClassificationDef`（即标签类型）

- 修改继承关系、属性定义、约束等

`GraphBackedSearchIndexer#onChange(ChangedTypeDefs)` 的作用是：

- 检测到类型定义变更后，

  重建或更新搜索索引的 schema/mapping

  - 例如：在 Solr 中更新 `managed-schema`（添加/删除字段）
  - 在 ES 中更新 index mapping（动态模板或显式字段）

- 它**不直接更新实体数据索引内容**，而是更新索引的**结构**。

3）JanusGraph图事务的自动同步

- Atlas使用JanusGraph作为图数据库，当实体数据在图数据库中更新时，JanusGraph会自动将这些变更同步到配置的后端索引（如Elasticsearch）中

- 这种同步是通过JanusGraph的事务机制实现的，在事务提交时自动触发索引更新

4）SolrIndexHelper 类实现了IndexChangeListener接口，是onChange方法的补充操作。当类型定义发生变更时，先调用 `GraphBackedSearchIndexer#onChange` ，再调用` IndexChangeListener#onChange`

SolrIndexHelper 主要负责处理Solr索引的搜索权重和建议字段配置。geIndexFieldNamesWithSearchWeights会收集所有需要设置搜索权重的索引字段。getIndexFieldNamesForSuggestions会收集搜索权重大于等于8的字段，用于搜索建议功能。

执行逻辑：1 获取图索引客户端，2 应用搜索权重配置，3 应用建议字段配置





### 2. 实现向量搜索处理器（新增）

需要创建一个新的搜索处理器，类似于现有的`EntitySearchProcessor`，专门处理向量搜索。org/apache/atlas/discovery/VectorSearchProcessor.java  新增检索处理器

org/apache/atlas/discovery/SearchProcessor.java    在SearchProcessorFactory中添加向量搜索处理器的创建逻辑

```
public class VectorSearchProcessor extends SearchProcessor {
    private final AtlasIndexQuery indexQuery;
    
    public VectorSearchProcessor(SearchContext context) {
        super(context);
        this.indexQuery = buildVectorQuery(context);
    }
    
    private AtlasIndexQuery buildVectorQuery(SearchContext context) {
        SearchParameters params = context.getSearchParameters();
        String vectorFieldName = params.getVectorAttributeName();
        float[] queryVector = params.getQueryVector();
        int k = params.getVectorK();
        String similarity = params.getVectorSimilarity();
        
        // 构建Elasticsearch向量查询DSL
        String queryString = buildElasticsearchVectorQuery(vectorFieldName, queryVector, k, similarity);
        return context.getGraph().indexQuery(Constants.VERTEX_INDEX, queryString);
    }
    
    private String buildElasticsearchVectorQuery(String fieldName, float[] vector, int k, String similarity) {
        // 构建Elasticsearch向量查询DSL
        // 例如: {"knn": {"field": fieldName, "query_vector": vector, "k": k, "similarity": similarity}}
        // 具体实现取决于Elasticsearch版本
    }
    
    @Override
    public List<AtlasVertex> execute() throws AtlasBaseException {}
}
```



### 3. 搜索服务层适配（修改）

org/apache/atlas/discovery/EntityDiscoveryService.java  添加向量搜索方法

```
// 修改searchWithSearchContext方法以支持向量搜索
private AtlasSearchResult searchWithSearchContext(SearchContext searchContext) throws AtlasBaseException {
    // 现有逻辑保持不变，因为VectorSearchProcessor继承自SearchProcessor
    // 所以现有代码可以处理向量搜索结果
}
```

org/apache/atlas/web/resources/DiscoveryREST.java   添加向量搜索REST方法



### 4. 检索相关文件（修改）

org/apache/atlas/model/discovery/SearchParameters.java  检索相关参数

```
public class SearchParameters {
    // 现有参数
    // 添加向量搜索相关字段
    private float[] queryVector;
    private int topK;
    // 添加对应的getter和setter方法
}
```

org/apache/atlas/model/discovery/AtlasSearchResult.java   检索结果是否要增加字段

org/apache/atlas/model/discovery/AtlasSearchResult.AtlasQueryType   检索类型

client/client-v2/src/main/java/org/apache/atlas/AtlasClientV2.java  向量搜索API的客户端方法



### 5. 其他文件 

模型定义文件，例如hive_model.json

配置文件，atlas-application.properties

测试用例，前端支持



## 方案二  利用AtlasArrayType

### 1. Atlas数组类型的能力

`AtlasArrayType.java` 支持`array<float>`格式存储浮点数数组

### 2. 存储层支持

`GraphBackedSearchIndexer.java`分析

- Atlas使用JanusGraph作为图数据库，支持Elasticsearch作为索引后端
- 数组类型会被序列化为JSON字符串存储
- 可以通过`createIndexForAttribute`方法为数组属性创建索引

### 3. 实体类型定义

实体类型的属性定义增加array\<float\>属性

```json
{
  "entityDefs": [
    {
      "name": "vector_document",
      "superTypes": ["DataSet"],
      "serviceType": "custom",
      "typeVersion": "1.0",
      "description": "支持向量检索的文档实体",
      "attributeDefs": [
        {
          "name": "embedding",
          "typeName": "array<float>",
          "cardinality": "SINGLE",
          "isIndexable": true,
          "isOptional": true,
          "isUnique": false,
          "description": "文档的向量嵌入"
        },
        {
          "name": "content",
          "typeName": "string",
          "cardinality": "SINGLE", 
          "isIndexable": true,
          "isOptional": false,
          "isUnique": false,
          "description": "文档内容"
        }
      ]
    }
  ]
}
```

### 4. 检索流程设计

```java
public List<VectorDocument> searchSimilarDocuments(float[] queryVector, int topK) {
    // 1. 从Atlas获取候选文档（可选的元数据过滤）
    List<VectorDocument> candidates = atlasClient.searchDocuments(metadataFilter);
    
    // 2. 调用向量相似度服务计算相似度
    List<SimilarityResult> similarities = vectorService.computeSimilarities(
        queryVector, 
        candidates.stream().map(d -> d.getEmbedding()).collect(Collectors.toList())
    );
    
    // 3. 返回最相似的topK结果
    return similarities.stream()
        .sorted(Comparator.comparing(SimilarityResult::getScore).reversed())
        .limit(topK)
        .map(result -> candidates.get(result.getIndex()))
        .collect(Collectors.toList());
}
```


