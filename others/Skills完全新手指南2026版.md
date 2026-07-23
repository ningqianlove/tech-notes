# Skills 完全新手指南(2026版)：从入门到精通

> 原创 康健康健  程序员AI破局指南  
> 2026年4月7日 06:00 · 北京

Skills 是 Claude Code 最被低估的功能——看似简单，却解决了两大 AI 使用的核心痛点：**重复输入** 和 **通用与专业的矛盾**。

它的逻辑很直接：在 Skills 目录下写一个 `SKILL.md` 文件，告诉 Claude「什么情况下该做什么」，它会在合适的时机自动加载，你不需要每次重复提示。

本文从零开始，系统讲透这套机制。读完你会掌握：

- Skills 的运行架构：元数据注入、按需加载、脚本执行的完整链路
- SKILL.md 的所有配置项（13 个 frontmatter 字段逐一解析）
- 4 种存放位置的优先级规则，以及 Monorepo 的自动发现机制
- 三种动态内容机制：参数变量、`${CLAUDE_SKILL_DIR}` 路径引用、Shell 命令注入
- `context: fork` 子代理隔离执行的使用场景与配置方式
- 5 个内置 Skills 的实际效果（`/batch` 并行重构、`/simplify` 代码优化等）
- 技能的三种分发方式：Git 版本控制、插件打包、企业管理配置

全文共 **10 章 + 附录**，按需跳读：

| 你的情况 | 推荐入口 |
|---|---|
| 刚接触 Skills，想建立整体认知 | 第一章，约 10 分钟 |
| 想立刻动手，跟着步骤做 | 第二章，创建第一个技能 |
| 想搞懂底层运行原理 | 第一章 1.4 节，有架构流程图 |
| 配置写了但不生效 | 第四章 Frontmatter 详解 + 第十章故障排查 |
| 需要把技能共享给团队 | 第九章，三种分发方式对比 |
| 碎片时间快速查配置项 | 直接翻附录速查卡 |

---

## 目录

**第一章：什么是 Claude Code Skills？**
- 1.1 Skills 能解决什么问题？
- 1.2 Skills 的核心概念
- 1.3 Skills 与其他功能的关系
- 1.4 架构原理：技能如何运行？

**第二章：快速上手——创建你的第一个 Skill**
- 2.1 Skill 的文件结构
- 2.2 SKILL.md 的结构解析
- 2.3 动手实践：创建 explain-code 技能

**第三章：Skill 存放位置详解**
- 3.1 四个存放位置
- 3.2 优先级与同名冲突
- 3.3 Monorepo 的子目录自动发现
- 3.4 --add-dir 的特殊行为

**第四章：Frontmatter 配置详解**
- 4.1 完整配置参考表
- 4.2 重点字段详解

**第五章：变量替换与动态内容**
- 5.1 参数变量
- 5.2 Shell 命令注入（动态上下文）

**第六章：支持文件——构建强大技能**
- 6.1 为什么需要支持文件？
- 6.2 推荐目录结构
- 6.3 在 SKILL.md 中引用支持文件
- 6.4 实战案例：带可视化的代码库分析技能

**第七章：高级模式——子代理与权限控制**
- 7.1 在子代理中运行技能（context: fork）
- 7.2 工具访问控制
- 7.3 扩展思维模式

**第八章：内置 Skills 介绍**
- 8.1 内置技能列表
- 8.2 /batch 详解——最强大的内置技能

**第九章：分享与分发技能**
- 9.1 三种分发方式
- 9.2 Agent Skills 开放标准

**第十章：故障排查与最佳实践**
- 10.1 常见问题排查
- 10.2 技能编写最佳实践
- 10.3 技能开发工作流建议

**附录：完整 Frontmatter 速查卡**

**参考资源**

---

## 第一章：什么是 Claude Code Skills？

如果你刚开始接触 Claude Code，可能会好奇 "Skills" 究竟是什么。简单来说：Skills 就是给 Claude 定制的"专项技能包"，让它学会某一特定领域的知识或工作流程，从而在处理相关任务时更专业、更高效。

