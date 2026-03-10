# 营小助配置中心实施计划

> 关联 Spec：spec/smart-operation/spec.md
> 创建日期：2026-03-10
> 状态：草稿

## 1. 技术方案概述

营小助配置中心采用前后端分离架构，后端使用 Spring Boot + Java 提供配置管理、规则引擎和 API 服务，前端使用 React + Ant Design 构建可视化配置中心，浏览器插件基于现有插件扩展实现页面识别、规则匹配和自动作业能力。系统采用 TDSQL（生产）和 H2（开发）存储配置数据和审计日志，开发环境使用 H2 数据库无需安装即可快速启动，Redis 用于缓存和规则分发，Elasticsearch 用于高频规则触发日志的写入与检索，确保配置在插件侧快速生效并支撑运行日志查询。一期优先交付智能提示（哨卫）完整能力，包括后端的页面资源管理、哨卫规则引擎、规则提示管理和哨卫浏览器插件。

## 2. 技术决策

### 2.1 方案选型

#### 后端框架选择
| 方方案 | 优势 | 劣势 | 最终选择 |
|------|------|------|---------|
| Spring Boot + Java | 银行内部常用技术栈，成熟稳定，生态完善，安全机制健全 | 启动较慢，内存占用较高 | ✅ 选择 |
| Node.js + NestJS | TypeScript 全栈，开发效率高 | 银行内部支持度可能不足 | - |
| Go + Gin/Echo | 高性能，部署简单 | 银行生态支持较弱 | - |

**理由**：银行内部项目首选 Spring Boot，符合现有技术栈标准，安全性和稳定性要求得到保障。

#### 前端框架选择
| 方案 | 优势 | 劣势 | 最终选择 |
|------|------|------|---------|
| React + Ant Design | 企业级 UI 组件库，拖拽编排可用 React Flow，生态成熟 | 学习曲线相对陡峭 | ✅ 选择 |
| Vue 3 + Element Plus | 上手简单，Vue Flow 支持编排 | 银行内部 React 占比较高 | - |
| Angular + NG-ZORRO | 完整框架，适合大型适合应用 | 学习曲线陡峭，开发效率较低 | - |

**理由**：React + Ant Design 是当前银行前端主流技术栈，React Flow 提供完善的拖拽编排能力。

#### 浏览器插件方案
| 方案 | 优势 | 劣势 | 最终选择 |
|------|------|------|---------|
| 基于现有插件扩展 | 复用现有代码，开发周期短，与业务系统兼容性高 | 受限于现有架构 | ✅ 选择 |
| Chrome Extension Manifest V3 | 技术栈现代，官方支持 | 开发工作量大，Manifest V2 迁移成本高 | - |
| Chrome Extension Manifest V2 | 兼容性好 | 即将废弃，不建议新项目使用 | - |

**理由**：CLAUDE.md 提到既有浏览器插件，在现有基础上扩展可大幅降低开发成本和风险。

#### 数据库方案
| 方案 | 优势 | 劣势 | 最终选择 |
|------|------|------|---------|
| TDSQL (生产) + H2 (开发) + Redis | 生产使用 TDSQL，开发使用 H2 无需安装数据库，Redis 缓存性能好 | H2 与 TDSQL 存在 SQL 方言差异 | ✅ 选择 |
| PostgreSQL + Redis | JSON 类型支持好，查询能力强 | 银行内部支持度可能不足 | - |
| Oracle + Redis | 银行常用，安全性高 | 授权成本高，开发效率相对较低 | - |

**理由**：TDSQL 是银行生产环境标准数据库，开发环境使用 H2 可快速启动无需安装数据库，通过 Spring Profile 区分环境，开发效率高。

#### 规则触发日志存储方案
| 方案 | 优势 | 劣势 | 最终选择 |
|------|------|------|---------|
| Elasticsearch | 适合高频写入、全文检索与多条件查询，便于承载规则触发日志 | 运维复杂度高于单库表存储 | ✅ 选择 |
| TDSQL 业务表存储 | 架构简单，便于统一管理 | 高写入量和复杂检索压力大 | - |
| Redis 临时存储 | 写入快 | 不适合作为长期审计检索存储 | - |

**理由**：规则触发日志写入量大、查询维度多，使用 Elasticsearch 更适合承担高频运行日志存储与检索压力；规则触发日志主体与字段匹配明细合并存入同一个索引文档，简化写入链路与查询模型。

#### 场景编排引擎选择
| 方案 | 优势 | 劣势 | 最终选择 |
|------|------|------|---------|
| React Flow | 成熟的流程编排库，支持自定义节点，社区活跃 | 文档相对简单 | ✅ 选择 |
| X6 (AntV) | 功能强大，文档完善 | 学习成本较高，包体积较大 | - |
| jsPlumb | 轻量级 | 功能相对简单，自定义节点复杂 | - |

**理由**：React Flow 与 React 技术栈深度集成，提供拖拽编排所需的核心功能，社区活跃度高。

#### 规则引擎选择
| 方案 | 优势 | 劣势 | 最终选择 |
|------|------|------|---------|
| JavaScript 规则引擎 | 轻量级，灵活性强，适合前端执行 | 性能相对较低 | ✅ 选择 |
| Drools (后端 Java) | 功能强大，性能优秀 | 重量级，规则需后端执行 | - |
| 自研规则引擎 | 完全自定义 | 开发成本高，稳定性待验证 | - |

**理由**：用户明确选择 JavaScript 规则引擎，支持在浏览器插件侧执行规则匹配，减少后端压力。

#### 脚本沙箱方案
| 方案 | 优势 | 劣势 | 最终选择 |
|------|------|------|---------|
| 首期不实现沙箱 | 开发成本低，快速交付 | 安全性不足 | ✅ 选择 |
| vm2 / isolated-vm | 轻量级沙箱 | vm2 已废弃，isolated-vm 兼容性问题 | - |
| Web Worker 隔离 | 安全性高 | 性能开销大，通信复杂 | - |

**理由**：用户明确选择首期不实现沙箱，后续迭代再补充安全限制。

#### 项目结构选择
| 方案 | 优势 | 劣势 | 最终选择 |
|------|------|------|---------|
| Multi-repo (多仓库) | 模块独立，权限控制精细，团队分工清晰 | 依赖管理复杂，版本协调困难 | ✅ 选择 |
| Monorepo | 统一版本管理，依赖共享 | 权限控制复杂，仓库体积大 | - |
| 单仓库 | 简单直接 | 缺乏模块隔离，不适合大型项目 | - |

**理由**：用户明确选择多仓库结构，符合银行内部项目组织习惯，便于权限管理和团队分工。

### 2.2 外部依赖

#### 后端依赖
- Spring Boot 3.x（基础框架）
- Spring Security（安全认证）
- MyBatis-Plus（ORM 框架）
- H2 Database（开发环境数据库，可选）
- TDSQL/MySQL Connector（生产环境数据库驱动）
- Redisson（Redis 客户端）
- Lombok（代码简化）
- Jackson（JSON 序列化）
- Hutool（工具类库）
- Logback（日志框架）

