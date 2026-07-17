# Module Parser Contract RFC

> 类型：RFC（Request for Comments）
> 日期：2026-07-17（对齐 codex/agent-collaboration-aligned 架构）
> 定位：Module Parser Agent 的 Stage IO Contract 与 ModuleContent 数据契约
> 消费者：成员 B（确定性引擎，消费 ModuleContent）
> 生产者：成员 C（Module Parser Agent）
>
> **框架对齐说明**：本文档定义的 Stage IO 对象中，`ModuleContent` 实现在 `contracts/module.py`（B/C 共享契约，`ContractModel` 使用 `frozen=True` + `tuple`）。`ModuleDraft`、`ValidationReport`、`ReviewReport` 是 C 私有模型，应定义在 `module/` 目录，不进入 `contracts/`。Part 2 的字段定义包含 Phase 2 目标字段（如 SanTrigger、Pregen、Asset、public_persona、stats、is_ending），这些在 `contracts/module.py` 中尚未实现。

---

## 〇、文档约定

本文档定义 Module Parser Agent 的**外部契约**——每个阶段的输入输出对象、ModuleContent 的字段语义、以及 Runtime 如何消费这些数据。内部实现（Prompt 设计、代码组织、校验算法）不在本文范围。

**术语对齐**：

| 术语 | 定义 |
|------|------|
| ModulePack | 数据库中的发布元数据（title, authors, version, players_min/max）。不直接被 Runtime 加载。 |
| ModuleContent | Runtime 实际加载的完整模组数据。包含 scenes, entities, checkpoints, win_conditions 等。 |
| ModuleDraft | Parser Pass 的原始输出。携带 source_references 和 confidence_notes。不可被 Runtime 加载。 |
| ValidationReport | Validation 阶段的产物。errors 阻断，warnings 提醒。确定性代码生成。 |
| ReviewReport | Review Pass 的产物。包含 A/B/C/D 覆盖度评估和 human_review_checklist。LLM 生成。 |
| ApprovedModule | 通过全部质量门（或显式接受 warnings）的 ModuleContent。版本化、不可变。 |

---

## Part 1: Stage IO Contract

### 1.1 Preprocess

| 属性 | 值 |
|------|-----|
| Purpose | 将原始模组文件转换为干净的、结构化分段的文本，为 Parser Pass 提供可追溯的输入。 |
| Input | `RawDocument`（PDF/Markdown/TXT 文件路径 + SHA256 checksum） |
| Output | `SourceFragment[]`（分段的文本块列表） |
| Owner | 确定性代码（PyMuPDF + 分段算法） |
| LLM | 否 |
| Failure Strategy | 阻断。PDF 无法解析 → 报告错误，停止流水线。 |
| Persisted | 否。SourceFragment 是流水线内部对象，Parser Pass 完成后可丢弃。 |
| Runtime Visible | 否 |

**SourceFragment 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 稳定可引用的片段 ID，格式 `src_{chapter}_{page}_{seq}` |
| locator | string | 人类可读的定位信息，如"第三章 / 第12页 / 第5段" |
| text | string | 段落原文 |
| section | "keeper_info" \| "player_info" \| "unclassified" | 原文区域分类 |

**设计说明**：SourceFragment 的 ID 必须稳定——同一份 PDF 两次运行的 ID 必须相同。这是来源追溯的基础。LLM 不能做分段，因为 LLM 的输出不可复现。

---

### 1.2 Parser Pass

| 属性 | 值 |
|------|-----|
| Purpose | 从模组原文中提取结构化数据。单次 LLM 调用，输出 ModuleDraft。 |
| Input | `SourceFragment[]` + skill catalog + hook catalog + op catalog |
| Output | `ModuleDraft`（携带 source_references + confidence_notes + unresolved_questions） |
| Owner | Module Parser Agent（LLM 调用） |
| LLM | **是**。PydanticAI Agent，model: Claude Opus 4。 |
| Failure Strategy | LLM 调用失败 → 重试（最多 2 次）。重试仍失败 → 降级：标记为 blocked，等待人工介入。Parser 输出不合法 JSON → Pydantic model_validate 拒绝，打回 LLM 重试。 |
| Persisted | 否。ModuleDraft 是候选结构，不应进入 Content Repository。 |
| Runtime Visible | **禁止。** ModuleDraft 未通过 Validation，绝不可被 Runtime 加载。 |