把 Skills 想象成这样一个场景：你请来了一位全能助理（Claude），他非常聪明，但对你的公司、你的项目、你的习惯一无所知。而 Skills 就相当于你给他准备的"岗前培训手册"——告诉他你们团队用什么技术栈、代码风格如何、如何部署项目……有了这份手册，助理就能立刻进入状态，不需要每次都重新解释。

### 1.1 Skills 能解决什么问题？

在没有 Skills 之前，使用 Claude Code 时你可能会遇到这些烦恼：

- **重复解释**：每次开新会话都要告诉 Claude "我们用 TypeScript，遵循 AirBnb 风格规范……"
- **上下文丢失**：跨项目时，Claude 对不同项目的特定约定毫不知情
- **工作流碎片化**：部署、代码审查、提交等操作需要手动逐步提示
- **知识无法沉淀**：团队积累的最佳实践只存在于人的脑子里，无法系统化传递给 AI

Skills 正是为了解决这些问题而生。它让你把反复使用的知识和流程"固化"成可复用的配置文件，Claude 会在需要时自动加载并应用。

### 1.2 Skills 的核心概念

| 概念 | 说明 | 类比 |
|---|---|---|
| SKILL.md | 技能的核心文件，包含指令和元数据 | 岗位说明书 |
| frontmatter | SKILL.md 顶部的 YAML 配置区域 | 档案封面 |
| 斜杠命令 | /skill-name 形式的调用方式 | 快捷拨号 |
| 自动触发 | Claude 根据场景判断是否加载技能 | 情景感应 |
| 支持文件 | 技能目录内的额外文件（模板、脚本等） | 工具箱 |

### 1.3 Skills 与其他功能的关系

Claude Code 中有几个相似的概念，容易混淆：

| 功能 | 本质 | Skills 的区别 |
|---|---|---|
| CLAUDE.md | 项目级持久记忆 | Skills 是可复用的能力包，不只是记忆 |
| 自定义命令（Commands） | 单文件的斜杠命令 | Skills 是命令的超集，支持目录结构和更多特性 |
| 子代理（Subagents） | 独立执行任务的 AI 实例 | Skills 可以在子代理中运行，也可以独立运行 |
| 插件（Plugins） | 可分发的扩展包 | Plugins 内部可以包含 Skills |

> 💡 **向后兼容提示**
>
> Skills 与旧版自定义命令完全兼容。你在 `.claude/commands/deploy.md` 创建的命令，和在 `.claude/skills/deploy/SKILL.md` 创建的技能，效果完全一样，`/deploy` 命令都会生效。无需迁移已有配置！

### 1.4 架构原理：技能如何运行？

了解 Skills 背后的运行机制，有助于你写出更高效的技能配置。技能运行于代理的虚拟机环境中。以 Claude 平台为例，代理启动时会扫描已安装技能目录，将每个技能的元数据（name、description）注入系统提示；只有当用户请求符合某技能触发条件时，代理才用 Bash 命令读取该技能的 SKILL.md 并将其内容加载到上下文。

如果指令中引用了附加文件（参考文档、示例、模式模板等），代理也会通过 Bash 动态读取；如指令中提到可执行脚本，代理会直接运行脚本，获取输出而不将脚本源代码载入上下文。这种"逐步加载"使得代理可以包含大量参考内容但不会一次性占用上下文窗口。

技能体系的完整工作流程：

1. 代理启动 → 扫描技能目录 → 注入元数据到系统提示
2. 用户发起请求 → 匹配技能触发条件 → 按需加载 SKILL.md
3. 引用支持文件 → 动态读取 → 补充上下文
4. 执行脚本 → 获取输出 → 注入结果
5. 技能组合协同 → 完成复杂任务

此外，技能可以组合使用（多个技能同时在容器中可用），复杂任务可由多技能协同处理。例如在财务分析场景中，可以同时载入 Excel 处理技能和自定义的财务估值技能，代理根据触发条件并行调用各自的能力。

总体来看，技能架构的核心设计哲学是：需求驱动时才加载相关知识，大幅减少对话上下文的占用，且相同技能可在不同场景下重复复用。

---

## 第二章：快速上手——创建你的第一个 Skill

