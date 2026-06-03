# 多框架组合实战：Harness Engineering 的协奏曲

## 引子：单一框架解决不了所有问题

学完前面的文档，你应该已经发现：**没有任何一个框架能覆盖所有问题**。

| 框架 | 强项 | 弱项 |
|------|------|------|
| **Spec-Kit** | 绿地规范管理、企业治理 | 棕地友好度低、执行隔离弱 |
| **OpenSpec** | 棕地增量变更、变更审计 | 绿地启动重、执行隔离弱 |
| **GSD** | 上下文管理、并行执行、验证 | 不管规范、启动成本高 |
| **ECC** | 能力扩展、安全审计 | 不是流程框架、维护成本高 |

**结论**：在真实复杂项目中，**多框架组合是必然选择**。

但组合不是简单的"全都用上"——错误的组合会**互相干扰**，造成 1+1<2 的结果。本文剖析正确的组合策略。

---

## 一、组合的底层逻辑

### 三大框架家族的职责切分



理解组合的前提：每类框架解决**完全不同维度**的问题。

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│          【规范层】SDD 框架                              │
│          解决：写什么（What & Why）                      │
│          代表：Spec-Kit、OpenSpec                       │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│          【执行层】上下文工程框架                         │
│          解决：怎么做（How to Execute）                  │
│          代表：GSD                                      │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│          【能力层】增强类框架                             │
│          解决：用什么做（What Tools）                    │
│          代表：ECC                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**这就是组合的根本基础**：三层各司其职，互不冲突。

### 组合时的"水平"和"垂直"原则



#### 水平原则：同层不要重叠

```
❌ 错误：Spec-Kit + OpenSpec 一起用（都是规范层，会冲突）
❌ 错误：GSD + 另一个上下文工程框架（执行层重叠）
✅ 正确：Spec-Kit + GSD（规范层 + 执行层，互补）
```

**为什么？**
同层的两个框架都试图管理相同的事，会出现：
- 规范放在哪？
- 谁是真相源？
- AI 该听谁的？

### 垂直原则：跨层组合最有效

```
✅ 推荐：SDD（规范） + 上下文工程（执行） + 能力增强（工具）
✅ 推荐：OpenSpec + GSD + ECC
✅ 推荐：Spec-Kit + GSD + ECC
```

**为什么？**
跨层组合让每个框架专注自己擅长的层，**互不抢戏**。

---

## 二、四种经典组合策略



### 组合 1：极简组合（OpenSpec + ECC）

**适用场景**：中小型团队、棕地项目、希望快速接入

**架构**：
```
OpenSpec        → 管理变更生命周期（propose/apply/archive）
   ↓ 在 apply 阶段调用
ECC 的 Agents   → 提供专业能力（coder、security-reviewer 等）
   ↓ Hooks 自动触发
ECC 的 Hooks    → 执行 lint、安全扫描等自动化任务
```

**实战流程**：
```bash
# 1. 用 OpenSpec 提案变更
/opsx:propose 添加用户头像上传功能

# 2. 人类审阅 proposal/specs/design/tasks

# 3. 进入实现阶段，主动调用 ECC 的 security-reviewer
@security-reviewer 评估 design.md 中的安全风险

# 4. 实现代码（PostEdit Hook 自动触发安全扫描）
/opsx:apply

# 5. 提交前 PreCommit Hook 自动跑测试和 lint

# 6. 归档变更
/opsx:archive
```

**优点**：
- 学习曲线适中
- 棕地友好
- 安全和质量自动保障

**缺点**：
- 没有强力的上下文管理
- 适合小到中等规模任务

---

### 组合 2：完整组合（OpenSpec + GSD + ECC）



**适用场景**：中大型项目、长周期开发、高质量要求

**架构**：
```
OpenSpec     → 规范层：管变更契约（specs/）
   ↓ 提供"做什么"
GSD          → 执行层：管上下文+并行+验证（.planning/）
   ↓ 调用
ECC          → 能力层：提供专业 Agents 和 Hooks
```

**关键映射**：

| OpenSpec 阶段 | GSD 阶段 | 协同方式 |
|--------------|---------|----------|
| **Propose** | Discuss + Plan | OpenSpec 的 proposal.md = GSD 的 CONTEXT.md |
| **Apply** | Execute | GSD 把 OpenSpec 的 tasks.md 拆为波次执行 |
| **Apply 中** | - | 调用 ECC 的 security-reviewer、coder 等 Agent |
| **Apply 后** | Verify | GSD 的 Goal-Backward 验证 |
| **Archive** | Ship | GSD 自动归档并生成 PR |

