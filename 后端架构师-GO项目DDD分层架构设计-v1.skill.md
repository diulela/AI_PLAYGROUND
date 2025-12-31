# 系统后端

## 项目概述

基于 Go-Kratos 微服务框架的系统后端，采用领域驱动设计 (DDD) 和 CQRS 模式。

**框架文档**: [go-kratos.dev](https://go-kratos.dev/docs/)

## 项目结构

```
├── api/                 // Protobuf API定义
├── cmd/backend/         // 入口 (main.go, wire.go)
├── configs/             // 配置文件
├── internal/
│   ├── application/     // 应用层：用例编排 (CQRS)
│   │   └── <domain>/
│   │       ├── command/ // 命令处理器(写操作)
│   │       └── query/   // 查询处理器(读操作)
│   ├── application_event/ // 事件处理器(同步EventBus/异步MQ)
│   ├── domain/          // 领域层：业务核心
│   │   └── <domain>/
│   │       ├── entity, valueobject, event
│   │       ├── repository/  // 仓储接口
│   │       └── service/     // 领域服务
│   ├── infra/           // 基础设施层：技术实现
│   │   ├── data/        // 仓储实现、DB模型
│   │   ├── bus/         // 事件总线
│   │   └── messaging/   // 消息队列
│   ├── pkg/             // 通用工具包
│   ├── server/          // HTTP/gRPC服务器
│   └── service/         // API服务实现(连接API与应用层)
└── openapi.yaml
```

## 核心设计原则

### 1. 限界上下文 (Bounded Context)
- 按业务领域划分独立上下文 (如 `internal/domain/user`, `internal/domain/filesystem`)
- **禁止**领域服务直接调用其他上下文的领域服务
- 跨上下文交互通过**应用层编排**或**领域事件**实现

### 2. CQRS 模式
- 应用层严格分离：
  - **Command** (`command/`): 写操作，管理事务
  - **Query** (`query/`): 读操作，无事务

### 3. 事件驱动
- 领域服务发布事件 → 应用层订阅处理
- 实现上下文间松耦合通信
- 优先异步处理

### 4. 依赖倒置
- 领域层定义接口 (如仓储接口)
- 基础设施层实现接口
- 避免包循环依赖

### 5. 领域层与应用层协同
- **领域服务职责**: 执行业务规则，返回变更后的实体或ID列表
- **应用层职责**: 编排用例，管理事务，协调持久化
- **关键**: 领域服务标识受影响实体，应用层统一持久化

## 分层架构

**依赖方向**: 接口层 → 应用层 → 领域层 ← 基础设施层

### 领域层 (Domain Layer)
- **内容**: 实体、值对象、领域服务、仓储接口、领域事件
- **职责**: 执行核心业务规则，处理内存状态变更
- **约束**: 
  - 不依赖具体实现，保持纯粹性
  - 可通过仓储接口**读取**数据，但**禁止直接写入**
  - 不能跨限界上下文调用
- **返回值**: 
  - 单实体操作: 返回变更后的实体 (如 `*entity.FsNode`)
  - 批量操作: 返回主实体和待操作ID列表
- **示例**: `internal/domain/filesystem/service/fs_node_service.go`

### 应用层 (Application Layer)  
- **职责**: 用例编排、事务管理、跨上下文协调
- **流程**: 参数校验 → 预检查 → 调用领域服务 → (事务内)持久化 → 返回结果
- **注意**: 接收领域服务返回的变更实体，在事务中统一持久化
- **示例**: `internal/application/filesystem/command/soft_delete_node.go`

### 基础设施层 (Infrastructure Layer)
- **职责**: 实现领域层定义的接口，提供技术能力
- **设计原则**:
  - **实现领域接口**: 仓储、事件总线、消息队列等接口在领域层定义，基础设施层实现
  - **技术细节封装**: 隐藏数据库、MQ、缓存等技术实现细节
  - **可替换性**: 通过接口隔离，支持技术选型切换（如MySQL→PostgreSQL）
- **使用方式**:
  - 通过**依赖注入**将实现注入到应用层和领域层
  - 仓储实现提供高效的查询方法（如批量操作、递归查询）
  - 避免业务逻辑泄露到基础设施层
- **示例**: `internal/infra/data/*_repo.go` 实现 `internal/domain/*/repository/*.go` 定义的接口

### 接口层 (Interface Layer)
- **职责**: API定义、DTO转换、Protobuf映射
- **原则**: 薄层，对应应用层CQRS用例，内容由应用层组装

## 代码规范

### 1. 错误处理
- 业务错误必须在 `internal/pkg/errs/biz_types.go` 定义
- API错误码必须唯一，不允许重复

### 2. ID类型
- 使用 `pkgEntity.DomainEntityID`
- 与基础类型交互时显式转换 (`.Int64()`)

### 3. 事务管理  
- Command Handler 通过 `IDataTransaction.WithTransaction` 管理事务

### 4. 事件处理 (`internal/application_event`)

**定位**: 应用层的事件订阅处理中心

**处理器类型**:
- **EventBusHandler**: 同步内存事件，处理服务内跨领域逻辑
  - 示例: 用户注册后创建个人空间
- **MessageQueueHandler**: 异步消息队列，处理跨服务通信
  - 示例: 同步外部文档更新到文件系统

**规范**:
- Handler负责编排，不包含核心业务规则
- 异步Handler必须实现**幂等性**
- 管理自己的事务边界
- **禁止**领域层反向依赖 `application_event`

### 5. API规范
- 必须使用Protobuf定义API
- 遵循Kratos错误码规范
- 包含依赖注入配置

## 性能优化策略 (大规模数据操作)

### 默认模式
领域服务返回受影响实体 → 应用层统一持久化 (适用于实体数量可控场景)

### 优化策略

**1. 基于ID的批量操作** (推荐)
- 领域服务返回ID列表而非完整实体
- 应用层在事务中协调仓储执行批量DB操作
- 示例: `FsNodeService.SoftDeleteNode` 返回ID列表 → `FsNodeRepo.BatchMarkAsDeleted` 批量更新

**2. 异步处理**
- 超大规模操作(如删除百万级文件)采用后台任务
- 接受最终一致性，提升响应速度

**权衡原则**: 在职责清晰、数据一致性与性能、内存效率间平衡

## Git提交规范
- `feat`: 新功能
- `fix`: 修复bug  
- `refactor`: 重构
- `docs`: 文档
- `test`: 测试