理论够了，动手最重要。这一章我们从零开始，手把手创建一个真实可用的 Skill。

### 2.1 Skill 的文件结构

一个 Skill 就是一个目录，最简单的形式只需要一个文件：

```
my-skill/
├── SKILL.md          ← 必需，技能的核心文件
├── template.md       ← 可选，Claude 填充的模板
├── examples/
│   └── sample.md     ← 可选，示例输出
└── scripts/
    └── helper.sh     ← 可选，可执行脚本
```

其中 **SKILL.md** 是唯一必需的文件。其他文件可以根据需求添加，用来支撑更强大的能力。

### 2.2 SKILL.md 的结构解析

每个 SKILL.md 由两部分组成：

```yaml
---
name: my-skill           ← 技能名（生成 /my-skill 命令）
description: 描述这个技能的作用和触发时机
---

## 这里是正文

当 Claude 调用这个技能时，会按照以下指令执行：

1. 步骤一
2. 步骤二
...
```

两个 `---` 之间的部分叫做 **frontmatter**（前言），使用 YAML 格式，用来告诉 Claude 技能的元信息。frontmatter 之后的 Markdown 内容，就是 Claude 执行技能时遵循的指令。

### 2.3 动手实践：创建 explain-code 技能

我们来创建一个"代码讲解"技能，让 Claude 在解释代码时总是使用类比和 ASCII 图示。

**步骤一：创建技能目录**

个人技能存放在 `~/.claude/skills/` 目录下，对所有项目生效：

```bash
mkdir -p ~/.claude/skills/explain-code
```

**步骤二：编写 SKILL.md**

创建文件 `~/.claude/skills/explain-code/SKILL.md`，内容如下：

```yaml
---
name: explain-code
description: 用类比和图示解释代码原理。当用户问"这段代码是什么意思？"、
"这个函数怎么工作？"时自动触发。
---

解释代码时，必须包含以下四个部分：

1. **生活类比**：用日常生活的例子打比方
2. **ASCII 图示**：用字符画出结构或流程
3. **逐步解析**：一行一行解释代码做了什么
4. **常见误区**：指出这里最容易搞错的地方

语气要通俗易懂，适合初学者。
```

**步骤三：测试技能**

技能创建完成后，有两种方式触发它：

1. **自动触发**：直接问 Claude 相关问题
   ```
   /chat 这段 Promise 代码是什么意思？
   ```

2. **手动触发**：使用斜杠命令
   ```
   /explain-code src/auth/login.ts
   ```

Claude 在回答时，就会按照你的技能指令，必然包含类比和图示。

> ✅ **验证技能已加载**
>
> 在 Claude Code 中输入 "What skills are available?" 可以查看已加载的所有技能列表，确认你的技能出现在其中。

---

## 第三章：Skill 存放位置详解

不同的存放位置决定了 Skill 的作用范围——是个人独享、项目专属，还是团队共享。

### 3.1 四个存放位置

| 位置 | 路径 | 作用范围 | 适用场景 |
|---|---|---|---|
| 企业级 | 通过管理配置下发 | 所有组织成员 | 公司统一规范、安全策略 |
| 个人级 | `~/.claude/skills/*/` | 你的所有项目 | 个人习惯、通用工具 |
| 项目级 | `.claude/skills/*/` | 当前项目 | 项目特定知识、团队协作 |
| 插件内 | `/skills/*/` | 插件启用的地方 | 随插件一起分发 |

### 3.2 优先级与同名冲突

当多个位置存在同名技能时，优先级从高到低：

**企业级 > 个人级 > 项目级**

插件内的技能使用 `plugin-name:skill-name` 的命名空间，不会与其他级别冲突。

### 3.3 Monorepo 的子目录自动发现

如果你的项目是 Monorepo 结构，Claude Code 会自动发现子目录中的技能。例如，当你在编辑 `packages/frontend/` 内的文件时，Claude Code 会自动查找并加载 `packages/frontend/.claude/skills/` 中的技能。

```
my-monorepo/
├── .claude/skills/              ← 根目录技能（所有子包可用）
│   └── deploy/
├── packages/
│   ├── frontend/
│   │   └── .claude/skills/      ← 前端专属技能
│   │       └── react-patterns/
│   └── backend/
│       └── .claude/skills/      ← 后端专属技能
│           └── api-conventions/
```

