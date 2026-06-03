# GSD 深度剖析：上下文工程的极致代表

## 引子：为什么 GSD 值得单独写一篇？

如果说 SDD 框架（Spec-Kit、OpenSpec）解决的是"规范怎么管"，能力增强框架（ECC）解决的是"能力怎么扩"，那么 **GSD 解决的是 AI 编程最底层、最根本的问题：上下文窗口怎么用**。

选择 GSD 作为代表的原因：

1. **问题最深刻**：直接攻击 Context Rot 这个 AI 编程的核心矛盾
2. **设计最纯粹**：所有机制都为"上下文管理"服务，没有冗余
3. **可工业化**：把"AI 唠嗑式编程"改造为"工业流水线"
4. **三大原理的极致体现**：决策外化、流程阶段化、任务原子化在 GSD 中都被推到极致
5. **可与 SDD/ECC 组合**：与其他类型框架天然互补

---

## 一、GSD 的设计哲学

### 核心定位

> **GSD 是"AI 编程的工业流水线"——它把不可控的对话式编程，改造为可观测、可调度、可恢复的工程化流程。**

它不关心"规范怎么写"（那是 SDD 的事），不关心"能力怎么扩"（那是 ECC 的事），它**只关心一件事**：

> **如何让 AI 在长项目中保持高质量产出，对抗 Context Rot。**

### 设计哲学

#### 哲学 1：上下文是稀缺资源，必须显式管理

通用 AI 把上下文当成无限资源用，结果总是耗尽
GSD 的做法 = 把上下文当成**预算**来管理

具体体现：
- 主会话上下文严格控制在 30-40% 占用率
- 每个子任务有 200K Token 的独立上下文上限
- 阶段切换时**强制 `/clear`**，重置上下文
- 信息存储在文件系统，按需加载，不长期占用上下文

#### 哲学 2：流水线优于工匠

通用 AI 编程像工匠手作——质量看心情、看运气
GSD 的做法 = 像工业流水线——稳定、可预测、可重复

具体体现：
- 五阶段标准化（Discuss → Plan → Execute → Verify → Ship）
- 每阶段有明确的输入、输出、验收标准
- 阶段间强制隔离，不互相污染
- 失败可局部回滚，不影响其他阶段

#### 哲学 3：跨会话记忆基于文件系统而非对话历史

通用 AI 跨会话失忆——每次都要重新讲一遍
GSD 的做法 = 文件系统是"长期记忆"

具体体现：
- `.planning/` 目录是项目的"单一真相源"
- `STATE.md` 实时记录当前进度
- `CONTEXT.md` 记录关键决策
- 新会话自动加载这些文件，无需用户重述

---

## 二、GSD 的核心架构

### 顶层目录结构

```
your-project/
├── .planning/               ← GSD 的"操作系统"
│   ├── PROJECT.md          ← 项目愿景与全局约束
│   ├── ROADMAP.md          ← 阶段路线图
│   ├── STATE.md            ← 实时进度（自动维护）
│   ├── CONTEXT.md          ← 关键决策记录
│   └── milestones/
│       ├── M1-foundation/
│       │   ├── phases/
│       │   │   ├── phase-1-auth/
│       │   │   │   ├── plan.xml         ← 带验证标准的 XML 计划
│       │   │   │   ├── tasks/           ← 原子任务清单
│       │   │   │   └── results/         ← 每个任务的执行结果
│       │   │   └── phase-2-payment/
│       │   └── overview.md
│       └── M2-features/
└── src/                     ← 实际代码
```

### 关键概念解析

#### 概念 1：`.planning/` 目录

**定位**：GSD 框架的"操作系统级"存储

**特点**：
- **不在代码仓库的核心目录**（.planning 名字开头表示元数据）
- **AI 和人类共同维护**
- **结构化、可遍历、可索引**

它就像 Git 的 `.git/` 目录之于代码仓库——**支撑整个工作流的基础设施**。

#### 概念 2：里程碑（Milestone）→ 阶段（Phase）→ 任务（Task）

GSD 的核心拆解模型：

```
项目 (Project)
  └── 里程碑 1 (Milestone)         ← 大约 1-2 周的范围
      ├── 阶段 1 (Phase)           ← 大约 1-3 天的范围
      │   ├── 任务 1 (Task)        ← 30 分钟内可完成
      │   ├── 任务 2
      │   └── 任务 3
      └── 阶段 2
  └── 里程碑 2
```

