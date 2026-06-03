# OpenSpec 深度剖析：规范驱动框架的代表作

## 引子：为什么选 OpenSpec 作为 SDD 类的代表？

规范驱动开发（SDD）类 Harness 框架有多个：Spec-Kit、Kiro、OpenSpec。选择 OpenSpec 作为代表的原因：

1. **设计最纯粹**：OpenSpec 把"规范驱动"的核心理念——**Delta Spec（增量规范）**——做到了极致
2. **棕地友好**：现实世界绝大多数项目是棕地（已有代码），OpenSpec 比 Spec-Kit 更贴合实际
3. **工具无关**：不绑定特定 IDE 或 AI 模型，方法论价值最高
4. **设计可迁移**：理解 OpenSpec 的设计哲学后，反推 Spec-Kit、Kiro 等都很容易

---

## 一、OpenSpec 的设计哲学

### 核心定位

> **OpenSpec 是面向 AI 编程时代的"变更管理"基础设施。**

它不是一个 IDE 插件，不是一个 AI 工具，而是**一种规范代码变更的工程化方法**——只是这个方法天然适配 AI 协作。

### 设计哲学

#### 哲学 1：规范是契约，不是文档

传统观念：规范是写给人看的文档（可有可无）
OpenSpec 观念：**规范是 AI 和人类共同遵守的契约**（强制性的）

具体体现：
- 主规范（`specs/`）描述当前系统的真实行为
- 任何变更必须先修改规范，再改代码
- AI 在 Apply 阶段必须严格按规范执行，不能"自由发挥"

这就像 TypeScript 的类型系统——**规范不是事后注释，而是事前约束**。

#### 哲学 2：变更是一等公民

传统观念：变更是 commit message 里的一句话
OpenSpec 观念：**变更是独立、可审计、可回滚的实体**

具体体现：
- 每个变更有完整产物（proposal + specs + design + tasks）
- 每个变更在 `changes/` 目录中独立存储
- 归档后保留完整审计历史

这就像 Git 把"提交"变成一等公民改变了软件协作；OpenSpec 把"变更"变成一等公民改变了规范协作。

#### 哲学 3：增量优于全量

传统观念：每次都重新描述完整系统
OpenSpec 观念：**只描述这次变了什么**

具体体现：
- Delta Spec 只用 `## ADDED` / `## MODIFIED` / `## REMOVED` 三种段落
- 主规范由历次 delta 累积合成
- 系统的"完整画像"是涌现的，不是预先定义的

这就像数据库的 WAL（Write-Ahead Log）——通过增量日志能恢复任意时间点的状态。

---

## 二、OpenSpec 的架构剖析

### 顶层目录结构

```
openspec/
  specs/                     ← 主规范目录（系统当前真实行为）
    auth/
      user-login.md
      session-management.md
    payments/
      checkout.md
  changes/                   ← 变更目录（每个变更独立空间）
    add-dark-mode/
      proposal.md            ← 变更提案
      specs/                 ← Delta Specs
        ui/dark-mode.md
      design.md              ← 技术设计
      tasks.md               ← 任务清单
    refactor-payment-strategy/
      ...
  config.yaml                ← 项目配置（技术栈、规范）
  README.md                  ← OpenSpec 自身的元数据
```

### 关键概念解析

#### 概念 1：主规范（Main Specs）

**职责**：描述系统**当前**的真实行为，作为权威真相源

**特点**：
- 按领域组织（`auth/`、`payments/`、`api/`）
- 内容是"完整描述"，不含变更标记
- 由多次变更归档累积形成
- AI 在做新变更时，先读主规范了解现状

**示例（`specs/auth/user-login.md`）**：
```markdown
# 用户登录规范

## 功能描述
用户通过邮箱+密码登录系统。

## 行为契约
- 登录成功：返回 JWT token，有效期 24 小时
- 密码错误：返回 401，第 5 次失败锁定账号 30 分钟
- 邮箱不存在：返回 401（不暴露具体原因）

## 数据模型
- users 表：id, email, password_hash, locked_until
- sessions 表：user_id, token, expires_at
```

#### 概念 2：变更（Changes）

**职责**：描述一次具体的变更意图、设计和任务

