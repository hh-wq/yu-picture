# 智能协同云图库 —— 面试项目指南

> 面向 Java 后端工程师的项目介绍、亮点提炼与面试表达指南。
> 所有代码引用均对应 `yu-picture-backend` 模块的真实类/文件路径。

---

## 一、项目定位与核心业务

### 一句话定位

**企业级智能协同云图库平台**：支持公共图片素材检索与管理、个人私有相册、团队协同图片编辑，同时集成 AI 扩图能力。

### 核心用户路径

| 角色 | 主要路径 |
|------|---------|
| 普通用户 | 注册/登录 → 搜索公共图片 → 上传图片到公共库/私有空间 → 管理图片（编辑/删除/分享） |
| 管理员 | 审核用户上传的图片（通过/拒绝）→ 分析平台用量 |
| 团队成员 | 创建/加入团队空间 → 实时协同编辑图片 → 查看空间用量分析 |

### 四大核心功能

1. **公共图库**：所有用户均可上传、检索、浏览公共图片素材（含图片审核机制）。
2. **私有空间**：用户开通私有空间，对图片进行批量管理、多维检索、编辑分析。
3. **团队空间**：企业可开通团队空间，邀请成员，基于 WebSocket 实时协同编辑图片。
4. **AI 扩图**：接入阿里云 AI 大模型，支持图片智能扩展/外绘功能。

---

## 二、关键架构与模块划分

### 整体分层（传统 MVC 版本：`yu-picture-backend`）

```
com.yupi.yupicturebackend
├── annotation/          # 自定义注解（@AuthCheck）
├── aop/                 # AOP 切面（AuthInterceptor 权限拦截）
├── api/                 # 外部 API 对接
│   ├── aliyunai/        # 阿里云 AI（扩图大模型）
│   └── imagesearch/     # 以图搜图（百度 API 门面）
├── common/              # 通用响应结构（BaseResponse、ResultUtils）
├── config/              # 配置类（COS、跨域、MyBatis-Plus、JSON 等）
├── constant/            # 常量（UserConstant）
├── controller/          # HTTP 接口层（5 个 Controller）
├── exception/           # 统一异常（ErrorCode、BusinessException、GlobalExceptionHandler）
├── manager/             # 核心基础设施层
│   ├── auth/            # Sa-Token 空间权限管理
│   ├── sharding/        # ShardingSphere 动态分表
│   ├── upload/          # 图片上传模板（模板方法模式）
│   └── websocket/       # WebSocket + Disruptor 协同编辑
├── mapper/              # MyBatis-Plus Mapper 接口
├── model/               # 数据模型（entity/dto/vo/enums）
├── service/             # 业务逻辑（Service + ServiceImpl）
└── utils/               # 工具类（颜色相似度、颜色转换）
```

### DDD 版本（`yu-picture-backend-ddd`）分层

```
com.yupi.yupicture
├── interfaces/          # 接口层（Controller、VO）
├── application/         # 应用服务层（编排领域服务）
├── domain/              # 领域层（entity、repository 接口、domainService、valueObject）
│   ├── picture/
│   ├── space/
│   └── user/
├── infrastructure/      # 基础设施层（repository 实现、Mapper、外部 API 适配）
└── shared/              # 公共组件（auth、sharding、websocket）
```

### 五个 Controller（接口入口）

| Controller | 前缀 | 职责 |
|-----------|------|------|
| `UserController` | `/api/user` | 注册、登录、用户管理 |
| `PictureController` | `/api/picture` | 图片上传/查询/审核/AI扩图 |
| `SpaceController` | `/api/space` | 空间创建/管理/查询 |
| `SpaceUserController` | `/api/spaceUser` | 团队成员邀请/移除/角色管理 |
| `SpaceAnalyzeController` | `/api/spaceAnalyze` | 空间用量分析（图表数据） |

---

## 三、面试亮点（≥6 条，带落点与验证）

### 亮点 1：模板方法模式实现多来源图片上传

**具体落点**

- 抽象基类：`manager/upload/PictureUploadTemplate.java`
- 文件上传实现：`manager/upload/FilePictureUpload.java`
- URL 上传实现：`manager/upload/UrlPictureUpload.java`
- 调用处：`service/impl/PictureServiceImpl.java`（根据 `inputSource` 类型自动选择策略，约第 168 行）

**设计权衡**

