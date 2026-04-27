# 智能协同云图库 —— Java 后端学习路径（Windows 环境）

> 目标：在 Windows 本地把 `yu-picture-backend` 跑起来，读懂主流程，具备二次开发能力。
> 预计总耗时：3 天（每天 4～6 小时）

---

## 零、前置准备（首次运行只做一次）

### 0.1 安装必要工具

| 工具 | 版本要求 | 说明 |
|------|---------|------|
| JDK | 1.8（必须） | 项目 `pom.xml` 指定 `<java.version>1.8</java.version>` |
| Maven | 3.8+ | 或直接使用项目根目录下 `mvnw.cmd`（免安装） |
| IntelliJ IDEA | 2022+ | Community 版即可；推荐 Ultimate（Spring 支持更好） |
| Docker Desktop for Windows | 最新版 | 用于一键启动 MySQL + Redis |
| Apifox / Postman | 任意版本 | 接口测试 |

**IDEA 必装插件**

- Lombok（处理 `@Data` 等注解）
- MybatisX（Mapper ↔ XML 跳转）

### 0.2 配置 JAVA_HOME（如尚未配置）

```cmd
# 以管理员身份打开命令提示符
setx JAVA_HOME "C:\Program Files\Java\jdk1.8.0_xxx" /M
setx Path "%Path%;%JAVA_HOME%\bin" /M
# 重开命令窗口验证
java -version
```

---

## 一、第 1 天：跑起来（目标：本地能通一个核心接口）

### 1.1 启动依赖服务（MySQL + Redis）

项目根目录提供了 `docker-compose.yml`，直接运行：

```cmd
# 在项目根目录（yu-picture/）执行
docker compose up -d
```

验证启动：

```cmd
docker ps
# 应看到 yu_mysql（3306）和 yu_redis（6379）两个容器
```

### 1.2 初始化数据库

数据库 DDL 脚本位于 `yu-picture-backend/sql/create_table.sql`。

```cmd
# 进入 MySQL 容器
docker exec -it yu_mysql mysql -uroot -p123456

# 在 MySQL shell 中执行（把路径替换为你实际的项目目录，例如：）
# source C:/projects/yu-picture/yu-picture-backend/sql/create_table.sql
# 或者直接用 IDEA 的 Database 工具、Navicat 等打开并运行该 SQL 文件（推荐）
```

> **注意**：`create_table.sql` 包含多条 `ALTER TABLE` 语句，请**完整执行**，否则缺少列会导致启动后接口报错。

### 1.3 复制并填写配置

```cmd
# 复制示例配置（已在本仓库提供）
copy docs\application-dev.yml.example yu-picture-backend\src\main\resources\application-dev.yml
```

打开 `application-dev.yml`，**最少需要修改以下内容**：

```yaml
# COS 对象存储（必填，否则上传图片会报错）
cos:
  client:
    host: https://your-bucket-name.cos.ap-xxx.myqcloud.com
    secretId: YOUR_SECRET_ID
    secretKey: YOUR_SECRET_KEY
    region: ap-xxx
    bucket: your-bucket-name

# 阿里云 AI（可选，不填则 AI 扩图功能不可用）
aliYunAi:
  apiKey: YOUR_API_KEY
```

> **说明**：MySQL 和 Redis 使用 `docker-compose.yml` 默认配置（root/123456，127.0.0.1:3306/6379），无需修改。

### 1.4 在 IDEA 中启动后端

1. 打开 IDEA → `Open` → 选择 `yu-picture-backend/` 目录（不要选根目录）。
2. 等待 Maven 依赖下载完成（首次约 5～15 分钟）。
3. 打开 `YuPictureBackendApplication.java`，在类名上右键 → `Run`。
4. 或者在 `Edit Configurations` 中添加 VM 参数：
   ```
   -Dspring.profiles.active=dev
   ```

**成功标志**：控制台出现类似日志：

```
Started YuPictureBackendApplication in 12.345 seconds
```

### 1.5 验证接口

打开浏览器访问接口文档：

```
http://localhost:8123/api/doc.html
```

用 Apifox/Postman 测试第一个接口：

```http
GET http://localhost:8123/api/
响应：{"code":0,"data":"ok","message":""}
```

---

### 1.6 本地可运行检查清单

- [ ] `docker ps` 看到 MySQL + Redis 容器正在运行
- [ ] `create_table.sql` 已完整执行，数据库中有 `user`、`picture`、`space`、`space_user` 等表
- [ ] IDEA 启动无 `SQLException` / `ConnectionRefused` 报错
- [ ] `GET http://localhost:8123/api/` 返回 `{"code":0,"data":"ok"}`
- [ ] Knife4j 文档页面正常加载

