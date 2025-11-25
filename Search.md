# EntityDiscoveryService.java
searchWithSearchContext内部查询逻辑分析

## 1. getSearchProcessor 的链表结构
getSearchProcessor 方法返回的是一个处理器链，采用了责任链模式。在 SearchContext 类中，处理器通过 addProcessor 方法添加到链表中，每个 SearchProcessor 对象都有一个 nextProcessor 字段，指向链中的下一个处理器：
<img width="939" height="152" alt="image" src="https://github.com/user-attachments/assets/1baa251a-6453-4846-b9f7-7de4daeb789c" />


##  2. 处理器使用方式（多个处理器而非择一）
搜索使用的是多个处理器组合的方式，而不是择一处理器。根据搜索条件，系统会按顺序添加多个处理器到链中：
<img width="940" height="561" alt="image" src="https://github.com/user-attachments/assets/ea3e52d8-384b-4eea-835d-4da371ef6850" />

执行时，第一个处理器执行查询，然后将结果传递给链中的下一个处理器进行进一步过滤：
<img width="940" height="165" alt="image" src="https://github.com/user-attachments/assets/c7f16cea-9ec0-4ac9-b275-98c13e17e20f" />

## 3. 检索处理器类型
系统包含以下几种处理器类型：
- TermSearchProcessor: 处理术语搜索，用于查找与特定术语关联的实体
- FreeTextSearchProcessor: 处理自由文本搜索，使用自由文本索引
- FullTextSearchProcessor: 处理全文搜索，使用全文索引
- ClassificationSearchProcessor: 处理分类搜索，根据分类类型过滤实体
- EntitySearchProcessor: 处理实体搜索，根据实体类型和属性过滤实体

## 4. Elasticsearch 支持
系统支持 Elasticsearch 作为索引后端。从代码中可以看出：
 1. 定义了 FULLTEXT_INDEX = "fulltext_index" 常量，用于全文搜索索引
 2. 在 FreeTextSearchProcessor 中，有针对 Solr 索引后端的特殊处理：
    <img width="1073" height="247" alt="image" src="https://github.com/user-attachments/assets/335e4131-decb-4f60-ac6b-9fd0abeb9540" />
 3. 在索引查询参数准备时，会根据索引后端类型添加不同参数：
    <img width="952" height="195" alt="image" src="https://github.com/user-attachments/assets/474784f3-a65d-4dc4-8fee-a17b876af8e1" />

## 5. 查询执行流程
- 初始化阶段: 根据搜索参数创建 SearchContext 对象，并构建处理器链
- 执行阶段: 调用 searchContext.getSearchProcessor().execute() 执行第一个处理器
- 过滤阶段: 每个处理器执行自己的查询逻辑，然后将结果传递给下一个处理器进行过滤
- 结果收集: 最终收集所有符合条件的实体顶点
这种设计允许系统灵活地组合不同类型的搜索条件，每个处理器专注于处理特定类型的搜索逻辑，通过链式调用实现复杂的复合搜索功能。
