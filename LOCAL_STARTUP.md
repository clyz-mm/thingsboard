# 本地启动指南（Windows + IntelliJ IDEA）

本文记录在 Windows 上用 IDEA 从源码启动 ThingsBoard 4.4.0.2 单机版的完整过程，包含实际踩到的 5 个坑和对应解法。

适用范围：本地开发调试，使用内存队列（in-memory）+ caffeine 缓存 + PostgreSQL，非生产部署。

---

## 0. 前置条件

| 组件 | 要求 | 本机实际 |
|---|---|---|
| JDK | **25**（`pom.xml` 中 `<release>25</release>`） | `D:\Program Files\Java\jdk-25` |
| Maven | 3.9.x | `D:\Apache\apache-maven-3.9.16` |
| PostgreSQL | 12+ | Docker 容器 `tb-postgres`（`postgres:16`，`0.0.0.0:5432`） |
| Docker | 用于跑数据库 | Rancher Desktop |

> 本机同时装了 JDK 21 和 JDK 27。**21 编不了（release 25 不支持），27 也编不了（Lombok 不兼容）**，只能用 25。

---

## 1. 让本项目使用 JDK 25，其他项目保持 21

不要把系统级 `JAVA_HOME` 改成 25，否则会影响其他项目。分两处配置：

### 1.1 IDEA 侧（三处都要改，改完点"确定"）

1. `项目结构 → 项目 → SDK`：选 25
2. `设置 → 构建、执行、部署 → 构建工具 → Maven → 运行程序(Runner) → JRE`：`使用项目 JDK`
3. `设置 → 构建工具 → Maven → 导入(Importing) → 导入器的 JDK`：选 25

改完可以用文件确认是否生效：

```powershell
Get-Content D:\github\thingsboard\.idea\misc.xml
# 应看到 project-jdk-name="25 ..."，若仍是 openjdk-27 说明没保存
```

### 1.2 终端侧（只在当前窗口生效）

```powershell
$env:JAVA_HOME = "D:\Program Files\Java\jdk-25"
mvn -version   # 确认 runtime 是 jdk-25
```

想长期方便，可在 PowerShell 配置文件里加一个只在需要时切换的函数（不影响其他项目）：

```powershell
function mvn25 { $env:JAVA_HOME = "D:\Program Files\Java\jdk-25"; & "$env:MAVEN_HOME\bin\mvn.cmd" @args }
```

---

## 2. 构建

```powershell
mvn clean install "-DskipTests" "-Dpkg.skip=true" "-Dskip.ui.build=true"
```

- `-Dpkg.skip=true`：跳过 Boot Jar / DEB / RPM / ZIP，日常开发够用。
- `-Dskip.ui.build=true`：跳过 Angular/Yarn 构建，省大量时间。

> **注意**：`-Dpkg.skip=true` 会同时跳过 packaging profile 里的 `copy-data` 步骤，这是后面"缺 sql 脚本"的根因，见第 3 节。

### 已知可忽略的告警

- `net.bytebuddy:byte-buddy` / `byte-buddy-agent` duplicate declaration
- `junit:junit` 版本为 LATEST/RELEASE 的提示
- `WARNING: A terminally deprecated method in sun.misc.Unsafe has been called`（Lombok / Netty 正常现象）

---

## 3. 补齐安装数据目录（关键，只做一次）

安装器在 `${dataDir}/sql/` 下读取建表脚本，`dataDir` 在 IDEA 运行时解析为：

```
D:\github\thingsboard\application\src\main\data
```

但仓库里 DDL 实际放在 `dao/src/main/resources/sql/`，只有打包时才会被根 `pom.xml` 的 `copy-data` 复制到 `application/target/data`。因为构建时加了 `-Dpkg.skip=true`，这个步骤永远不执行，所以 `src/main/data/sql` 不存在。

补齐（仓库根目录执行）：

```powershell
New-Item -ItemType Directory -Force -Path application\src\main\data\sql | Out-Null
Copy-Item dao\src\main\resources\sql\*.sql application\src\main\data\sql\
Get-ChildItem application\src\main\data\sql
```

应得到 8 个文件：

```
schema-entities.sql              schema-entities-idx.sql
schema-entities-idx-psql-addon.sql   schema-functions.sql
schema-timescale.sql             schema-ts-psql.sql
schema-ts-latest-psql.sql        schema-views.sql
```

说明：

- 该目录**未被 git 跟踪**，`git status` 会显示 `?? application/src/main/data/sql/`。不想看到可以加进 `.gitignore`。
- `mvn clean` 只删 `target`，不会删掉这里的文件，复制一次即可长期有效。
- 替代方案：不加 `-Dpkg.skip=true` 完整打包一次，生成 `application/target/data`，然后给运行配置加 `-Dinstall.data_dir=D:\github\thingsboard\application\target\data`，就不用往源码目录放文件了（代价是完整打包很慢）。

---

## 4. IDEA 运行配置：安装器

主类：`org.thingsboard.server.ThingsboardInstallApplication`
工作目录：`$MODULE_DIR$`（即 `application`，必须是这个，安装器靠 `user.dir` 定位数据目录）

### 4.1 VM 选项

```
-Dserver.port=0
```

安装器会顺带启动内嵌 Tomcat，但装完就 `SpringApplication.exit()` 退出，不需要对外提供端口。`0` 表示随机空闲端口，避免和本机其他服务抢 8080。

