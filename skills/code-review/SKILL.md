---
name: comprehensive-code-review
description: 五角色代码审查：基于 docs/rules/ 项目规则，从架构/技术/规范/性能/业务维度深度分析代码。支持规则库初始化和知识积累。触发词："审查代码"、"code review"、"检查提交"。输出中文。
---

# 全面代码审查技能

角色驱动的规则合规性审查流程。纯流程引擎，0%规则内容，100%依赖项目 `docs/rules/`。

**重要**：`docs/rules/` 位于被审查项目的根目录下。例如审查 `backend/` 项目，规则库应在 `backend/docs/rules/`。

---

## 工作流程

### 阶段 1: 规则库准备

#### 检查和加载规则

```bash
# 在项目根目录下读取
read {项目根目录}/docs/rules/README.md
```

- **存在**：解析 README 索引，加载所有规则文件（Glob扫描目录或按清单读取）
- **不存在**：执行初始化

#### 初始化引导（首次使用）

**Step 1: 确定项目根目录**
```
从当前工作目录或用户指定的审查范围推断项目根目录：
- 如果在 backend/ 目录下执行审查 → 项目根 = backend/
- 如果审查指定 backend/ 下的文件 → 项目根 = backend/
- 如果在 monorepo/services/api/ 下 → 项目根 = services/api/
- 规则库路径 = {项目根}/docs/rules/
```

**Step 2: 自动检测项目特征**（在项目根目录下）
- 读取项目概要文档 README.md
- 统计文件扩展名分布，确定主要语言（Glob查找 `**/*.{ext}`）
- 读取依赖文件（go.mod/package.json等），提取框架
- 推断架构风格（从目录结构）

**Step 3: 展示计划并确认**
```
项目 [{项目名}] 无规则库

[检测结果]
项目根: {项目根目录}
语言: {语言列表}
框架: {框架列表}
架构: {推断结果}

[将创建]
{项目根}/docs/rules/README.md
{项目根}/docs/rules/best-practices/README.md         # 最佳实践案例索引
{项目根}/docs/rules/architecture/principles.md
{项目根}/docs/rules/languages/coding-standards.md
{项目根}/docs/rules/languages/{lang}-standards.md
...

是否创建？[是/稍后]
```

- **是**：创建规则库（遵循规则编写规范），继续审查
- **否**：暂停，提示无规则库无法审查

#### 规则编写规范（初始化时遵循）

**原则**：≤150行/文件，用文件引用替代示例代码，创建 `best-practices/README.md`

**规则模板**：
```markdown
# 规则标题
**严重性**: critical/high/medium  
**适用**: 路径模式

> **参考**: docs/rules/best-practices/README.md

## 1. 规则组
### 1.1 规则名
**规则**: 一句话描述  
**参考**: `文件路径:行号`

## 检查点
- [ ] 检查项
```

