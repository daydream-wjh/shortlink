# ShortLink 短链接系统

## 项目简介

基于 Spring Cloud 微服务架构的短链接生成与访问统计系统。用户可以把长网址压缩成 `nurl.ink` 域名下的短码，系统记录每一次跳转的 PV / UV / UIP、地区、浏览器、操作系统、设备类型、网络类型等维度数据，并在 Vue3 控制台里以图表形式呈现。

技术形态上有两套部署方式：

- **微服务模式**：`gateway`（网关）+ `admin`（后台管理服务）+ `project`（短链跳转与统计服务）
- **单体聚合模式**：`aggregation` 直接把 admin 与 project 的包扫进同一个进程启动，网关默认指向它

前端 `console-vue` 是一个独立的 Vite + Vue3 单页应用（包名 `smallmark`）。

## 功能模块

| 模块 | 主要能力 | 关键代码位置 |
| --- | --- | --- |
| 用户体系 | 注册、登录（Redis 存 token）、退出、修改信息、用户名占用校验、手机号脱敏 | `admin/controller/UserController.java` |
| 分组管理 | 分组的增删改查、排序、分组下短链数量统计（单用户最多 20 个分组） | `admin/controller/GroupController.java` |
| 短链管理 | 单条创建、批量创建（EasyExcel 导出）、分页查询、修改、按分组统计 | `admin/controller/ShortLinkController.java` |
| 短链跳转 | `GET /{short-uri}` 302 跳转，布隆过滤器防穿透、空值缓存、Redisson 分布式锁 | `project/controller/ShortLinkController.java` |
| 访问统计 | 单链/分组维度的 PV、UV、UIP、地区分布、浏览器、OS、设备、网络、日访问曲线、访问明细 | `project/controller/ShortLinkStatsController.java` |
| 回收站 | 移入回收站、分页、恢复、彻底删除 | `admin`/`project` 的 `RecycleBinController.java` |
| 网页标题抓取 | 输入长链接自动解析标题（jsoup） | `UrlTitleController.java` |
| 跳转白名单 | 只允许跳转到白名单域名（拿个offer / 知乎 / 掘金 / 博客园） | `GotoDomainWhiteListConfiguration` |
| 限流风控 | Sentinel 规则 + Lua 脚本实现的用户访问频控 | `SentinelRuleConfig`、`lua/user_flow_risk_control.lua` |
| 异步统计 | RocketMQ 生产/消费统计事件，延迟队列补算，消费幂等 | `project/mq/**` |
| 控制台前端 | 首页、我的空间（创建/编辑/二维码/图表）、回收站、个人中心、登录 | `console-vue/src/views/**` |

## 技术栈

全部依据 `pom.xml` 与配置文件实际内容：

| 类别 | 选型 | 版本 |
| --- | --- | --- |
| 语言 / 构建 | Java、Maven 多模块 | Java 17 |
| 基础框架 | Spring Boot | 3.0.7 |
| 微服务 | Spring Cloud / Spring Cloud Alibaba | 2022.0.3 / 2022.0.0.0-RC2 |
| 注册中心 | Nacos Discovery | 随 SCA 版本 |
| 网关 | Spring Cloud Gateway（自定义 `TokenValidate` 过滤器） | — |
| 持久层 | MyBatis-Plus | 3.5.3.1 |
| 分库分表 | ShardingSphere-JDBC（`HASH_MOD` 分表 + AES 字段加密） | 5.3.2 |
| 数据库 | MySQL 8（`utf8mb4_0900_ai_ci`） | — |
| 缓存 | Redis + Redisson（布隆过滤器、分布式锁、读写锁） | Redisson 3.27.2 |
| 消息队列 | RocketMQ（rocketmq-spring-boot-starter） | 2.2.3 |
| 熔断限流 | Sentinel（`spring-cloud-starter-alibaba-sentinel`） | — |
| 工具库 | Hutool 5.8.20、Guava 30.0-jre、fastjson2 2.0.36、jsoup 1.15.3、EasyExcel 3.1.3、jjwt 0.9.1 | — |
| 服务调用 | OpenFeign + LoadBalancer | — |
| 模板引擎 | Thymeleaf（404 降级页 `/page/notfound`） | — |
| 代码规范 | Spotless Maven Plugin（license header + 格式化） | 2.2.1 |
| 前端 | Vite 4、Vue 3.3、Vue Router 4、Vuex 4、Element Plus 2.3、ECharts 4.8、qrcode、vanta/three、axios | — |