### 3.4 --add-dir 的特殊行为

当你用 `--add-dir` 标志添加额外目录时，该目录内的 `.claude/skills/` 会被自动加载，这是 Skills 特有的例外规则（其他配置如 subagents、commands 不会被加载）。

这个特性非常适合共享 Skill 库的场景：

```bash
claude --add-dir /path/to/shared-skills-repo
```

而且支持"热更新"——在会话中修改这些技能文件后，不需要重启 Claude Code，修改会立即生效。

---

## 第四章：Frontmatter 配置详解

Frontmatter 是控制 Skill 行为的"控制面板"。掌握这些配置项，才能真正发挥 Skills 的威力。

### 4.1 完整配置参考表

| 字段 | 是否必需 | 默认值 | 说明 |
|---|---|---|---|
| name | 否 | 目录名 | 技能显示名，生成 /name 斜杠命令。只能含小写字母、数字、连字符，最长 64 字符 |
| description | 推荐 | 正文第一段 | Claude 用此决定是否自动加载。前置关键词，因为超过 250 字符会被截断 |
| argument-hint | 否 | 无 | 自动补全时显示的参数提示，如 `[filename] [format]` |
| disable-model-invocation | 否 | false | 设为 true 时，只有你能手动触发，Claude 不会自动加载 |
| user-invocable | 否 | true | 设为 false 时，从 / 菜单隐藏，只由 Claude 自动调用 |
| allowed-tools | 否 | 继承全局 | 此技能激活时，Claude 可无需询问直接使用的工具列表 |
| model | 否 | 继承会话 | 此技能激活时使用的模型 |
| effort | 否 | 继承会话 | 努力级别：low / medium / high / max |
| context | 否 | 内联 | 设为 fork 时，在隔离的子代理中运行 |
| agent | 否 | general-purpose | 与 context: fork 配合，指定子代理类型 |
| hooks | 否 | 无 | 技能生命周期钩子配置 |
| paths | 否 | 无 | Glob 模式，限定技能只在特定文件类型时激活 |
| shell | 否 | bash | 用于 `` `command` `` 注入的 shell，可选 powershell |

### 4.2 重点字段详解

**description —— 最关键的字段**

这是整个 frontmatter 中最重要的字段。Claude 依赖它来判断"什么时候该加载这个技能"。写好 description 的原则：

- 前置关键词：最重要的触发关键词放在最前面，因为超过 250 字符会被截断
- 描述使用场景：明确说明"何时使用"，如 "当用户问……时使用"
- 避免过于宽泛：太宽泛会导致技能频繁被触发，影响性能

反面示例 vs 正面示例：

```yaml
# ❌ 太模糊，容易过度触发
description: 帮助 React 开发

# ✅ 精确描述触发场景
description: React 组件性能优化指导。当代码包含 useEffect、useMemo、useCallback，
或用户询问组件卡顿、重复渲染问题时使用。
```

**disable-model-invocation vs user-invocable**

这两个字段控制"谁能触发技能"，很多人搞混：

| 配置 | 你能 /调用 | Claude 能自动调用 | 出现在 / 菜单 |
|---|---|---|---|
| （默认） | ✅ | ✅ | ✅ |
| disable-model-invocation: true | ✅ | ❌ | ✅ |
| user-invocable: false | ❌ | ✅ | ❌ |

> 💡 **使用场景建议**
>
> `disable-model-invocation: true` 适合有副作用的操作（部署、发送消息），你不希望 Claude 自动执行。
>
> `user-invocable: false` 适合背景知识型技能（如 legacy-system-context），Claude 应该知道，但这不是一个有意义的"命令"。

**paths —— 按文件类型激活**

让技能只在处理特定文件时才被激活，避免污染不相关的任务：

```yaml
---
name: react-patterns
description: React 最佳实践
paths: "**/*.tsx,**/*.jsx"
---

# 这个技能只在处理 React 文件时才会被 Claude 考虑加载
```

**effort —— 控制思考深度**