**实战流程**：
```bash
# 第一阶段：用 OpenSpec 锁定规范
/opsx:propose 重构支付模块支持多渠道支付

# 第二阶段：把 OpenSpec 产出导入 GSD
/gsd-import-from-openspec changes/refactor-payment

# 这一步会：
# - 把 proposal.md 内容写入 .planning/CONTEXT.md
# - 把 design.md 转为 plan.xml
# - 把 tasks.md 拆解为带依赖图的原子任务

# 第三阶段：用 GSD 执行
/gsd-execute-phase 1

# 这一步会：
# - 自动并行启动子代理（每个子代理可调用 ECC Agents）
# - 每个任务原子提交
# - 主代理保持上下文轻盈

# 第四阶段：Goal-Backward 验证
/gsd-verify-work 1

# 第五阶段:Ship 并归档
/gsd-ship 1
/opsx:archive
```

**优点**：
- 三层各司其职
- 完整的工程化保障
- 适合长期维护

**缺点**：
- 学习成本高（需要熟悉三个框架）
- 适合已有工程文化的团队

---

### 组合 3：绿地组合（Spec-Kit + GSD + ECC）

**适用场景**：从零开始的新项目、需要企业治理

**架构**：
```
Spec-Kit  → 项目宪法 + 完整规范（绿地优势）
   ↓
GSD       → 执行管理 + 上下文优化
   ↓
ECC       → 专业能力扩展
```

**关键映射**：

| Spec-Kit 阶段 | GSD 阶段 | 协同方式 |
|--------------|---------|----------|
| **Constitution** | (设置 PROJECT.md) | Constitution 内容写入 PROJECT.md |
| **Specify** | Discuss | spec.md 即 Discuss 阶段产出 |
| **Plan** | Plan | plan.md 转换为 plan.xml |
| **Tasks** | Plan（任务部分） | tasks.md 拆为 GSD 的 Task |
| **Implement** | Execute | GSD 接管执行 |
| - | Verify | Spec-Kit 没有，GSD 补充 |
| - | Ship | Spec-Kit 没有，GSD 补充 |

**实战流程**：
```bash
# Spec-Kit 启动新项目
specify init my-project --ai claude

# Spec-Kit 五阶段
/speckit.constitution
/speckit.specify
/speckit.plan
/speckit.tasks

# 切换到 GSD 执行
/gsd-import-from-speckit specs/feature-A
/gsd-execute-phase 1
/gsd-verify-work 1
/gsd-ship 1

# 后续迭代如果需要变更管理，可切换到 OpenSpec
# 但通常 Spec-Kit 内的迭代用 /speckit.specify 增量补充即可
```

---

### 组合 4：渐进组合（先 ECC，再加 OpenSpec/GSD）

**适用场景**：现有项目想接入 Harness、团队还在适应中

**渐进步骤**：

#### Step 1: 只用 ECC（第一周）
- 安装 ECC
- 团队学习几个核心 Agents（coder、security-reviewer、code-reviewer）
- 配置基础 Hooks（PreCommit lint）

#### Step 2: 加入 OpenSpec（第 2-4 周）
- 在新功能开发中开始用 `/opsx:propose`
- 不强制立即用于所有功能
- 让团队感受规范驱动的好处

#### Step 3: 加入 GSD（第 2-3 个月）
- 在大型功能开发中用 GSD 做执行管理
- 体验上下文管理的价值
- 逐步把所有项目纳入 GSD

**这种渐进路径降低了团队抵触**，让组合方案自然落地。

---

## 三、组合时的常见陷阱



### 陷阱 1：规范源冲突

**症状**：Spec-Kit 的 `spec.md` 和 OpenSpec 的 `specs/*` 同时存在，AI 不知道听哪个。

**根因**：违反了"水平原则"——同层不要重叠。

**解决**：选一个 SDD 框架就够了。如果绿地用 Spec-Kit 启动，后期可平滑迁移到 OpenSpec（导出 spec.md 为 OpenSpec 主规范）。

### 陷阱 2：上下文重复加载

