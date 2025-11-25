## 当前问题

**目前Atlas不支持Elasticsearch的向量检索功能**。虽然Atlas可以使用Elasticsearch作为索引后端，但现有接口主要针对传统文本搜索和属性查询，没有专门支持向量搜索的功能。

从代码分析中可以看出：

1. Atlas通过JanusGraph使用Elasticsearch作为索引后端，但支持的版本是5.6.4，这个版本对向量搜索的支持有限
2. 现有的搜索接口（如`AtlasGraph.indexQuery()`）虽然可以接受Elasticsearch查询DSL，但没有专门为向量搜索设计的API
3. 现有的搜索处理器（如`EntitySearchProcessor`、`FullTextSearchProcessor`等）主要处理基于文本和属性的搜索，没有向量相似度计算逻辑



## 适配估计

### 1. 向量/混合索引的创建和维护（新增，约200行）

首先需要扩展Elasticsearch索引配置，为需要的属性创建向量字段。在以下位置添加相关逻辑：

org/apache/atlas/repository/graphdb/janus/AtlasJanusGraphDatabase.java   

添加向量字段类型的映射定义setupVectorFieldMapping，在初始化方法中调用向量字段映射设置

org/apache/atlas/repository/graph/GraphBackedSearchIndexer.java 

添加向量索引的创建逻辑createVectorIndex，并修改onEntityChanged方法以处理向量字段



### 2. 实现向量搜索处理器（新增，约300行）

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



### 3. 搜索服务层适配（修改，约100行）

org/apache/atlas/discovery/EntityDiscoveryService.java  添加向量搜索方法

```
// 修改searchWithSearchContext方法以支持向量搜索
private AtlasSearchResult searchWithSearchContext(SearchContext searchContext) throws AtlasBaseException {
    // 现有逻辑保持不变，因为VectorSearchProcessor继承自SearchProcessor
    // 所以现有代码可以处理向量搜索结果
}
```

org/apache/atlas/web/resources/DiscoveryREST.java   添加向量搜索REST方法



### 4. 检索相关文件（修改，约100行）

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

