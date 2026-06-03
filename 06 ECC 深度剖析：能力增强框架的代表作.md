# ECC 深度剖析：能力增强框架的代表作

## 引子：为什么选 ECC 作为能力增强类的代表？

能力增强类（Augmentation）Harness 框架包括：ECC、OMC、Trellis 等。选择 ECC 的原因：

1. **设计最完整**：ECC 是目前规模最大的 Claude Code 增强系统（48 个 Agents + 182 个 Skills + 14 套 Rules）
2. **三层架构清晰**：Skills/Agents/Commands 的分层是能力增强类框架的经典范式
3. **独有创新**：红蓝队对抗审计（AgentShield）是其他框架没有的
4. **跨工具兼容**：不仅服务 Claude Code，还能延伸到 Cursor、Codex 等
5. **可学习性强**：ECC 的设计可以拆解、拷贝、改造为团队自己的增强包

---

## 一、ECC 的设计哲学

### 核心定位

> **ECC 不是工作流，而是 Claude Code 的"能力扩展包"——把通用 AI 改造成专业开发引擎。**

它**不主张**特定的开发流程（不像 Spec-Kit 强制阶段化、不像 OpenSpec 强制 Delta），而是提供一个**能力工具箱**，让你在任何流程中都能调用专业能力。

### 设计哲学

#### 哲学 1：能力是可组合的

通用 AI = 一个"多功能瑞士军刀"
ECC 的做法 = 把 AI 拆成"专业工具集"——每个工具专注一件事，可按需组合

具体体现：
- 写代码用 `coder` Agent
- 审核安全用 `security-reviewer` Agent
- 修构建错误用 `build-error-resolver` Agent
- 这些 Agent 组合起来形成完整开发流程

#### 哲学 2：上下文必须纯净

通用 AI 一个会话承担所有任务，导致上下文混杂污染
ECC 的做法 = 让每个 Agent 在自己的领域里**单一职责**，上下文极度聚焦

具体体现：
- `security-reviewer` 只看安全相关的内容，不被业务逻辑分心
- `tdd-tester` 只关注测试，不被 UI 设计分心
- 主代理只负责调度，不持有具体实现细节

#### 哲学 3：经验必须可沉淀

通用 AI 每次会话都是"从头开始"
ECC 的做法 = 通过 Skills（技能库）和 Hooks（钩子）把经验固化为可复用资产

具体体现：
- 一次踩坑的经验 → 写成 Skill 文档 → 下次自动加载
- 一次代码审查发现的模式 → 写成 Rule → 自动应用到所有项目
- 通过 `/instinct-export` 命令把会话经验导出为新 Skill

---

## 二、ECC 的三层架构

### 顶层架构图

```
┌──────────────────────────────────────────────────────────┐
│  Commands（命令层）                                        │
│  /security-scan  /multi-plan  /tdd-feature  ...           │
│  ↓ 调用                                                    │
├──────────────────────────────────────────────────────────┤
│  Agents（代理层）                                          │
│  planner  architect  security-reviewer  tdd-tester  ...   │
│  ↓ 使用                                                    │
├──────────────────────────────────────────────────────────┤
│  Skills（技能层）                                          │
│  tdd-workflow  security-review  api-design  ...           │
│  ↓ 配合                                                    │
├──────────────────────────────────────────────────────────┤
│  Rules（规则层）                                           │
│  TypeScript-rules  Python-rules  general-rules  ...       │
└──────────────────────────────────────────────────────────┘
                  ↑ 通过 Hooks 触发
┌──────────────────────────────────────────────────────────┐
│  Hooks（钩子）+ MCP Configs（外部集成）                    │
│  PostEdit、PreCommit、SessionStart  +  GitHub、Database   │
└──────────────────────────────────────────────────────────┘
```

### 项目目录结构