**最佳实践模板**：
```markdown
# 最佳实践案例库
## 架构
- **领域服务**: `internal/domain/*/service/*.go` - 返回实体不持久化
## 技术  
- **错误**: `internal/pkg/errs/biz_types.go` - 集中管理
## 反面案例
- (审查补充)
```

---

### 阶段 2: 分析审查范围

#### 确定审查对象

| 请求 | 对象 | 命令 |
|-----|------|------|
| "审查代码" | 未提交变更 | `git diff HEAD` |
| "审查提交" | 最近提交 | `git show HEAD` |
| "审查分支" | 分支差异 | `git diff main...branch` |
| "审查文件" | 指定文件 | 读取文件 |
| "审查PR" | PR变更 | `gh pr diff` |

#### 分析变更特征

**收集统计数据**：
```bash
# 1. 统计文件类型和数量
grep "^\+\+\+ b/" diff.txt | awk -F. '{print $NF}' | sort | uniq -c
# 输出示例: 15 .go, 2 .proto, 1 .yaml

# 2. 统计变更行数
git diff --stat | tail -1
# 输出示例: 51 files changed, 2866 insertions(+), 698 deletions(-)

# 3. 提取变更文件路径
grep "^\+\+\+ b/" diff.txt | sed 's/^+++ b\///'
# 输出示例: internal/domain/user/entity/user.go
```

**分析维度**：
- 文件类型分布（.go/.proto/.yaml等）
- 变更规模（行数、文件数）
- 涉及层次（domain/application/infra/bootstrap/conf等）
- 关键词（transaction/repository/service/config等）

**确定审查深度**：
- <50行 → quick（快速审查）
- 50-200行 → standard（标准审查）
- 200-500行 → deep（深度审查）
- >500行 → comprehensive（全面审查）

#### 角色需求评估 (必须执行)

**原则**：根据变更内容评估，非机械应用行数规则。

| 角色 | 触发条件（满足任一）（参考）|
|-----|------------------|
| 🏛️ 架构 | 跨层改动 / 接口重构 / 新增组件 / >200行多层次 |
| 🔧 技术 | 外部调用 / 事务并发 / 错误处理 / 配置安全 / 有代码改动 |
| 📏 规范 | 有代码改动（几乎总是需要） |
| ⚡ 性能 | infra/data/ / 数据库查询 / 批量操作 / >200行 |
| 💼 业务 | domain/ / 业务规则 / application/command|query |

**示例**：
- 配置修改30行 → 技术+规范
- 仓储重构200行 → 架构+技术+规范+性能
- 新增系统功能2866行 → 全部5角色

---

### 阶段 3: 双轨制并行审查

**两轨策略**：
1. **第一轨**：规则驱动审查（5角色并行，必须引用规则）
2. **第二轨**：通用实践补充（主流程扫描，标注"规则外发现"）

---

#### 第一轨：规则驱动审查

| 角色 | 审查范围 | 典型规则库路径 |
|-----|---------|--------------|
| 🏛️ 架构 | 分层/依赖/CQRS/接口 | architecture/** |
| 🔧 技术 | 正确性/安全/错误/并发 | languages/**, security/**, process/error-handling.md |
| 📏 规范 | 命名/注释/格式 | languages/coding-standards.md, *-standards.md |
| ⚡ 性能 | 查询/批量/并发/缓存 | performance/**, architecture/database-design.md |
| 💼 业务 | 领域模型/业务规则 | business/**, architecture/principles.md |

**工作流**：读取README → 选择2-5个规则文件 → 分析diff → 输出JSON（必须引用规则）

---

#### 第二轨：通用实践补充

**检查清单**（6类18项）：
- **代码结构**: 空代码块、注释与实现不符、无限循环
- **资源管理**: 资源泄漏、循环中创建资源未释放、上下文未传递超时
- **数据一致性**: 事务边界不明确、部分操作失败未补偿、幂等性未实现
- **并发安全**: 共享状态无保护、goroutine panic未恢复、死锁风险
- **日志告警**: 日志级别使用不当、敏感信息泄漏、错误日志缺少上下文
- **远程调用**: 循环中调用远程服务、远程调用无超时、批量操作未分片

**原则**: ✅ 明显缺陷直接指出 | ❌ 主观偏好/深度业务理解不检查

---

#### 执行流程

```
启动5角色Task（第一轨）→ 等待JSON → 主流程扫描diff（第二轨）→ 合并去重 → 生成报告
```

**去重**: 同一位置规则驱动优先，按严重性排序

---

### 阶段 4: 汇总生成报告

**流程**: 收集JSON → 合并去重 → 分级 → 生成简洁报告

**报告路径**: `{项目根}/docs/code-review/YYYY-MM-DD-{主题}.md`

**报告原则**：
- **极简格式**: 问题直接列要点，不重复背景
- **引用规则**: 必须引用 `docs/rules/xxx.md:行号`
- **精准定位**: 文件路径:行号，核心代码片段（≤5行）
- **修复优先**: 重点是如何修复，不是长篇分析

**报告模板**（双轨制）：

```markdown
# 代码审查：[主题]
**时间**: YYYY-MM-DD | **范围**: +X -Y lines, Z files

## 📊 综合评估
| 维度 | 分数 | 规则内 | 规则外 | 说明 |
|-----|------|-------|-------|------|
| 架构 | 3/5 | 5 | 2 | 跨层依赖、空代码块 |
| 技术 | 3/5 | 8 | 4 | 错误处理、资源泄漏 |

**总计**: X个问题（Y critical, Z high, ...）

## 🔴 关键问题 (Critical)
### [ARCH-01] 标题 【规则驱动】
**规则**: `docs/rules/xxx.md:L123` | **位置**: `file.go:45`  
**问题**: 一句话 | **修复**: 代码示例

### [TECH-05] 标题 【规则外发现】
**类别**: 资源管理 | **位置**: `file.go:123`  
**问题**: 一句话 | **修复**: defer resp.Body.Close()  
**建议规则**: 补充到 `docs/rules/languages/go-standards.md`

## 💡 亮点 (X个)
- `file.go:23` - 配置驱动设计 ✅

## 📚 规则演进
- [ ] 补充现有: `xxx.md` - 内容
- [ ] 新增文件: `yyy.md` - 原因
- [ ] 最佳实践: 补充X个案例

## 总结
[功能范围] | [整体评价] | **是否合并**: ✅/⚠️/❌

**审查模式**: 双轨制（规则驱动 + 通用实践补充）
```

---

## 阶段 5: 知识积累

**询问用户**：发现X个新模式，是否更新？  
**确认后**：插入规则 → 更新README.md日期  
**最佳实践**：补充到 `best-practices/README.md`

---

## 执行细节

### 角色输出格式（JSON）

**第一轨**（规则驱动，必须引用规则）：
```json
{
  "role": "角色名", "score": 1-5,
  "issues": [{"severity": "critical/high/medium/low", "title": "标题", 
    "rule_source": "docs/rules/xxx.md:行号", "problem": "一句话", 
    "file": "路径", "line": 行号, "code_snippet": "≤5行", "fix": "修复方案"}],
  "highlights": [{"title": "标题", "file": "路径", "line": 行号, "description": "描述"}],
  "rule_gaps": [{"description": "规则空白", "suggestion": "建议"}],
  "summary": "总体评价2-3句话"
}
```

**第二轨**（通用实践补充，标注类别和建议规则）：
```json
{
  "beyond_rules_issues": [{"severity": "critical/high/medium/low", "category": "6类之一", 
    "title": "标题", "problem": "一句话", "file": "路径", "line": 行号, 
    "code_snippet": "≤5行", "fix": "修复方案", "suggest_rule": "建议补充的规则"}]
}
```

### 文件类型-规则映射（参考）

| 类型/路径 | 关注规则 | 优先级 |
|---------|---------|--------|
| .proto | api-design | high |
| .yaml/.env | configuration | critical |
| domain/*/service/ | principles, domain-rules | critical |
| infra/data/repo/ | database-design, database-optimization | high |