**三层粒度的意义**：
- **Milestone**：业务交付单位（可演示、可验收的功能集）
- **Phase**：技术实现单位（架构上独立、可测试）
- **Task**：执行单位（30 分钟内、单上下文可完成）

#### 概念 3：XML 格式的 plan.xml

GSD 选择 XML 而不是 Markdown 的原因：
- **强结构化**：明确的字段、嵌套关系
- **机器可解析**：AI 不会"理解错"格式
- **支持验证**：可以自动校验完整性

**示例（简化版）**：
```xml
<phase id="phase-1-auth" milestone="M1">
  <objective>实现用户认证系统</objective>

  <validation_criteria>
    <criterion>用户可以注册、登录、登出</criterion>
    <criterion>JWT 有效期 24 小时</criterion>
    <criterion>所有 API 单元测试通过</criterion>
    <criterion>集成测试覆盖关键路径</criterion>
  </validation_criteria>

  <tasks>
    <task id="task-1" status="completed">
      <description>创建 users 表 schema 和 migration</description>
      <validation>migration 可成功运行</validation>
      <context_files>
        <file>prisma/schema.prisma</file>
      </context_files>
    </task>

    <task id="task-2" status="in_progress" depends_on="task-1">
      <description>实现 AuthService.register 方法</description>
      <validation>测试 auth.service.test.ts 通过</validation>
      <context_files>
        <file>src/services/auth.service.ts</file>
        <file>src/services/auth.service.test.ts</file>
      </context_files>
    </task>

    <task id="task-3" depends_on="task-2">
      <description>实现 AuthService.login 方法</description>
      ...
    </task>
  </tasks>
</phase>
```

**关键字段解析**：

| 字段 | 作用 |
|------|------|
| `validation_criteria` | 阶段完成的验收标准（Goal-Backward 验证） |
| `tasks` | 原子任务列表 |
| `task.status` | 任务执行状态 |
| `task.validation` | 单任务的验收标准 |
| `task.depends_on` | 依赖关系，决定执行顺序 |
| `task.context_files` | 这个任务需要加载的上下文文件 |

#### 概念 4：状态机驱动的进度管理

`STATE.md` 不是一个简单的文档，而是一个**状态机**：

```markdown
# 项目状态

## 当前位置
- Milestone: M1-foundation
- Phase: phase-1-auth
- Task: task-2 (in_progress)

## 已完成
- M1.phase-1.task-1 ✅ (2026-06-01)

## 待执行
- M1.phase-1.task-3
- M1.phase-1.task-4
- M1.phase-2 (整个 phase 待启动)

## 阻塞
- task-3 等待 task-2 完成

## 上下文文件
- 当前 phase 计划: .planning/milestones/M1/phases/phase-1-auth/plan.xml
- 全局上下文: .planning/CONTEXT.md
```

新会话启动时，AI 自动读取这个文件，**3 秒内完全恢复工作状态**。

---

## 三、GSD 的五阶段工作流

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Discuss  │→ │  Plan    │→ │ Execute  │→ │ Verify   │→ │  Ship    │
│ (讨论)   │  │ (规划)   │  │ (执行)   │  │ (验证)   │  │ (交付)   │
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

### 阶段 1：Discuss（讨论）

**命令**：`/gsd-discuss-phase N`

**目标**：明确需求细节，锁定决策

**核心活动**：
1. AI 主动询问需求中的模糊点
2. 用户与 AI 反复对话，澄清边界
3. 关键决策被记录到 `CONTEXT.md`

**产出**：
- `CONTEXT.md` 更新（含本阶段的关键决策）
- 模糊点全部消除，可进入 Plan 阶段

**反模式**：
- 跳过 Discuss 直接 Plan → AI 会自己脑补需求
- Discuss 阶段动手写代码 → 决策不稳定

### 阶段 2：Plan（规划）

**命令**：`/gsd-plan-phase N`

**目标**：拆解任务为原子单元，生成可执行计划

**核心活动**：
1. AI 读取 CONTEXT.md，理解决策
2. 分析现有代码库（如果是棕地）
3. 拆解为原子任务（每个 30 分钟内可完成）
4. 标注任务依赖关系
5. 定义每个任务的验收标准