```
everything-claude-code/
├── agents/          # 专业子代理（48 个）
│   ├── planner.md
│   ├── architect.md
│   ├── security-reviewer.md
│   ├── tdd-tester.md
│   └── build-error-resolver.md
├── skills/          # 可复用技能（182 个）
│   ├── tdd-workflow.md
│   ├── security-review.md
│   ├── api-design.md
│   └── refactoring-patterns.md
├── commands/        # 斜杠命令
│   ├── security-scan.md
│   ├── multi-plan.md
│   └── tdd-feature.md
├── hooks/           # 自动化钩子
│   ├── post-edit-security-scan.sh
│   ├── pre-commit-lint.sh
│   └── session-start-load-context.sh
├── rules/           # 编码规范
│   ├── languages/
│   │   ├── typescript.md
│   │   └── python.md
│   └── general/
│       └── solid-principles.md
├── mcp-configs/     # 外部服务集成
│   ├── github.json
│   └── postgres.json
└── contexts/        # 动态提示注入
    └── project-context-template.md
```

---

## 三、四个核心概念深度解析

### 概念 1：技能库

**定义**：可复用的工作流定义或领域知识，类似业务开发中的"公共函数包"

**特点**：
- 描述**怎么做某件事**（流程、步骤、最佳实践）
- 不是 Agent，自己不主动执行，被 Agent 调用
- 类似软件工程中的"代码库"

**示例**：`skills/tdd-workflow.md`

```markdown
# TDD 工作流技能

## 适用场景
任何新功能开发或 bug 修复

## 核心步骤
1. **Red**：先写失败的测试
   - 命名格式：describe('FunctionName', () => { it('should ...') })
   - 必须断言具体行为，不要测试实现细节
2. **Green**：写最少代码让测试通过
   - 不追求优雅，只追求绿色
3. **Refactor**：重构代码保持测试绿色
   - 命名优化、提取函数、消除重复

## 反模式
- 先写代码再补测试
- 测试覆盖率追求 100%（应该追求关键路径覆盖）
- 测试实现细节而非行为
```

### 概念 2：Agents（子代理）

**定义**：独立的专家角色，每个 Agent 专注单一任务领域

**特点**：
- 描述**它是谁、能做什么**（身份、职责、能力）
- 拥有独立的上下文（不污染主代理）
- 可调用 Skills 完成具体任务

**ECC 的 Agent 分类**：

| 分类 | 代表 Agent | 职责 |
|------|-----------|------|
| **规划类** | planner、architect | 需求分析、架构设计 |
| **执行类** | coder、tdd-tester | 写代码、写测试 |
| **审查类** | security-reviewer、code-reviewer | 安全/质量审查 |
| **修复类** | build-error-resolver、bug-hunter | 错误修复、问题诊断 |
| **运维类** | deploy-helper、log-analyzer | 部署、日志分析 |

**示例**：`agents/security-reviewer.md`

```markdown
# Security Reviewer Agent

## 角色定义
你是一名资深的应用安全工程师，拥有 OWASP Top 10 等安全标准的深入知识。

## 核心职责
- 审查代码中的安全漏洞
- 识别敏感信息泄露
- 检查认证授权逻辑
- 评估第三方依赖的安全风险

## 调用的 Skills
- skills/security-review.md（标准审查流程）
- skills/owasp-checklist.md（OWASP 检查清单）
- rules/security-rules.md（项目安全规范）

## 输出格式
- 严重程度（Critical / High / Medium / Low）
- 漏洞类型（SQL注入 / XSS / CSRF / ...）
- 位置（文件:行号）
- 修复建议（具体代码示例）

## 不做什么
- 不修改代码（只审查不修复）
- 不评估业务逻辑（除非涉及安全）
```

### 概念 3：Commands（命令）

**定义**：用户主动触发的快捷指令，对应一个或多个 Agent 的协作

**特点**：
- 是用户的**入口点**
- 通常调度多个 Agents 协作
- 类似 Unix 命令，简洁高效

**示例**：

```bash
/security-scan          # 启动 security-reviewer Agent，扫描全项目
/tdd-feature 用户登录   # 启动 planner + tdd-tester + coder 协作开发
/multi-plan             # 多 Agent 并行规划，对比方案
/build-fix              # 启动 build-error-resolver 修复编译错误
```

### 概念 4：Hooks（钩子）

**定义**：事件驱动的自动化触发器