上传来源有"本地文件"和"URL 抓取"两种，二者共享"校验 → 生成路径 → 上传 COS → 获取元数据 → 清理临时文件"的主流程，但校验逻辑和文件预处理逻辑各自不同。模板方法将公共流程固定在父类，子类只覆写差异步骤（`validPicture`、`getOriginFilename`、`processFile`），做到**开闭原则**。

```java
// PictureServiceImpl.java  约第 168 行
PictureUploadTemplate pictureUploadTemplate = filePictureUpload;
if (inputSource instanceof String) {
    pictureUploadTemplate = urlPictureUpload;
}
UploadPictureResult uploadPictureResult =
        pictureUploadTemplate.uploadPicture(inputSource, uploadPathPrefix);
```

**如何验证**

调用 `POST /api/picture/upload`（multipart 文件）和 `POST /api/picture/upload/url`（URL 字符串），两条链路均进入各自子类的 `processFile`，但共享相同的 COS 上传和结果封装逻辑。可在父类 `uploadPicture` 方法打断点，观察两次调用进入同一主流程。

---

### 亮点 2：COS 端云数据处理（压缩 + 缩略图一次请求完成）

**具体落点**

- COS 客户端封装：`manager/CosManager.java`（`putPictureObject` 方法）
- 上传结果解析：`manager/upload/PictureUploadTemplate.java`（`buildResult` 方法）
- 配置：`config/CosClientConfig.java` + `application.yml`（`cos.client.*`）

**设计权衡**

传统做法需要在应用层先压缩再生成缩略图，多次 I/O 且占用服务器 CPU。项目利用腾讯云 COS 数据万象（CI）的"上传时处理"功能：一次 `putPictureObject` 请求即可异步完成格式转换、压缩和缩略图生成，返回 `CIObject` 列表中分别包含压缩图和缩略图的 key/大小信息。服务器零 CPU 消耗，存储成本更低。

**如何验证**

上传一张大图后，在响应体中观察 `url`（压缩后的图）和 `thumbnailUrl`（缩略图）为不同路径；对比原始文件体积与 COS 上存储的文件体积（通过腾讯云控制台）。

---

### 亮点 3：Redis + Caffeine 多级缓存加速图片列表查询

**具体落点**

- 本地缓存定义：`controller/PictureController.java`（`LOCAL_CACHE` Caffeine 实例，最大 10000 条，5 分钟过期）
- Redis 缓存操作：`controller/PictureController.java`（`listPictureVOByPageWithCache` 方法）
- 缓存 Key 策略：`queryCondition + pageNum + pageSize` 的 MD5 哈希

**设计权衡**

图片列表为高频读取接口。Redis 作为分布式二级缓存保证多实例一致；Caffeine 作为进程内一级缓存避免每次查询都走网络到 Redis。请求先查本地缓存（微秒级），命中则直接返回；未命中再查 Redis（毫秒级），再未命中才查 DB 并回写两级缓存。大幅降低数据库压力和 P99 延迟。

```java
// PictureController.java
// 1. 查本地缓存
String cacheValue = LOCAL_CACHE.getIfPresent(cacheKey);
if (cacheValue != null) { return ResultUtils.success(...); }
// 2. 查 Redis
cacheValue = valueOperations.get(cacheKey);
if (cacheValue != null) {
    LOCAL_CACHE.put(cacheKey, cacheValue);
    return ResultUtils.success(...);
}
// 3. 查数据库并回写
```

**如何验证**

连续两次调用 `GET /api/picture/list/page/vo/cache`，第一次走 DB（日志可见 SQL），第二次在本地缓存命中（无 SQL 日志），响应时间明显缩短。

---

### 亮点 4：Sa-Token + JSON 配置驱动的空间 RBAC 权限体系

**具体落点**

- 权限配置文件：`resources/biz/spaceUserAuthConfig.json`（角色 → 权限列表的 JSON 映射）
- 权限管理器：`manager/auth/SpaceUserAuthManager.java`
- Sa-Token 接口实现：`manager/auth/StpInterfaceImpl.java`（`getPermissionList` / `getRoleList`）
- 自定义注解：`manager/auth/annotation/SaSpaceCheckPermission.java`
- 多 StpLogic：`manager/auth/StpKit.java`（与系统登录态隔离的 `SPACE` 类型 StpLogic）

**设计权衡**

空间角色（viewer/editor/admin）及其对应权限通过外置 JSON 文件配置，无需修改代码即可调整权限矩阵，符合**开闭原则**和**配置驱动**思想。Sa-Token 的多 `StpLogic` 机制将"空间成员鉴权"与"系统用户登录"完全隔离，避免鉴权逻辑耦合。注解 `@SaSpaceCheckPermission` 与 AOP 结合，侵入性极低。