**产出**：
- `plan.xml`（带验证标准的 XML 计划）
- 任务依赖图（DAG）

**关键点**：Plan 阶段**不写代码**，只规划。这是 GSD 的纪律。

### 阶段 3：Execute（执行）

**命令**：`/gsd-execute-phase N`

**目标**：通过子代理并行处理任务

**核心活动**：
1. 主代理读取 plan.xml，识别任务依赖图
2. 把无依赖的任务分组为"波次"（Wave）
3. 为每个任务**启动独立子代理**
4. 子代理在干净的上下文中执行任务
5. 每个任务完成后**独立提交 Git commit**
6. 主代理收集结果，更新 STATE.md

**关键机制**：
- **波次执行（Wave Execution）**：无依赖任务并行
- **上下文隔离**：每个子代理只看到自己任务相关的文件
- **原子提交**：每个任务一个 commit，便于回滚

#### 波次执行示意

假设一个 Phase 有 8 个任务，依赖关系：

```
Task 1 ─┐
Task 2 ─┼─→ Task 5 ─┐
Task 3 ─┘           ├─→ Task 7 ─→ Task 8
                    │
Task 4 ───→ Task 6 ─┘
```

GSD 自动分波次：

```
波次 1（4 个任务并行）：Task 1, 2, 3, 4
波次 2（2 个任务并行）：Task 5, 6
波次 3（1 个任务）   ：Task 7
波次 4（1 个任务）   ：Task 8
```

**理论加速比**：8 个任务串行执行需要 8N 时间，波次并行只需要 4N 时间（**2 倍加速**）。

### 阶段 4：Verify（验证）

**命令**：`/gsd-verify-work N`

**目标**：用 Goal-Backward 验证确保交付质量

**核心活动**：
1. 读取 plan.xml 中的 `validation_criteria`
2. **从目标倒推**：测试用户能否完成承诺的功能
3. 测试**行为而非实现**

**Goal-Backward 验证 vs 传统验证**：

| 维度 | 传统验证 | Goal-Backward 验证 |
|------|---------|-------------------|
| **关注点** | 实现是否正确 | 用户能否达成目标 |
| **测试方式** | 单元测试为主 | E2E 测试为主 |
| **失败类型** | 函数返回错误值 | 用户场景不可达 |
| **测试视角** | 开发者视角 | 产品视角 |

**示例**：
- 传统验证："`AuthService.login` 函数返回 token"
- Goal-Backward："**用户能否登录后访问个人主页？**"

后者会发现：login 函数正确，但 token 没存到 cookie，用户**事实上**无法访问主页。

### 阶段 5：Ship（交付）

**命令**：`/gsd-ship N`

**目标**：生成 PR，归档里程碑

**核心活动**：
1. 自动生成 PR 描述（含本次变更摘要）
2. 关联相关 Issue（如果有）
3. 归档当前 Phase 到 milestones/
4. 更新 ROADMAP.md（标记完成状态）

---

## 四、GSD 对抗 Context Rot 的工程实践

### 实践 1：上下文隔离

```
传统做法（污染累积）：
会话开始 → Task1 → Task2 → Task3 → ... → 上下文塞满 → 质量下降

GSD 做法（隔离执行）：
主会话只调度 → 每个 Task 启动子代理 → 子代理执行完毕销毁 → 主会话保持轻盈
```

**实现方式**：
- 阶段间强制 `/clear` 清空上下文
- 子代理只加载任务相关的文件（通过 `task.context_files` 字段）
- 主会话窗口长期保持 30-40% 占用率

### 实践 2：跨会话记忆

```
传统做法：
新会话 → 用户："我们之前在做 X 项目，已经完成了 A、B" → AI 重新理解 → 浪费 20K token

GSD 做法：
新会话 → AI 自动读取 .planning/STATE.md → 立刻知道当前位置 → 0 token 浪费
```

**实现方式**：
- `STATE.md` 是项目的"当前状态快照"
- `CONTEXT.md` 是项目的"决策历史"
- `plan.xml` 是当前 Phase 的"执行图"
- 三者共同构成完整的项目记忆

### 实践 3：并行执行而非线性