---

## 二、第 2 天：读懂主流程（目标：能回答"一张图片是怎么上传进来的"）

### 2.1 分层地图（先建立整体认知）

按以下顺序浏览各层目录（不要急着读每行代码，先看类名/方法名）：

```
1. model/entity/     → 4 个核心实体：User、Picture、Space、SpaceUser
2. mapper/           → 4 个 Mapper 接口（MyBatis-Plus）
3. service/          → 5 个 Service 接口（看方法签名）
4. controller/       → 5 个 Controller（看接口路径和参数）
5. manager/          → 基础设施：upload、auth、sharding、websocket
6. config/           → 配置：COS、Redis、MyBatis-Plus、CORS
7. exception/        → 错误码 ErrorCode、全局处理 GlobalExceptionHandler
```

### 2.2 核心链路：图片上传（精读，约 2 小时）

**文件阅读顺序**：

```
1. controller/PictureController.java
   → 找方法 uploadPicture()（约第 92 行）
   → 看参数：MultipartFile + PictureUploadRequest + HttpServletRequest

2. service/impl/PictureServiceImpl.java
   → 找方法 uploadPicture()（约第 108 行）
   → 关键步骤：校验空间额度 → 选择上传模板 → 上传 COS → 构造 Picture 实体 → 事务写库 → 更新空间用量

3. manager/upload/PictureUploadTemplate.java
   → 看 uploadPicture() 主流程（模板方法）
   → 看 abstract 方法：validPicture / getOriginFilename / processFile

4. manager/upload/FilePictureUpload.java
   → 看 processFile() 如何把 MultipartFile 写入临时文件
   → 看 validPicture() 校验文件类型/大小

5. manager/CosManager.java
   → 看 putPictureObject() 如何调用 COS SDK
   → 看返回的 CIObject 列表（压缩图 + 缩略图）
```

**在你的笔记里写下这条调用链**：

```
PictureController.uploadPicture()
  └─ PictureServiceImpl.uploadPicture()
       ├─ spaceService.getById()            // 校验空间
       ├─ FilePictureUpload.uploadPicture() // 上传到 COS
       │    ├─ validPicture()               // 校验
       │    ├─ processFile()                // MultipartFile → 临时文件
       │    └─ CosManager.putPictureObject()// 上传 + CI 处理
       ├─ transactionTemplate.execute()     // 事务：写 picture 表 + 更新 space 用量
       └─ PictureVO.objToVo()              // 返回 VO
```

### 2.3 次要链路：图片列表（带缓存，约 1 小时）

- `PictureController.listPictureVOByPageWithCache()` — 两级缓存查询
- `PictureServiceImpl.getQueryWrapper()` — 动态查询条件构建（多字段模糊/精确查询）

### 2.4 权限体系（约 1 小时）

- `manager/auth/SpaceUserAuthManager.java` — `getPermissionList()` 如何根据空间类型和角色返回权限列表
- `resources/biz/spaceUserAuthConfig.json` — 角色权限配置（JSON）
- `manager/auth/StpInterfaceImpl.java` — Sa-Token 回调接口
- `manager/auth/annotation/SaSpaceCheckPermission.java` — 注解定义

### 2.5 知识点自查（读完第 2 天后能回答）

1. `PictureUploadTemplate` 用了什么设计模式？为什么这样设计？
2. 图片列表接口缓存 Key 是如何生成的？过期时间是多少？
3. `spaceUserAuthConfig.json` 中 `editor` 角色有哪些权限？
4. 公共图库和私有空间在上传路径上有什么区别？（见 `PictureServiceImpl.java` 约第 159 行）

---

## 三、第 3 天：具备二次开发能力（目标：能稳定加功能并验证）

### 3.1 练习 1（低风险）：修改图片上传大小限制

**目标**：将最大上传文件从 10 MB 改为 20 MB，同时修改业务层校验。

**改动点定位**：

1. `application.yml`（或 `application-dev.yml`）：
   ```yaml
   spring:
     servlet:
       multipart:
         max-file-size: 20MB   # 改这里
   ```

2. `manager/upload/FilePictureUpload.java`：
   找 `validPicture()` 方法，修改文件大小校验阈值（当前约为 `2 * 1024 * 1024` 或类似常量）。