**特点**：
- 在特定事件发生时**自动**执行
- 不需要用户手动调用
- 是 ECC 的"幕后英雄"

**ECC 中的常见 Hook**：

| Hook 触发时机 | 自动执行 |
|--------------|---------|
| `SessionStart` | 加载项目上下文、规则、技能清单 |
| `PostEdit`（编辑文件后） | 运行安全扫描、Lint 检查 |
| `PreCommit`（提交前） | 运行测试、检查 lint、扫描敏感信息 |
| `PostBuild`（构建后） | 运行 E2E 测试、检查 bundle 大小 |
| `OnError`（出错时） | 自动调用 build-error-resolver |

**示例**：`hooks/post-edit-security-scan.sh`

```bash
#!/bin/bash
# 文件编辑后自动安全扫描
if [[ $EDITED_FILE =~ \.(ts|js|py)$ ]]; then
  claude-agent invoke security-reviewer \
    --target $EDITED_FILE \
    --mode quick
fi
```

**Hook 的价值**：把"应该做但容易忘"的事变成**自动化**。例如安全扫描，没人愿意每次手动跑，但 Hook 让它在每次编辑后自动跑。

---

## 四、ECC 的杀手级特性：AgentShield

### 什么是 AgentShield？

ECC 独有的**红蓝队对抗审计**子系统。

### 三方角色

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Red Team       │ vs  │  Blue Team      │  →  │  Auditor        │
│  红队：攻击方    │     │  蓝队：防御方    │     │  审计员：仲裁    │
│                 │     │                 │     │                 │
│  模拟黑客找漏洞  │     │  评估现有防护    │     │  综合双方报告    │
│  尝试攻击链      │     │  识别盲点        │     │  生成优先级清单  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

### 工作流程

```bash
npx ecc-agentshield scan --opus --stream
```

1. **Red Team Agent** 启动：
   - 角色：模拟有恶意的攻击者
   - 任务：在代码中找漏洞、尝试攻击链
   - 重点：CLAUDE.md 注入、Hook 脚本利用、密钥泄露、MCP 配置弱点

2. **Blue Team Agent** 启动：
   - 角色：项目的安全防御方
   - 任务：评估现有的防护措施
   - 重点：认证逻辑、权限控制、日志记录、监控告警

3. **Auditor Agent** 综合：
   - 角色：中立的审计员
   - 任务：综合红蓝队报告，去重、排序、生成可操作清单
   - 输出：优先级排序的风险报告

### 为什么红蓝对抗有效？

**单一视角的局限**：
- 只让 AI "审查代码安全" → AI 倾向于挑常见问题，深层漏洞可能漏掉
- 只让 AI "模拟攻击" → 可能产生不切实际的威胁场景

**红蓝对抗的价值**：
- 红队**主动找漏洞**——会想办法绕过防御
- 蓝队**评估防御有效性**——会指出防御措施的局限
- 审计员**综合视角**——避免任一方的偏见

这本质上是把**对抗性测试（Adversarial Testing）**的思想引入到 AI 代码审查。

### AgentShield 扫描范围（102 条规则）

- **CLAUDE.md 注入**：恶意 prompt 隐藏在配置中
- **Hook 脚本漏洞**：钩子脚本被注入恶意代码
- **MCP 配置弱点**：外部集成的认证薄弱
- **敏感信息泄露**：API key、密码、token 硬编码
- **依赖供应链**：第三方包的已知漏洞
- **OWASP Top 10**：SQL 注入、XSS、CSRF 等

---

## 五、ECC 工作流实战

### 场景：实现一个新功能（用户头像上传）

#### Step 1: 用户输入需求

```
用户：实现用户头像上传功能，要求支持 JPG/PNG，限制 5MB，存到 S3
```

#### Step 2: 主代理识别意图，调度 Agents

主代理（Orchestrator）的内部决策：
```
1. 这是一个新功能开发 → 启动 planner 规划
2. 涉及文件上传 → 启动 security-reviewer 评估安全
3. 需要测试 → 启动 tdd-tester
4. 需要写代码 → 启动 coder
```

#### Step 3: planner Agent 规划