**ModuleDraft 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| draft | `ModuleContent` | Parser 提取的结构化内容 |
| source_references | `dict[str, str]` | 字段路径 → SourceFragment.id 映射 |
| confidence_notes | `dict[str, float]` | 字段路径 → 置信度（0.0-1.0） |
| unresolved_questions | `list[str]` | Parser 无法确定、需要人工介入的段落描述 |

**硬约束**：
1. 原文不含的信息不得凭空补造
2. `Entity.secrets` 和 `Entity.public_persona`（如果模型支持）必须分离
3. Parser 不得因为"这样更好玩"而补造 Rule 或结局

---

### 1.3 Validation

| 属性 | 值 |
|------|-----|
| Purpose | 发现确定性的错误。所有错误在数学上可证明——不需要"理解"模组，只检查数据结构。 |
| Input | `ModuleDraft.draft`（即 `ModuleContent` 对象或 dict） |
| Output | `ValidationReport`（errors 阻断 + warnings 警告 + status） |
| Owner | 确定性代码。无 LLM。 |
| LLM | 否 |
| Failure Strategy | errors 不为空 → status = `needs_revision` 或 `blocked`，打回 Parser 或人工修订。warnings 仅为提醒，不阻断。 |
| Persisted | 否。但 ValidationReport 可在 Review Pass 和 Human Approval 中引用。 |
| Runtime Visible | 否 |

**ValidationReport 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| status | `"pass"` \| `"needs_revision"` \| `"blocked"` | pass: 无 error；needs_revision: 有 error 但可修；blocked: 缺少关键信息无法修 |
| errors | `list[ValidationIssue]` | 阻断项。必须修复才能进入 Review Pass。 |
| warnings | `list[ValidationIssue]` | 提醒项。可接受，但需要显式记录。 |

**ValidationIssue 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| severity | `"error"` \| `"warning"` | 严重级别 |
| code | string | 稳定错误码，如 `"scene.ref.entity_not_found"`。用于统计回归。 |
| path | string | 出错字段路径，如 `"scenes[0].entity_ids[2]"` |
| message | string | 人类可读的错误描述 |

**三层校验**：

| 层级 | 检查内容 | 阻断？ | 示例 |
|------|---------|--------|------|
| L1 Schema | 字段类型、必填项、枚举值、额外字段拒绝 | 阻断 | `Entity.kind` 不在枚举中 |
| L2 Reference | 跨引用完整性：Scene→Entity、Checkpoint→target、Rule→State、WinCondition→State | 阻断 | Scene 引用了不存在的 entity_id |
| L3 Semantic | Expr 语法、State 引用完整性（未被引用的 key 应降级）、可达性、循环依赖 | 警告 | 柜子需要钥匙，钥匙在柜子里；WinCondition 不可达 |

---

### 1.4 Review Pass

| 属性 | 值 |
|------|-----|
| Purpose | 对已通过 Validation 的 ModuleDraft 进行语义审查——检测 Parser 无法自检的遗漏（B/C 类规则、秘密泄漏、过度结构化）。 |
| Input | `ModuleDraft` + 原始 `SourceFragment[]`（或原文） |
| Output | `ReviewReport`（errors + warnings + human_review_checklist + mechanism_abcd_coverage） |
| Owner | Module Parser Agent（LLM 调用） |
| LLM | **是**。PydanticAI Agent，model: Claude Sonnet 5。 |
| Failure Strategy | errors 不为空 → status = `needs_revision`，打回 Parser 或标记为待人工修订。LLM 调用失败 → 降级：标注"Review 未完成"，自动发布为 Layer 1。 |
| Persisted | 否。但 ReviewReport 应保留至 Human Approval 完成。 |
| Runtime Visible | 否 |