可以为特定技能设置不同的思考强度，平衡速度和质量。注意 max 级别仅 Claude Opus 4.6 可用。

```yaml
---
name: deep-review
description: 深度代码审查
effort: max
---
```

---

## 第五章：变量替换与动态内容

静态指令有限制——有时你需要把用户输入的内容传递给技能，或者注入实时数据。Skills 提供了两种机制来实现这一点。

### 5.1 参数变量

当用户调用 `/skill-name` 时，可以传入参数，技能内用变量引用：

| 变量 | 说明 | 示例 |
|---|---|---|
| `$ARGUMENTS` | 所有参数，合并为字符串 | `/fix-bug 1234` → `"1234"` |
| `$ARGUMENTS[N]` | 第 N 个参数（从 0 开始） | `$ARGUMENTS[0]` → 第 1 个参数 |
| `$0, $1, $2...` | `$ARGUMENTS[N]` 的简写 | `$0` 同 `$ARGUMENTS[0]` |
| `${CLAUDE_SESSION_ID}` | 当前会话 ID | 用于日志、临时文件命名 |
| `${CLAUDE_SKILL_DIR}` | 技能目录的绝对路径 | 引用捆绑脚本时使用 |

**单参数示例**

```yaml
---
name: fix-issue
description: 修复 GitHub Issue
disable-model-invocation: true
---

修复 GitHub Issue #$ARGUMENTS，遵循以下步骤：

1. 阅读 Issue 描述
2. 分析根本原因
3. 实现修复
4. 编写测试
5. 提交代码
```

执行 `/fix-issue 42` 时，Claude 收到的指令变为 "修复 GitHub Issue #42，遵循以下步骤……"

**多参数示例**

```yaml
---
name: migrate-component
description: 将组件从一个框架迁移到另一个框架
---

将 $0 组件从 $1 迁移到 $2。

保留所有现有功能和测试，不引入行为变更。
```

执行 `/migrate-component SearchBar React Vue` 时，Claude 收到："将 SearchBar 组件从 React 迁移到 Vue……"

### 5.2 Shell 命令注入（动态上下文）

使用 `` !`command` `` 语法，可以在技能被发送给 Claude 之前，先执行一段 Shell 命令，并将输出注入到技能内容中。

> ⚠️ **重要区别**
>
> 这不是 Claude 执行命令——而是 Claude Code 在发送技能给 Claude 之前预处理。Claude 只看到最终输出的文本，不知道有命令被执行过。

**内联注入**

```yaml
---
name: pr-summary
description: 总结当前 Pull Request 的变更
context: fork
allowed-tools: Bash(gh *)
---

## PR 上下文

- 变更差异：!`gh pr diff`
- PR 评论：!`gh pr view --comments`
- 修改文件：!`gh pr diff --name-only`

## 你的任务

请总结这个 PR 的主要变更、影响范围和潜在风险。
```

**多行命令注入**

对于多行命令，使用 ```! 代码块：

```markdown
## 运行环境

```!
node --version
npm --version
git log --oneline -5
```
```

**禁用 Shell 注入**

管理员可以在设置中添加 `"disableSkillShellExecution": true` 来禁用此功能（适合安全敏感环境）。被禁用时，注入点会替换为 `[shell command execution disabled by policy]`。

---

## 第六章：支持文件 —— 构建强大技能

单个 SKILL.md 适合简单任务，但复杂能力需要多个文件配合。这一章介绍如何组织技能目录，让技能更强大、更易维护。

### 6.1 为什么需要支持文件？

有几个场景，单文件 SKILL.md 会遇到瓶颈：

- **内容太长**：详细的 API 文档、参考规范塞进一个文件，每次触发技能都全量加载，占用大量 context
- **输出需要模板**：代码审查报告、技术文档等有固定格式，需要模板文件
- **需要执行脚本**：生成可视化、数据处理等任务，需要 Python/Shell 脚本支持

解决方案：把这些内容拆分到独立文件中，在 SKILL.md 中引用，让 Claude 按需加载。

### 6.2 推荐目录结构

```
code-review/
├── SKILL.md                    ← 核心指令（控制在 500 行以内）
├── checklist.md                ← 详细审查清单（按需加载）
├── templates/
│   └── report.md               ← 审查报告模板
├── examples/
│   ├── good-example.ts         ← 合格代码示例
│   └── bad-example.ts          ← 需要改进的示例
└── scripts/
    └── complexity.py           ← 计算代码复杂度的脚本