## 模块说明

```
shortlink-all (父 POM，packaging=pom)
├── admin/                        短链后台管理服务，端口 8002
│   ├── controller / service / dao(UserDO, GroupDO)
│   ├── remote/                   OpenFeign 客户端 @FeignClient("short-link-project", url="${aggregation.remote-url:}")
│   └── shardingsphere-config-*.yaml   t_user / t_group 分表 + 字段加密
├── project/                      短链跳转与统计服务，端口 8001
│   ├── controller / service      短链创建、跳转、统计、回收站
│   ├── mq/                       RocketMQ 生产者、消费者、幂等处理
│   └── shardingsphere-config-*.yaml   t_link / t_link_goto 分表
├── gateway/                      网关，端口 8000，默认 profile=aggregation
├── aggregation/                  单体聚合启动模块，端口 8003
│   └── 依赖 admin + project，scanBasePackages 同时扫描两个服务的包
└── console-vue/                  前端控制台（Vite + Vue3，与 Maven 无关）
```

模块依赖关系：

| 模块 | 依赖 |
| --- | --- |
| `shortlink-admin` | spring-web、mybatis-plus、shardingsphere-jdbc、redis、redisson、nacos-discovery、openfeign、loadbalancer、easyexcel |
| `shortlink-project` | spring-web、mybatis-plus、shardingsphere-jdbc、redis、redisson、thymeleaf、jsoup、sentinel、nacos-discovery、rocketmq |
| `shortlink-gateway` | spring-cloud-starter-gateway、loadbalancer、nacos-discovery、redis、netty-all |
| `shortlink-aggregation` | `shortlink-admin` + `shortlink-project` |

网关路由（`gateway/src/main/resources/application.yaml`）：

| 路由 | 断言 | 目标 |
| --- | --- | --- |
| short-link-admin | `Path=/api/short-link/admin/**` | `lb://short-link-admin`（aggregation profile 下为 `lb://short-link-aggregation`） |
| short-link-project | `Path=/api/short-link/**` | `lb://short-link-project`（aggregation profile 下为 `lb://short-link-aggregation`） |

`TokenValidate` 过滤器从请求头取 `username` 与 `token`，到 Redis Hash `short-link:login:{username}` 校验，通过后注入 `userId`、`realName` 头；白名单为 `/api/short-link/admin/v1/user/login` 与 `/api/short-link/admin/v1/user/has-username`。

## 数据库与配置

### 需要的中间件

| 组件 | 默认地址 | 用途 |
| --- | --- | --- |
| MySQL 8 | `127.0.0.1:3306`，库名 `shortlink` | 业务数据 |
| Redis | `127.0.0.1:6379` | 登录 token、跳转缓存、布隆过滤器、分布式锁 |
| Nacos | `127.0.0.1:8848` | 服务注册发现 |
| RocketMQ | `127.0.0.1:9876` | 访问统计异步落库（project / aggregation 必需） |

### 分表结构

ShardingSphere 配置了 16 片 `HASH_MOD` 分表，逻辑表与分片键：

| 逻辑表 | 真实表 | 分片键 | 所在配置 |
| --- | --- | --- | --- |
| `t_user` | `t_user_0` ~ `t_user_15` | `username` | `admin/shardingsphere-config-*.yaml` |
| `t_group` | `t_group_0` ~ `t_group_15` | `username` | 同上 |
| `t_link` | `t_link_0` ~ `t_link_15` | `gid` | `project/shardingsphere-config-*.yaml` |
| `t_link_goto` | `t_link_goto_0` ~ `t_link_goto_15` | `full_short_url` | 同上 |

不分表的统计类表：`t_link_access_logs`、`t_link_access_stats`、`t_link_browser_stats`、`t_link_device_stats`、`t_link_locale_stats`、`t_link_network_stats`、`t_link_os_stats`、`t_link_stats_today`（统计表实体见 `project/dao/entity/`）。