**ReviewReport 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| status | `"pass"` \| `"needs_revision"` \| `"blocked"` | 同 ValidationReport |
| errors | `list[ValidationIssue]` | 阻断项（如秘密泄漏到公开字段） |
| warnings | `list[ValidationIssue]` | 提醒项（如过度结构化） |
| human_review_checklist | `list[ChecklistItem]` | 需要人工逐条核查的 B/C 类遗漏提示 |
| mechanism_abcd_coverage | `dict[str, bool]` | A/B/C/D 四类机制是否都有示例 |

**ChecklistItem 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| category | `"A"` \| `"B"` \| `"C"` \| `"D"` \| `"hook_gap"` | 遗漏类型 |
| entity_id | string \| null | 关联的 Entity ID |
| question | string | 人工需要回答的问题，如"管家在玩家进入书房后是否必须主动开口？" |

---

### 1.5 Human Approval

| 属性 | 值 |
|------|-----|
| Purpose | 对 Review Pass 无法确定的问题做人工最终裁决。B/C 类遗漏 LLM 漏报率最高，需要人类的反事实想象能力兜底。 |
| Input | `ReviewReport`（特别是 human_review_checklist）+ 原始 `ModuleDraft` + 原文 |
| Output | Human Approval Decision：`批准` \| `打回` \| `有条件批准` |
| Owner | 人工审核员 |
| LLM | 否 |
| Failure Strategy | 打回 → 返回 Parser 或人工修订，重新跑 Validation + Review Pass。 |
| Persisted | 审批记录应持久化（谁、何时、批准了什么、接受了哪些 warnings）。 |
| Runtime Visible | 否（审批元数据不进入 Runtime） |

**质量分层**（Human Approval 是可选 Layer 3，不是发布前置条件）：

| 层级 | 触发条件 | 标注 | 体验 |
|------|---------|------|------|
| Layer 1 | Validation 通过 | "可运行" | 结构正确，能跑 |
| Layer 2 | + Review Pass 通过 | "AI 已审查" | B/C 遗漏率降低 |
| Layer 3 | + Human Approval 通过 | "人工认证" | 作者意图 100% 保留 |

---

### 1.6 Publish

| 属性 | 值 |
|------|-----|
| Purpose | 将通过质量门的 ModuleContent 版本化、冻结、交付给 Runtime。 |
| Input | `ModuleContent`（通过 Validation，可能也通过 Review Pass 和 Human Approval） + 质量标注 |
| Output | `ApprovedModule`（版本化的 ModuleContent + content_hash + 质量标注 + 冻结的 fixtures/evals） |
| Owner | 确定性代码 |
| LLM | 否 |
| Failure Strategy | 版本冲突（同 module_id + version 已存在）→ 拒绝，要求更新版本号。 |
| Persisted | **是。** ApprovedModule 存入 Content Repository，是 Runtime 加载的权威来源。 |
| Runtime Visible | **是。** 这是 Module Parser Agent 对外的唯一交付物。 |

**ApprovedModule 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| content | `ModuleContent` | 通过质量门的完整模组数据 |
| version | string | 语义版本号（如 "1.0.0"） |
| content_hash | string | `ModuleContent` 序列化后的 SHA256 |
| quality_level | `"layer_1"` \| `"layer_2"` \| `"layer_3"` | 通过的最高质量层级 |
| approved_by | string \| null | Layer 3 时为审批人标识，Layer 1/2 为 null |
| frozen_fixtures | `list[string]` | 随此版本冻结的 fixture 文件列表 |

---

### 1.7 Stage IO 总览

```text
C 私有（module/）                        B/C 共享（contracts/module.py）
─────────────────────────                ─────────────────────────────

RawDocument                   外部输入，只读
    ↓ Preprocess
SourceFragment[]              C 私有，Parse 完成后可丢弃
    ↓ Parser Pass
ModuleDraft                   C 私有，不可进入 Runtime
    ↓ Validation
ValidationReport              C 私有，确定性代码生成
    ↓ Review Pass
ReviewReport                  C 私有，LLM 生成
    ↓ Human Approval
Human Approval Decision       审批记录，持久化
    ↓ Publish
ApprovedModule                对外交付物，持久化，Runtime 可加载
    ↓
ModuleContent                 B/C 共享契约（contracts/module.py）
                              被 B 消费执行规则
```