```
传统做法：
"实现这 8 个任务" → AI 一个一个做 → 8N 时间 → 上下文累积污染

GSD 做法：
解析任务依赖图 → 分波次并行 → 4N 时间 → 每个任务独立上下文
```

**关键点**：GSD 不是简单地"快"——它是通过**并行化避免单会话过载**，速度只是副产品。

---

## 五、GSD 与三大核心原理的关系

GSD 是**唯一一个把三大原理推到极致**的框架：

### 决策外化为文件

GSD 的外化层次最丰富：
- **PROJECT.md**：项目级愿景
- **ROADMAP.md**：里程碑级路径
- **CONTEXT.md**：决策历史
- **STATE.md**：实时进度
- **plan.xml**：阶段级执行计划

**评价**：在 SDD 之外，GSD 增加了**状态文件**这个独特的外化层。

### 流程结构化为阶段

GSD 的五阶段是 SDD 框架中最完整的：
- **Discuss** 在 SDD 中很弱（Spec-Kit 的 specify 即是）
- **Plan** 与 SDD 类似
- **Execute** 在 SDD 中通常是黑盒，GSD 显式拆为波次
- **Verify** 是 GSD 独有的强项（Goal-Backward 验证）
- **Ship** 把 PR/归档纳入流程

**评价**：GSD 的阶段化**最完整、最严谨**。

### 任务原子化为单元

GSD 把这条做到了极致：
- 显式的 200K Token 单任务上限
- 子代理强制隔离
- 波次并行调度
- DAG 依赖管理

**评价**：GSD 的原子化**是行业最高标准**。

---

## 六、GSD 的精妙设计

### 设计 1：XML 而非 Markdown

为什么计划用 XML？
- **结构强制**：AI 必须按格式生成，不会随意发挥
- **可解析**：调度器可以自动遍历任务、识别依赖
- **可验证**：完整性可机器校验

这是一个**"为机器优化"的决策**——牺牲一点人类可读性，换取系统的稳定性。

### 设计 2：Goal-Backward 验证

普通验证容易陷入"实现细节"：
> "Login 函数返回了 token" ✅ → "但用户其实登录不成功"

Goal-Backward 强制从用户视角验证：
> "用户能不能完成登录并访问主页？" → 任何中间环节失败都会暴露

这本质上是把 BDD（行为驱动开发）的思想嵌入到 AI 工作流。

### 设计 3：状态机化的进度管理

`STATE.md` 不仅是"记录"，更是"调度依据"：
- 主代理读取 STATE.md 决定下一步做什么
- 子代理完成任务后**回写 STATE.md**
- 任何会话都可以从 STATE.md 恢复

**这让 GSD 具备了"断点续传"能力**——任何时候关掉会话，下次开新会话能精确接续。

### 设计 4：纪律即代码

GSD 把"应该这样做"的纪律编码为命令：
- `/gsd-clear-and-load` 强制清空上下文并重新加载
- `/gsd-verify-context` 检查上下文是否被污染
- `/gsd-checkpoint` 在关键节点保存状态

**人类纪律不可靠，但命令是可靠的**——GSD 通过命令化让纪律自动生效。

### 设计 5：自动化的 Git 集成

每个原子任务一个 commit，commit message 自动生成（基于 task description），让 git history 成为天然的开发日志。

```
git log
  feat(M1.P1.T8): implement logout API
  feat(M1.P1.T7): add JWT validation middleware
  feat(M1.P1.T6): implement login API
  feat(M1.P1.T5): implement register API
  feat(M1.P1.T4): add JWT signing utility
  feat(M1.P1.T3): create auth.service.ts skeleton
  feat(M1.P1.T2): implement password hashing
  feat(M1.P1.T1): create users table migration
```

**git history 本身就是项目执行历史**，不需要额外的进度报告。

---

## 七、GSD 的局限和适用边界

### 局限 1：启动成本最高

GSD 需要把项目拆解到 Milestone → Phase → Task 三层，初次设置需要大量思考。
**对策**：从一个小 Milestone 开始试点，不要一上来就规划整个项目。

### 局限 2：纪律要求最严

GSD 的所有收益建立在"严格执行流程"之上。如果团队跳过 Discuss、跳过 Verify，效果会显著退化。
**对策**：把 GSD 命令固化为 CI/CD 流程的一部分，强制执行。

### 局限 3：对 AI 工具的能力依赖最强