---

## 关键机制

**评分**: 5完美/4良好/3合格/2不足/1严重  
**编号**: [ARCH/TECH/STD/PERF/BIZ-序号]  
**引用**: `docs/rules/xxx.md:行号` + 原文

**演进触发**: 重复问题→新规则、反模式→patterns-to-avoid、优秀代码→best-practices  
**瘦身**: >150行→提取示例，用引用替代

---

## 特殊场景

**规则冲突**: 标注冲突规则，建议优先级  
**规则过时**: 标注并建议修订  
**规则空白**: 第一轨标注rule_gaps；第二轨通用实践清单覆盖则直接指出  
**两轨冲突**: 规则驱动优先，同一问题标注【规则驱动】或【规则外发现】

---

## 核心原则

### 双轨制原则

**第一轨（规则驱动）**：
- 以 {项目根}/docs/rules/ 为准，每个问题必须引用规则文件:行号
- 规则空白标注到rule_gaps

**第二轨（通用实践）**：
- 明显缺陷直接指出（空块/资源泄漏/幂等性bug）
- 为每个"规则外发现"建议补充规则
- 避免主观判断和深度业务理解问题

**通用**：提供修复方案 | 识别亮点补充best-practices | 促进规则演进 | 智能筛选规则（≤150行/文件）

---

## 质量保证

### 规则瘦身
**检查**: `wc -l docs/rules/**/*.md | awk '$1>150'`  
**策略**: 提取示例→best-practices/，用引用替代

### 最佳实践维护
**时机**: 审查发现优秀代码  
**格式**: `- **场景**: 文件:行号 - 亮点`  
**标准**: 清晰、典型、可参考