**症状**：每个 ECC Agent 都加载完整的 spec.md + plan.md + 整个代码库，token 成本暴涨。

**根因**：没有用 GSD 的子代理隔离机制。

**解决**：用 GSD 包装 ECC Agent 调用，确保每个 Agent 只看到自己需要的上下文文件。

### 陷阱 3：阶段重复执行

**症状**：OpenSpec 在 propose 阶段已经设计了，GSD 又在 Plan 阶段重新设计一遍。

**根因**：没有把两者的产出**互相导入**。

**解决**：用 `/gsd-import-from-openspec` 这样的桥接命令，让 GSD 直接复用 OpenSpec 的产物。

### 陷阱 4：纪律松散

**症状**：团队"看心情"用某个框架，有时用 OpenSpec 有时直接 vibe coding。

**根因**：组合方案没有写成强制规则。

**解决**：在 CLAUDE.md 或 constitution.md 中明确规定：
> 所有功能开发**必须**走 OpenSpec → GSD 流程。
> bug 修复 < 30 行**可以**跳过 OpenSpec，直接 GSD 快速执行。

### 陷阱 5：能力过剩

**症状**：项目还很小，就上了 OpenSpec + GSD + ECC 全套，团队被流程拖累。

**根因**：盲目追求"完整"。

**解决**：按项目规模选择组合：
- 小项目（< 1 人月）：纯 vibe coding 或 ECC
- 中项目（1-3 人月）：OpenSpec + ECC
- 大项目（> 3 人月）：OpenSpec + GSD + ECC

---

## 四、决策树：你该用什么组合？



```
项目规模？
├── 小（< 1 人月）
│   └── 纯 ECC（保留一些专业能力即可）
│
├── 中（1-3 人月）
│   ├── 棕地（已有项目）
│   │   └── OpenSpec + ECC
│   └── 绿地（新项目）
│       └── Spec-Kit + ECC
│
└── 大（> 3 人月）
    ├── 棕地（已有项目）
    │   ├── 高质量要求？
    │   │   ├── 是 → OpenSpec + GSD + ECC（完整组合）
    │   │   └── 否 → OpenSpec + ECC（极简组合）
    │   └── 微服务架构？
    │       └── OpenSpec + GSD + ECC（GSD 管多服务调度）
    └── 绿地（新项目）
        ├── 企业级合规要求？
        │   ├── 是 → Spec-Kit + GSD + ECC
        │   └── 否 → OpenSpec + GSD + ECC（更轻便）
        └── 团队偏好？
            ├── 习惯瀑布 → Spec-Kit + GSD + ECC
            └── 习惯敏捷 → OpenSpec + GSD + ECC
```

---

## 五、完整剧本：用 OpenSpec + GSD + ECC 实现一个功能

### 场景

> **项目**：电商平台（已运行 1 年）
> **需求**：实现"用户行为分析"——记录用户点击、浏览、购买行为，并提供分析报表

### 阶段 0：环境准备

```bash
# 已有项目接入 OpenSpec
openspec init

# 已有项目接入 GSD
/gsd-init-existing-project

# 已有项目接入 ECC
git submodule add https://github.com/.../everything-claude-code.git .claude-tools
# 配置 PreCommit Hook、PostEdit Hook
```

### 阶段 1：OpenSpec Propose

```bash
/opsx:propose 添加用户行为分析模块，记录点击/浏览/购买事件，提供报表
```

AI 生成：
```
changes/add-behavior-analytics/
  proposal.md
  specs/
    analytics/event-tracking.md
    analytics/reporting.md
    api/analytics-endpoints.md
  design.md
  tasks.md
```

人类审阅 → 批准

### 阶段 2：导入 GSD

```bash
/gsd-import-from-openspec changes/add-behavior-analytics
```

GSD 自动转换：
- `proposal.md` → `.planning/CONTEXT.md`（追加）
- `design.md` → `.planning/milestones/M5/phases/phase-1/plan.xml`
- `tasks.md` → 解析任务依赖图，生成原子任务列表

### 阶段 3：GSD Discuss（如有遗漏）

```bash
/gsd-discuss-phase 1
```

AI：
> 我注意到 design.md 没有提到事件采集的频率限制。如果用户高频点击会产生大量事件，需要确定：
> 1. 客户端是否做防抖？防抖时间多少？
> 2. 服务端是否做限流？每用户每秒最多多少事件？