GSD 严重依赖**子代理（Sub-Agent）能力**——必须能在独立上下文中执行任务。
**对策**：选用支持子代理的 AI 工具（Claude Code 是首选）。

### 局限 4：不擅长快速 prototype

如果只想"快速搞个原型"，GSD 的流程会显得过重。
**对策**：原型阶段用 Spec-Kit 或纯 vibe coding，进入正式开发后切换到 GSD。

### 局限 5：缺乏规范管理

GSD 关注"如何执行"，不关注"规范如何写"。如果项目需要长期维护规范契约，需要配合 SDD 框架。
**对策**：GSD + OpenSpec 组合（详见多框架组合文档）。

---

## 八、GSD 适用场景判断表

| 场景特征 | 推荐度 |
|---------|--------|
| 长周期项目（数月到一年） | ★★★★★ |
| 复杂功能开发（涉及多模块） | ★★★★★ |
| 需要严格质量保证 | ★★★★★ |
| 多人协作（需要状态共享） | ★★★★ |
| 快速 prototype | ★★ |
| 单文件简单修改 | ★ |
| 团队接受流程化 | ★★★★★ |
| 团队偏好"自由发挥" | ★★ |

---

## 九、与其他框架的对比

| 维度 | GSD | OpenSpec | Spec-Kit | ECC |
|------|-----|----------|----------|-----|
| **核心关注** | 上下文管理 | 规范管理 | 规范管理 | 能力管理 |
| **解决主问题** | Context Rot | 棕地变更 | 绿地启动 | 能力扩展 |
| **状态管理** | 极强（STATE.md） | 中（changes/） | 弱 | 弱 |
| **并行能力** | 极强（Wave） | 弱 | 弱 | 中 |
| **验证机制** | 极强（Goal-Backward） | 中 | 中 | 中 |
| **学习曲线** | 陡 | 中 | 较陡 | 陡 |
| **适用项目大小** | 中-大 | 中-大 | 中-大 | 任意 |

---

## 十、GSD 快速上手

### 安装

```bash
npx get-shit-done-cc@latest
```

### 初始化项目

```bash
/gsd-new-project
```

AI 会引导你：
1. 描述项目愿景 → 生成 PROJECT.md
2. 拆解里程碑 → 生成 ROADMAP.md
3. 进入第一个里程碑

### 自动化模式

```bash
/gsd-auto
```

让 AI 自动推进所有阶段，你只在关键节点确认（Discuss 完成、Verify 通过、Ship 发布）。

### 半自动模式（推荐）

逐阶段手动执行：

```bash
/gsd-discuss-phase 1
# 与 AI 对话澄清需求...

/gsd-plan-phase 1
# AI 生成 plan.xml，你审阅...

/gsd-execute-phase 1
# AI 启动子代理并行执行...

/gsd-verify-work 1
# AI Goal-Backward 验证...

/gsd-ship 1
# 生成 PR、归档...
```

---

## 十一、关键启发

通过 GSD 这个代表，能提炼出**上下文工程类框架的共性设计原则**：

1. **上下文是稀缺资源**——必须像内存一样精打细算
2. **流水线优于工匠**——稳定可预测的流程胜过依赖灵感的工匠
3. **状态外化是断点续传的关键**——文件系统是项目的"硬盘"
4. **并行化是质量保障**——而不仅仅是为了速度
5. **目标驱动验证**——从用户价值倒推，而非从实现细节出发
6. **纪律必须可执行**——把规范编码为命令，让流程自动生效

这些原则不只属于 GSD——它们是**任何上下文工程框架的共同基础**。

---

## 十二、扩展阅读

- [GSD 官方文档](https://github.com/gsd-build/get-shit-done)
- [GSD 框架实战：解决 AI 编程的 context rot 问题](https://blog.zhijun.io/posts/gsd-get-shit-done-project-framework)
- [掘金 - GSD 使用指南](https://juejin.cn/post/7635965174852911155)
- [CSDN - 深度解析 GSD 上下文工程](https://blog.csdn.net/weixin_45888077/article/details/159474157)
- [Zread - GSD 官方文档概述](https://zread.ai/gsd-build/get-shit-done)
- [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172)
- [Anthropic - Subagents in Claude Code](https://docs.claude.com/en/docs/claude-code/sub-agents)