`t_user` 的 `phone`、`mail` 字段启用 ShardingSphere AES 加密（`common_encryptor`），查询走密文列。

### 配置项与需要改动的地方

1. 数据源：`shardingsphere-config-dev.yaml` 里的 `jdbcUrl` / `username` / `password`，默认使用 `root` + 弱口令，需改成自己的库。
2. 环境切换：`spring.datasource.url = jdbc:shardingsphere:classpath:shardingsphere-config-${database.env:dev}.yaml`，用 `-Ddatabase.env=prod` 切到生产配置。
3. 短链域名：`short-link.domain.default`（project 为 `nurl.ink:8001`，aggregation 为 `nurl.ink:8003`），必须改成自己的域名/端口，否则生成的短链无法跳回本机。
4. Redis / Nacos / RocketMQ 地址：各模块 `application.yaml` 中手写为 `localhost`、`127.0.0.1`。
5. 高德地图 Key：`short-link.stats.locale.amap-key`（用于 IP 归属地解析，接口地址见 `ShortLinkConstant.AMAP_REMOTE_URL`）。
6. 网关默认 `spring.profiles.active: aggregation`；只想跑 admin + project 微服务时改为 `dev`。

### 建表脚本

仓库**未包含任何 SQL 脚本**。16 张分片表的 DDL 需要自己补齐，可以参考 `admin/src/test/java/com/daydream/shortlink/admin/test/UserTableShardingTest.java` 里用 `String.format` 拼出的模板（注意该类当前的 DDL 内容其实是 `t_link_stats_today`），其余表结构按 `dao/entity` 的字段推导。启动前请确保 `shortlink` 库中 16 张分片表与统计表都已创建。

## 快速开始

### 后端

```bash
# 0. 环境：JDK 17、Maven 3.8+、MySQL 8、Redis、Nacos、RocketMQ
# 1. 建库建表（仓库无脚本，需自行准备，见上文）
mysql -uroot -p -e "CREATE DATABASE shortlink DEFAULT CHARSET utf8mb4;"

# 2. 改配置：数据源、Redis、Nacos、RocketMQ、short-link.domain.default
#    admin/src/main/resources/shardingsphere-config-dev.yaml
#    project/src/main/resources/application.yaml

# 3. 编译（Spotless 需要 format/license-header，见「已知问题」，可先跳过）
mvn clean install -DskipTests -Dspotless.check.skip=true -Dspotless.apply.skip=true

# 4. 启动
#    admin 与 gateway 的 pom 里声明了 spring-boot-maven-plugin，可直接跑 jar：
java -jar admin/target/shortlink-admin.jar          # 8002
java -jar gateway/target/shortlink-gateway.jar      # 8000

#    project 与 aggregation 的 pom 没有 spring-boot-maven-plugin（见「已知问题」），
#    只能用 IDE 直接运行启动类，或自己给 pom 补上插件后再打包：
#      project     -> com.daydream.shortlink.project.ShortLinkApplication        (8001)
#      aggregation -> com.daydream.shortlink.aggregation.AggregationServiceApplication (8003)

#    两种运行方式二选一：
#      微服务模式：admin(8002) + project(8001)，并把 gateway 的 profiles.active 改成 dev
#      聚合模式：aggregation(8003) + gateway(8000)，gateway 默认 profile 就是 aggregation
```

服务端口：`gateway 8000`、`project 8001`、`admin 8002`、`aggregation 8003`。

### 前端

```bash
cd console-vue
npm install          # 或 pnpm install
npm run dev          # Vite dev server
# 后端地址在 vite.config.js 中代理：/api -> http://127.0.0.1:8000
```

## 接口文档

