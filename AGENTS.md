# AGENTS.md

本文件面向在本仓库工作的 AI 编程助手和新贡献者，说明 ThingsBoard 仓库的结构、开发命令、代码规范和变更验证要求。优先遵循本文件；未覆盖的内容以 `CONTRIBUTING.md`、相邻代码和各模块 `pom.xml` 为准。

## 1. 项目概览

ThingsBoard 是一个用于设备接入、遥测采集、规则引擎、告警、可视化和设备管理的一体化 IoT 平台。当前仓库是 Maven 多模块单仓，版本见根目录 `pom.xml`（当前为 `4.4.0.2`）。

主要技术栈：

- 后端：Java 25、Spring Boot 3.5.x、Maven、Lombok、JUnit 4/5、Testcontainers。
- 前端：Angular 20、TypeScript 5.9、RxJS 7.8、Angular Material、NgRx、SCSS、Tailwind CSS。
- 前端工具链：Node.js `v22.22.2`、Yarn `v1.22.22`，Maven 构建可自动下载。
- 数据与中间件：PostgreSQL/TimescaleDB、Cassandra、Redis/Valkey、Kafka、MQTT/CoAP/LwM2M/SNMP/HTTP 等。
- 打包：Spring Boot Boot Jar、DEB/RPM/ZIP、Docker 镜像和微服务部署制品。

## 2. 环境要求

- JDK 25。
- Maven 3.9.x 较新版本；仓库根目录没有 Maven Wrapper。
- Docker Desktop/Engine：DAO、Application 集成测试和黑盒测试依赖 Testcontainers 或 Compose。
- Python 3：用于 YAML 配置描述检查和许可证头 provenance 检查。
- 直接在 `ui-ngx` 迭代时可本机安装 Yarn 1.x；只跑 Maven 构建时无需手动安装 Node/Yarn。

## 3. 仓库结构与模块边界

| 路径 | 职责 |
|---|---|
| `common/` | 共享领域模型、DAO API、消息、队列、传输 API、脚本、Protobuf、缓存、工具类等底层模块。 |
| `common/data/` | DTO、实体 ID、分页对象、枚举、异常、通用数据结构。 |
| `common/dao-api/` | DAO 和实体服务接口，避免上层直接依赖具体持久化实现。 |
| `common/message/`、`common/queue/` | 内部消息、队列抽象、服务类型条件注解等。 |
| `common/transport/` | MQTT、HTTP、CoAP、LwM2M、SNMP 传输的主要实现。 |
| `common/proto/` | `.proto` 定义、Protobuf/gRPC 相关转换代码。 |
| `dao/` | DAO API 的 SQL/NoSQL 实现、缓存、持久化服务和数据库 schema 资源。 |
| `rule-engine/rule-engine-api/` | 规则节点 SPI，如 `TbNode`、`TbContext`、配置接口。 |
| `rule-engine/rule-engine-components/` | 内置规则节点实现和规则节点配置。 |
| `transport/` | 各传输服务的 Spring Boot 启动模块；具体协议逻辑优先放在 `common/transport/`。 |
| `integration/` | 第三方平台集成实现和 integration executor。 |
| `application/` | 主服务/单体应用，包含 REST Controller、业务服务、Actor、安装升级逻辑和大量集成测试。 |
| `edqs/` | Entity Data Query Service。 |
| `ui-ngx/` | Angular Web UI；详细路由和前端架构见 `ui-ngx/structure.md`。 |
| `rest-client/` | 手写维护的 Java REST 客户端，核心入口是 `RestClient.java`。 |
| `netty-mqtt/` | 独立 Netty MQTT 客户端库。 |
| `msa/` | 微服务 Docker 打包、各服务镜像模块和黑盒测试。 |
| `monitoring/`、`migrator/`、`report/` | 独立监控、数据迁移、报表服务。 |
| `packaging/` | 由 Maven 调用的 Gradle 打包逻辑，用于 DEB/RPM/ZIP 等制品。 |
| `tools/` | 开发/CI 工具、Python 检查脚本、MQTT 示例工具。 |

