# 系统设计文档

## 1. 概述

本设计文档基于用户服务系统的技术架构与实现方案，主要涵盖分库分表策略、RPC 调用链路设计、MQ 消息机制及可靠性保障策略。系统采用 Spring Boot 框架构建，结合 ShardingSphere 实现数据分片，通过 Spring Cloud Feign 进行服务间通信，并使用 RabbitMQ 实现异步消息传递，最终由日志服务完成操作记录的持久化。

## 2. 架构设计

![架构图](https://open-api.cloudlab.top/fss/resource/1ab706318b818001/AYsiLY/summary/workdir/%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

### 2.1 服务描述

- **用户服务 (user-service)**：系统的核心，负责处理用户注册、登录、信息管理、密码重置等操作，并通过 Feign 客户端调用权限服务。同时，作为消息生产者，将用户操作日志异步发送至 RabbitMQ。
- **权限服务 (permission-service)**：负责管理用户角色和权限。提供 RPC 接口供用户服务调用，以处理用户角色绑定、权限查询、角色升降级等业务。
- **日志服务 (logging-service)**：作为消息消费者，负责监听消息队列中的日志消息，并将解析后的操作日志持久化到数据库中，实现核心业务与日志记录的解耦。
- **消息队列 (RabbitMQ)**：用于服务间的异步通信，主要承担用户操作日志的缓冲和传递。

## 3. 技术选型

- **整体框架**：Spring Boot (2.2.6) + MyBatis-Plus (3.4.0) + MySQL (8)
- **服务发现与配置中心**：Nacos (1.2.1)
- **声明式 REST 调用**：OpenFeign (2.2.1)
- **数据分片**：ShardingSphere (5.1.0)
- **分布式事务**：Seata (1.6.1)
- **异步消息处理**：RabbitMQ (2.2.6)
- **安全认证**：JWT (Java Web Token)

## 4. 数据存储设计

### 4.1 用户服务 (user-service)

- **数据源配置**：配置 `db0` 和 `db1` 两个数据源，分别指向 `user_db0` 和 `user_db1` 数据库。
- **分片规则**：
  - **users 表**：按 `user_id` 进行分库，不分表。实际数据节点为 `db${0..1}.users`。
  - **分片算法**：采用 INLINE 行表达式算法，通过 `user_id % 2` 的方式决定数据路由到哪个数据库。
    - `user_id = 1` → `1 % 2 = 1` → 路由至 `db1`
    - `user_id = 2` → `2 % 2 = 0` → 路由至 `db0`
- **主键生成**：采用雪花算法（Snowflake）生成分布式唯一主键，保证分库后主键的全局唯一性。

### 4.2 其他服务

- **权限服务 (permission-service)**：使用独立的 `permission_db` 数据库，包含 `roles`、`user_roles` 等表，不进行分库分表。
- **日志服务 (logging-service)**：使用独立的 `logging_db` 数据库，包含 `operation_logs` 表，用于存储用户操作日志，不进行分库分表。

## 5. RPC 调用链路设计

系统内的服务间通信主要通过 OpenFeign 实现。核心调用方为 `user-service`，被调用方为 `permission-service`。

### 5.1 Feign 接口定义（RoleFeignService）

`user-service` 中定义的 Feign 客户端，用于调用 `permission-service` 提供的 REST 接口：

- `bindDefaultRole(userId)`：注册时为用户绑定默认角色。
- `getUserRoleCode(userId)`：获取用户角色码，用于权限判断。
- `upgradeToAdmin(userId)`：提升用户为管理员。
- `downgradeToUser(userId)`：降级管理员为普通用户。
- `getUserIdsByRoleId(roleId)`：根据角色 ID 获取用户 ID 列表。
- `getUserRoles()`：获取所有用户角色信息。

### 5.2 主要调用流程

#### 用户注册

1. `user-service`：检查用户名是否重复。
2. `user-service`：将用户信息（密码加密后）保存到数据库。此过程可由 Seata 管理分布式事务。
3. `user-service`：通过 Feign 调用 `permission-service` 的 `bindDefaultRole` 接口，为新用户绑定"普通用户"角色。
4. `user-service`：发送消息到 RabbitMQ，记录注册操作日志。

#### 用户登录

1. `user-service`：验证用户名和密码。
2. `user-service`：通过 Feign 调用 `permission-service` 的 `getUserRoleCode` 接口获取用户角色。
3. `user-service`：生成 JWT 令牌并返回给客户端。
4. `user-service`：发送消息到 RabbitMQ，记录登录操作日志。

#### 分页查询用户

1. `user-service`：根据当前登录用户的角色（普通用户 / 管理员）构建查询。
2. 若为管理员，通过 Feign 调用 `permission-service` 的 `getUserIdsByRoleId` 接口，获取所有普通用户的 ID 列表，再进行查询。
3. `user-service`：发送消息到 RabbitMQ，记录查询操作日志。

## 6. 异步消息与日志记录

系统采用 RabbitMQ 实现业务操作与日志记录的解耦。

### 6.1 消息发送方（user-service）

- **MQSender**：消息发送工具类，负责将 `MessageVo` 对象封装成 Map，并调用 `RabbitTemplate` 发送。
- **RabbitMQConfig**：
  - **Exchange**：定义名为 `log_direct_exchange` 的 Direct 类型交换机。
  - **Queue**：定义名为 `log_queue` 的队列。
  - **Binding**：将队列与交换机通过路由键 `log` 进行绑定。
- **消息内容**：包含 `userId`、`action`（操作类型）、`ip`、`detail`（详情）、`timestamp` 等。

### 6.2 消息接收方（logging-service）

- **MQReceiver**：消息接收类，使用 `@RabbitListener` 注解监听 `log_queue` 队列。
- **处理流程**：
  1. 接收到消息（Map 格式）。
  2. 将 Map 数据转换为 `OperationLogs` 实体对象。
  3. 调用 `OperationLogsService` 将日志对象存入 `logging_db` 数据库。

## 7. 消息可靠性保障

- **消息持久化**：交换机、队列和消息本身都设置为持久化（Durable）。
- **消费者确认机制（Consumer Acknowledgements）**：`logging-service` 当前采用自动确认（AcknowledgeMode.AUTO）。

## 8. 安全与认证

系统通过 JWT（JSON Web Token）实现无状态的 API 认证。

- **JwtUtil**：提供生成、解析和校验 JWT 的工具方法。
- **JwtAuthInterceptor**：Spring MVC 拦截器，在请求到达 Controller 前，对请求头中的 JWT 进行校验。
  - 校验失败则拦截请求，返回认证失败的响应。
  - 校验成功则从 JWT 中解析出用户信息，用户信息存储在 HttpServletRequest 的属性，供后续业务逻辑使用。
- **登录流程**：用户登录成功后，`user-service` 会生成一个包含用户 ID 和角色的 JWT 令牌，并返回给客户端。客户端在后续请求中需在 `Authorization` 请求头中携带此令牌。