```

### 6.3 在 SKILL.md 中引用支持文件

在 SKILL.md 里明确告诉 Claude 每个文件的内容和加载时机：

```yaml
---
name: code-review
description: 全面的代码审查，包含安全、性能、可维护性检查
---

执行代码审查，聚焦以下维度：安全、性能、可读性。

## 附加资源

- 详细审查清单：见 [checklist.md](checklist.md)（执行审查时加载）
- 报告模板：见 [templates/report.md](templates/report.md)（生成报告时使用）
- 运行复杂度分析：python ${CLAUDE_SKILL_DIR}/scripts/complexity.py

## 输出要求

使用模板格式输出审查报告。
```

> 📌 **最佳实践**
>
> 将 SKILL.md 保持在 500 行以内，详细参考资料移入独立文件。Claude 会在真正需要时才加载它们，节省 context 窗口空间。

### 6.4 实战案例：带可视化的代码库分析技能

下面是一个使用支持文件的完整示例——一个能生成可交互 HTML 可视化的技能：

**目录结构**

```
codebase-visualizer/
├── SKILL.md
└── scripts/
    └── visualize.py
```

**SKILL.md**

```yaml
---
name: codebase-visualizer
description: 生成代码库的可交互 HTML 树状可视化。
探索新代码库、了解项目结构时使用。
allowed-tools: Bash(python *)
---

# 代码库可视化

运行可视化脚本：

```bash
python ${CLAUDE_SKILL_DIR}/scripts/visualize.py .
```

这将在当前目录生成 `codebase-map.html` 并在浏览器中打开。
```

注意 `${CLAUDE_SKILL_DIR}` 的用法——无论当前工作目录在哪，这个变量始终指向技能文件所在的目录，确保脚本路径正确。

---

## 第七章：高级模式 —— 子代理与权限控制

掌握基础后，这一章介绍两个让 Skills 真正强大的高级特性：在独立子代理中运行技能，以及精细控制工具权限。

### 7.1 在子代理中运行技能（context: fork）

`context: fork` 让技能在隔离的子上下文中执行，不影响主对话。这就像派遣一个"特种兵"去完成特定任务，完成后汇报结果。

**何时使用 context: fork？**

- 任务独立性强：需要大量探索、不应干扰主对话的研究任务
- 需要特定工具：任务需要只读工具集（如 Explore 代理）
- 避免上下文污染：PR 分析、代码库扫描等不需要对话历史的任务

**示例：深度研究技能**

```yaml
---
name: deep-research
description: 对指定主题进行深入研究，分析代码库
context: fork
agent: Explore
---

深入研究：$ARGUMENTS

1. 用 Glob 和 Grep 找到相关文件
2. 读取并分析代码
3. 总结发现，附上具体文件引用
```

**可用的内置代理类型**

| 代理类型 | 工具集 | 适用场景 |
|---|---|---|
| Explore | 只读工具（Read, Grep, Glob） | 代码库研究、文档生成 |
| Plan | 读取 + 规划工具 | 任务分解、架构设计 |
| general-purpose | 完整工具集 | 通用任务（默认） |
| 自定义代理 | 由 .claude/agents/ 定义 | 项目特定的专属代理 |

### 7.2 工具访问控制

**allowed-tools 限制工具**

在技能激活期间，可以限制 Claude 只使用特定工具，这对只读技能或安全敏感操作非常有用：

```yaml
---
name: safe-reader
description: 只读模式，不修改任何文件
allowed-tools: Read Grep Glob
---

在只读模式下分析代码...
```

多个工具可以空格分隔，也可以用 YAML 列表格式：

```yaml
allowed-tools:
  - Read
  - Grep
  - Bash(gh *)
  - Bash(python *)
```

**限制技能的全局访问**

除了在单个技能里控制，还可以在权限配置中全局管理哪些技能 Claude 可以调用：

```
# 在 /permissions 中禁用所有技能
Skill