推荐依赖方向：

1. 通用数据结构放 `common/data`，跨服务接口放 `common/dao-api`、`common/queue` 或对应 API 模块。
2. 持久化接口定义在 `common/dao-api`，具体实现放 `dao`。
3. 规则节点契约放 `rule-engine-api`，内置实现放 `rule-engine-components`。
4. 协议复用逻辑放 `common/transport/*`，`transport/*` 只保留启动装配和模块特有代码。
5. REST Controller 和应用编排逻辑放 `application`，不要让 `common` 或 `dao` 反向依赖 `application`。
6. 前端通过 `ui-ngx/src/app/core/http` 的服务访问 `/api`，组件不要直接散落调用 `HttpClient`。

## 4. 后端开发规范

### 4.1 通用风格

- Java 代码使用 4 空格缩进，保持与相邻文件一致的 import、注解和换行风格。
- 受 license plugin 管理的新源文件必须带 SPDX 许可证头；优先运行 `mvn license:format` 自动修复，不要手写猜测不同许可证头。
- 使用 Lombok 减少样板代码，例如 `@Slf4j`、`@RequiredArgsConstructor`、`@Getter`、`@Setter` 等；修改构造器注入时注意 Spring Bean 依赖关系。
- 项目已启用 Java 25 编译和 `-parameters`，但公共 API 仍应保持显式、可读、向后兼容。
- 优先复用 `common/util`、`JacksonUtil`、`StringUtils` 等现有工具，避免引入重复工具类。
- 不要大范围重新格式化未修改代码；不要修改 `target/`、生成代码或 `node_modules/`。
- 新增依赖必须先确认是否已有同类依赖，并检查依赖冲突和许可证影响；版本统一在根 `pom.xml` 管理。

### 4.2 REST Controller

- Controller 位于 `application/src/main/java/org/thingsboard/server/controller`。
- 典型类使用 `@RestController`、`@RequestMapping("/api")`、`@TbCoreComponent`。
- 每个端点必须有明确的 `@PreAuthorize` 权限控制，并补充 OpenAPI/Swagger 注解（如 `@ApiOperation`、`@Parameter`）。
- Controller 负责参数解析、权限和 HTTP 语义，复杂业务逻辑下沉到 service。
- 新增或修改 REST API 时，同步更新 `rest-client/src/main/java/org/thingsboard/rest/client/RestClient.java`，并在交付说明中提醒 Python client/API 文档是否需要同步。
- 保持 URL、请求/响应 DTO 和分页结构向后兼容；破坏性变更必须提供升级方案。

### 4.3 Service、DAO 和数据库变更

- 应用业务服务常见模式是接口 + `Default*` 实现，例如 `TbAlarmService` 与 `DefaultTbAlarmService`。
- 实体业务编排位于 `application/src/main/java/org/thingsboard/server/service/entitiy`；注意包名 `entitiy` 是仓库现状，不要顺手重命名。
- DAO 接口放在 `common/dao-api`，JPA/SQL/NoSQL 实现放在 `dao`。
- 数据库 schema 资源位于 `dao/src/main/resources/sql` 和 `dao/src/main/resources/cassandra`。
- 涉及 schema、视图、函数、索引或升级路径时，同时检查 `application/src/main/java/org/thingsboard/server/service/install` 下的 schema/upgrade service，以及 `DefaultDatabaseSchemaSettingsService` 中支持升级版本。
- 任何持久化行为变更都要考虑旧版本数据升级、租户隔离、缓存失效和回滚风险。

### 4.4 微服务条件注解

新增 Spring Bean 时，根据其运行位置选择条件注解，而不是只使用裸 `@Component`：

- `@TbCoreComponent`：单体或 `tb-core` 中运行。
- `@TbRuleEngineComponent`：规则引擎服务。
- `@TbTransportComponent` 以及 MQTT/CoAP/LwM2M/SNMP 对应传输注解：传输服务。
- `@TbIntegrationComponent`、`@TbIntegrationExecutorComponent`：集成相关服务。
- `@TbReportComponent`、`@TbVersionControlComponent` 等：对应独立服务。

