# 规则库初始化模板

项目首次审查时在**项目根目录**下创建 `docs/rules/` 的模板内容。

**重要**：规则库位于被审查项目的根目录下，例如：
- 审查 `backend/` → 创建 `backend/docs/rules/`
- 审查 `monorepo/services/api/` → 创建 `monorepo/services/api/docs/rules/`

**策略**：最小化、模板化、引导性、可演进

---

## 文件 1: docs/rules/README.md

```markdown
# 项目规则体系

本目录是规范权威来源，所有开发和审查必须遵守。

## 目录结构
```
docs/rules/
├── README.md              (本文件)
├── architecture/          架构规则
├── languages/             编码规范
├── business/              业务规则
├── security/              安全规范
├── performance/           性能要求
└── process/               流程标准
```

## 规则清单

### 架构规则 (architecture/)
| 文件 | 严格度 | 说明 | 更新 |
|-----|--------|------|------|
| principles.md | 🔴强制 | 架构原则 | YYYY-MM-DD |

### 语言规范 (languages/)
| 文件 | 严格度 | 说明 | 更新 |
|-----|--------|------|------|
| coding-standards.md | 🔴强制 | 通用规范 | YYYY-MM-DD |
| {lang}-standards.md | 🔴强制 | 语言规范 | YYYY-MM-DD |

**严格度**：🔴强制（必修）/🟡推荐（可豁免需注释）/🟢参考（可讨论）

## 规则优先级
业务>技术、安全>性能、显式>隐式、项目>通用

## 变更日志
### YYYY-MM-DD - 初始化
创建规则库框架

**待办**：
- [ ] TODO: 补充业务规则
- [ ] TODO: 细化技术规范
```

---

## 文件 2: docs/rules/architecture/principles.md

```markdown
# 架构设计原则

## 原则 1: 模块化和职责分离
每个模块有明确职责边界。
❌ Handler中混杂业务逻辑和数据访问
✅ Handler→Service→Repository 职责分离

## 原则 2: 依赖方向
外层依赖内层，业务层依赖抽象接口，不依赖具体实现。
❌ Service依赖PostgresDatabase
✅ Service依赖RepositoryInterface

## 原则 3: TODO - 补充项目特定原则
提示：架构模式？模块划分？依赖管理？

待完善：
- [ ] 数据一致性原则
- [ ] 可扩展性原则
- [ ] 安全架构原则
```

---

## 文件 3: docs/rules/languages/coding-standards.md

```markdown
# 编码规范

## 命名
- 文件：小写+分隔符
- 变量：小驼峰 `userName`
- 常量：大写+下划线 `MAX_COUNT`
- 布尔值：`is/has/should` 前缀
- 函数：动词开头 `createUser`

## 格式
- TODO: 缩进（2空格/4空格/Tab）
- TODO: 行长度限制
- TODO: 花括号风格

## 注释
✅ 必须：公开API、复杂逻辑、非显而易见决策、TODO/FIXME
❌ 禁止：显而易见代码、重复叙述

TODO: 文档注释风格（Godoc/JSDoc/Docstring）

## 错误处理
- 不忽略错误
- 错误消息清晰可操作
- 适当层次处理

TODO: 错误码系统、日志格式、上报策略

待完善：
- [ ] 项目命名约定
- [ ] 格式规则
- [ ] 错误处理策略
- [ ] 日志规范
- [ ] 测试规范
```

---

## 文件 4: docs/rules/languages/{lang}-standards.md

动态生成，按检测语言创建（go-standards.md / typescript-standards.md 等）