# 只允许特定技能
Skill(commit)
Skill(review-pr *)

# 禁止特定技能
Skill(deploy *)
```

### 7.3 扩展思维模式

在技能内容中任意位置包含 "ultrathink" 这个词，就能激活 Claude 的深度思考模式，适合需要复杂推理的技能：

```yaml
---
name: architecture-review
description: 深度架构审查
effort: max
---

对这个系统进行架构审查。ultrathink

分析：可扩展性、安全性、可维护性、性能...
```

---

## 第八章：内置 Skills 介绍

Claude Code 预置了几个高质量的内置技能，可以直接使用，无需任何配置。

### 8.1 内置技能列表

| 命令 | 功能 | 典型用法 |
|---|---|---|
| `/batch <指令>` | 并行大规模重构：自动拆解任务，为每个子任务创建独立分支并行处理 | `/batch 将 src/ 下所有组件从类组件迁移到函数组件` |
| `/claude-api` | 加载 Claude API 参考文档（支持 Python、TS、Go 等 8 种语言） | 当代码 import anthropic 时自动触发 |
| `/debug [描述]` | 启用 debug 日志，分析当前会话的调试信息 | `/debug 为什么这个工具调用总是失败？` |
| `/loop [间隔] <提示>` | 按间隔重复执行提示，适合监控类任务 | `/loop 5m 检查部署是否完成` |
| `/simplify [焦点]` | 并行启动三个代理审查最近修改的代码，发现并修复质量问题 | `/simplify 专注于内存效率` |

### 8.2 /batch 详解 —— 最强大的内置技能

/batch 是内置技能中最令人印象深刻的一个，它能把大型重构任务自动化：

**工作流程：**

1. Claude 研究代码库，理解任务范围
2. 将工作分解为 5~30 个独立单元
3. 展示计划，等待你确认
4. 为每个单元创建独立的 git worktree
5. 并行启动多个后台代理，各自处理一个单元
6. 每个代理：实现 → 测试 → 提交 → 开 PR

> ⚠️ **使用前提**
>
> /batch 需要 git 仓库环境。适合大型、相对独立的批量修改任务，不适合需要严格依赖顺序的变更。

---

## 第九章：分享与分发技能

好的技能应该被共享，而不是只留给自己用。

### 9.1 三种分发方式

**方式一：Git 版本控制（团队协作）**

把项目级技能提交到代码仓库，团队成员 clone 后即可使用：

```bash
git add .claude/skills/
git commit -m "add: code-review and deploy skills"
git push
```

所有团队成员的 Claude Code 都会自动加载这些技能，无需单独配置。

**方式二：插件（Plugins）**

如果你的技能与其他配置（MCP 服务器、钩子等）配套使用，可以打包成插件：

```
my-plugin/
├── claude-plugin.json    ← 插件清单
├── skills/
│   ├── deploy/
│   └── review/
└── hooks/
    └── post-commit.sh