---

## Part 2: ModuleContent Contract

### 2.1 Scene

**Purpose**：描述游戏中的一个空间位置。Runtime 通过 Scene 确定当前玩家可以看到什么。

**Required Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 唯一标识 |
| name | string | 场景名称 |
| content | string | 场景的文学描述。自由文本，LLM 可演绎。 |

**Optional Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| entity_ids | `list[string]` | 本场景包含的 Entity ID 列表 |
| checkpoint_ids | `list[string]` | 本场景可触发的 Checkpoint ID 列表 |

**Runtime Usage**：

- ContextAssembler 根据当前 `Character.location` 加载对应 Scene，组装 `TurnContext.visible_entities` 和 `TurnContext.checkpoint_options`
- Scene.content 进入 Planner 和 Narrator 的上下文，作为场景氛围描述

**Initialization**：不参与。Scene 是静态内容。

**References**：

| 引用 | 指向 | 用途 |
|------|------|------|
| entity_ids | Entity.id | 确定场景中有哪些实体可见 |
| checkpoint_ids | Checkpoint.id | 确定场景中有哪些动作可用 |

**Lifecycle**：

```text
Parser 创建 → Validation 校验引用完整性 → Publish 固化 → Runtime 加载
```

---

### 2.2 Entity

**Purpose**：描述游戏世界中的一个可交互对象——NPC、怪物、物品、线索、动物、场景物体。Entity 是内容层最核心的模型：Rule 挂在 Entity 上，EntityState 由 Entity.state 初始化。

**Required Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 唯一标识 |
| kind | `"npc"` \| `"monster"` \| `"item"` \| `"clue"` \| `"animal"` \| `"object"` | 实体类型 |
| name | string | 实体名称 |
| content | string | 玩家可见的表面描述。自由文本。 |

**Optional Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| public_persona | string \| null | NPC 的表面人设。🟢 可进入 Planner/Narrator 上下文。 |
| secrets | string \| null | NPC 的真实底牌或隐藏信息。🔴 绝不可进入 Planner/Narrator 上下文。 |
| stats | `StatBlock` \| null | NPC/怪物的属性块。可为 null（大部分 NPC 没有属性）。 |
| state | `dict[str, Primitive]` | D 类键的初始值。游戏开始时拷贝到 Room.entity_states。 |
| refuse_ops | `list[string]` | A 类：引擎必须拒绝的操作类型列表。 |
| rules | `list[Rule]` | B/C 类规则。`(hook, when, then, mode, priority)` 五元组。 |
| aliases | `list[string]` | 别名列表，用于 Intent 匹配（如 "书架" 匹配 "藏书架"）。 |

**Runtime Usage**：

- `content` 和 `public_persona` → 进入 Planner 上下文（可见实体描述）
- `secrets` → 仅引擎可读。玩家发现后由引擎写入 Event，Narrator 通过 Event 间接触达
- `state` → Room 初始化时拷贝到 `Room.entity_states`
- `refuse_ops` → 引擎在执行 Op 前校验
- `rules` → 引擎在每个 hook 上收集并求值
- `aliases` → IntentParser 匹配玩家输入中的别名

**Initialization**：

- **EntityState 初始化**：Room 创建时，引擎遍历 `ModuleContent.entities`，将每个 `Entity.state` 的键值拷贝到 `Room.entity_states[entity.id]`。
- **不参与 Character 初始化**。Character 的属性来自车卡过程，不来自 Entity。

**References**：

| 引用 | 指向 | 用途 |
|------|------|------|
| Rule.hook | World.hooks | 校验 hook 名是否在合法列表中 |
| Rule.when 中的 state path | Entity.state 的键空间 | 校验表达式引用的键是否存在 |

**Lifecycle**：