### 2.4 本地开发环境配置

#### 2.4.1 H2 数据库配置（开发环境）

开发环境使用 H2 数据库，无需安装数据库即可启动项目，通过 Spring Profile 区分开发和生产环境。

**application-dev.yml 配置：**
```yaml
spring:
  datasource:
    # 使用 H2 内存数据库，MODE=MySQL 模拟 MySQL 语法
    url: jdbc:h2:mem:yxzconfig;MODE=MySQL;DB_CLOSE_DELAY=-1
    # 或使用文件模式（数据持久化到本地文件）：jdbc:h2:file:./data/yxzconfig;MODE=MySQL
    driver-class-name: org.h2.Driver
    username: sa
    password:
  h2:
    console:
      enabled: true  # 启用 H2 Console 管理界面，访问地址：http://localhost:8080/h2-console
      path: /h2-console
  jpa:
    hibernate:
      ddl-auto: update  # 开发环境自动更新表结构
```

**application-prod.yml 配置：**
```yaml
spring:
  datasource:
    # 生产环境使用 TDSQL（或 MySQL）
    url: jdbc:mysql://tdsql-host:3306/yxzconfig
    # 或：jdbc:mysql://mysql-host:3306/yxzconfig
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate  # 生产环境仅验证表结构，不自动更新
```

**application.yml（环境选择）：**
```yaml
spring:
  profiles:
    active: dev  # 开发环境使用 dev profile
```

#### 2.4.2 Maven 依赖配置

```xml
<dependencies>
    <!-- H2 数据库（仅开发环境） -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>

    <!-- MyBatis-Plus -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    </dependency>

    <!-- TDSQL/MySQL 驱动（生产环境） -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

#### 2.4.3 H2 与 TDSQL 兼容性注意事项

| 注意事项 | 说明 | 建议 |
|---------|------|------|
| **SQL 方言差异** | H2 使用 MySQL 模式但不完全兼容 TDSQL 特有语法 | 使用标准 SQL 语法，关键 SQL 在 TDSQL 环境验证 |
| **数据类型** | 某些 TDSQL 特有数据类型 H2 不支持 | 使用标准 SQL 类型（VARCHAR、BIGINT、INT、TEXT 等） |
| **索引/约束** | 复杂索引语法可能不同 | 使用 MyBatis-Plus 注解或简单 SQL |
| **函数差异** | TDSQL 特有函数 H2 不支持 | 尽量使用标准 SQL 函数 |
| **字符集** | H2 默认 UTF-8，TDSQL 可能不同 | 统一使用 UTF-8 字符集 |
| **事务隔离** | 隔离级别可能不同 | 使用 Spring 默认事务配置 |
| **自增策略** | H2 和 TDSQL 自增主键生成策略不同 | 使用 MyBatis-Plus 的 IdType.AUTO 自动适配 |

#### 2.4.4 推荐的开发环境替代方案

如果希望更接近生产环境，可以考虑以下方案：

**方案 A：使用 Docker MySQL**
```bash
# 运行 MySQL 容器（兼容 TDSQL）
docker run -d \
  --name yxzconfig-db \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=yxzconfig \
  -e MYSQL_CHARSET=utf8mb4 \
  mysql:8.0