**如何验证**

以 viewer 角色的成员身份调用 `POST /api/picture/upload`（需要 `PICTURE_UPLOAD` 权限），期望返回 403；以 editor 角色调用则成功。修改 `spaceUserAuthConfig.json` 中 viewer 的权限列表，无需重启可验证（需重新加载，或重启后立即生效）。

---

### 亮点 5：WebSocket + Disruptor 无锁队列实现实时协同编辑

**具体落点**

- WebSocket 处理器：`manager/websocket/PictureEditHandler.java`
- 握手拦截器：`manager/websocket/WsHandshakeInterceptor.java`（连接时注入 `user`、`pictureId`）
- Disruptor 配置：`manager/websocket/disruptor/PictureEditEventDisruptorConfig.java`（Ring Buffer 大小 256 * 1024）
- 事件生产者：`manager/websocket/disruptor/PictureEditEventProducer.java`
- 事件消费者：`manager/websocket/disruptor/PictureEditEventWorkHandler.java`
- 消息模型：`model/PictureEditRequestMessage.java` / `PictureEditResponseMessage.java`

**设计权衡**

WebSocket 消息接收和广播若在同一线程中串行执行，在高并发场景（多人同时编辑）容易成为瓶颈。引入 LMAX Disruptor 无锁队列：消息到来后生产者仅将事件写入 RingBuffer（纳秒级），消费者异步批量处理并广播，彻底解耦接收与广播，消除锁竞争，吞吐量远高于 `BlockingQueue`。

**如何验证**

打开两个浏览器标签页，均进入同一图片的编辑页面。在一个标签操作（移动/缩放图层），另一个标签页应实时同步。查看服务端日志，Disruptor 的 WorkHandler 线程（名称前缀 `pictureEditEventDisruptor`）会异步打印事件处理日志。

---

### 亮点 6：ShardingSphere 动态分库分表（按 spaceId 路由）

**具体落点**

- 自定义分片算法：`manager/sharding/PictureShardingAlgorithm.java`（实现 `StandardShardingAlgorithm<Long>`）
- 动态建表/更新路由：`manager/sharding/DynamicShardingManager.java`（`createSpacePictureTable`、`updateShardingTableNodes`）
- ShardingSphere 配置：`application.yml`（`spring.shardingsphere.*`）

**设计权衡**

随着团队空间图片数量增长，单表查询性能会下降。项目为"旗舰版团队空间"动态创建 `picture_{spaceId}` 分表，写入和查询按 `spaceId` 路由，做到空间级数据隔离，互不影响。通过 ShardingSphere 的 ContextManager API 在运行时动态更新 `actual-data-nodes`，无需重启服务即可扩容新分表。

**如何验证**

创建一个旗舰版团队空间，观察数据库中生成 `picture_{spaceId}` 新表；向该空间上传图片，通过 ShardingSphere 日志（`sql-show: true`）观察 SQL 被路由到对应分表。

---

### 亮点 7：以图搜图（Facade 门面模式封装外部 API）

**具体落点**

- 门面类：`api/imagesearch/ImageSearchApiFacade.java`
- 子步骤：`api/imagesearch/sub/GetImagePageUrlApi.java`、`GetImageFirstUrlApi.java`、`GetImageListApi.java`
- 调用处：`controller/PictureController.java`（`searchPictureByPicture` 方法）

**设计权衡**

以图搜图需要三步：获取搜索页 URL → 获取第一张图 URL → 获取图片列表。三步分别实现为独立的 API 子类，通过门面类 `ImageSearchApiFacade` 统一对外暴露一个 `searchPicture` 方法，Controller 无需了解内部分步细节，符合**单一职责**和**门面模式**。

**如何验证**

调用 `POST /api/picture/search/picture`，传入任意图片 URL，观察返回相似图片列表；在 `ImageSearchApiFacade` 打断点，逐步跟踪三个子步骤的执行。

---

### 亮点 8：全局统一异常处理 + 语义化错误码

**具体落点**

- 错误码枚举：`exception/ErrorCode.java`
- 业务异常：`exception/BusinessException.java`
- 全局拦截：`exception/GlobalExceptionHandler.java`（`@RestControllerAdvice`）
- 工具类：`exception/ThrowUtils.java`（断言式抛出，减少 if-throw 样板代码）

**设计权衡**