```markdown
# 规划产出

## 任务拆解
1. 数据模型：users 表添加 avatar_url 字段
2. API 端点：POST /api/users/avatar
3. 文件验证：类型、大小检查
4. S3 集成：上传到 avatars/<user_id>.jpg
5. 前端组件：AvatarUploader

## 依赖
- 已有的 auth middleware
- 已有的 S3 客户端
- 新增：multer（文件上传中间件）
```

#### Step 4: security-reviewer Agent 安全评估

```markdown
# 安全检查清单

## 必须实现
- [ ] 文件类型白名单（不仅靠扩展名，还要查 magic bytes）
- [ ] 文件大小限制（前端 + 后端双重校验）
- [ ] 用户只能改自己的头像（认证检查）
- [ ] S3 路径不可猜测（用 UUID 而非 user_id）
- [ ] 限流（每用户每天最多上传 10 次）

## 红队视角发现的潜在威胁
- SVG 文件可包含 XSS（建议拒绝 SVG）
- ImageMagick 漏洞（建议用 sharp 库处理）
- ZIP Bomb（虽然限制了大小，但仍要检查解压后大小）
```

#### Step 5: tdd-tester Agent 写测试

```typescript
// 先写测试（Red 阶段）
describe('AvatarUploader', () => {
  it('should accept valid JPG under 5MB', async () => { ... });
  it('should reject files over 5MB', async () => { ... });
  it('should reject non-image types', async () => { ... });
  it('should reject malicious SVG', async () => { ... });
  it('should require authentication', async () => { ... });
});
```

#### Step 6: coder Agent 实现代码

按 plan + 测试 + 安全清单实现，让所有测试通过。

#### Step 7: Hooks 自动触发

- `PostEdit` Hook → 自动运行安全扫描
- `PreCommit` Hook → 运行 lint + 测试 + 敏感信息扫描

#### Step 8: code-reviewer Agent 最终审查

在提交前做最后一道审查，确保代码质量。

---

## 六、ECC 的精妙设计

### 设计 1：上下文纯净度极高

每个 Agent 只看到自己需要的内容：
- `planner` 只看需求和现有架构
- `coder` 只看 plan 和当前要实现的任务
- `security-reviewer` 只看代码和安全规则

主代理（Orchestrator）只持有"调度信息"，不持有具体实现细节。
**结果**：主代理上下文长期保持在 30-40% 占用率。

### 设计 2：能力可独立演进

某个 Agent 表现不好？只改那一个 .md 文件即可。
某个 Skill 过时了？替换那个 Skill 文件即可。
新需求？写一个新 Agent 加入。

**这种"插件式"架构让 ECC 可以渐进演进**，不需要重写整个系统。

### 设计 3：Hooks 实现"零纪律"

人类的纪律是不可靠的：
- "提交前要跑测试"——总有人忘记
- "敏感文件不能提交"——总有人疏忽

Hooks 让规则**自动执行**：
- `PreCommit` 自动跑测试，失败则不能提交
- `PreCommit` 自动扫描敏感信息，发现则阻止
- 不依赖纪律，依赖工程

### 设计 4：经验沉淀机制

```bash
/instinct-export    # 把当前会话的经验导出为新 Skill
/evolve            # 让 ECC 自我演进，根据使用模式优化
```

**ECC 是会"学习"的**——你用得越久，它越懂你的项目。

### 设计 5：跨工具兼容

虽然名字叫 Everything **Claude Code**，但实际上：
- Skills 是 Markdown，任何工具都能读
- Agents 定义可移植到 Cursor、Codex
- Hooks 用 shell 脚本，与工具解耦

**这让 ECC 的投资具有长期价值**——即使换工具，能力库依然有用。

---

## 七、ECC 在三大原理上的体现

### 决策外化为文件

ECC 把**决策、能力、经验**全部外化为文件：
- `agents/*.md`：每个角色的能力定义
- `skills/*.md`：每个流程的最佳实践
- `rules/*.md`：编码规范
- `contexts/*`：项目上下文模板

**评价**：决策外化做得**最彻底**——连"AI 应该是谁"都外化了。