人类回答 → 决策写入 CONTEXT.md

### 阶段 4：GSD Plan（如需细化）

```bash
/gsd-plan-phase 1
```

AI 进一步拆解任务，标注依赖：
```xml
<task id="t1" status="pending">
  <description>创建 events 表 schema 和 migration</description>
  <validation>migration 可成功运行</validation>
</task>

<task id="t2" status="pending" depends_on="t1">
  <description>实现 EventService.track 方法（含限流）</description>
  <validation>单元测试覆盖正常和限流场景</validation>
  <agent_hint>调用 @security-reviewer 评估限流安全性</agent_hint>
</task>

<!-- ... 共 12 个任务 -->
```

### 阶段 5：GSD Execute（含 ECC 调用）

```bash
/gsd-execute-phase 1
```

GSD 主代理：
1. 解析任务依赖图，分波次：
   - 波次 1：t1, t9（无依赖）
   - 波次 2：t2, t3, t10（依赖波次 1）
   - 波次 3：...

2. 对每个任务启动子代理：
```
子代理 1（执行 t1）：
  - 加载 plan.xml 中 t1 的描述
  - 加载 prisma/schema.prisma（context_files）
  - 调用 @coder Agent 实现
  - 调用 @code-reviewer Agent 审查
  - 提交 commit "feat(M5.P1.T1): create events table"

子代理 2（执行 t2）：
  - 加载 t2 描述
  - 加载 src/services/event.service.ts
  - 调用 @coder Agent 实现
  - 调用 @security-reviewer Agent 评估限流（来自 agent_hint）
  - 调用 @tdd-tester Agent 写测试
  - 提交 commit "feat(M5.P1.T2): implement EventService.track"
```

3. PostEdit Hook 自动触发：
   - 每次文件编辑后自动运行 lint
   - 安全扫描自动检查敏感信息

### 阶段 6：GSD Verify（Goal-Backward）

```bash
/gsd-verify-work 1
```

AI 从用户视角验证：
- 用户点击商品 → 事件能否被记录？
- 管理员查看报表 → 数据能否正确聚合？
- 高频点击 → 限流是否生效？

发现：
- ✅ 单事件记录正常
- ✅ 报表聚合正确
- ❌ 限流生效但前端没有友好提示 → 创建一个修复任务

修复后再次 verify → 通过

### 阶段 7：Ship

```bash
/gsd-ship 1
```

自动：
1. 生成 PR：`feat: 添加用户行为分析模块`
2. PR 描述包含本次 12 个任务的摘要
3. 关联相关 Issue
4. 标记 phase-1 为 completed

### 阶段 8：OpenSpec Archive

```bash
/opsx:archive add-behavior-analytics
```

自动：
1. 把 Delta Specs 合并到主规范：
   - `specs/analytics/event-tracking.md` → 系统正式规范
   - `specs/analytics/reporting.md` → 系统正式规范
   - `specs/api/analytics-endpoints.md` → 追加到现有 API 规范
2. 把 `changes/add-behavior-analytics/` 移到 `changes/archived/`

### 完整时序图

```
人类           OpenSpec              GSD                 ECC                Git
 │                │                  │                   │                  │
 ├─propose──────→│                  │                   │                  │
 │                ├─generate specs   │                   │                  │
 │←──review──────┤                  │                   │                  │
 │approve──────→│                  │                   │                  │
 │                │                  │                   │                  │
 ├─import──────────────────────────→│                   │                  │
 │                │                  ├─convert to plan.xml                  │
 │                │                  │                   │                  │
 ├─execute─────────────────────────→│                   │                  │
 │                │                  ├─wave-1 starts────│                  │
 │                │                  │  ├─sub-agent-1──→│                  │
 │                │                  │  │   call @coder │                  │
 │                │                  │  │←─done─────────┤                  │
 │                │                  │  │──commit T1───────────────────────→│
 │                │                  │  ├─sub-agent-2──→│                  │
 │                │                  │  │   ...          │                  │
 │                │                  ├─wave-2 starts────│                  │
 │                │                  │  ...              │                  │
 │                │                  │                   │                  │
 ├─verify──────────────────────────→│                   │                  │
 │                │                  ├─Goal-Backward test                   │
 │←──issues──────────────────────────┤                   │                  │
 │fix issues...                                                              │
 │                │                  │                   │                  │
 ├─ship────────────────────────────→│                   │                  │
 │                │                  ├─generate PR──────────────────────────→│
 │                │                  │                   │                  │
 ├─archive─────→│                  │                   │                  │
 │                ├─merge to main specs                  │                  │
```