统一错误码让前端可以按 `code` 做精确的错误提示，同时方便监控系统按错误码聚合告警。`ThrowUtils` 提供断言式 API（如 `ThrowUtils.throwIf(condition, ErrorCode.PARAMS_ERROR, "msg")`），消除大量 `if-throw` 样板代码，代码更简洁，可读性高。

---

## 四、面试用项目介绍材料

### 项目简介

> 基于 Spring Boot 2 + Vue 3 + 腾讯云 COS 的企业级智能协同云图库平台。支持公共图库的上传/检索/审核、私有空间的个人图片管理，以及企业团队空间的实时协同图片编辑（WebSocket），同时集成阿里云 AI 扩图能力。后端采用传统 MVC 与 DDD 双架构版本。

### 技术栈（后端）

| 分类 | 技术 |
|------|------|
| 核心框架 | Spring Boot 2.7.6 / JDK 8 |
| 持久层 | MySQL 8 + MyBatis-Plus 3.5.9 |
| 缓存 | Redis（分布式缓存 + Spring Session） + Caffeine（本地缓存） |
| 对象存储 | 腾讯云 COS + 数据万象 CI（图片压缩/缩略图） |
| 权限 | Sa-Token 1.39 + 自定义 RBAC（空间角色权限 JSON 配置） |
| 实时协同 | Spring WebSocket + LMAX Disruptor 无锁队列 |
| 分库分表 | ShardingSphere-JDBC 5.2（按 spaceId 动态分表） |
| AI 能力 | 阿里云 AI 扩图大模型 API |
| 接口文档 | Knife4j（OpenAPI 2） |
| 架构设计 | MVC（传统版） + DDD（领域驱动版） |

### 你负责的内容可如何描述

> 参与企业级智能协同云图库平台的后端全程设计与开发，主要工作包括：
>
> 1. **图片上传链路设计**：基于模板方法模式设计 `PictureUploadTemplate` 抽象基类，统一"本地文件上传"和"URL 抓取上传"两种来源的处理流程，结合腾讯云 COS 数据万象实现上传时自动压缩与缩略图生成。
> 2. **多级缓存方案落地**：在图片列表查询接口引入 Caffeine + Redis 两级缓存，本地缓存最大 10000 条（5 分钟 TTL），Redis 缓存分布式共享，查询延迟降低显著。
> 3. **RBAC 权限体系**：基于 Sa-Token 多 `StpLogic` 机制设计空间成员权限体系（viewer / editor / admin），权限配置外置 JSON 文件，支持无代码修改地调整权限矩阵。
> 4. **实时协同编辑**：基于 Spring WebSocket + LMAX Disruptor 无锁队列实现团队成员实时协同编辑图片，消息接收与广播解耦，高并发下吞吐量优于 `BlockingQueue`。
> 5. **动态分库分表**：基于 ShardingSphere-JDBC 为旗舰版团队空间动态创建 `picture_{spaceId}` 分表，通过 ContextManager API 运行时扩容新分表，无需重启。
> 6. **DDD 架构重构**：将传统 MVC 项目重构为 DDD 四层（interfaces / application / domain / infrastructure），提升大型模块的可维护性与可测试性。

### 难点与解决方案

| 难点 | 解决方案 |
|------|---------|
| 多人并发编辑时消息广播延迟高 | 引入 LMAX Disruptor 无锁 RingBuffer，将消息接收与广播异步解耦，消除锁竞争 |
| 图片上传来源多样（文件/URL），代码重复 | 模板方法模式抽象公共上传流程，子类只实现差异化步骤 |
| 空间图片量大导致单表查询慢 | ShardingSphere 动态分表，按 `spaceId` 路由，旗舰版空间数据完全隔离 |
| 列表接口高频访问，DB 压力大 | Caffeine + Redis 两级缓存，进程内命中率高，避免频繁网络 I/O |
| 空间角色权限逻辑复杂、易错 | 权限矩阵外置 JSON 配置，运行时加载，避免硬编码，支持灵活调整 |

### 结果指标（可替代表述）

- 图片列表查询在缓存命中时响应时间从 DB 查询的数十毫秒降低至本地缓存的亚毫秒级（提升 10x+）。
- 基于 ShardingSphere 动态分表，理论上支持按空间数水平扩展图片存储，单空间查询不随全局数据量增长而劣化。
- WebSocket + Disruptor 架构在多用户并发编辑场景下，消息处理延迟稳定在个位数毫秒，满足实时协同体验要求。
- 图片上传借助 COS 数据万象端侧处理，服务端 CPU 消耗接近零，上传处理能力随 COS 带宽线性扩展。