**验证方式**：尝试上传一个 15 MB 的图片文件，期望上传成功而非被拒绝。

---

### 3.2 练习 2（中等难度）：图片列表新增"文件大小"排序支持

**目标**：让 `GET /api/picture/list/page/vo` 支持按 `picSize` 字段排序。

**改动点定位**：

1. `model/dto/picture/PictureQueryRequest.java`：确认是否已有 `sortField`/`sortOrder` 字段（可能已存在）。

2. `service/impl/PictureServiceImpl.java`：找 `getQueryWrapper()` 方法，在 QueryWrapper 构建处增加：
   ```java
   wrapper.orderBy(StrUtil.isNotBlank(sortField),
       "ascend".equals(sortOrder), sortField);
   ```

3. **安全校验**：在接受 `sortField` 前，加白名单校验（防止 SQL 注入）：
   ```java
   List<String> ALLOWED_SORT_FIELDS = Arrays.asList("picSize", "createTime", "editTime");
   ThrowUtils.throwIf(!ALLOWED_SORT_FIELDS.contains(sortField), ErrorCode.PARAMS_ERROR, "非法排序字段");
   ```

**验证方式**：请求 `?sortField=picSize&sortOrder=descend`，响应列表应按文件从大到小排序（可通过 SQL 日志确认 `ORDER BY` 子句）。

---

### 3.3 练习 3（完整功能）：实现图片"主色调"按颜色相似度搜索

**目标**：调用已有的 `ColorSimilarUtils` 实现按颜色相似度过滤图片列表。

**背景**：项目已在 `utils/ColorSimilarUtils.java` 和 `utils/ColorTransformUtils.java` 中实现了主色调标准化与相似度计算逻辑；`picture` 表已有 `picColor` 字段。

**改动点定位**：

1. `controller/PictureController.java`：参考已有的 `searchPictureByColor` 方法（若已存在），或新增接口：
   ```java
   @PostMapping("/search/color")
   public BaseResponse<List<PictureVO>> searchPictureByColor(
       @RequestBody SearchPictureByColorRequest request,
       HttpServletRequest httpServletRequest)
   ```

2. `model/dto/picture/`：新增 `SearchPictureByColorRequest.java`（含 `picColor`、`spaceId` 字段）。

3. `service/PictureService.java` + `service/impl/PictureServiceImpl.java`：
   - 查出同 space 下所有有 `picColor` 的图片
   - 用 `ColorSimilarUtils.calculateSimilarity()` 计算相似度
   - 按相似度降序排序，返回前 N 条

**验证方式**：上传几张图片，调用颜色搜索接口传入一个颜色值（如 `0xFF0000` 红色），检查返回的图片主色调与目标颜色是否接近。

---

## 四、开发调试技巧

### 断点调试（IDEA）

1. 在 `PictureController.uploadPicture()` 方法入口打断点。
2. IDEA 以 Debug 模式启动应用（Shift+F9 或点 Debug 按钮）。
3. 用 Apifox 发送上传请求，IDEA 会在断点处暂停，可逐步 Step Into 追踪。

### 查看 SQL 日志

`application.yml` 中已配置：
```yaml
mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

控制台会打印所有执行的 SQL，方便排查 ORM 问题。

### ShardingSphere SQL 路由日志

```yaml
spring:
  shardingsphere:
    props:
      sql-show: true
```

控制台会打印逻辑 SQL → 物理 SQL 的路由结果，可观察分表路由是否正确。

### 接口文档

启动后访问：`http://localhost:8123/api/doc.html`（Knife4j UI）

---

## 五、常见问题排查

| 问题现象 | 可能原因 | 解决方法 |
|---------|---------|---------|
| 启动报 `Communications link failure` | MySQL 未启动 | 运行 `docker compose up -d`，确认容器状态 |
| 启动报 `Connection refused 6379` | Redis 未启动 | 同上 |
| 上传图片报 `CosClientException` | COS 配置未填 | 检查 `application-dev.yml` 中 `cos.client.*` 是否正确 |
| Knife4j 页面 404 | 端口或路径错误 | 确认访问 `http://localhost:8123/api/doc.html` |
| 接口返回 `{"code":40100}` | 未登录 | 先调用 `/api/user/login` 接口获取 Session |
| 接口返回 `{"code":40101}` | 权限不足 | 确认用户角色（管理员接口需 `userRole=admin`） |
| ShardingSphere 报 `Table or view not found` | 分表未创建 | 先创建普通空间测试，旗舰版空间才会触发建表 |