统一前缀 `/api/short-link`（经网关转发），返回体为 `Result<T>`（`code` / `message` / `data`）。

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| POST | `/api/short-link/admin/v1/user/login` | 用户登录，返回 token |
| POST | `/api/short-link/admin/v1/user` | 用户注册 |
| GET | `/api/short-link/admin/v1/user/has-username` | 校验用户名是否可用（白名单） |
| POST | `/api/short-link/admin/v1/group` | 新建短链分组 |
| GET | `/api/short-link/admin/v1/group` | 查询当前用户分组列表 |
| POST | `/api/short-link/admin/v1/create` | 创建短链接 |
| POST | `/api/short-link/admin/v1/create/batch` | 批量创建并导出 Excel |
| GET | `/api/short-link/admin/v1/page` | 短链分页查询 |
| GET | `/api/short-link/admin/v1/stats` | 单条短链访问统计 |
| GET | `/api/short-link/admin/v1/stats/access-record` | 访问明细分页 |
| POST | `/api/short-link/admin/v1/recycle-bin/save` | 移入回收站 |
| GET | `/api/short-link/{short-uri}` | 短链跳转（302，project 服务） |

## 已知问题

1. **硬编码数据库口令**：`admin`、`project`、`aggregation` 下的 `shardingsphere-config-dev.yaml` / `-prod.yaml` 都直接写着数据源账号密码，生产配置文件中同样存在明文口令，建议改为环境变量或配置中心注入。
2. **加密密钥明文入库**：ShardingSphere 手机号/邮箱字段的 AES 密钥 `aes-key-value` 直接写在 yaml 里，等于加密失效。
3. **第三方 Key 泄漏**：`project/application.yaml` 与 `aggregation/application.yaml` 中的高德 `amap-key` 为明文真实 Key。
4. **用户密码明文存储与比对**：`UserServiceImpl.login` 直接用 `.eq(UserDO::getPassword, ...)` 明文匹配，`register` 也只是把 DTO 原样 `BeanUtil.toBean` 落库，没有任何哈希，建议改为 BCrypt/Argon2。同时 `UserRegisterReqDTO` 一个校验注解都没有，`BaseErrorCode` 里定义的 `PASSWORD_SHORT_ERROR` 实际未被使用。
5. **前端内网地址残留**：`console-vue/.env.development` 里的 `VITE_LOGIN_PATH` 指向一台内网 SSO 主机（`http://10.**.**.**:8200/sso/login/1`），属于开发机环境信息，公开仓库里应删除。
6. **`.idea/` 部分文件已提交入库**：`dataSources.xml`、`deployment.xml`、`sqldialects.xml`、`encodings.xml`、`vcs.xml`、`misc.xml` 共 7 个 IDE 文件在版本控制中（`.gitignore` 只排除了其中几个），会泄露本地数据源与部署信息，建议 `git rm -r --cached .idea` 后补全忽略规则。
7. **编译期会因 Spotless 失败**：父 POM 的 spotless 配置引用 `${maven.multiModuleProjectDirectory}/format/license-header`，但仓库里没有 `format/` 目录，`mvn compile` 会在 `spotless:apply` 阶段报错。临时绕过：`-Dspotless.check.skip=true -Dspotless.apply.skip=true`，或自行补一个 license header 文件。
8. **缺少 SQL 脚本与初始化数据**：无建库建表脚本、无测试数据；分表 DDL 只有 `UserTableShardingTest` 里一段，且类名（User）与内容（`t_link_stats_today`）不一致。
9. **前端遗留 Vite 模板 README**：`console-vue/README.md` 仍是 `npm create vue` 的默认文档；同时存在 `package-lock.json` 与 `pnpm-lock.yaml` 两套锁文件，包管理器不统一。
10. **配置重复**：`admin`、`project`、`aggregation` 各有一份 `application.yaml`，`shardingsphere-config-dev/prod.yaml` 在三个模块里各复制了一份，内容大量重叠，改一处容易漏改；`aggregation` 中的 `aggregation.remote-url` 指向自身（`http://127.0.0.1:${server.port}`），意味着聚合模式下 Feign 调用会绕回本进程，调试时容易误判链路。
11. **两个模块打不出可执行 jar**：只有 `admin` 与 `gateway` 的 pom 里声明了 `spring-boot-maven-plugin` 和 `finalName`；`project`、`aggregation` 完全没有 `<build>` 段，`mvn package` 产出的是普通 jar，`java -jar` 会报 "no main manifest attribute"，本地只能靠 IDE 运行启动类。