---

## 六、组合的协同效应（1+1+1 > 3）



### 协同 1：OpenSpec 提供"为什么"，GSD 提供"怎么做"

OpenSpec 的 `proposal.md` 解释**变更的意图和影响范围**——这是 GSD 在 Discuss 阶段最需要的信息。
GSD 不再需要重新讨论"为什么做"，可以直接进入"怎么做"。

### 协同 2：GSD 的并行执行 + ECC 的专业 Agent

GSD 启动 5 个子代理并行处理 5 个任务。
每个子代理在自己的上下文中调用 ECC 的 `@coder`、`@security-reviewer` 等专业 Agent。
**结果**：5 倍速度 × 专业质量 = 高速高质量。

### 协同 3：ECC 的 Hooks + GSD 的状态管理

ECC 的 `PostEdit Hook` 自动跑安全扫描。
GSD 的 `STATE.md` 记录每次扫描结果。
**结果**：质量自动保障，进度全程可见。

### 协同 4：OpenSpec 的归档 + GSD 的 Verify

OpenSpec 的 archive 把 Delta 合并到主规范，**前提是**实现真的符合规范。
GSD 的 Goal-Backward Verify **保证**了这个前提。
**结果**：归档后的主规范真实反映系统行为，避免规范漂移。

---

## 七、组合的反协同（什么时候不该组合？）

### 反协同 1：项目太小

如果你的项目只有 200 行代码，搭三个框架的成本远大于收益。
**建议**：纯 vibe coding 或单 ECC。

### 反协同 2：团队还没适应

强行让团队同时学三个框架，会造成抵触。
**建议**：渐进引入（参见组合 4）。

### 反协同 3：工具不支持

如果你用的 AI 工具不支持子代理（Sub-Agent），GSD 的核心机制无法生效。
**建议**：要么换工具（推荐 Claude Code），要么去掉 GSD（用 OpenSpec + ECC 就够了）。

### 反协同 4：工程文化不匹配

如果团队偏好"快速试错、推倒重来"，规范驱动会让人感到束缚。
**建议**：尊重文化，从 ECC 这种灵活工具开始，慢慢演进。

---

## 八、组合方案的演进路径



成熟团队的典型演进路径：

```
Year 0   ─→  纯 vibe coding，初步接触 AI 编程
            ↓
Month 3  ─→  接入 ECC，体验专业能力
            ↓
Month 6  ─→  小范围试点 OpenSpec
            ↓
Month 9  ─→  全面推行 OpenSpec + ECC
            ↓
Month 12 ─→  大型项目接入 GSD
            ↓
Month 18 ─→  完整组合 OpenSpec + GSD + ECC
            ↓
Year 2   ─→  团队定制自己的扩展（基于这套基础）
```

**关键点**：演进是**渐进的**，不是一蹴而就的。每个阶段都要等团队真正掌握当前组合，再加入新框架。

---

## 九、关键启发



通过多框架组合的深入分析，得到的核心启发：

1. **没有银弹**——任何单一框架都有边界
2. **分层组合最有效**——规范层 + 执行层 + 能力层各司其职
3. **同层不要重叠**——避免冲突的关键
4. **桥接是关键**——框架间的产物互导是组合的成败所在
5. **渐进引入**——团队适应比一次到位更重要
6. **决策树思考**——根据项目特征选组合，而非追求"最先进"

---

## 十、扩展阅读

- [掘金 - 6 个代表性 AI 编程 Harness 工程化框架拆解](https://article.juejin.cn/post/7626998085928189998)
- [腾讯云 - OpenSpec vs Superpowers：2 套 AI 编码工作流怎么选？](https://developer.cloud.tencent.com/article/2649111)
- [GitHub - spec-kit](https://github.com/github/spec-kit)
- [GitHub - OpenSpec](https://github.com/Fission-AI/OpenSpec)
- [GSD 官方文档](https://github.com/gsd-build/get-shit-done)
- [Everything Claude Code](https://github.com/everything-claude-code/everything-claude-code)
- [Anthropic - Claude Code 文档](https://docs.claude.com/en/docs/claude-code)

---