```text
Parser 创建 → Validation 校验（kind 枚举、state 键名、Rule 引用完整性）
→ Publish 固化 → Runtime: Room 初始化时 state → entity_states
```

---

### 2.3 Checkpoint

**Purpose**：描述一个可由玩家触发的技能检定——成功和失败各自产生什么事实、什么状态变更、什么叙事约束。Checkpoint 是 Intent → Engine 的关键桥梁。

**Required Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 唯一标识 |
| scene_id | string | 所属 Scene ID |
| skill | `SkillRef` | 检定的技能，可以是具体技能名（如 "侦查"）或类别引用（如 "@交涉"） |
| on_success | `Outcome` | 成功时的结果（narration_context 描述 + ops 列表） |
| on_fail | `Outcome` | 失败时的结果 |

**Optional Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| match_hint | string | Intent 匹配提示，如"调查书架、检查书架、搜索书架" |
| difficulty | `"regular"` \| `"hard"` \| `"extreme"` \| null | 难度等级。**可为 null**——表示由运行时软判据（roleplay_tier）决定。 |
| hidden | bool | 是否暗骰。默认 false。true → roll_mode 必须为 "auto"。 |
| roll_mode | `"auto"` \| `"prompt"` | "auto": 引擎静默掷骰；"prompt": 创建 PendingCheck 等待玩家交互。 |
| target_id | string | 检定的目标 Entity ID。用于校验 target 是否在场景中可见。 |
| priority | int | 场景内有多个候选 Checkpoint 时的排序依据。 |

**Runtime Usage**：

- `match_hint` → Planner 用来匹配合适的 Checkpoint
- `skill` / `difficulty` → CheckResolver 确定检定参数
- `on_success` / `on_fail` → 引擎根据检定结果执行对应 Outcome 中的 ops 和 narration_constraints
- `hidden` / `roll_mode` → 决定是否创建 PendingCheck

**Initialization**：不参与。Checkpoint 是静态内容，引擎根据 Intent 动态加载。

**References**：

| 引用 | 指向 | 用途 |
|------|------|------|
| scene_id | Scene.id | 确定 Checkpoint 归属哪个场景 |
| target_id | Entity.id | 确定检定目标实体 |
| skill | Skill.id 或 Category | 确定检定使用的技能 |

**Lifecycle**：

```text
Parser 创建 → Validation 校验（scene_id 存在、target_id 存在、difficulty 可空）
→ Publish 固化 → Runtime: Planner 匹配 → Engine 执行 → Narrator 接收 Outcome
```

---

### 2.4 SanTrigger

**Purpose**：描述一个 SAN 检定触发器——玩家在什么条件下、以什么方式、损失多少 SAN。CoC 7e 有 6 种 SAN 损失形态，SanTrigger 承载其结构化表达。

**Required Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 唯一标识 |
| kind | `"check"` \| `"flat"` \| `"direct"` \| `"max_reduce"` \| `"gain"` \| `"capped"` | SAN 损失形态 |
| loss | `SanExpr` | SAN 损失表达式，如 "0/1d6"（成功/失败）或 "1d4"（固定） |

**Optional Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| source_tag | string \| null | 累计封顶的分组键。kind="capped" 时必须提供。 |
| condition | string | 触发条件。自由文本，描述玩家什么行为触发此 SAN 检定。 |

**Runtime Usage**：

- SANManager 根据 `kind` 选择处理逻辑
- `loss` → 确定 SAN 损失的具体数值
- `source_tag` → LedgerEntry 累计封顶：同 source_tag 的多次 SAN 损失合计不超过上限
- `condition` → 自由文本，由 Planner 判断当前行为是否触发

**Initialization**：不参与。

**References**：无（不引用其他 Content 对象）。

**Lifecycle**：

```text
Parser 创建 → Validation 校验（kind 枚举、loss 格式）
→ Publish 固化 → Runtime: SANManager 在触发时加载
```

---

### 2.5 WinCondition

**Purpose**：描述一个结局条件——当特定状态满足时触发结局叙事。

**Required Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 唯一标识 |
| when | `Condition` | 触发条件。引用 entity_states 中的键。 |
| text | string | 结局文本。LLM 可演绎。 |