```markdown
# {语言} 编码规范

建议参考：{语言}官方风格指南、社区最佳实践、团队约定

## 基础规范
- TODO: 命名风格（camelCase/snake_case/PascalCase）
- TODO: 函数/类型/常量命名
- TODO: 私有/公开标识符区分
- TODO: 缩进（空格数/Tab）
- TODO: 行长度
- TODO: 导入组织

## 错误处理
- TODO: 标准方式（异常/错误码/Result）
- TODO: 错误信息格式
- TODO: 捕获/传播策略
- TODO: 日志规范

## 并发/异步（如适用）
- TODO: 并发原语使用
- TODO: 共享数据保护
- TODO: 资源管理

## 类型系统（如适用）
- TODO: 类型使用规范
- TODO: 泛型使用
- TODO: 空值处理

## 测试规范
- TODO: 测试命名/组织
- TODO: 覆盖率要求
- TODO: Mock策略

待完善：
- [ ] 补充{语言}特性规范
- [ ] 提炼团队编码风格
- [ ] 记录常见问题和最佳实践
- [ ] 第三方库使用规范
```

**模板调整**：
- 静态类型语言：强调类型系统
- 动态类型语言：强调运行时验证
- 支持并发语言：包含并发章节

---

## 初始化流程

```
1. 确定项目根目录
   - 从当前工作目录或审查对象推断
   - 例如：审查 backend/app/service.go → 项目根 = backend/

2. 项目概要文档检查
   - 读取项目根 README.md

3. 检测技术栈（在项目根下）
   - 统计扩展名（.go/.ts/.py等）
   - 读取依赖文件（go.mod/package.json等）
   - 确定主要语言（按数量排序，取前2-3个）

4. 创建目录（在项目根下）
   mkdir -p {项目根}/docs/rules/{architecture,languages,business,security,performance,process}
   mkdir -p {项目根}/docs/code-review

5. 生成核心文件
   - {项目根}/docs/rules/README.md
   - {项目根}/docs/rules/architecture/principles.md
   - {项目根}/docs/rules/languages/coding-standards.md

6. 生成语言文件
   按检测语言：{项目根}/docs/rules/languages/{lang}-standards.md

7. 生成可选文件
   按用户选择：{项目根}/docs/rules/business/{domain}/

8. 更新README清单
   列出所有文件+严格度+日期

9. 提示用户
   "✅ 已在 {项目根}/docs/rules/ 创建 {N} 个模板文件"
```

---

## 可选扩展文件

按需建议创建：

**architecture/**
- database-design.md：查询性能、索引、事务、迁移
- api-design.md：接口规范、版本管理、兼容性
- patterns-to-avoid.md：反模式案例

**security/**
- requirements.md：认证授权、加密、脱敏、漏洞检查
- compliance.md：GDPR、等保、数据隐私、审计

**performance/**
- baseline.md：响应时间基线、资源限制、性能测试

**process/**
- testing-requirements.md：覆盖率、集成测试、Mock

---

## 业务领域模板

**金融**：business/financial/ → compliance.md（合规）, money-calculation.md（金额计算）, audit.md（审计）, transaction.md（交易一致性）

**电商**：business/ecommerce/ → order-processing.md（订单状态机）, inventory.md（库存）, pricing.md（定价折扣）, payment-flow.md（支付流程）

**SaaS**：business/saas/ → multi-tenancy.md（租户隔离）, rate-limiting.md（限流配额）, data-privacy.md（数据隐私）, subscription.md（订阅计费）

**内容平台**：business/content/ → moderation.md（审核）, recommendation.md（推荐算法）, ugc.md（UGC处理）

**工具/通用**：最小化结构，不预设业务目录

---

## 初始化交互示例

```
项目 [backend] 无规则库

[检测结果]
项目根: /path/to/backend/
语言: Go(45%), TypeScript(38%)
框架: Kratos, React
架构: 分层（app/domain/infra）
规模: 中型(~15K行)

[将在项目根创建]
backend/docs/rules/
├── README.md
├── architecture/principles.md
├── languages/
│   ├── coding-standards.md
│   ├── go-standards.md
│   └── typescript-standards.md
├── business/{domain}/  (可选)
├── security/  (可选)
└── performance/  (可选)

是否创建？[是/稍后]
```