这些注解定义在 `common/queue/src/main/java/org/thingsboard/server/queue/util`，通过 `service.type` 控制 Bean 是否加载。

### 4.5 规则节点

- 内置规则节点位于 `rule-engine/rule-engine-components`。
- 实现类使用 `@RuleNode` 描述组件类型、名称、配置类、UI directive、图标和文档地址，并实现 `TbNode`。
- 配置类实现项目要求的配置接口，提供默认配置；节点初始化在 `init(TbContext, TbNodeConfiguration)` 中完成。
- 消息成功/失败应通过 `TbContext#tellSuccess`、`TbContext#tellFailure` 等明确传递。
- 异步回调必须处理成功和失败路径，并避免阻塞规则引擎线程。
- 新增规则节点通常需要后端测试、前端配置组件、静态 UI 资源和用户文档说明。

### 4.6 传输与集成

- `transport/mqtt`、`transport/http`、`transport/coap`、`transport/lwm2m`、`transport/snmp` 主要是启动模块；可复用协议逻辑应放到 `common/transport/*`。
- 集成技术按目录拆分，例如 Kafka、MQTT、RabbitMQ、OPC UA、AWS、Azure、Pub/Sub 等；公共契约放 `common/integration/*`。
- 协议和集成代码要特别关注连接生命周期、背压、超时、重试、线程安全和资源释放。

## 5. 前端开发规范

前端详细导航见 `ui-ngx/structure.md`。核心路径：

- `src/app/core/http`：按实体/领域拆分的 HTTP 服务。
- `src/app/core/auth`、`guards`、`interceptors`、`services`、`ws`：认证、路由守卫、拦截器、全局服务和 WebSocket。
- `src/app/modules/home/pages`：页面级功能，按业务域拆分。
- `src/app/modules/home/components`：业务复用组件。
- `src/app/shared/components`：通用组件。
- `src/app/shared/models`：TypeScript 接口、枚举和模型。
- `src/assets/locale`：多语言 JSON。
- `src/theme`、`src/scss`：全局主题和 SCSS 资源。

规范：

- TypeScript/HTML/SCSS 使用 2 空格缩进；JSON 使用 4 空格缩进，遵循 `ui-ngx/.editorconfig`。
- 组件 selector 必须使用 `tb` 前缀，例如 `tb-entity-table`；ESLint 会强制检查。
- 路径优先使用 `@app/*`、`@core/*`、`@home/*`、`@modules/*`、`@shared/*`、`@env/*` 别名。
- HTTP 调用统一封装在 `core/http` 服务中，返回 RxJS `Observable`，并通过 `defaultHttpOptionsFromConfig` 处理 `RequestConfig`。
- 用户可见文案使用翻译 key 并更新 locale 文件，不要在模板中硬编码新文案。
- 项目以 NgModule 风格为主，大量组件显式 `standalone: false`；没有迁移要求时不要批量改 standalone。
- Tailwind CSS 的 `preflight` 已关闭，并且样式优先级限定在 `.tb-default`；不要启用全局 reset 或破坏既有作用域。
- 不要随意升级 Angular、Material、RxJS 或带 Git URL 的依赖；锁定版本由 `package.json` 和 `yarn.lock` 控制。
- 当前 `package.json` 没有前端单元测试脚本，前端变更至少运行 lint 和生产构建；复杂交互应补充可手动验证的步骤或截图。

## 6. 配置、许可证与安全要求

- 主配置是 `application/src/main/resources/thingsboard.yml`，各独立服务还有自己的 `tb-*.yml`。
- 新增 YAML 属性时，必须在属性附近写清楚描述，并使用环境变量占位符和合理默认值。
- 修改工作流列出的配置文件后运行：

```powershell
python tools/src/main/python/check_yml_file.py
```