```

**方案 B：使用 Testcontainers（推荐用于测试）**
```java
@Testcontainers
@SpringBootTest
class UserRepositoryTest {
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("yxzconfig")
            .withUsername("test")
            .withPassword("test");
}
```

#### 2.4.5 环境切换指南

| 操作 | 命令 | 说明 |
|------|------|------|
| 启动开发环境 | `mvn spring-boot:run -Dspring-boot.run.profiles=dev` | 使用 H2 内存数据库 |
| 启动生产环境 | `mvn spring-boot:run -Dspring-boot.run.profiles=prod` | 使用 TDSQL/MySQL 数据库 |
| 查看开发环境 H2 Console | 访问 http://localhost:8080/h2-console | 可视化管理 H2 数据库 |

#### 前端依赖
- React 18.x（UI 框架）
- Ant Design 5.x（UI 组件库）
- React Flow 11.x（场景编排）
- Axios（HTTP 请求）
- React Router DOM（路由管理）
- Zustand / Redux Toolkit（状态管理）
- React Hook Form（表单管理）
- Monaco Editor（代码编辑器）
- Day.js（日期处理）

#### 浏览器插件依赖
- Chrome Extension API（插件 API）
- React 18.x（UI 框架，与前端保持一致）
- Ant Design 5.x（UI 组件库）
- JavaScript 规则引擎（规则匹配）

### 2.3 内部依赖

#### 外部系统依赖
- TDSQL 数据库（生产环境）
- H2 数据库（开发环境，本地模拟）
- Redis 缓存
- Elasticsearch（规则触发日志检索）
- 统一身份认证平台（SSO）
- 统一权限管理平台
- HR 系统（员工身份标签数据，含"新员工"标识）
- 组织架构系统（用户、部门树、用户部门归属）
- 业务系统接口（页面编码 API、业务数据接口）

#### 模块间依赖关系
- 智能提示（哨卫）子系统：页面资源管理、系统管理、规则引擎、规则提示管理
- 智能作业子系统 → 智能提示（哨卫）子系统（事件产出模块，只读依赖）
- 哨卫浏览器插件 → 哨卫配置后台
- 智能作业浏览器插件 → 智能作业配置后台、哨卫浏览器插件（事件消费）

## 3. 任务拆解

### 3.1 智能提示（哨卫）子系统 - 后端配置中心

#### 3.1.1 基础架构与系统管理

- [ ] **TODO-B1: 项目基础架构搭建**
  - **描述**：搭建哨卫后端项目基础架构，包括 Spring Boot 项目初始化、MyBatis-Plus 配置、Redis 配置、统一异常处理、统一返回结果、统一日志配置、CORS 配置
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：新建 backend/sentry/pom.xml、backend/sentry/src/main/java/com/yxzconfig/sentry/、application.yml
  - **依赖**：无
  - **验收标准**：项目可正常启动，健康检查接口返回正常，日志正常输出

- [ ] **TODO-B2: 统一权限与认证集成**
  - **描述**：集成统一身份认证平台（SSO）和统一权限管理平台，实现用户登录、token 验证、角色权限校验、部门信息获取
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/security/、filter/
  - **依赖**：TODO-B1
  - **验收标准**：未登录用户无法访问受保护接口，已登录用户可正常访问，权限校验生效

- [ ] **TODO-B3: 数据库表结构设计与创建**
  - **描述**：设计并创建数据库表结构，包括监控网站表、监控网页表、本页元素表、用户表、部门表、角色表、权限表、规则表、规则条件表、规则提示配置表、规则状态变更记录表、版本记录表、审计日志表等核心表，使用 MyBatis-Plus 代码生成器生成实体类和 Mapper；规则触发日志及其字段匹配明细不落 TDSQL，统一写入单个 Elasticsearch 索引文档；其中规则提示配置依附规则存在，不单独建启停对象；条件模型底层允许多个条件标记可重复触发，但首期产品层仅开放单个条件配置
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/resources/db/migration/、backend/sentry/src/main/java/com/yxzconfig/sentry/entity/
  - **依赖**：TODO-B1
  - **验收标准**：数据库表创建成功，实体类和 Mapper 生成正确，可正常 CRUD；规则、规则提示配置、规则状态变更记录边界清晰；规则触发日志写入 Elasticsearch 成功并可检索；不存在“提示任务”“提示独立启停”相关表

- [ ] **TODO-B4: 系统管理 API - 用户与组织管理**
  - **描述**：实现用户和组织管理接口，从外部系统同步用户、部门树、用户部门归属、员工身份标签（含"新员工"标识），提供用户查询、部门查询、部门树查询接口
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/UserManagementController.java、service/UserManagementService.java
  - **依赖**：TODO-B3
  - **验收标准**：用户和组织数据正确同步，查询接口正常返回

- [ ] **TODO-B5: 系统管理 API - 角色与权限管理**
  - **描述**：实现角色和权限管理接口，定义系统角色（配置管理员、业务配置人员、审核人员、只读审计人员、分行管理人员、总行管理人员），并实现角色权限配置、用户角色绑定、权限校验
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/RoleManagementController.java、service/RoleManagementService.java
  - **依赖**：TODO-B4
  - **验收标准**：角色和权限配置正确，权限校验生效

- [ ] **TODO-B6: 版本管理基础服务**
  - **描述**：实现通用版本管理服务，支持配置对象的版本生成、版本查询、版本回滚、版本对比、版本发布记录，配置对象生命周期包括草稿、待发布、已发布、已下线
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/service/VersionManagementService.java、entity/VersionRecord.java
  - **依赖**：TODO-B3
  - **验收标准**：版本生成正确，发布记录完整，回滚功能正常

- [ ] **TODO-B7: 审计日志基础服务**
  - **描述**：实现通用审计日志服务，记录高风险操作（发布、回滚、下线）、配置变更、规则触发、提示展示等操作日志，支持按角色权限隔离查询，审计字段包括操作人、操作时间、操作对象、操作结果、前后版本信息
  - **涉及模块模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/service/AuditLogService.java、entity/AuditLog.java
  - **依赖**：TODO-B3
  - **验收标准**：审计日志正确记录，查询权限隔离生效，高风险操作完整留痕

#### 3.1.2 页面资源管理

- [ ] **TODO-P1: 页面资源管理 API - 监控网站管理**
  - **描述**：实现监控网站的 CRUD 接口，包括创建网站、编辑网站、删除网站（校验引用）、查询网站列表、启用/停用网站、权限校验（仅归属部门可编辑）
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/MonitoringWebsiteController.java、service/MonitoringWebsiteService.java
  - **依赖**：TODO-B3
  - **验收标准**：接口功能正常，权限校验生效，被引用的网站无法删除

- [ ] **TODO-P2: 页面资源管理 API - 监控网页管理**
  - **描述**：实现监控网页的 CRUD 接口，包括创建网页（绑定监控网站）、编辑网页（URL 匹配模式、页面编码、描述）、删除网页（校验引用）、查询网页列表、启用/停用网页，URL 匹配模式支持精确/通配/正则三种
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/MonitoringWebpageController.java、service/MonitoringWebpageService.java
  - **依赖**：TODO-P1
  - **验收标准**：接口功能正常，URL 匹配规则正确，被引用的网页无法删除

- [ ] **TODO-P3: 页面资源管理 API - 本页元素管理**
  - **描述**：实现本页元素的 CRUD 接口，包括创建元素（绑定监控网页、元素逻辑名、选择器、元素类型）、编辑元素、删除元素（校验被规则/场景引用）、查询元素列表、元素类型支持文本/下拉/单选/多选/日期等，并支持元素跨部门共享（只读复用、Fork 复制）
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/PageElementController.java、service/PageElementService.java
  - **依赖**：TODO-P2
  - **验收标准**：接口功能正常，元素类型校验正确，共享机制生效，被引用的元素无法删除

- [ ] **TODO-P4: 页面资源管理 API - 页面识别服务**
  - **描述**：实现页面识别服务接口，支持插件通过页面编码（API 获取）或 URL 匹配查找对应监控网页配置，优先使用页面编码，页面编码不可用时使用 URL 保底匹配
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/PageIdentificationController.java、service/PageIdentificationService.java
  - **依赖**：TODO-P2
  - **验收标准**：插件可通过页面编码正确识别网页，页面编码不可用时通过 URL 匹配正确识别

#### 3.1.3 规则引擎核心

- [ ] **TODO-R1: 规则引擎核心 - 条件模型设计**
  - **描述**：设计哨卫规则引擎核心数据模型，包括规则、条件、左值/右值来源（元素/固定值/接口）、运算符、预处理链、条件逻辑（AND/OR），规则状态包括草稿、已发布、已停用、已过期
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/entity/Rule.java、Condition.java、ValueSource.java、Operator.java
  - **依赖**：TODO-B3
  - **验收标准**：数据模型设计完整，支持 spec 要求的所有字段和状态

- [ ] **TODO-R2: 规则引擎核心 - 预处理器管理**
  - **描述**：实现预处理器管理功能，包括内置预处理器（字符串处理、日期格式化、数值转换、正则匹配等）和自定义预处理器管理，预处理器支持参数 schema 定义、脚本分类
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/service/PreprocessorService.java、entity/Preprocessor.java
  - **依赖**：TODO-R1
  - **验收标准**：内置预处理器正常工作，自定义预处理器可管理，参数 schema 校验正确

- [ ] **TODO-R3: 规则引擎核心 - 接口定义管理**
  - **描述**：实现接口定义管理功能，包括接口 CRUD、请求参数定义、入参来源配置、响应取值路径配置、超时配置（默认 5 秒）、重试配置，接口调用由服务端统一代调，启用 SSRF 防护（域名白名单、内网地址限制、回环地址限制）
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/InterfaceDefinitionController.java、service/InterfaceDefinitionService.java、service/ApiClientService.java
  - **依赖**：TODO-B1
  - **验收标准**：接口定义管理正常，服务端代调功能正常，SSRF 防护生效

- [ ] **TODO-R4: 规则引擎核心 - 规则模板管理**
  - **描述**：实现规则模板管理功能，包括模板 CRUD、占位元素定义、模板条件、模板提示，规则模板可通过占位元素映射复用到目标页面，映射时必须完成所有占位元素映射，未完成映射时禁止保存
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/RuleTemplateController.java、service/RuleTemplateService.java
  - **依赖**：TODO-R1
  - **验收标准**：规则模板管理正常，占位元素映射校验正确，映射完整校验生效

- [ ] **TODO-R5: 规则引擎核心 - 规则 CRUD API**
  - **描述**：实现规则的 CRUD 接口，包括创建规则（绑定监控网页）、编辑规则、删除规则（校验引用）、查询规则列表、启用/停用规则、规则状态流转（草稿→已发布、已发布→已停用、过期自动失效）
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/RuleController.java、service/RuleService.java
  - **依赖**：TODO-R1
  - **验收标准**：规则 CRUD 正常，状态流转正确，被引用的规则无法删除

- [ ] **TODO-R6: 规则引擎核心 - 规则匹配服务**
  - **描述**：实现规则匹配服务，支持插件传入页面上下文（页面 ID、元素值、用户信息），服务端执行规则匹配逻辑，包括条件运算（AND/OR/）、预处理链执行、接口调用、值比较，返回命中规则列表
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/RuleMatchingController.java、service/RuleMatchingService.java
  - **依赖**：TODO-R2, TODO-R3, TODO-R5
  - **验收标准**：规则匹配逻辑正确，预处理链正常执行，接口调用正常，返回结果正确

#### 3.1.4 规则提示管理

- [ ] **TODO-S1: 规则提示配置管理 API**
- **描述**：在规则管理基础上实现规则内提示配置管理能力，提示配置依附规则存在，不作为独立配置对象；支持维护提示内容（标题、正文、处理建议）、提示类型（静默、浮窗、外部通知）、提示标签、处理优先级、生效范围（部门及其子部门、指定用户）、适用节点、有效期、责任人、维护分行；三种提示类型命中后均必须写入规则触发日志，其中静默类型不展示浮窗，外部通知类型按配置执行站外通知，具体对接细节不在本计划展开
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/RulePromptController.java、service/RulePromptService.java
  - **依赖**：TODO-R5
- **验收标准**：规则提示配置管理正常，提示类型分类正确（静默、浮窗、外部通知），标签维护正常，生效范围计算正确（部门及子部门递归、指定用户生效）

- [ ] **TODO-S2: 规则提示管理 - 总行强控规则**
  - **描述**：实现总行强控规则功能，总行管理人员可下发全行强控提示规则，强控规则全行生效，分行仅可查看，不可停用或修改
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/service/HeadquartersStrongControlService.java
  - **依赖**：TODO-S1
  - **验收标准**：总行强控规则下发正常，分行权限隔离生效

- [ ] **TODO-S3: 规则管理 - 分行规则启停管理**
  - **描述**：实现分行管理人员规则启停管理功能，分行管理人员可对本分行规则执行启用、停用、延期操作；该能力维护规则在分行维度的生效治理状态，所有状态变化必须生成规则操作审计记录
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/BranchRuleManagementController.java、service/BranchRuleManagementService.java
  - **依赖**：TODO-S1
  - **验收标准**：分行规则启停管理正常，权限校验生效，规则操作审计完整且不影响规则内容本身

- [ ] **TODO-S4: 规则提示管理 - 提示冲突与优先级**
- **描述**：实现提示冲突与优先级处理逻辑，同一页面同一时刻命中多条规则时，按规则优先级排序；浮窗类型进入串行展示队列，静默类型只记日志不进入浮窗队列，外部通知类型不进入浮窗队列；标签仅用于检索和筛选，不参与运行时优先级计算；同一规则的重复提示需去重
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/service/PromptConflictResolutionService.java
  - **依赖**：TODO-S1
  - **验收标准**：提示优先级排序正确，重复提示去重生效

- [ ] **TODO-S5: 插件配置拉取 API**
  - **描述**：实现插件配置拉取接口，插件传入 userId/deptId/URL/localVersion，服务端返回当前用户在当前页面有效期内的规则集（含提示配置）、页面元素元数据，支持增量拉取（版本号比对）
）
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/controller/PluginConfigPullController.java、service/PluginConfigPullService.java
  - **依赖**：TODO-R5, TODO-S1
  - **验收标准**：插件可正常拉取配置，增量拉取生效，版本比对正确

- [ ] **TODO-S6: 规则触发日志记录**
- **描述**：实现规则触发日志记录功能，记录规则 ID、规则编码、规则名称、命中时规则版本、提示类型、触发原因、触发页面、操作人、分行、触发时间、处理结果等主字段；三种提示类型命中后均必须写入日志，成功触发时在同一 Elasticsearch 文档内记录当次所有字段匹配值；其中静默类型记录“已静默处理”，浮窗类型记录“已展示/已关闭”等处理结果，外部通知类型记录对应处理结果；该日志与规则操作审计分开存储，并统一写入单个 Elasticsearch 索引文档，支持按规则、页面、分行、操作人、时间范围检索
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：backend/sentry/src/main/java/com/yxzconfig/sentry/service/RuleTriggerLogService.java、backend/sentry/src/main/java/com/yxzconfig/sentry/service/RuleTriggerLogIndexService.java
  - **依赖**：TODO-S1
  - **验收标准**：规则触发日志正确记录，成功触发字段明细完整，单索引写入与按规则/页面/分行/时间维度查询功能正常

### 3.2 智能提示（哨卫）子系统 - 哨卫浏览器插件

- [ ] **TODO-E1: 哨卫插件项目初始化**
  - **描述**：创建哨卫浏览器插件项目，基于现有插件扩展，配置 manifest.json、背景脚本、内容脚本、弹出页面，设置权限（storage、activeTab、tabs），采用常驻监听模式并支持在命中页面时进入实际工作状态
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：新建 plugins/sentry/manifest.json、background.js、content.js、popup.html
  - **依赖**：无
  - **验收标准**：插件可正常安装和加载，基础功能正常

- [ ] **TODO-E2: 页面识别引擎**
  - **描述**：实现页面识别引擎，插件进入页面后先调用页面编码 API 获取页面标识，页面编码不可用时使用 URL 匹配规则查找对应监控网页配置，支持精确/通配/正则三种匹配模式，并兼容传统多页跳转和 SPA 路由切换场景
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：plugins/sentry/src/content/PageIdentificationEngine.js
  - **依赖**：TODO-E1, TODO-P4
  - **验收标准**：页面识别正确，页面编码优先逻辑生效，URL 保底匹配正确

- [ ] **TODO-E3: 元素解析引擎**
  - **描述**：实现元素解析引擎，根据页面配置的元素选择器解析页面元素，获取元素当前值，支持文本/下拉/单选/多选/日期等元素类型，对缺失或不可读元素标记为异常
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：plugins/sentry/src/content/ElementParser.js
  - **依赖**：TODO-E2
  - **验收标准**：元素解析正确，各种元素类型支持支持正常，异常元素标记正确

- [ ] **TODO-E4: 配置拉取与缓存管理**
  - **描述**：实现配置拉取与缓存管理，插件进入页面后拉取当前用户在当前页面的配置（规则及其提示配置、元素元数据），采用“进页面拉取一次”的模式；支持本地缓存（localStorage，按监控网页维度缓存，单页面 100KB），支持增量拉取（版本号比对），缓存连续可用时长 60 分钟
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：plugins/sentry/src/content/ConfigPullManager.js
  - **依赖**：TODO-E1, TODO-S5
  - **验收标准**：配置拉取正常，缓存机制生效，增量拉取正确，超时降级正确

- [ ] **TODO-E5: 规则匹配执行器（前端）**
  - **描述**：实现前端规则匹配执行器，使用 JavaScript 规则引擎，执行条件运算（AND/OR）、预处理链执行、接口调用（服务端代理）、值比较，支持失败策略（失败不触发），记录失败原因日志；页面首次加载时执行一次全量判定，字段变化时仅继续判定未命中规则，已命中规则默认过滤；若规则中的某个条件标记为可重复触发，则在该条件涉及字段 DOM 实际取值发生变化后，将整条规则重新放入待触发队列。底层设计允许多个条件标记，但首期仅开放单个条件配置
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：plugins/sentry/src/content/RuleMatchingExecutor.js
  - **依赖**：TODO-E3, TODO-E4
  - **验收标准**：规则匹配逻辑正确，预处理正常执行，接口调用正常，失败策略生效

- [ ] **TODO-E6: 提示渲染组件**
- **描述**：实现提示渲染组件，仅处理浮窗类型提示；支持页面内浮框展示，提示内容包括标题、正文、处理建议，支持 XSS 白名单过滤（允许 p、br、strong、em、ul、ol、li、a 标签），同一时刻命中多条浮窗提示时按优先级排序串行展示，支持用户关闭操作；静默与外部通知类型不进入该组件
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：plugins/sentry/src/content/PromptRenderer.js
  - **依赖**：TODO-E1
- **验收标准**：浮窗提示展示正确，XSS 过滤生效，优先级排序正确，串行展示正常；静默与外部通知类型不会误进入浮窗渲染

- [ ] **TODO-E7: 事件产出模块**
  - **描述**：实现事件产出模块，页面变化触发规则命中后，产出触发事件，包含页面上下文（页面 ID、URL）、当前页面所有已配置元素的完整元数据（逻辑名、选择器、当前值）、触发规则信息，供智能作业插件消费
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：plugins/sentry/src/content/EventEmitter.js
  - **依赖**：TODO-E3, TODO-E5
  - **验收标准**：事件产出正确，元数据完整，智能作业插件可正常消费

- [ ] **TODO-E8: 页面变化监听**
  - **描述**：实现页面变化监听，基于 DOM 监听监听页面加载、字段变化、URL 变化等事件，触发规则匹配和事件产出；支持条件级可重复触发配置，即当被标记为可重复触发的条件所涉及字段 DOM 实际取值发生变化时，将对应规则重新加入待触发队列；并兼容传统多页与 SPA 页面
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：plugins/sentry/src/content/PageChangeMonitor.js
  - **依赖**：TODO-E5, TODO-E7
  - **验收标准**：页面变化监听正常，规则匹配触发正确，重复触发控制生效

- [ ] **TODO-E9: 规则触发日志上报**
- **描述**：实现规则触发日志上报，规则命中并完成对应处理后上报规则 ID、提示类型、触发原因、触发页面、操作人、分行信息、处理结果以及当次命中的字段匹配值到服务端；静默、浮窗、外部通知三种提示类型均必须上报日志
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：plugins/sentry/src/content/PromptTriggerLogReporter.js
  - **依赖**：TODO-E6, TODO-S6
- **验收标准**：日志上报正常，服务端正确记录

- [ ] **TODO-E10: 插件异常处理与降级**
  - **描述**：实现插件异常处理与降级，网络异常、配置异常、页面结构变化时给出可理解提示，自动降级为手工模式，支持一键重试，不影响用户继续手工作业
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：plugins/sentry/src/content/ExceptionHandler.js
  - **依赖**：TODO-E4, TODO-E5
  - **验收标准**：异常处理正确，降级机制生效，重试功能正常

### 3.3 智能提示（哨卫）子系统 - 前端配置中心

- [ ] **TODO-F1: 前端配置中心项目初始化**
  - **描述**：创建前端配置中心项目，使用 React + Ant Design，配置路由、状态管理（Zustand）、HTTP 请求（Axios）、统一异常处理、统一 Loading、统一消息提示
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：新建 frontend/config-center/package.json、src/App.tsx、src/router/、src/store/
  - **依赖**：无
  - **验收标准**：项目可正常启动，路由配置正确，状态管理正常

- [ ] **TODO-F2: 登录与权限校验**
  - **描述**：实现登录页面和权限校验，集成统一身份认证平台（SSO），登录成功后保存 token，实现路由守卫和接口权限校验，根据角色控制菜单和按钮显示
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Login/、src/components/AuthGuard.tsx
  - **依赖**：TODO-F1
  - **验收标准**：登录功能正常，权限校验生效，菜单和按钮权限控制正确

- [ ] **TODO-F3: 监控网站管理页面**
  - **描述**：实现监控网站管理页面，包括网站列表展示、创建/编辑网站弹窗、删除确认、启用/停用操作，展示网站名称、域名、归属部门、可见范围，支持按部门权限过滤
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Shared/MonitoringWebsite/、src/services/MonitoringWebsiteService.ts
  - **依赖**：TODO-F2, TODO-P1
  - **验收标准**：页面功能正常，增删改查正确，权限过滤生效

- [ ] **TODO-F4: 监控网页管理页面**
  - **描述**：实现监控网页管理页面，包括网页列表展示（按网站分组）、创建/编辑网页弹窗、删除确认、启用/停用操作，展示页面编码、URL 匹配规则、页面描述，URL 匹配模式支持精确/通配/正则三种
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Shared/MonitoringWebpage/、src/services/MonitoringWebpageService.ts
  - **依赖**：TODO-F3, TODO-P2
  - **验收标准**：页面功能正常，URL 匹配模式支持正确，增删改查正确

- [ ] **TODO-F5: 本页元素管理页面**
  - **描述**：实现本页元素管理页面，包括元素列表展示（按网页分组）、创建/编辑元素弹窗、删除确认，展示元素逻辑名、选择器、元素类型、备注、是否必填，支持元素跨部门共享（只读复用、Fork 复制）
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center-center/src/pages/Shared/PageElement/、src/services/PageElementService.ts
  - **依赖**：TODO-F4, TODO-P3
  - **验收标准**：页面功能正常，元素类型支持正确，共享机制生效

- [ ] **TODO-F6: 规则模板管理页面**
  - **描述**：实现规则模板管理页面，包括模板列表展示、创建/编辑模板弹窗、删除确认，展示模板名称、占位元素列表、模板条件、模板提示，支持占位元素添加/删除/编辑
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Sentry/RuleTemplate/、src/services/RuleTemplateService.ts
  - **依赖**：TODO-F2, TODO-R4
  - **验收标准**：页面功能正常，占位元素管理正确，增删改查正确

- [ ] **TODO-F7: 接口定义管理页面**
  - **描述**：实现接口定义管理页面，包括接口列表展示、创建/编辑接口弹窗、删除确认、测试接口功能，展示接口名称、请求方法、请求 URL、请求参数、响应取值路径、超时配置、重试配置
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Sentry/InterfaceDefinition/、src/services/InterfaceDefinitionService.ts
  - **依赖**：TODO-F2, TODO-R3
  - **验收标准**：页面功能正常，接口参数配置正确，测试功能正常

- [ ] **TODO-F8: 预处理器管理页面**
  - **描述**：实现预处理器管理页面，包括预处理器列表展示（内置预处理器只读、自定义预处理器可编辑）、创建/编辑自定义预处理器弹窗、删除确认，展示预处理器名称、分类、参数 schema、脚本内容（使用 Monaco Editor）
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Sentry/Preprocessor/、src/services/PreprocessorService.ts
  - **依赖**：TODO-F2, TODO-R2
  - **验收标准**：页面功能正常，内置预处理器不可编辑，自定义预处理器管理正确

- [ ] **TODO-F9: 规则配置页面**
  - **描述**：实现规则配置页面，包括规则列表展示、创建/编辑规则弹窗、删除确认、发布/停用操作，展示规则名称、绑定的监控网页、条件列表、条件逻辑、提示配置、生效范围、有效期、状态
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Sentry/Rule/、src/services/RuleService.ts
  - **依赖**：TODO-F5, TODO-F6, TODO-F7, TODO-R5
  - **验收标准**：页面功能正常，条件配置正确，引用选择正常，状态流转正确

- [ ] **TODO-F10: 规则编辑器 - 条件配置**
  - **描述**：实现规则编辑器中的条件配置功能，支持添加/删除/编辑条件，配置左值/右值来源（元素/固定值/接口）、运算符（等于、不等于、包含、不包含、大于、小于等）、预处理链，支持条件逻辑（AND/OR）
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Sentry/Rule/RuleEditor.tsx
  - **依赖**：TODO-F9
  - **验收标准**：条件配置正确，来源选择正常，预处理链配置正确，逻辑配置正确

- [ ] **TODO-F11: 规则编辑器 - 提示配置**
- **描述**：实现规则编辑器中的提示配置功能，配置提示内容（富文本编辑器）、提示类型（静默、浮窗、外部通知）、提示标签、优先级、处理方式、适用范围、有效期
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Sentry/Rule/RuleEditor.tsx
  - **依赖**：TODO-F9
- **验收标准**：提示配置正确，富文本编辑正常，提示类型选择正确（静默、浮窗、外部通知）

- [ ] **TODO-F12: 规则提示管理页面**
- **描述**：实现规则提示管理页面，面向已配置规则管理其提示配置，包括规则列表展示、提示内容编辑、发布/停用操作，展示规则名称、提示类型、提示标签、提示内容、生效范围、责任人、维护分行、状态，支持总行强控规则标记、分行规则启停管理
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Sentry/RulePrompt/、src/services/RulePromptService.ts
  - **依赖**：TODO-F2, TODO-S1
  - **验收标准**：页面功能正常，规则提示配置展示和编辑正确，总行强控标记正确，分行规则启停正确

- [ ] **TODO-F13: 提示审计查询页面**
- **描述**：实现提示审计查询页面，支持按规则、提示类型、提示标签、分行、页面、操作人、时间范围检索提示全生命周期记录，展示规则触发日志、规则操作记录，支持导出
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Sentry/PromptAudit/、src/services/PromptAuditService.ts
  - **依赖**：TODO-F2, TODO-S6
  - **验收标准**：查询功能正常，多条件过滤正确，导出功能正常

- [ ] **TODO-F14: 版本管理页面**
  - **描述**：实现版本管理页面，展示配置对象的版本历史、版本对比、版本发布记录，支持版本回滚操作，回滚需记录回滚原因、操作人、时间与影响范围
  - **涉及模块**：智能提示（哨卫）子系统
  - **涉及文件**：frontend/config-center/src/pages/Shared/VersionManagement/、src/services/VersionManagementService.ts
  - **依赖**：TODO-F2, TODO-B6
  - **验收标准**：版本历史展示正确，版本对比正确，回滚功能正常

## 4. 依赖关系与执行顺序

### 后端任务依赖关系
```
TODO-B1 → TODO-B2
TODO-B1 → TODO-B3
TODO-B1 → TODO-R3
TODO-B3 → TODO-B4
TODO-B3 → TODO-B5
TODO-B3 → TODO-B6
TODO-B3 → TODO-B7
TODO-B3 → TODO-P1
TODO-B3 → TODO-R1
TODO-P1 → TODO-P2 → TODO-P3
TODO-P2 → TODO-P4
TODO-R1 → TODO-R2
TODO-R1 → TODO-R4
TODO-R1 → TODO-R5
TODO-R2 → TODO-R6
TODO-R3 → TODO-R6
TODO-R5 → TODO-R6
TODO-R5 → TODO-S1
TODO-S1 → TODO-S2
TODO-S1 → TODO-S3
TODO-S1 → TODO-S4
TODO-R5 → TODO-S5
TODO-S1 → TODO-S5
TODO-S1 → TODO-S6
```

### 哨卫浏览器插件任务依赖关系
```
TODO-E1
TODO-P4 → TODO-E2
TODO-E2 → TODO-E3
TODO-E1 → TODO-E4
TODO-S5 → TODO-E4
TODO-E3 → TODO-E5
TODO-E4 → TODO-E5
TODO-E1 → TODO-E6
TODO-E3 → TODO-E7
TODO-E5 → TODO-E7
TODO-E5 → TODO-E8
TODO-E7 → TODO-E8
TODO-E6 → TODO-E9
TODO-S6 → TODO-E9
TODO-E4 → TODO-E10
TODO-E5 → TODO-E10
```

### 前端配置中心任务依赖关系
```
TODO-F1
TODO-F1 → TODO-F2
TODO-F2 → TODO-F3
TODO-P1 → TODO-F3
TODO-F3 → TODO-F4
TODO-P2 → TODO-F4
TODO-F4 → TODO-F5
TODO-P3 → TODO-F5
TODO-F2 → TODO-F6
TODO-R4 → TODO-F6
TODO-F2 → TODO-F7
TODO-R3 → TODO-F7
TODO-F2 → TODO-F8
TODO-R2 → TODO-F8
TODO-F5 → TODO-F9
TODO-F6 → TODO-F9
TODO-F7 → TODO-F9
TODO-R5 → TODO-F9
TODO-F9 → TODO-F10
TODO-F9 → TODO-F11
TODO-F2 → TODO-F12
TODO-S1 → TODO-F12
TODO-F2 → TODO-F13
TODO-S6 → TODO-F13
TODO-F2 → TODO-F14
TODO-B6 → TODO-F14
```

### 可并行执行的任务组
```
组 1（可并行）：TODO-B1, TODO-F1, TODO-E1
组 2（B1 完成后可并行）：TODO-B2, TODO-B3, TODO-R3
组 3（B3 完成后可并行）：TODO-P1, TODO-R1
组 4（B3、F1 完成后可并行）：TODO-P4, TODO-F2
组 5（B3 完成后可并行）：TODO-B4, TODO-B5, TODO-B6, TODO-B7
组 6（P1 完成后可并行）：TODO-P2, TODO-F3
组 7（P2 完成后可并行）：TODO-P3, TODO-F4
组 8（P3 完成后可并行）：TODO-P4, TODO-F5
组 9（R1 完成后可并行）：TODO-R2, TODO-R4, TODO-R5
组 10（R3 完成后可并行）：TODO-R6
组 11（R5 完成后可并行）：TODO-S1
组 12（S1 完成后可并行）：TODO-S2, TODO-S3, TODO-S4, TODO-S5, TODO-S6
组 13（R5、S1 完成后可并行）：TODO-S5
组 14（F2 完成后可并行）：TODO-F6, TODO-F7, TODO-F8
组 15（R4 完成后可并行）：TODO-F6
组 16（R3 完成后可并行）：TODO-F7
组 17（R2 完成后可并行）：TODO-F8
组 18（P4、E1 完成后可并行）：TODO-E2
组 19（S5、E1 完成后可并行）：TODO-E4
组 20（R2、R3、R5 完成后可并行）：TODO-R6
```

## 5. 测试标准

### 5.1 单元测试标准

#### 后端单元测试
- **TODO-B1**：健康检查接口返回正常，日志输出正常，异常处理生效
- **TODO-B2**：未登录访问受保护接口返回 401，权限校验失败返回 403，token 解析正确
- **TODO-B3**：实体类和 Mapper 生成正确，CRUD 操作正常，数据库约束生效
- **TODO-B4**：用户和组织数据同步正确，查询接口正常返回
- **TODO-B5**：角色和权限配置正确，权限校验生效，用户角色绑定正确
- **TODO-B6**：版本生成正确，版本回滚正常，版本对比正常，版本发布记录完整
- **TODO-B7**：审计日志记录正确，查询权限隔离生效，高风险操作完整留痕
- **TODO-P1**：监控网站 CRUD 正常，权限校验生效（非归属部门无法编辑），被引用的网站无法删除
- **TODO-P2**：监控网页 CRUD 正常，URL 匹配规则正确（精确/通配/正则），被引用的网页无法删除
- **TODO-P3**：本页元素 CRUD 正常，元素类型校验正确，共享机制生效（只读、Fork），被引用的元素无法删除
- **TODO-P4**：页面识别服务正确（页面编码优先、URL 保底），URL 匹配规则正确
- **TODO-R1**：规则数据模型完整，条件模型完整，值来源模型完整，运算符枚举正确
- **TODO-R2**：内置预处理器正常工作，自定义预处理器管理正常，参数 schema 校验正确
- **TODO-R3**：接口定义管理正常，服务端代调功能正常，SSRF 防护生效（域名白名单、内网地址限制、回环地址限制）
- **TODO-R4**：规则模板管理正常，占位元素映射正确，映射完整校验生效
- **TODO-R5**：规则 CRUD 正常，状态流转正确（草稿→已发布、已发布→已停用），被引用的规则无法删除
- **TODO-R6**：规则匹配逻辑正确，AND/OR 逻辑正确，预处理链执行正常，接口调用正常，失败策略生效
- **TODO-S1**：规则提示配置管理正常，提示类型分类正确（静默、浮窗、外部通知），标签维护正常，生效范围计算正确（部门及子部门递归、指定用户生效）
- **TODO-S2**：总行强控规则下发正常，分行权限隔离生效（分行不可停用总行强控规则）
- **TODO-S3**：分行规则启停管理正常，权限校验生效（仅分行管理人员可操作），规则操作审计完整，且不修改规则内容
- **TODO-S4**：提示优先级排序正确（按规则优先级），标签不参与运行时排序，重复提示去重生效
- **TODO-S5**：插件配置拉取正常，增量拉取生效（版本号比对），用户和部门过滤正确
- **TODO-S6**：规则触发日志记录正确，处理结果字段语义准确，成功触发字段明细完整，单个 Elasticsearch 索引检索正常，且可与规则操作记录区分查询

#### 哨卫浏览器插件单元测试
- **TODO-E1**：插件可正常安装和加载，manifest.json 配置正确
- **TODO-E2**：页面识别正确，页面编码优先逻辑生效，URL 保底匹配正确（精确/通配/正则）
- **TODO-E3**：元素解析正确，各种元素类型支持正常（文本/下拉/单选/多选/日期），异常元素标记正确
- **TODO-E4**：配置拉取正常，进页面拉取一次生效，缓存机制生效（按监控网页维度、localStorage、100KB 限制），增量拉取正确，超时降级正确（60 分钟）
- **TODO-E5**：规则匹配逻辑正确，AND/OR 逻辑正确，预处理正常执行，接口调用正常，失败策略生效（失败不触发），首次加载全量判定与字段变化增量判定逻辑正确；条件级可重复触发时，相关字段 DOM 实际取值变化后规则重新进入待触发队列；首期 UI 仅开放单个可重复触发条件配置
- **TODO-E6**：浮窗提示展示正确，XSS 过滤生效（仅允许安全标签），优先级排序正确，串行展示正常，静默与外部通知类型不会误渲染
- **TODO-E7**：事件产出正确，元数据完整（页面 ID、URL、元素逻辑名、选择器、当前值），智能作业插件可正常消费
- **TODO-E8**：页面变化监听正常（页面加载、字段变化、URL 变化），基于 DOM 监听工作正常，传统多页与 SPA 页面兼容；条件级可重复触发时，相关字段 DOM 实际取值变化可正确驱动规则重新入队
- **TODO-E9**：日志上报正常，服务端正确写入 Elasticsearch 并记录成功触发字段明细
- **TODO-E10**：异常处理正确，降级机制生效，重试功能正常，不影响用户手工作业

#### 前端配置中心单元测试
- **TODO-F1**：项目启动正常，路由配置正确，状态管理正常
- **TODO-F2**：登录功能正常，权限校验生效，菜单和按钮权限控制正确
- **TODO-F3**：监控网站管理页面功能正常，增删改查正确，权限过滤生效
- **TODO-F4**：监控网页管理页面功能正常，URL 匹配模式支持正确，增删改查正确
- **TODO-F5**：本页元素管理页面功能正常，元素类型支持正确，共享机制生效
- **TODO-F6**：规则模板管理页面功能正常，占位元素管理正确，增删改查正确
- **TODO-F7**：接口定义管理页面功能正常，接口参数配置正确，测试功能正常
- **TODO-F8**：预处理器管理页面功能正常，内置预处理器不可编辑，自定义预处理器管理正确
- **TODO-F9**：规则配置页面功能正常，条件配置正确，引用选择正常，状态流转正确
- **TODO-F10**：规则编辑器条件配置正确，来源选择正常，预处理链配置正确，逻辑配置正确
- **TODO-F11**：规则编辑器提示配置正确，富文本编辑正常，提示类型选择正确（静默、浮窗、外部通知）
- **TODO-F12**：规则提示管理页面功能正常，规则提示配置展示和编辑正确，总行强控标记正确，分行规则启停正确
- **TODO-F13**：提示审计查询页面查询功能正常，可区分规则触发日志与规则操作记录，多条件过滤正确，导出功能正常
- **TODO-F14**：版本管理页面版本历史展示正确，版本对比正确，回滚功能正常

### 5.2 集成 / 场景验证标准

#### 场景 1：页面识别与配置拉取
- **前置条件**：已配置监控网站、监控网页、本页元素，插件已安装
- **操作步骤**：用户打开已配置的业务页面
- **期望结果**：插件正确识别页面，进入页面时拉取对应配置，缓存机制生效，页面加载额外耗时不超过 2000ms

#### 场景 2：规则匹配与提示展示
- **前置条件**：已配置规则（绑定页面、条件、提示内容），插件已安装并拉取配置
- **操作步骤**：用户在页面操作，触发规则匹配条件
- **期望结果**：规则匹配正确，提示按优先级排序展示，提示内容正确，XSS 过滤生效

#### 场景 3：规则触发日志记录
- **前置条件**：已配置规则，插件已安装
- **操作步骤**：用户触发规则命中，提示展示
- **期望结果**：规则触发日志正确上报，服务端写入 Elasticsearch 并记录完整（规则 ID、提示类型、触发原因、触发页面、操作人、分行、处理结果、成功触发字段匹配值）

#### 场景 4：多提示冲突处理
- **前置条件**：已配置多条不同处理类型的提示规则（静默、浮窗、外部通知），插件已安装
- **操作步骤**：用户触发多条提示规则命中
- **期望结果**：多条提示按规则优先级排序，标签不影响运行时排序，重复提示去重，串行展示正常

#### 场景 5：分行规则启停管理
- **前置条件**：已配置规则提示内容，分行管理人员已登录
- **操作步骤**：分行管理人员执行规则启用/停用/延期操作
- **期望结果**：操作成功，权限校验生效（仅分行管理人员可操作），生成规则操作审计记录且不改动规则内容

#### 场景 6：总行强控规则下发
- **前置条件**：总行管理人员已登录
- **操作步骤**：总行管理人员创建并发布全行强控规则
- **期望结果**：规则全行生效，分行仅可查看，不可停用或修改

#### 场景 7：规则模板使用
- **前置条件**：已创建规则模板（含占位元素），已配置本页元素
- **操作步骤**：配置人员选择规则模板，完成占位元素映射到目标页面元素
- **期望结果**：映射完整校验生效，规则生成正确，保存成功

#### 场景 8：接口定义与代调
- **前置条件**：已创建接口定义，规则引用接口
- **操作步骤**：规则匹配时触发接口调用
- **期望结果**：服务端代调功能正常，SSRF 防护生效，接口返回值正确参与规则匹配

#### 场景 9：元素共享与 Fork
- **前置条件**：已配置本页元素（归属部门 A），部门 B 获得共享权限
- **操作步骤**：部门 B 配置人员查看共享元素，选择 Fork 复制到本部门
- **期望结果**：共享元素仅可查看，Fork 复制成功，不影响原元素

#### 场景 10：版本回滚
- **前置条件**：已发布规则版本 V1，发布新版本 V2
- **操作步骤**：配置管理人员执行回滚到 V1，填写回滚原因
- **期望结果**：回滚成功，规则回退到 V1，回滚原因、操作人、时间与影响范围记录完整

#### 场景 11：插件异常处理与降级
- **前置条件**：插件已安装，配置服务短时不可用
- **操作步骤**：用户打开页面，插件拉取配置失败
- **期望结果**：插件使用缓存继续运行，缓存不可用时降级为手工模式，提示用户，不影响手工作业

#### 场景 12：页面结构变化处理
- **前置条件**：插件已安装并拉取配置
- **操作步骤**：目标页面结构发生变化，元素选择器失效
- **期望结果**：元素解析失败，标记为异常，规则匹配失败（失败不触发），提示用户，不影响手工作业

#### 场景 13：超时缓存降级
- **前置条件**：插件已安装并拉取配置
- **操作步骤**：插件与服务端断连超过 60 分钟
- **期望结果**：插件停止自动执行，降级为手工模式，提示"当前配置可能已过期"

#### 场景 14：审计查询
- **前置条件**：已执行多项高风险操作（发布、回滚、停用）
- **操作步骤**：审计人员按时间范围、操作人、操作类型检索审计记录
- **期望结果**：查询结果正确，高风险操作（发布、回滚、下线）完整留痕，越权角色不可见

#### 场景 15：权限校验
- **前置条件**：已配置多个角色（配置管理员、业务配置人员、审核人员、只读审计人员、分行管理人员、总行管理人员）
- **操作步骤**：不同角色用户登录，执行操作
- **期望结果**：权限校验生效，配置管理员可发布/回滚/下线，业务配置人员可编辑场景，审核人员可执行/查看结果，只读审计人员仅可查看，分行管理人员可管理本分行提示，总行管理人员可下发全行强控规则

## 6. 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 浏览器插件与业务系统不兼容 | 高 | 基于现有插件扩展，充分测试兼容性，提供降级机制 |
| 常驻监听导致浏览器资源占用增加 | 中 | 仅在命中页面或页面变化事件时进入实际工作状态，避免无关页面执行重逻辑 |
| 页面结构变化导致规则失效 | 高 | 提供异常处理和降级机制，提示配置人员更新选择器，后续迭代支持自动再写规则 |
| SPA 路由切换导致页面识别遗漏 | 中 | 将 URL 变化、DOM 变化、首屏加载统一纳入页面变化监听范围 |
| 规则引擎性能不足，影响用户体验 | 中 | 优化规则匹配算法，使用缓存，限制规则条件数量（1~10），设置超时时间 |
| 接口调用失败导致规则无法匹配 | 中 | 采用"失败不触发"策略，记录失败原因，配置重试机制 |
| XSS 攻击导致提示内容注入恶意脚本 | 高 | 使用 XSS 白名单过滤，仅允许安全标签，转义特殊字符 |
| SSRF 攻击导致内网泄露 | 高 | 启用 SSRF 防护，域名白名单，内网地址限制，回环地址限制 |
| 配置拉取失败导致插件无法工作 | 中 | 提供本地缓存机制，缓存连续可用 60 分钟，超过后降级为手工模式 |
| 权限配置不当导致越权操作 | 高 | 严格的权限校验，角色权限隔离，高风险操作审计留痕 |
| 版本回滚导致线上配置混乱 | 中 | 回滚以版本为单位，记录回滚原因和影响范围，提供回滚确认机制 |
| 多提示并发展示导致用户体验差 | 中 | 提示按优先级排序，串行展示，同类提示去重，避免重复干扰 |
| 首期不实现再箱导致脚本执行风险 | 中 | 限制脚本使用场景，后续迭代补充沙箱隔离能力 |
| 配置发布后插件侧未及时生效 | 中 | 配置发布后插件侧应在 10 分钟内感知并生效，使用增量拉取和版本比对 |
| 页面识别失败导致整个流程无法执行 | 中 | 页面编码优先，URL 保底，识别失败时降级为手工模式，不影响手工作业 |
| 大量配置导致插件拉取性能下降 | 中 | 使用增量拉取，版本号比对，单页面缓存限制 100KB，缓存自动清理最旧数据 |

## 7. 开放问题

无