### 4.2 环境变量（填在"环境变量"字段，**不是** VM 选项）

| 变量 | 值 | 说明 |
|---|---|---|
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://localhost:5432/thingsboard` | 必填 |
| `SPRING_DATASOURCE_USERNAME` | `postgres` | 必填 |
| `SPRING_DATASOURCE_PASSWORD` | `postgres` | 必填 |
| `NON_PRODUCTION_USE` | `true` | 无 license 时必填，见下方注意 |

可选项（`thingsboard.yml` 里默认值就是这三个，写了也是同样效果，可以完全不填）：

- `TB_SERVICE_TYPE=monolith`
- `TB_QUEUE_TYPE=in-memory`
- `CACHE_TYPE=caffeine`

> **关于 license**：`NON_PRODUCTION_USE` 是**布尔值**，只能写 `true`/`false`。它有代价：界面带水印、不能白标，且**累计运行满 30 天实例会自动关闭**。有正式 license 时不要用它，改为设置 `TB_LICENSE_SECRET=<密钥>`（密钥来自 https://license.thingsboard.io ）。

### 4.3 常见误配

`KEY=VALUE` 形式的变量**不能**写进 VM 选项或程序实参。JVM 启动器会把第一个不带 `-` 前缀的 token 当成主类名，报：

```
错误: 找不到或无法加载主类 NON_PRODUCTION_USE=TB_LICENSE_SECRET
原因: java.lang.ClassNotFoundException: NON_PRODUCTION_USE=TB_LICENSE_SECRET
```

---

## 5. 运行安装器

数据库需为空库（全新安装）。检查：

```powershell
docker exec tb-postgres psql -U postgres -d thingsboard -c "\dt"
# 期望输出：Did not find any relations.
```

启动 `ThingsboardInstallApplication`，成功标志：

```
Installation finished successfully!
```

日志顺序大致为：Hibernate 校验 → HikariPool 连接成功 → 建表/索引/视图 → 加载系统数据 → 退出（退出码 0）。

---

## 6. 启动服务

安装完成后，把主类换成 `org.thingsboard.server.ThingsboardServerApplication`，环境变量沿用 4.2 那套，额外加端口（如果 8080 被占用）：

| 变量 | 值 |
|---|---|
| `HTTP_BIND_PORT` | `8081` |

对应 `application/src/main/resources/thingsboard.yml` 第 13 行 `port: "${HTTP_BIND_PORT:8080}"`。也可以用 VM 选项 `-Dserver.port=8081`。

访问 `http://localhost:8081`，默认账号：

| 账号 | 用户名 | 密码 |
|---|---|---|
| 系统管理员 | `sysadmin@thingsboard.org` | `sysadmin` |
| 租户管理员 | `tenant@thingsboard.org` | `tenant` |

---

## 7. 排错速查表

| 报错 | 根因 | 处理 |
|---|---|---|
| `Fatal error compiling: java.lang.ExceptionInInitializerError: com.sun.tools.javac.tree.EndPosTable` | 用 JDK 27 构建，该版本 javac 移除了 `EndPosTable`，而 Lombok 1.18.46 仍引用它 | 改用 JDK 25；或把 `pom.xml` 的 `lombok.version` 升到 1.18.48（实测可在 JDK 27 编译） |
| `Fatal error compiling: 错误: 不支持发行版本 25` | 用 JDK 21 构建 | 切到 JDK 25 |
| `错误: 找不到或无法加载主类 XXX=YYY` | 环境变量写进了 VM 选项/程序实参 | 移到"环境变量"字段 |
| `Web server failed to start. Port 8080 was already in use.` | 8080 被占用（本机是 matomo 容器，经 Rancher Desktop 的 `wslrelay`/`host-switch` 监听） | 安装器加 `-Dserver.port=0`；或 `docker stop matomo-app`；正式服务用 `HTTP_BIND_PORT=8081` |
| `java.nio.file.NoSuchFileException: ...\application\src\main\data\sql\schema-entities.sql` | `-Dpkg.skip=true` 跳过 `copy-data`，`src/main/data/sql` 从未生成 | 见第 3 节，复制 dao 的 sql 到 `application/src/main/data/sql` |
| `Not valid working directory: ...` | 运行配置的工作目录不是 `application` 模块根 | 工作目录改回 `$MODULE_DIR$`，或指定 `-Dinstall.data_dir` |

排查小技巧：

```powershell
# 看端口被谁占了
Get-NetTCPConnection -LocalPort 8080 -State Listen | ForEach-Object { Get-Process -Id $_.OwningProcess } | Select-Object Id,ProcessName,Path
# 看 Docker 容器端口映射
docker ps --format "{{.Names}}`t{{.Image}}`t{{.Ports}}"
```

---

## 8. 环境变量与配置文件对照

`application/src/main/resources/thingsboard.yml` 关键项：

| 属性 | 环境变量 | 默认值 |
|---|---|---|
| `server.port` | `HTTP_BIND_PORT` | 8080 |
| `service.type` | `TB_SERVICE_TYPE` | monolith |
| `queue.type` | `TB_QUEUE_TYPE` | in-memory |
| `cache.type` | `CACHE_TYPE` | caffeine |
| `license.secret` | `TB_LICENSE_SECRET` | 空 |
| `license.non_production_use` | `NON_PRODUCTION_USE` | false |
| `spring.datasource.url` | `SPRING_DATASOURCE_URL` | — |

更多构建/测试/代码规范见 `AGENTS.md`。