**Optional Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| is_ending | bool | 是否终局。默认 true。false 表示状态回滚（如"没救猫→被抓回房间→重来"）。 |

**Runtime Usage**：

- WinConditionEvaluator 在每个回合结束时求值 `when`
- `is_ending=true` → 触发结局，Room phase → "ended"
- `is_ending=false` → 触发状态回滚，不结束游戏
- `text` → Narrator 生成结局叙事

**Initialization**：不参与。

**References**：

| 引用 | 指向 | 用途 |
|------|------|------|
| when 中的 path | Entity.state 的键空间 | 校验表达式引用的键是否存在 |

**Lifecycle**：

```text
Parser 创建 → Validation 校验（when 引用的 state key 存在、is_ending 语义）
→ Publish 固化 → Runtime: WinConditionEvaluator 每回合求值
```

---

### 2.6 Pregen

**Purpose**：模组附带的预设角色卡。玩家可选择预设角色快速开始游戏。

**Required Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 唯一标识 |
| name | string | 角色名称 |
| attributes | `dict[str, int]` | 八大属性值 |
| skills | `dict[str, int]` | 技能值 |

**Optional Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| occupation | string \| null | 职业名称或引用 |
| equipment | `list[string]` \| null | 初始装备 |

**Runtime Usage**：Character 创建时作为模板填充初始属性、技能和装备。

**Initialization**：参与 Character 初始化。`Character.based_on_pregen_id` 指向 Pregen.id。

**References**：

| 引用 | 指向 | 用途 |
|------|------|------|
| occupation | Occupation.id | 职业引用 |

**Lifecycle**：

```text
Parser 创建 → Validation 校验（attributes 八大属性齐全、skills 键合法）
→ Publish 固化 → Runtime: 玩家选角色 → Character 初始化
```

---

### 2.7 Asset

**Purpose**：模组附带的资源文件引用——地图、图片、文字材料（Handout）。Asset 本身不包含二进制数据，只包含引用指针。

**Required Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | string | 唯一标识 |
| ref | string | 资源引用。指向 blob_assets 的 storage_key。 |

**Optional Fields**：

| 字段 | 类型 | 说明 |
|------|------|------|
| kind | `"image"` \| `"map"` \| `"handout"` \| `"other"` | 资源类型 |
| label | string \| null | 人类可读的标签，如"一楼地图""文字材料 1" |

**Runtime Usage**：按需通过 `ref` 加载二进制数据，投递给客户端展示。

**Initialization**：不参与。

**References**：

| 引用 | 指向 | 用途 |
|------|------|------|
| ref | blob_assets.storage_key | 获取实际文件 |

**Lifecycle**：

```text
Parser 创建 → Validation 校验（ref 格式合法）
→ Publish 固化 → Runtime: 客户端按需加载
```

---

## 特别回答

### 1. Runtime Keeper 如何消费 ModuleContent？

```text
ModuleContent 经 Publish 后成为 ApprovedModule，存入 Content Repository。

Room 创建时：
  Room.module_pack_id → 加载对应的 ApprovedModule.content

每回合开始时：
  ContextAssembler 根据 Character.location 定位 Scene
  → 遍历 Scene.entity_ids，组装 VisibleEntity 列表
  → 遍历 Scene.checkpoint_ids，组装 CheckpointOption 列表
  → 返回 TurnContext

引擎执行时：
  根据 Intent.check 加载对应的 Checkpoint
  → CheckResolver 执行检定
  → RuleEvaluator 收集 Entity.rules + World.world_rules，按 priority 求值
  → StateManager 校验 Op，写入 entity_states
  → EventLogger 写入 Event
  → WinConditionEvaluator 求值 WinCondition.when

Narrator 生成时：
  接收 ActionResult（含 confirmed_facts、player_visible_information、narration_constraints）
  → 生成自然语言叙事
```

### 2. Scene 如何驱动 Runtime？