**结构**：每个变更目录包含 4 个核心文件：

| 文件 | 职责 |
|------|------|
| `proposal.md` | 为什么要这次变更（动机、影响、风险） |
| `specs/*.md` | 这次变更的 Delta Spec（增量规范） |
| `design.md` | 技术设计（架构、数据流、容错） |
| `tasks.md` | 原子化的任务清单 |

#### 概念 3：Delta Spec（增量规范）

**这是 OpenSpec 最核心的创新**。

**三种操作**：

```markdown
# changes/add-dark-mode/specs/ui/theme.md

## ADDED Requirements
### 暗色模式开关
- 用户在设置页面可切换暗色模式
- 偏好保存在 localStorage
- 系统自动跟随 OS 主题（可选）

## MODIFIED Requirements
### 颜色变量定义 [MODIFIED]
- 旧：硬编码 #FFFFFF, #000000
- 新：使用 CSS 变量 --bg-primary, --text-primary

## REMOVED Requirements
### 强制白色主题
（移除，由暗色模式开关替代）
```

**Delta Spec 的合并规则**（归档时）：
- `ADDED` → 追加到主规范对应章节
- `MODIFIED` → 替换主规范的对应内容
- `REMOVED` → 从主规范中删除

#### 概念 4：配置（config.yaml）

**职责**：告诉 AI 项目的全局上下文

**示例**：
```yaml
project:
  name: "电商平台"
  type: "monorepo"

stack:
  backend: "Node.js + TypeScript + Express"
  frontend: "React + TypeScript + Vite"
  database: "PostgreSQL + Prisma"

conventions:
  naming: "camelCase for JS/TS, snake_case for SQL"
  testing: "Jest for unit, Playwright for E2E"
  api: "RESTful, OpenAPI documented"

ai_instructions:
  - 严禁使用 any 类型
  - 所有 API 必须有错误处理
  - 提交前必须通过 lint
```

这相当于 Spec-Kit 的 `constitution.md`，但更轻量——不是"宪法"，而是"项目元数据"。

---

## 三、OpenSpec 的工作流深度解析

### 核心命令体系

OpenSpec 有两套 Profile：

#### Profile 1：Quick（快速模式，默认）

适合**简单变更**，3 个核心命令：

```bash
/opsx:propose <change-name>   # 创建变更提案
/opsx:apply                   # 执行任务实现
/opsx:archive                 # 归档变更
```

#### Profile 2：Expanded（扩展模式）

适合**复杂变更**，提供更细的步骤：

```bash
/opsx:explore         # 探索现有代码，生成初始规范
/opsx:new <name>      # 脚手架创建变更目录
/opsx:continue        # 逐步生成 proposal → specs → design → tasks
/opsx:verify          # 校验实现与规范一致性
/opsx:apply           # 执行实现
/opsx:archive         # 归档
```

切换：`openspec config profile`

---

### 完整工作流详解（Quick 模式）

#### Step 1: `/opsx:propose` 内部发生了什么

当你执行：
```
/opsx:propose add-dark-mode 用户可以切换暗色模式
```

OpenSpec 内部执行的步骤：

```
1. 读取 openspec/config.yaml （获取项目上下文）
2. 读取 openspec/specs/ （了解系统现状）
3. 分析现有代码库（识别相关文件、模式、约定）
4. 生成 changes/add-dark-mode/ 目录
5. 创建 proposal.md（变更动机和影响范围）
6. 创建 specs/ 下的 Delta 规范
7. 创建 design.md（技术设计）
8. 创建 tasks.md（任务清单）
```

**关键点**：AI 不是凭空想象，而是**先理解现状**再设计变更。这就是 OpenSpec 的"棕地友好"基因。

#### Step 2: 人类审阅与迭代

这是**最关键但常被忽略**的环节。

人类需要审查：
- `proposal.md` — 变更动机是否合理？影响范围是否准确？
- `specs/*.md` — Delta 是否完整？是否遗漏边界情况？
- `design.md` — 技术方案是否可行？性能/安全考虑是否充分？
- `tasks.md` — 任务粒度是否合适？依赖关系是否清晰？