- 不要提交真实密钥、令牌、密码、客户数据或私有证书。
- 安全漏洞不要通过公开 issue 披露，遵循 `security.md`。
- 许可证头是 CI 的一部分。常规 Java/TS/JS/SCSS/Proto 等文件由 Maven license plugin 维护；PE/CE provenance 检查见 `check-license-headers.sh` 和 `tools/src/main/python/license-headers`。
- 不要为了通过检查随意修改 license curation 或排除规则；应先理解文件来源。

## 7. 常用构建和测试命令

以下命令在仓库根目录执行，PowerShell、macOS/Linux Shell 均可使用 Maven 参数形式。

### 7.1 后端/全仓构建

跳过测试和分发打包，适合日常开发：

```powershell
mvn clean install -DskipTests -Dpkg.skip=true
```

后端快速构建并跳过 Angular/Yarn：

```powershell
mvn clean install -DskipTests -Dpkg.skip=true -Dskip.ui.build=true
```

编译某个模块及其依赖：

```powershell
mvn -pl application -am -DskipTests -Dpkg.skip=true test-compile
mvn -pl common/data -am -DskipTests -Dpkg.skip=true test-compile
```

`-Dpkg.skip=true` 会跳过 Boot Jar、DEB、RPM、ZIP。需要构建 `msa/*` Docker 镜像或黑盒测试时不要跳过这些制品。

### 7.2 后端测试

运行某个模块的测试：

```powershell
mvn -pl common/data test
```

运行单个测试类：

```powershell
mvn -pl application -Dtest=AdminControllerTest test -Dsurefire.failIfNoSpecifiedTests=false
```

运行单个测试方法：

```powershell
mvn -pl common/data -Dtest=DockerComposeConfigTest#testGetEdgeRoutingKey test
```

完整后端测试的并行和内存参数见 `CONTRIBUTING.md`。小改动不需要跑全仓，优先选择受影响模块和包。

测试约定：

- 测试源码镜像主代码包路径，类名通常为 `*Test.java`；测试套件为 `*TestSuite.java`，黑盒测试使用 TestNG。
- 新的纯单元测试通常使用 JUnit 5；`application` 中大量遗留 Controller/集成测试使用 JUnit 4，跟随相邻测试风格。
- 部分模块在 `junit-platform.properties` 中启用并行执行，测试不能依赖可变静态状态或固定端口。
- Controller 测试优先复用 `AbstractControllerTest`、`AbstractWebTest` 等基类。
- DAO/Application 集成测试可能通过 Testcontainers 启动真实数据库或 Broker，需要 Docker。
- 如果遇到 Testcontainers 与 Docker API 版本不兼容，按 `CONTRIBUTING.md` 调整 Docker `min-api-version`。

### 7.3 前端命令

在 `ui-ngx` 目录执行：

```powershell
yarn install
yarn lint
yarn build:prod
```

本地开发服务器（代理到 `http://localhost:8080`，WebSocket 代理到 `ws://localhost:8080`）：

```powershell
yarn start
```

Maven 构建会自动安装 `ui-ngx/pom.xml` 中固定的 Node.js 和 Yarn 版本；手动使用 Yarn 时应保持 Yarn 1.x。

### 7.4 许可证和配置检查

格式化许可证头：

```powershell
mvn license:format
```

黑盒测试模块有独立 Maven 项目时：

```powershell
mvn license:format -f msa/black-box-tests/pom.xml
```

检查 YAML 属性描述：

```powershell
python tools/src/main/python/check_yml_file.py
```

### 7.5 Protobuf/gRPC

修改 `common/proto/src/main/proto/*.proto` 后，使用 Bash 执行：

```bash
./build_proto.sh
```

该脚本会通过 Maven 编译 `common/proto` 及相关消息、传输模块。不要手改 `target/` 下的生成类。

### 7.6 Docker 黑盒测试

黑盒测试说明以 `msa/black-box-tests/README.md` 为准。基本流程：

