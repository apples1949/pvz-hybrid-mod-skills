# 贡献指南 —— Issue 与 PR 规范

> 本仓库是《植物大战僵尸杂交版》(Godot 4 + C#) 的 WorkBuddy Agent Skills 集合。
> **⚠️ 本仓库的 `SKILL.md` 会被 AI 直接当作指令加载**——一条错的结论会持续误导后续**所有**会话。
> 因此本仓库对「证据」和「可复现性」的要求高于普通文档仓库：**不接受凭印象的改动**。

---

## 0. 收录范围

| 目录 | 收录 | 说明 |
|---|---|---|
| `skills/pvz-hybrid-mod-authoring/` | 通用 `.pmod` 包格式、打包、资源覆盖、托管插件 | 主百科 |
| `skills/pvz-hybrid-plant-authoring/` | 植物 Mod 端到端流程 | |
| `skills/pvz-hybrid-zombie-authoring/` | 僵尸 Mod 端到端流程 | |
| `tools/` | 生成器、工具链、离线闸门、实战案例文档 | |
| 根目录 `README.md` / `README_EN.md` / `GETTING_STARTED.md` | 对外说明与入门 | 中英双语需同步 |

**不受理**：游戏本体资源的分发/破解、与本仓库 mod 制作流程无关的讨论、要求改写官方构建器 exe（`pvzhe-level-builder.exe` 为二进制，纯 config 不可达的能力属既有边界）、未获授权的商业素材（贴图/音频/字体）。

---

## 1. 通用要求（Issue 与 PR 共通）

1. **结论先行**：先写「预期 vs 实际」，再写过程。
2. **必须给证据**：代码文件 + 行号、游戏内实测现象、日志片段、命令输出。**「我记得」「应该是」「一般会」一律视为无效证据。**
3. **必须可复现**：给出最小复现步骤。涉及环境差异的必须写清版本（见第 4 节）。
4. **一事一议**：一个 Issue / 一个 PR 只解决一件事。多个不相关改动请拆开。
5. **语言**：中文叙述；代码、路径、字段名、JSON 标识符保留英文原文。
6. **不确定就标注**：无法验证的部分请显式写【待确认】，不要写成结论。

---

## 2. Issue 提交规范

### 2.1 必须使用模板

空白 Issue 已关闭。请从 `.github/ISSUE_TEMPLATE/` 里选对应表单：

| 模板 | 适用场景 | 自动标签 |
|---|---|---|
| **内容纠错** | 技能/文档里的结论、口径、命令、行号与实际不符 | `bug` `documentation` |
| **工具缺陷** | `tools/` 下脚本报错、产物不正确、闸门失败 | `bug` `tools` |
| **提案** | 新技能 / 新案例文档 / 新工具 | `enhancement` |
| **用法提问** | 按文档操作卡住、不确定选哪条路线 | `question` |

### 2.2 标题格式

```
[类型] 一句话结论
```

类型 ∈ `纠错` / `工具` / `提案` / `提问`。模板会自动带上前缀，请在后面补上**结论**而不是现象。

- ✅ `[纠错] 僵尸技能里 offset 的口径与引擎实际绘制原点不一致`
- ❌ `[纠错] 有个地方好像不对`

### 2.3 必填内容（内容纠错类）

1. **涉及文件**（skill / references / tools / 根文档）
2. **具体位置**：章节标题或行号
3. **原文摘录**：直接粘贴，不要转述
4. **错误证据**：代码行号 / 游戏实测 / 日志
5. **正确做法**：你验证过的结论
6. **影响面**：按错内容操作会导致什么后果
7. **环境**：游戏版本 / 构建器版本 / .NET 或 Godot 版本 / 操作系统

### 2.4 优先级与响应

| 级别 | 判据 | 处理 |
|---|---|---|
| **P0** | 按文档操作会直接失败、崩溃或损坏存档 | 优先修 |
| **P1** | 结论错误但使用者可自行绕过 | 排期修 |
| **P2** | 表述不清、错别字、排版 | 随手修 |

维护者一般会在若干工作日内给出首个回复；长时间无回复可礼貌 ping 一次。

### 2.5 直接关闭的情形

- 无复现步骤、无证据，且追问后不补充
- 与游戏本体机制无关（如游戏平衡性意见、联机功能请求）
- 要求收录第三方付费或未授权素材
- 重复提交（会打上 `duplicate`）

---

## 3. PR 提交规范

### 3.1 分支命名

```
<type>/<scope>-<short-desc>
```

`type` ∈ `feat` | `fix` | `docs` | `tools` | `chore`

示例：`docs/plant-fix-typo`、`fix/zombie-offset-origin`、`tools/pmod-guardrail`

### 3.2 提交信息

```
<scope>: <imperative summary>      ← 首行 ≤ 72 字符，英文，不加句号

为什么改 + 关键结论
验证证据：命令与结果（PASS 计数）
影响面 / 破坏性变更（无则省略）
```

- `<scope>` = 技能名 / 目录名 / 主题，例如 `pvz-hybrid-zombie-authoring`、`tools/README.md`、`GETTING_STARTED`。
- 参考本仓库既有风格：`GETTING_STARTED: switch the hands-on template to SuperGatlingPea`。
- **一个提交只做一件事**；大改动请拆成可独立验证的若干提交。

### 3.3 PR 必填项

PR 描述由 `.github/PULL_REQUEST_TEMPLATE.md` 强制，包含：**变更类型、动机与结论、变更文件清单、验证证据、不变量自查、影响面、关联 Issue**。

### 3.4 验证证据（硬要求）

| 变更类型 | 必须提供 |
|---|---|
| 纯文档 / 错别字 / 排版 | 无需跑闸门，但需说明影响面 |
| 技能内容修正（口径、判据、行号） | 证据来源 + **修改前后对照** |
| 生成器 / 工具链改动 | **on-disk 断言 + 负向测试** 的运行输出（含 PASS 计数） |
| 新增技能 / 新案例 | 对应闸门结果 + 至少一个真实跑通案例 |

**⚠️ 不接受「假绿」验证。** 断言必须：

1. 与 **GOLD 字面量**比较（不能拿实现产物的输出和自己比）；
2. **「实现产物」自己也有一条对账断言**——只断言「抄写目标 vs 金标」而放空实现产物 = 无效；
3. 能**容忍上游被改坏**：循环上界须做长度守卫，不得因索引越界把整个自检崩掉（应报 FAIL 而非 `IndexError`）；
4. 判据不足时必须**删掉该用例**，而不是留一条永远为真的假用例。

### 3.5 不变量自查（模板会逐条勾选）

- [ ] 未提交构建产物：`**/obj/`、`**/.build/`、`bin/`、生成的 mod 工程、构建好的 `.pmod`、对比 PNG
- [ ] 未提交 `.workbuddy/` 会话数据
- [ ] 未提交一次性探针 / 废弃脚本（`_*` 临时脚手架已在 `.gitignore`）
- [ ] `.bat` / `.cmd`：**纯 ASCII + CRLF 行尾 + 括号平衡**（cmd 按当前代码页逐字节读脚本，非 ASCII 会让整行被吞，报错极具误导性）
- [ ] 写文件用 `newline=""`（避免行尾被静默转换）
- [ ] 计数、指纹、项数**由脚本生成**，不手写
- [ ] 涉及「两份构建」（remake / console）的改动，两侧保持一致
- [ ] 改了对外能力 / 目录树 / 邀请码时，`README.md` 与 `README_EN.md` **双语同步**
- [ ] 脚本内不 spawn git 子进程、不把 token 写进仓库

### 3.6 评审与合并

- 至少 **1 名维护者 Approve**；**凡改动 `SKILL.md` 指令内容的 PR，维护者必须逐行过目**（这类文件等价于代码）。
- 合并策略：**Squash merge**，保持 `main` 线性。
  ⇒ **PR 标题会被当作最终提交信息**，因此标题必须符合 3.2 的格式。
- 合并前需满足：无冲突、闸门/自检通过、不变量自查完成。
- 直推 `main` 仅限维护者做纯文档或脚手架改动。

---

## 4. 环境声明要求

报告问题或提交 PR 时，请注明以下版本（缺失会显著拖慢定位）：

| 项 | 示例 |
|---|---|
| 游戏版本 | V0.28 / V0.29 |
| 关卡构建器版本 | v0.28 |
| .NET / Godot 版本 | .NET 9 / Godot 4.x |
| 操作系统 | Windows 11 |

本仓库内容默认对应 **游戏 V0.28/V0.29 + 构建器 v0.28**。其他版本请显式说明差异。

---

## 5. 目录与命名约定

```
skills/<skill-name>/SKILL.md        # 技能主体（被 AI 加载的指令）
skills/<skill-name>/references/     # 参考资料
skills/<skill-name>/assets/         # 技能自带素材
tools/<category>/                   # 生成器 / 工具链 / 闸门
tools/case-docs/<类型>Mod-<名称>.md  # 实战案例文档
```

---

## 6. English summary (TL;DR)

- **Every `SKILL.md` in this repo is loaded by an AI as instructions.** A wrong statement keeps misleading every future session — evidence is mandatory.
- **Issues**: use the templates under `.github/ISSUE_TEMPLATE/` (blank issues are disabled). Title: `[type] one-line conclusion`. Provide location, verbatim quote, verifiable evidence, correct behaviour, impact, and environment.
- **PRs**: branch `<type>/<scope>-<desc>`; commit subject `<scope>: <imperative summary>` (≤72 chars, no trailing period); fill the PR template including **verification evidence** and the **invariant checklist**.
- **Evidence must not be "false green"**: assert against GOLD literals, give the *produced artefact* its own reconciliation assertion, keep assertions robust to upstream breakage, and drop any test case that cannot actually fail.
- **Never commit** build artifacts (`obj/`, `.build/`, `bin/`, generated mod projects, built `.pmod`, diff PNGs) or `.workbuddy/` session data. `.bat`/`.cmd` must be ASCII-only with CRLF line endings.
- Merging is **squash-only**; the PR title becomes the commit subject, so keep it in the required format.