```

**方式三：企业级管理配置**

企业版用户，管理员可以通过管理配置文件为所有成员统一下发技能，新成员入职即自动获得团队标准技能集。

### 9.2 Agent Skills 开放标准

Anthropic 已将 Skills 发布为开放标准（agentskills.io）。这意味着：

- **跨平台兼容**：同一个 SKILL.md 可以在 Claude Code、其他支持该标准的 AI 工具中使用
- **生态共享**：社区可以贡献和分发通用技能包
- **未来兼容**：随着更多工具支持该标准，你的技能投资不会过时

---

## 第十章：故障排查与最佳实践

技能不按预期工作？这一章提供系统化的排查方法和经验总结。

### 10.1 常见问题排查

**问题一：技能没有触发**

按以下步骤排查：

1. 确认技能文件存在：检查路径和文件名是否正确（SKILL.md，注意大小写）
2. 验证技能已被加载：在 Claude Code 中问 "What skills are available?"
3. 检查 description 是否包含用户可能使用的关键词
4. 尝试直接手动触发：`/skill-name`，确认技能语法正确
5. 检查是否设置了 `disable-model-invocation: true`（如果设了，Claude 不会自动触发）

**问题二：技能触发太频繁**

解决方案：

1. 让 description 更具体，减少误触发关键词
2. 设置 `disable-model-invocation: true`，改为手动触发
3. 使用 `paths` 字段限制文件类型

**问题三：description 被截断**

症状：技能名称出现在列表中，但关键触发词丢失，导致 Claude 不能正确匹配。

解决方案：

1. 把最重要的触发关键词移到 description 的最前面
2. 每个技能 description 严格控制在 250 字符以内（这是硬性限制）
3. 如需增加总预算，设置环境变量：`export SLASH_COMMAND_TOOL_CHAR_BUDGET=16000`

**问题四：Shell 注入命令失败**

排查步骤：

1. 在终端手动运行命令，确认命令本身正确
2. 检查技能目录内的脚本文件权限：`chmod +x scripts/xxx.sh`
3. 确认 `disableSkillShellExecution` 未被管理员禁用

### 10.2 技能编写最佳实践

| 方面 | 推荐做法 | 避免 |
|---|---|---|
| 文件大小 | SKILL.md 控制在 500 行以内 | 把所有内容都堆在一个文件里 |
| description | 前置关键词，具体描述使用场景，<250 字符 | 模糊描述如"帮助开发" |
| 副作用操作 | 设置 disable-model-invocation: true | 让 Claude 自动触发部署/发送等操作 |
| 工具权限 | 用 allowed-tools 最小化授权 | 给只读技能授予写入权限 |
| 脚本路径 | 使用 ${CLAUDE_SKILL_DIR} 引用捆绑脚本 | 使用硬编码的绝对路径 |
| 技能范围 | 一个技能只做一件事 | 把不相关功能塞进同一个技能 |
| 版本控制 | 项目技能提交到 git | 技能只存在于本地 |

### 10.3 技能开发工作流建议

这是一个经过验证的技能开发流程：

1. **明确需求**：写下你想让 Claude 做什么，以及触发时机
2. **从简单开始**：先写最小可用版本，只有 name、description 和核心指令
3. **测试触发**：自动触发和手动触发都测一遍
4. **收集反馈**：实际使用几天，观察哪里不符合预期
5. **迭代优化**：根据反馈调整 description、指令细节
6. **添加支持文件**：需要时才添加模板、示例、脚本，不要过度设计
7. **提交分享**：好的技能提交到代码库，与团队共享

---

## 附录：完整 Frontmatter 速查卡

```yaml
---
# ── 基本信息 ──────────────────────────────────────────
name: skill-name          # 技能名，生成 /skill-name 命令
description: >            # Claude 的触发判断依据，<250字符
  关键词优先，描述何时使用。

# ── 调用控制 ───────────────────────────────────────────
disable-model-invocation: false   # true = 只能手动 /调用
user-invocable: true              # false = 从菜单隐藏，仅 Claude 自动调用

# ── 工具与模型 ─────────────────────────────────────────
allowed-tools: Read Grep Glob     # 无需审批的工具（空格分隔）
model: claude-sonnet-4-6          # 覆盖默认模型
effort: medium                    # low / medium / high / max

# ── 执行环境 ──────────────────────────────────────────
context: fork                     # 在隔离子代理中运行
agent: Explore                    # 指定子代理类型

# ── 激活范围 ──────────────────────────────────────────
paths: "**/*.tsx,**/*.jsx"       # 仅在匹配文件时激活

# ── 其他 ─────────────────────────────────────────────
argument-hint: "[issue-number]"   # 自动补全提示
shell: bash                       # bash 或 powershell
---

## 技能正文内容（Markdown）

使用 $ARGUMENTS 引用用户传入的参数。
使用 !`shell-command` 注入动态上下文。
使用 ${CLAUDE_SKILL_DIR} 引用技能目录路径。
```

---

## 参考资源

- 官方文档：code.claude.com/docs/en/skills
- Agent Skills 开放标准：agentskills.io
- 技能生态 cookbook：platform.claude.com/docs/en/build-with-claude/skills-guide
- 子代理文档：code.claude.com/docs/en/sub-agents

---

*来源：程序员AI破局指南 · 康健康健*