任何一项不满意，可以：
- 直接编辑文件（人 + AI 都可读写）
- 让 AI 补充：`/opsx:continue 补充 design.md 中的缓存策略`

#### Step 3: `/opsx:apply` 内部发生了什么

```
1. 读取 changes/add-dark-mode/ 全部产物
2. 按 tasks.md 顺序逐个执行任务
3. 每个任务完成后：
   - 在 tasks.md 中标记 [x]
   - 提交一个 atomic git commit
4. 全部完成后，标记变更可归档
```

**关键点**：
- 任务原子化执行（每个任务一个 commit）
- 严格按 spec 实现，不"自由发挥"
- 实现过程中如发现规范有问题，**先修规范再改代码**

#### Step 4: `/opsx:archive` 内部发生了什么

```
1. 验证所有任务都已完成
2. 运行测试（确保实现符合规范）
3. 合并 Delta Specs 到主规范：
   - ADDED 部分 → 追加到 specs/
   - MODIFIED 部分 → 替换 specs/ 中的对应内容
   - REMOVED 部分 → 从 specs/ 中删除
4. 把 changes/add-dark-mode/ 移动到 changes/archived/
5. 生成归档摘要（可选）
```

**关键点**：归档不是简单的目录移动，而是**真相源更新**。归档后，系统的"当前真实行为"就包含了暗色模式。

---

## 四、OpenSpec 的精妙设计

### 设计 1：变更隔离

每个变更在自己的目录中独立存在，**互不干扰**。

**好处**：
- 多个变更可以并行开发（多个分支同时进行）
- 任何变更可以独立放弃（删除目录即可）
- 变更间的冲突在合并时显式暴露

### 设计 2：归档保留历史

归档后的变更不是删除，而是移到 `changes/archived/`。

**好处**：
- 完整的审计历史（谁、什么时候、改了什么、为什么）
- 可以回溯任何决策的原始动机
- 替代了一部分 ADR（架构决策记录）的作用

### 设计 3:Delta 比 Diff 更高层

Git diff 是**字符级**的差异，难以理解语义。
OpenSpec Delta 是**需求级**的差异，直接表达意图。

```diff
-超时时间 60 分钟
+超时时间 30 分钟
```
vs
```markdown
## MODIFIED Requirements
### 会话超时 [MODIFIED]
- 旧：60 分钟无活动后过期
- 新：30 分钟无活动后过期
- 原因：响应安全合规要求
```

后者**人和 AI 都能理解**。

### 设计 4：先理解后设计

`/opsx:propose` 强制 AI 先读取现有 specs 和代码，再生成变更。

**这避免了 AI 的两个常见错误**：
1. 凭空设计（不考虑现有架构）
2. 重复造轮子（已存在的功能再实现一遍）

### 设计 5：渐进式接入

不需要一次性把整个系统的规范都写出来。

**典型接入路径**：
```
第 1 个月：只为新功能写规范
第 2 个月：修改 Bug 时顺便补上相关模块的规范
第 6 个月：核心模块的规范已基本完整
```

主规范是**涌现的**，不是预先定义的。这是 OpenSpec 比 Spec-Kit 更友好的根本原因。

---

## 五、OpenSpec 在三大原理上的体现

### 决策外化为文件

| 决策类型 | 外化文件 |
|---------|---------|
| 全局约定 | `config.yaml` |
| 系统现状 | `specs/*` |
| 变更动机 | `changes/*/proposal.md` |
| 技术决策 | `changes/*/design.md` |
| 执行进度 | `changes/*/tasks.md` |
| 历史记录 | `changes/archived/*` |

**评价**：决策外化做得**全面且分层**。每个决策都有明确的归属文件。

### 流程结构化为阶段

```
Quick:     Propose → Apply → Archive
Expanded:  Explore → New → Continue → Verify → Apply → Archive
```

**评价**：阶段化做得**清晰但灵活**。Quick 模式的三阶段是 SDD 类框架中最少的，但通过 Expanded 模式提供了细粒度选项。

### 任务原子化为单元

OpenSpec 在 `tasks.md` 中拆分原子任务，但**不强制**子代理隔离执行。

**评价**：任务原子化做得**到位但执行隔离不强**。这是 OpenSpec 的弱点——它假设你用 Claude Code 等工具的子代理能力配合。

