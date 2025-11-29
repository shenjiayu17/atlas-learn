# Apache Atlas WebApp 模块 - 后端REST服务请求入口分析

## 目录
1. [系统架构概览](#系统架构概览)
2. [Web请求处理流程](#web请求处理流程)
3. [REST服务入口清单](#rest服务入口清单)
4. [核心REST控制器详解](#核心rest控制器详解)
5. [请求路由映射](#请求路由映射)
6. [过滤器和监听器](#过滤器和监听器)
7. [安全和认证机制](#安全和认证机制)

---

## 系统架构概览

### 模块结构
```
webapp/
├── pom.xml                          # Maven配置，定义Jersey和Spring依赖
├── src/
│   ├── main/
│   │   ├── java/org/apache/atlas/
│   │   │   ├── Atlas.java          # 应用启动入口
│   │   │   ├── web/
│   │   │   │   ├── rest/           # REST控制器（核心）
│   │   │   │   ├── servlets/       # Servlet处理（登录、错误）
│   │   │   │   ├── filters/        # 请求过滤器
│   │   │   │   ├── listeners/      # 事件监听器
│   │   │   │   ├── security/       # 安全相关
│   │   │   │   ├── service/        # 业务服务层
│   │   │   │   └── util/           # 工具类
│   │   │   └── notification/       # 通知模块
│   │   ├── resources/
│   │   │   ├── spring-security.xml # Spring Security配置
│   │   │   └── atlas-buildinfo.properties
│   │   └── webapp/
│   │       └── WEB-INF/
│   │           └── web.xml         # Web应用配置（核心）
│   └── test/
```

### 技术栈
- **Web框架**: Jersey (JAX-RS) - REST框架
- **Spring集成**: Spring Security, Spring容器
- **Servlet容器**: Jetty (EmbeddedServer)
- **端口**: 
  - HTTP: 21000 (默认)
  - HTTPS: 21443 (默认)

---

## Web请求处理流程

### 整体流程图
```
HTTP Request
    ↓
Jetty Server (EmbeddedServer)
    ↓
URL Pattern: /api/atlas/*
    ↓
Web Container (web.xml)
    ↓
┌─ Filters (按顺序执行) ────────────────────────────────────┐
│ 1. SpringSecurityFilterChain (Spring Security)             │
│ 2. AuditFilter (审计)                                      │
│ 3. AtlasHeaderFilter (指定路径)                            │
└─────────────────────────────────────────────────────────────┘
    ↓
RequestContextListener (请求上下文)
    ↓
Jersey Servlet (com.sun.jersey.spi.spring.container.servlet.SpringServlet)
    ↓
REST Controller (@Path注解映射)
    ↓
Service Layer (业务逻辑)
    ↓
HTTP Response
```

---

## REST服务入口清单

### 主入口配置 (web.xml)

```xml
<!-- Jersey Servlet配置 -->
<servlet>
    <servlet-name>jersey-servlet</servlet-name>
    <servlet-class>
        com.sun.jersey.spi.spring.container.servlet.SpringServlet
    </servlet-class>
    <init-param>
        <param-name>com.sun.jersey.api.json.POJOMappingFeature</param-name>
        <param-value>true</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<!-- 请求URL映射 -->
<servlet-mapping>
    <servlet-name>jersey-servlet</servlet-name>
    <url-pattern>/api/atlas/*</url-pattern>
</servlet-mapping>
```

### REST控制器清单

所有REST控制器位置: `webapp/src/main/java/org/apache/atlas/web/rest/`

#### 1. **DiscoveryREST.java** - 搜索相关接口
基础路径: `/api/atlas/v2/search`

| 方法 | HTTP | 路径 | 功能 | 备注 |
|------|------|------|------|------|
| searchUsingDSL | GET | `/v2/search/dsl` | DSL查询 | 支持元数据查询语言 |
| dslSearchCreateFile | POST | `/v2/search/dsl/download/create_file` | 创建DSL查询下载任务 | 异步导出 |
| searchUsingFullText | GET | `/v2/search/fulltext` | 全文搜索 | 模糊匹配 |
| searchUsingBasic | GET | `/v2/search/basic` | 基础搜索 | 按类型、分类过滤 |
| searchUsingAttribute | GET | `/v2/search/attribute` | 属性搜索 | 按属性值搜索 |
| searchWithParameters | POST | `/v2/search/basic` | 参数化搜索 | 复杂查询条件 |
| basicSearchCreateFile | POST | `/v2/search/basic/download/create_file` | 创建基础搜索下载任务 | 异步导出 |
| getSearchResultDownloadStatus | GET | `/v2/search/download/status` | 查询下载状态 | 异步任务状态 |
| downloadSearchResultFile | GET | `/v2/search/download/{filename}` | 下载文件 | 获取导出结果 |
| relationSearch | POST | `/v2/search/relations` | 关系搜索 | 边搜索 |
| relationSearchWithGuid | GET | `/v2/search/relations` | 关系查询（GUID） | 关系列表 |
| getRelationshipCountByFilter | GET | `/v2/search/relationship` | 关系计数 | 统计 |
| addSavedSearch | POST | `/v2/search/saved` | 保存搜索 | 持久化 |
| updateSavedSearch | PUT | `/v2/search/saved` | 更新搜索 | 修改保存 |
| getSavedSearch | GET | `/v2/search/saved/{name}` | 获取搜索 | 按名称 |
| getSavedSearches | GET | `/v2/search/saved` | 列表搜索 | 所有保存的搜索 |
| deleteSavedSearch | DELETE | `/v2/search/saved/{guid}` | 删除搜索 | - |
| executeSavedSearchByName | GET | `/v2/search/saved/execute/{name}` | 执行保存的搜索 | 按名称执行 |
| executeSavedSearchByGuid | GET | `/v2/search/saved/execute/guid/{guid}` | 执行保存的搜索 | 按GUID执行 |
| quickSearch | GET/POST | `/v2/search/quick` | 快速搜索 | 简化查询 |
| getSuggestions | GET | `/v2/search/suggestions` | 搜索建议 | 自动完成 |

#### 2. **EntityREST.java** - 实体操作接口
基础路径: `/api/atlas/v2/entity`

| 方法 | HTTP | 路径 | 功能 |
|------|------|------|------|
| getEntityById | GET | `/v2/entity/guid/{guid}` | 按GUID获取实体 |
| getEntityByUniqueAttribute | GET | `/v2/entity/uniqueAttribute/type/{typeName}` | 按唯一属性获取 |
| getClassifications | GET | `/v2/entity/guid/{guid}/classifications` | 获取实体分类 |
| classifyEntity | POST | `/v2/entity/guid/{guid}/classification` | 添加分类 |
| classifyEntities | POST | `/v2/entity/bulk/classification` | 批量分类 |
| createEntity | POST | `/v2/entity` | 创建实体 |
| updateEntity | PUT | `/v2/entity` | 更新实体 |
| updateEntity | PUT | `/v2/entity/guid/{guid}` | 按GUID更新 |
| updateEntity | POST | `/v2/entity/importjson` | 导入JSON |
| updateEntity | POST | `/v2/entity/upload` | 上传文件导入 |
| deleteEntity | DELETE | `/v2/entity/guid/{guid}` | 删除实体 |
| deleteEntity | DELETE | `/v2/entity/uniqueAttribute/type/{typeName}` | 按唯一属性删除 |
| getEntityAudit | GET | `/v2/entity/guid/{guid}/audit` | 获取审计日志 |

#### 3. **TypesREST.java** - 类型定义接口
基础路径: `/api/atlas/v2/types`

| 方法 | HTTP | 路径 | 功能 |
|------|------|------|------|
| getTypeByName | GET | `/v2/types/typedef/name/{name}` | 按名称获取类型 |
| getTypeByGuid | GET | `/v2/types/typedef/guid/{guid}` | 按GUID获取类型 |
| createTypeDefinition | POST | `/v2/types/typedefs` | 创建类型定义 |
| updateTypeDefinition | PUT | `/v2/types/typedefs` | 更新类型定义 |
| deleteTypeDefinition | DELETE | `/v2/types/typedefs/name/{name}` | 删除类型定义 |
| getAllTypeDefinitions | GET | `/v2/types/typedefs` | 获取所有类型 |
| getAllTypeHeaders | GET | `/v2/types/typedefs/headers` | 获取类型头 |

#### 4. **LineageREST.java** - 血缘关系接口
基础路径: `/api/atlas/v2/lineage`

| 方法 | HTTP | 路径 | 功能 |
|------|------|------|------|
| lineageInfo | GET | `/v2/lineage/guid/{guid}` | 获取血缘信息 |
| lineageInfoOnDemand | POST | `/v2/lineage/guid/{guid}/on-demand` | 按需血缘 |

#### 5. **RelationshipREST.java** - 关系操作接口
基础路径: `/api/atlas/v2/relationship`

| 方法 | HTTP | 路径 | 功能 |
|------|------|------|------|
| create | POST | `/v2/relationship` | 创建关系 |
| update | PUT | `/v2/relationship` | 更新关系 |
| delete | DELETE | `/v2/relationship/guid/{guid}` | 删除关系 |
| getById | GET | `/v2/relationship/guid/{guid}` | 获取关系 |

#### 6. **GlossaryREST.java** - 业务词汇表接口
基础路径: `/api/atlas/v2/glossary`

| 方法 | HTTP | 路径 | 功能 |
|------|------|------|------|
| createGlossary | POST | `/v2/glossary` | 创建词表 |
| updateGlossary | PUT | `/v2/glossary/{glossaryGuid}` | 更新词表 |
| deleteGlossary | DELETE | `/v2/glossary/{glossaryGuid}` | 删除词表 |
| getTerm | GET | `/v2/glossary/terms/{termGuid}` | 获取术语 |
| createTerm | POST | `/v2/glossary/terms` | 创建术语 |
| updateTerm | PUT | `/v2/glossary/terms/{termGuid}` | 更新术语 |
| deleteTerm | DELETE | `/v2/glossary/terms/{termGuid}` | 删除术语 |

#### 7. **IndexRecoveryREST.java** - 索引恢复接口
基础路径: `/api/atlas/v2/index`

| 方法 | HTTP | 路径 | 功能 |
|------|------|------|------|
| reindexByQuery | GET/POST | `/v2/index/reindex/search` | 重新索引搜索 |
| reindexByType | GET/POST | `/v2/index/reindex/type/{typeName}` | 按类型重新索引 |

#### 8. **NotificationREST.java** - 通知接口
基础路径: `/api/atlas/v2/notification`

| 方法 | HTTP | 路径 | 功能 |
|------|------|------|------|
| getTopics | GET | `/v2/notification/topics` | 获取消息主题 |

---

## 核心REST控制器详解

### 1. DiscoveryREST - 搜索入口

```java
@Path("v2/search")
@Singleton
@Service
@Consumes({Servlets.JSON_MEDIA_TYPE, MediaType.APPLICATION_JSON})
@Produces({Servlets.JSON_MEDIA_TYPE, MediaType.APPLICATION_JSON})
public class DiscoveryREST {
    private static final Logger LOG = LoggerFactory.getLogger(DiscoveryREST.class);
    
    @Inject
    private AtlasDiscoveryService discoveryService;
    
    // ========== 搜索接口 ==========
    
    /**
     * DSL查询接口
     * @param query DSL查询字符串
     * @param limit 返回结果数
     * @param offset 偏移
     */
    @GET
    @Path("/dsl")
    @Timed
    public AtlasSearchResult searchUsingDSL(
        @QueryParam("query") String query,
        @QueryParam("typeName") String typeName,
        @QueryParam("classification") String classification,
        @QueryParam("limit") int limit,
        @QueryParam("offset") int offset) throws AtlasBaseException {
        // 核心逻辑：调用discoveryService.searchUsingDslQuery()
    }
    
    /**
     * 全文搜索接口
     * @param query 搜索关键词
     * @param excludeDeletedEntities 是否排除删除的实体
     */
    @GET
    @Path("/fulltext")
    @Timed
    public AtlasSearchResult searchUsingFullText(
        @QueryParam("query") String query,
        @QueryParam("excludeDeletedEntities") boolean excludeDeletedEntities,
        @QueryParam("limit") int limit,
        @QueryParam("offset") int offset) throws AtlasBaseException {
        // 核心逻辑：调用discoveryService.searchUsingFullTextQuery()
    }
    
    /**
     * 基础搜索接口
     * @param query 搜索查询
     * @param typeName 实体类型
     * @param classification 分类标签
     */
    @GET
    @Path("/basic")
    @Timed
    public AtlasSearchResult searchUsingBasic(
        @QueryParam("query") String query,
        @QueryParam("typeName") String typeName,
        @QueryParam("classification") String classification,
        @QueryParam("sortBy") String sortByAttribute,
        @QueryParam("sortOrder") SortOrder sortOrder,
        @QueryParam("excludeDeletedEntities") boolean excludeDeletedEntities,
        @QueryParam("limit") int limit,
        @QueryParam("offset") int offset,
        @QueryParam("marker") String marker) throws AtlasBaseException {
        // 核心逻辑：调用discoveryService.searchUsingBasicQuery()
    }
    
    /**
     * 参数化搜索接口 - 最灵活的搜索方式
     * @param parameters 搜索参数对象
     */
    @Path("basic")
    @POST
    @Timed
    public AtlasSearchResult searchWithParameters(SearchParameters parameters) 
            throws AtlasBaseException {
        // 核心逻辑：调用discoveryService.searchWithParameters()
        // 支持复杂过滤条件、排序、分页等
    }
}
```

### 2. EntityREST - 实体操作入口

```java
@Path("v2/entity")
@Singleton
@Service
@Consumes({Servlets.JSON_MEDIA_TYPE, MediaType.APPLICATION_JSON})
@Produces({Servlets.JSON_MEDIA_TYPE, MediaType.APPLICATION_JSON})
public class EntityREST {
    
    @Inject
    private AtlasEntityStore entityStore;
    
    @Inject
    private EntityGraphRetriever entityRetriever;
    
    /**
     * 创建实体
     * @param request HTTP请求
     * @param entities 实体数据
     */
    @POST
    @Timed
    @Path("")
    public EntityMutationResponse createEntity(
        @Context HttpServletRequest request,
        final AtlasEntitiesWithExtInfo entities) throws AtlasBaseException {
        // 核心逻辑：entityStore.createOrUpdate()
    }
    
    /**
     * 获取实体 by GUID
     * @param guid 实体全局唯一标识
     * @param minExtInfo 最少返回属性信息
     * @param ignoreRelationships 是否忽略关系
     */
    @GET
    @Path("/guid/{guid}")
    @Timed
    public AtlasEntityWithExtInfo getEntityById(
        @PathParam("guid") String guid,
        @QueryParam("minExtInfo") @DefaultValue("false") boolean minExtInfo,
        @QueryParam("ignoreRelationships") @DefaultValue("false") boolean ignoreRelationships) 
            throws AtlasBaseException {
        // 核心逻辑：entityRetriever.toAtlasEntityWithExtInfo()
    }
    
    /**
     * 删除实体 by GUID
     * @param guid 实体GUID
     */
    @DELETE
    @Path("/guid/{guid}")
    @Timed
    public EntityMutationResponse deleteEntity(@PathParam("guid") String guid) 
            throws AtlasBaseException {
        // 核心逻辑：entityStore.deleteById()
    }
    
    /**
     * 批量导入实体
     * @param input 上传的文件流
     * @param fileDetail 文件信息
     */
    @POST
    @Path("/upload")
    @Consumes("multipart/form-data")
    public BulkImportResponse uploadEntity(
        @FormDataParam("file") InputStream input,
        @FormDataParam("file") FormDataContentDisposition fileDetail) 
            throws AtlasBaseException {
        // 核心逻辑：bulkImportService.importEntities()
    }
}
```

### 3. TypesREST - 类型定义入口

```java
@Path("v2/types")
@Singleton
@Service
public class TypesREST {
    
    @Inject
    private AtlasTypeDefStore typeDefStore;
    
    /**
     * 创建类型定义
     * @param typeDef 类型定义对象
     */
    @POST
    @Path("/typedefs")
    @Timed
    public AtlasTypesDef createTypeDefinition(AtlasTypesDef typeDef) 
            throws AtlasBaseException {
        // 核心逻辑：typeDefStore.createTypesDef()
    }
    
    /**
     * 获取类型定义 by 名称
     * @param name 类型名称
     */
    @GET
    @Path("/typedef/name/{name}")
    @Timed
    public AtlasBaseTypeDef getTypeByName(@PathParam("name") String name) 
            throws AtlasBaseException {
        // 核心逻辑：typeDefStore.getTypeDefByName()
    }
    
    /**
     * 获取所有类型定义
     */
    @GET
    @Path("/typedefs")
    @Timed
    public AtlasTypesDef getAllTypeDefinitions(
        @QueryParam("type") String typeCategory) 
            throws AtlasBaseException {
        // 核心逻辑：typeDefStore.searchTypesDef()
    }
}
```

### 4. LineageREST - 血缘关系入口

```java
@Path("v2/lineage")
@Singleton
@Service
public class LineageREST {
    
    @Inject
    private AtlasLineageService atlasLineageService;
    
    /**
     * 获取实体血缘信息
     * @param guid 实体GUID
     * @param direction 方向：UPSTREAM/DOWNSTREAM/BOTH
     * @param depth 深度
     */
    @GET
    @Path("/guid/{guid}")
    @Timed
    public AtlasLineageInfo lineageInfo(
        @PathParam("guid") String guid,
        @QueryParam("direction") @DefaultValue("BOTH") LineageDirection direction,
        @QueryParam("depth") @DefaultValue("3") int depth) 
            throws AtlasBaseException {
        // 核心逻辑：atlasLineageService.getLineageInfo()
    }
}
```

---

## 请求路由映射

### URL路由树

```
/api/atlas/
├── v2/search/                          [DiscoveryREST]
│   ├── dsl                             (GET)    - DSL查询
│   ├── dsl/download/create_file        (POST)   - DSL导出
│   ├── fulltext                        (GET)    - 全文搜索
│   ├── basic                           (GET/POST) - 基础搜索
│   ├── attribute                       (GET)    - 属性搜索
│   ├── relations                       (POST/GET) - 关系查询
│   ├── relationship                    (GET)    - 关系统计
│   ├── download/
│   │   ├── status                      (GET)    - 下载状态
│   │   └── {filename}                  (GET)    - 下载文件
│   └── saved/
│       ├── (POST/PUT/GET/DELETE)       - CRUD搜索
│       └── execute/{name|guid}         (GET)    - 执行保存的搜索
│
├── v2/entity/                          [EntityREST]
│   ├── (POST/PUT)                      - 创建/更新实体
│   ├── guid/{guid}                     (GET/PUT/DELETE) - 实体操作
│   ├── uniqueAttribute/type/{typeName} (GET/DELETE) - 唯一属性
│   ├── classification/                 - 分类操作
│   ├── audit/                          - 审计日志
│   └── upload                          (POST)   - 批量导入
│
├── v2/types/                           [TypesREST]
│   ├── typedefs                        (GET/POST/PUT) - CRUD类型
│   ├── typedef/name/{name}             (GET)    - 获取类型
│   ├── typedef/guid/{guid}             (GET)    - 获取类型
│   └── typedefs/headers                (GET)    - 类型头列表
│
├── v2/lineage/                         [LineageREST]
│   └── guid/{guid}                     (GET/POST) - 血缘查询
│
├── v2/relationship/                    [RelationshipREST]
│   ├── (POST/PUT)                      - 创建/更新关系
│   └── guid/{guid}                     (GET/DELETE) - 关系操作
│
├── v2/glossary/                        [GlossaryREST]
│   ├── (POST/PUT/DELETE)               - 词表操作
│   └── terms/                          - 术语操作
│
├── v2/notification/                    [NotificationREST]
│   └── topics                          (GET)    - 消息主题
│
└── v2/index/                           [IndexRecoveryREST]
    └── reindex/                        - 索引重建
```

---

## 过滤器和监听器

### 请求过滤链

#### 1. **SpringSecurityFilterChain** (Spring Security)
- **位置**: `spring-security.xml` 配置
- **功能**: 
  - 认证 (Authentication)
  - 授权 (Authorization)
  - 跨站请求伪造防护 (CSRF)
- **执行顺序**: 第1个
- **URL模式**: `/*` (所有请求)

```xml
<filter>
    <filter-name>springSecurityFilterChain</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>

<filter-mapping>
    <filter-name>springSecurityFilterChain</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

#### 2. **AuditFilter** (审计过滤器)
- **类**: `org.apache.atlas.web.filters.AuditFilter`
- **功能**:
  - 记录所有API请求的审计日志
  - 捕获请求参数、响应状态
  - 记录执行时间
- **执行顺序**: 第2个
- **URL模式**: `/*` (所有请求)

```xml
<filter>
    <filter-name>AuditFilter</filter-name>
    <filter-class>org.apache.atlas.web.filters.AuditFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>AuditFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

#### 3. **HeaderFilter** (请求头过滤器)
- **类**: `org.apache.atlas.web.filters.AtlasHeaderFilter`
- **功能**:
  - 验证请求头信息
  - 处理特定API的请求头要求
- **执行顺序**: 第3个
- **URL模式**: 
  - `/api/atlas/admin/metrics`
  - `/api/atlas/admin/status`

### 事件监听器

#### 1. **RequestContextListener** (Spring)
- **类**: `org.springframework.web.context.request.RequestContextListener`
- **功能**: 
  - 维护请求上下文 (RequestContext)
  - 支持在业务方法中访问 `RequestContextHolder.getRequestAttributes()`

#### 2. **KerberosAwareListener** (Kerberos认证)
- **类**: `org.apache.atlas.web.setup.KerberosAwareListener`
- **功能**:
  - Kerberos认证初始化
  - 可选的Hadoop集成认证

---

## 安全和认证机制

### 认证方式

#### 1. Spring Security配置
- **文件**: `src/main/resources/spring-security.xml`
- **支持的认证方式**:
  - LDAP
  - KERBEROS
  - BASIC
  - FORM

#### 2. Session管理
```xml
<session-config>
    <session-timeout>60</session-timeout>              <!-- 60分钟超时 -->
    <tracking-mode>COOKIE</tracking-mode>              <!-- Cookie跟踪 -->
    <cookie-config>
        <name>ATLASSESSIONID</name>                   <!-- Session Cookie名 -->
        <http-only>true</http-only>                   <!-- HttpOnly标志 -->
    </cookie-config>
</session-config>
```

### Authorization控制

所有REST接口都通过以下机制进行授权检查：

```java
// 在REST控制器或Service中
@Inject
private AtlasAuthorizationUtils authorizationUtils;

public void someRestMethod() throws AtlasBaseException {
    // 检查用户权限
    authorizationUtils.verifyAccess(
        new AtlasEntityAccessRequest(typeRegistry, AtlasPrivilege.ENTITY_READ, entityGuid)
    );
}
```

---

## 主要应用启动流程

### 入口: `Atlas.java` main()

```java
public class Atlas {
    public static void main(String[] args) throws Exception {
        // 1. 解析命令行参数
        CommandLine cmd = parseArgs(args);
        
        // 2. 获取应用配置
        Configuration configuration = ApplicationProperties.get();
        
        // 3. 启动Jetty服务器
        server = EmbeddedServer.newServer(
            appHost,           // 绑定地址
            appPort,           // 端口号
            appPath,           // WAR包路径
            enableTLS          // 是否启用TLS
        );
        
        // 4. 启动服务
        server.start();
        
        // 5. 安装日志桥接
        installLogBridge();
    }
    
    // 关闭钩子
    static {
        ShutdownHookManager.get().addShutdownHook(
            new Thread(() -> server.stop()),
            SHUTDOWN_PRIORITY
        );
    }
}
```

---

## 常见API调用示例

### 1. 搜索实体

```bash
# 全文搜索
curl -X GET "http://localhost:21000/api/atlas/v2/search/fulltext?query=database&limit=10&offset=0"

# 基础搜索
curl -X GET "http://localhost:21000/api/atlas/v2/search/basic?typeName=DataSet&limit=10"

# DSL查询
curl -X GET "http://localhost:21000/api/atlas/v2/search/dsl?query=DataSet&limit=10"

# 参数化搜索 (POST)
curl -X POST "http://localhost:21000/api/atlas/v2/search/basic" \
  -H "Content-Type: application/json" \
  -d '{
    "typeName": "DataSet",
    "classification": "Sensitive",
    "limit": 10,
    "offset": 0
  }'
```

### 2. 实体操作

```bash
# 获取实体
curl -X GET "http://localhost:21000/api/atlas/v2/entity/guid/{guid}"

# 创建实体
curl -X POST "http://localhost:21000/api/atlas/v2/entity" \
  -H "Content-Type: application/json" \
  -d '{...entity data...}'

# 更新实体
curl -X PUT "http://localhost:21000/api/atlas/v2/entity" \
  -H "Content-Type: application/json" \
  -d '{...updated entity...}'

# 删除实体
curl -X DELETE "http://localhost:21000/api/atlas/v2/entity/guid/{guid}"
```

### 3. 类型管理

```bash
# 获取所有类型
curl -X GET "http://localhost:21000/api/atlas/v2/types/typedefs"

# 创建类型
curl -X POST "http://localhost:21000/api/atlas/v2/types/typedefs" \
  -H "Content-Type: application/json" \
  -d '{...type definition...}'
```

### 4. 血缘查询

```bash
# 获取血缘信息
curl -X GET "http://localhost:21000/api/atlas/v2/lineage/guid/{guid}?direction=BOTH&depth=3"
```

---

## 扩展点

### 1. 自定义REST控制器

```java
@Path("v2/custom")
@Singleton
@Service
@Consumes({Servlets.JSON_MEDIA_TYPE, MediaType.APPLICATION_JSON})
@Produces({Servlets.JSON_MEDIA_TYPE, MediaType.APPLICATION_JSON})
public class CustomREST {
    @GET
    @Path("/hello")
    public Response hello() {
        return Response.ok("Hello Atlas").build();
    }
}
```

### 2. 自定义过滤器

```java
public class CustomFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        // 自定义逻辑
        chain.doFilter(request, response);
    }
}

// web.xml中注册
<filter>
    <filter-name>CustomFilter</filter-name>
    <filter-class>com.example.CustomFilter</filter-class>
</filter>
```

### 3. 自定义Service

```java
@Service
public class CustomService {
    @Inject
    private AtlasDiscoveryService discoveryService;
    
    public void customLogic() {
        // 业务逻辑
    }
}
```

---

## 性能监控

### @Timed 注解

所有REST方法都使用 `@Timed` 注解进行性能跟踪：

```java
@GET
@Path("/basic")
@Timed  // 自动记录方法执行时间
public AtlasSearchResult searchUsingBasic(...) {
    // 方法体
}
```

**性能日志**:
- 记录位置: `PERF_LOG` 
- 包含: 方法调用时间、参数信息
- 使用: `AtlasPerfTracer` 工具

---

## 错误处理

### 异常映射

REST接口抛出的异常通过全局异常处理器映射为HTTP响应：

| 异常类型 | HTTP状态码 | 示例 |
|---------|-----------|------|
| `AtlasBaseException` (BAD_REQUEST) | 400 | 非法参数 |
| `AtlasBaseException` (NOT_FOUND) | 404 | 资源不存在 |
| `AtlasBaseException` (UNAUTHORIZED) | 401 | 无权限 |
| `AtlasBaseException` (INTERNAL_SERVER_ERROR) | 500 | 服务器错误 |
| 其他异常 | 500 | 未预期的错误 |

---

## 总结

### 关键点:
1. ✅ **入口**: `/api/atlas/*` URL模式映射到Jersey Servlet
2. ✅ **控制器**: 8个主要REST控制器处理不同功能
3. ✅ **过滤链**: Spring Security + Audit + Header三层过滤
4. ✅ **认证**: Spring Security支持多种认证方式
5. ✅ **监听**: RequestContext和Kerberos支持
6. ✅ **启动**: EmbeddedServer在Atlas.main()中启动Jetty

### 核心控制器:
- `DiscoveryREST` - 搜索功能 (最常用)
- `EntityREST` - 实体CRUD
- `TypesREST` - 类型管理
- `LineageREST` - 血缘关系
- `RelationshipREST` - 关系操作
- `GlossaryREST` - 词汇表
- `NotificationREST` - 通知
- `IndexRecoveryREST` - 索引恢复