### 流程结构化为阶段

ECC 不强制特定流程，而是提供**可组合的阶段化能力**：
- 想用 SDD 流程？调用 planner → architect → coder → reviewer
- 想用 TDD 流程？调用 planner → tdd-tester → coder
- 想用 quick fix？直接调用 build-error-resolver

**评价**：阶段化做得**灵活但需要用户决定**。这是优点（自由）也是缺点（缺乏强约束）。

### 任务原子化为单元

每个 Agent 在独立上下文中执行，天然支持原子化。
通过 Hooks 实现自动化的小任务（如安全扫描）。

**评价**：任务原子化做得**到位**，是 ECC 的强项。

---

## 八、ECC 的局限和适用边界

### 局限 1：学习曲线陡

48 个 Agent + 182 个 Skill，新用户面对这么多选项会迷茫。
**对策**：从 5-10 个核心 Agent 开始用，逐步扩展。

### 局限 2：维护成本高

文件多、规则多，需要持续维护。
**对策**：以"够用就好"为原则，不追求覆盖所有场景。

### 局限 3：依赖 Claude Code 生态

虽然设计上跨工具，但深度功能（如多 Agent 协作）依赖 Claude Code 的子代理能力。
**对策**：选用支持子代理的 AI 工具。

### 局限 4：缺乏端到端流程

ECC 是工具箱，不是工作流。如果团队需要"开箱即用的流程"，可能更适合 GSD 或 Spec-Kit。
**对策**：ECC + Spec-Kit 组合使用——ECC 提供能力，Spec-Kit 提供流程。

---

## 九、ECC 适用场景判断表

| 场景特征 | 推荐度 |
|---------|--------|
| 安全敏感项目（金融、医疗） | ★★★★★ |
| 多人协作 | ★★★★ |
| 长期维护项目 | ★★★★★ |
| 快速 prototype | ★★ |
| 小型项目（单人短期） | ★★ |
| 重视代码质量 | ★★★★★ |
| 需要标准化流程 | ★★★ |
| 团队有时间投入维护 | ★★★★★ |

---

## 十、与 OpenSpec/Spec-Kit 的关系

ECC 与 SDD 框架是**互补关系**，不是替代关系：

```
SDD 框架（OpenSpec/Spec-Kit）→ 提供"做什么"和"怎么做"的流程骨架
ECC                          → 提供"用什么能力做"的工具箱
```

**经典组合**：
- 用 OpenSpec 管变更流程（propose → apply → archive）
- 在 apply 阶段调用 ECC 的 Agents（coder、security-reviewer）
- 用 ECC 的 Hooks 自动化执行安全扫描和 lint

这种组合让你既有**流程的严谨**，又有**能力的丰富**。

---

## 十一、关键启发

通过 ECC 这个代表，能提炼出**能力增强类框架的共性设计原则**：

1. **能力是可组合的资产**——不是单体 AI，而是模块化能力库
2. **角色专精胜过全能**——每个 Agent 单一职责，上下文聚焦
3. **经验必须可沉淀**——通过 Skills/Rules 把一次性经验变成长期资产
4. **自动化优于纪律**——Hooks 让正确的事情自动发生
5. **对抗优于审查**——红蓝队对抗比单视角审查更有效

这些原则不只属于 ECC——它们是**任何能力增强类框架的共同基础**。

---

## 十二、扩展阅读

- [Everything Claude Code 文档](https://article.juejin.cn/post/7612177493036859418)
- [Everything Claude Code：终极配置指南](https://blog.csdn.net/2401_83343725/article/details/159580552)
- [ECC：让 AI 代理真正为你工作的完整系统](https://fastbee.blog.csdn.net/article/details/159356773)
- [全栈后端开发资源全景图：18个Agent+28个Skill+14套Rules](https://juejin.cn/post/7632252682875994127)
- [Anthropic - Claude Code 自动化安全审查](https://support.anthropic.com/zh-CN/articles/11932705-claude代码中的自动化安全审查)
- [Agent 怎么用？区分 Commands、Skills、Agents](https://blog.csdn.net/weixin_70573287/article/details/161426957)