---

## 六、OpenSpec 的局限和适用边界

### 局限 1：规范维护成本

虽然 Delta 模式降低了启动成本，但**长期维护规范仍需投入**。
如果团队不严格执行"先改规范再改代码"，主规范会逐渐与代码脱节。

**对策**：把"更新规范"作为 PR 合并的硬性要求。

### 局限 2：对小变更过重

修复一个简单 typo，也要走 propose → apply → archive？
**对策**：定义"豁免范围"——纯 bug 修复、文档调整等可不走流程。

### 局限 3：跨服务协调能力有限

虽然天然支持跨服务变更（一个 change 可含多服务的 specs），但**没有内置的服务依赖编排**。
**对策**：在 design.md 中明确部署顺序，或与 OMC、GSD 等编排框架组合使用。

### 局限 4：依赖团队的工程文化

OpenSpec 的价值依赖于团队**愿意写规范、读规范、维护规范**。
如果团队仍然"只看代码不看 spec"，OpenSpec 会沦为形式。
**对策**：把规范评审纳入代码评审流程。

---

## 七、OpenSpec 适用场景判断表

| 场景特征 | 推荐度 |
|---------|--------|
| 已有项目（棕地） | ★★★★★ |
| 新项目（绿地） | ★★★ |
| 频繁迭代（每周多次变更） | ★★★★★ |
| 重构频繁（架构经常调整） | ★★★★★ |
| 多人协作 | ★★★★ |
| 单人开发 | ★★★ |
| 微服务架构 | ★★★★ |
| 单体应用 | ★★★★ |
| 需要变更审计 | ★★★★★ |
| 强调快速 prototype | ★★ |

---

## 八、与 Spec-Kit 的最终对比（专题深度版）

| 维度 | OpenSpec | Spec-Kit |
|------|----------|----------|
| **核心数据模型** | Delta Spec（增量） | Full Spec（全量） |
| **类比** | Git（快照+增量） | Word 文档（直接编辑） |
| **启动成本** | 低（直接 propose） | 高（先写宪法） |
| **棕地友好度** | ★★★★★ | ★★ |
| **绿地清晰度** | ★★★ | ★★★★★ |
| **变更审计** | ★★★★★（changes 目录天然存档） | ★★★（依赖 git 历史） |
| **学习曲线** | 中等（理解 Delta 概念） | 较陡（5 阶段+宪法） |
| **企业治理** | 中等 | 强 |
| **工具绑定** | 工具无关 | 倾向 Copilot |
| **适合"我修一个 bug"** | 重 | 重 |
| **适合"我加个大功能"** | ★★★★★ | ★★★★ |
| **适合"重构一个模块"** | ★★★★★ | ★★ |

---

## 九、关键启发

`[高置信][核心]`

通过 OpenSpec 这个代表，我们能提炼出**规范驱动框架的共性设计原则**：

1. **规范是契约不是文档**——它有强制性，是 AI 行为的约束
2. **先理解再设计**——AI 必须先读懂现状才能设计变更
3. **变更是独立实体**——可审查、可回滚、可审计
4. **真相源永远在文件**——不在对话里，不在记忆里
5. **人机共同维护**——文件格式必须人和 AI 都能读写

这些原则不只属于 OpenSpec——它们是**任何 SDD 框架的共同基础**。理解了 OpenSpec，反推 Spec-Kit、Kiro 都很容易。

---

## 十、扩展阅读

- [GitHub - OpenSpec](https://github.com/Fission-AI/OpenSpec)
- [掘金 - OpenSpec 从入门到精通](https://article.juejin.cn/post/7637800059725250606)
- [OpenSpec 实战指南：从工作流到落地](https://magicliang.github.io/2026/03/23/OpenSpec-实战指南-从工作流到落地/)
- [基于 OpenSpec 实现规范驱动开发](https://article.juejin.cn/post/7631008687277817902)
- [OpenSpec v1.1 使用心得](https://blog.csdn.net/Cjh20pan/article/details/158037808)
- [Vibe Coding - Fission-AI OpenSpec 面向 AI 编程助手的规范驱动开发](https://artisan.blog.csdn.net/article/details/153654303)