```text
Scene 是 Runtime 的空间组织单元：

1. 玩家所在 Scene 决定了可见实体集合和可用动作集合
2. 场景切换：Intent.action_type = "move" → 引擎更新 Character.location
   → 下一回合 ContextAssembler 加载新 Scene
3. Scene 的出口关系（Scene.exits）决定了玩家可以去哪些场景
4. Scene.checkpoint_ids 决定了当前场景有哪些模组预设动作可用

注意：不需要全局场景图。并非所有模组都有严格的邻接约束——
《追书人》《鬼屋》《蛙蛙村》都是城市/园区内自由移动，exits 恒为空。
```

### 3. Checkpoint 如何推进剧情？

```text
Checkpoint 不直接推进剧情——它不生成叙事，不写入状态。

Checkpoint 提供的是"玩家做 X → 引擎判定 Y → 事实 Z 成立"的映射：

1. Planner 匹配 Checkpoint → 生成 Intent
2. Engine 根据 Intent.check 加载 Checkpoint → 执行检定
3. 检定结果（success/fail）→ 引擎执行对应 Outcome 中的 ops
4. ops 写入 entity_states → 产生 Event
5. Event 可能触发 B 类 Rule（hook 上的必然事件）
6. WinConditionEvaluator 读取 entity_states 判断结局

剧情推进是 Rule + Event + WinCondition 协作的结果。
Checkpoint 只是检定入口，不是剧情引擎。
```

### 4. EntityState 如何由 Entity 初始化？

```text
Room 创建时（SessionManager.create_room）：

1. 加载 ApprovedModule.content
2. 遍历 ModuleContent.entities
3. 对每个 Entity e：
     Room.entity_states[e.id] = deep_copy(e.state)
4. 之后所有 entity_states 变更由引擎执行（经校验的 Op）

注意：
- Entity.state 是 Content 层，只读，由 Parser 在导入时定义
- Room.entity_states 是 GameState 层，可变，仅引擎可写
- EntityState 的当前值是物化视图——可从 EventLog 重建
```

### 5. 哪些对象属于静态内容？哪些对象属于运行时状态？

```text
静态内容（Content 层，Parser 创建，Publish 固化，只读）：
  Scene, Entity, Checkpoint, SanTrigger, WinCondition, Pregen, Asset
  Rule（挂在 Entity.rules 或 World.world_rules 上）

运行时状态（GameState 层，引擎创建和修改）：
  Room.entity_states（从 Entity.state 初始化，后续由引擎修改）
  Character.attributes / skills / conditions / ledger / equipment / location
  Character.derived_stats（HP / SAN / MP / LUCK，引擎修改）
  Player.pending_check（引擎创建，玩家提交后引擎消费）

权威历史（EventLog 层，引擎写入，只增）：
  Event（每条 event 携带 type + payload + cause + visibility）
```

---

## Open Questions

1. **ModulePack 归属**：`ModulePack` 的 Pydantic Model 应在 `contracts/module.py` 中定义，还是作为数据库层的 ORM Model？当前仅 `ModuleContent` 存在，缺失 `ModulePack`。Parser Pass 提取的 title、authors、players_min/max 目前无 Pydantic Model 承载。

2. **Entity.public_persona 和 Entity.stats**：`contracts/module.py` 的 `EntitySpec` 中缺失。Phase 1 Demo NPC 无属性块可接受，Phase 2 需补充。

3. **SanTrigger、Pregen、Asset 模型缺失**：`contracts/module.py` 中完全无对应 Spec 定义。Phase 2 真实模组导入时需要补上。

4. **CheckpointSpec.difficulty 的可空性**：当前 `contracts/module.py` 的 difficulty 不可空（`Literal["regular", "hard", "extreme"]`）。数据模型要求可空。需 B 共同确认。

5. **RuleSpec.when / WinConditionSpec.when**：当前为 `ConditionSpec {path, equals}`。数据模型要求 `Expr`（完整布尔表达式）。Issue 1 后需 B 共同评审升级路径。

6. **contracts/module.py 修改审批**：`architecture.md` 规定 `contracts/module.py` 是 B/C 共同评审区——C 的任何字段修改需 B 审批。