```powershell
# 根目录：构建本地服务镜像，不能使用 -Dpkg.skip=true
mvn clean install -Ddockerfile.skip=false

# 黑盒测试模块
Push-Location msa/black-box-tests
mvn clean install -DblackBoxTests.skip=false
Pop-Location
```

该测试会自动准备 Compose 栈并依赖本地 Docker 镜像，耗时较长；只有修改微服务打包、传输、集成、部署或端到端行为时才需要运行。

## 8. 变更影响清单

提交修改前，按变更范围检查：

### 通用

- [ ] diff 聚焦，没有无关格式化、生成文件或 IDE 文件。
- [ ] 受 license plugin 管理的新文件和修改文件具有正确 SPDX 头。
- [ ] 公共 API、配置、数据库结构保持向后兼容，或明确提供升级脚本/迁移说明。
- [ ] 没有硬编码密钥、环境专属地址或客户数据。
- [ ] 相关日志级别合适，异常路径不会吞掉原因。

### 后端

- [ ] 新 Bean 使用正确的 `@Tb*Component` 条件注解。
- [ ] REST 端点有 `@PreAuthorize` 和 OpenAPI 注解。
- [ ] 新增 REST API 已同步 `RestClient.java`。
- [ ] DTO、DAO 接口、实现、缓存和事务边界放在正确模块。
- [ ] 数据库变更同步 schema、安装/升级服务和升级版本约束。
- [ ] 新增/修改行为有单元或集成测试；无法添加测试时说明原因。

### 前端

- [ ] HTTP 调用放入 `core/http`，模型放入 `shared/models` 或对应领域模型文件。
- [ ] 组件 selector 使用 `tb` 前缀并遵循路径别名约定。
- [ ] 用户可见文案已更新 locale。
- [ ] `yarn lint` 和 `yarn build:prod` 通过。
- [ ] UI/API/Widget 行为变化已准备截图、使用说明或文档更新事项。

### 配置与打包

- [ ] 新 YAML 属性有描述、环境变量名和默认值。
- [ ] 已运行 YAML 检查脚本。
- [ ] 只有在需要 Docker/黑盒测试时才运行完整打包；普通验证使用 `-Dpkg.skip=true`。
- [ ] 修改 Docker、MSA、DEB/RPM/ZIP 后检查对应 `msa/*` 或 `packaging` 模块。

## 9. Agent 工作方式要求

- 先阅读相邻实现和模块 `pom.xml`，再做最小范围修改；不要仅凭类名猜测架构。
- 需要新建分支时使用 `codex/` 前缀，除非用户明确指定其他分支名。
- 不要自动执行 `git commit`、`git push`、发布、推送镜像或不可逆清理操作，除非用户明确要求。
- 不要为了让测试通过而删除测试、跳过断言或缩小测试范围；应修复实现或说明环境限制。
- 不要随意升级依赖、Java/Angular/Spring Boot 版本或修改大规模锁文件。
- 长耗时命令优先按模块执行；只有交付前或变更影响打包/端到端行为时才运行全量测试。
- 命令失败时先区分编译错误、测试失败、Docker/Testcontainers 环境问题和网络依赖问题，再决定是否重试。
- 最终回复应清楚列出修改文件、验证命令、未运行的高成本验证及原因。

## 10. 关键参考文件

- `LOCAL_STARTUP.md`：Windows + IDEA 本地启动、安装器运行和常见启动报错处理。
- `CONTRIBUTING.md`：官方构建、测试和提交流程。
- `pull_request_template.md`：后端、前端和通用 PR 检查项。
- `pom.xml`：模块列表、Java/Spring Boot/依赖版本、打包和插件配置。
- `ui-ngx/structure.md`：前端路由、页面和组件架构索引。
- `ui-ngx/eslint.config.mjs`：前端 ESLint 规则。
- `application/src/main/resources/thingsboard.yml`：主配置和环境变量约定。
- `msa/black-box-tests/README.md`：黑盒测试和 Compose 栈说明。
- `security.md`：安全漏洞披露流程。