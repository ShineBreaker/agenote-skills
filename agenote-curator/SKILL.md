---
name: agenote-curator
description: 跨 agent KB 健康度维护。**触发信号**：每周/长会话后例行维护 / 卡片 >50 张 / 检索质量明显下降 / 用户触发 `/agenote-curate` / 发现重复卡片或矛盾结论 / 新增了对话抽取源或记忆源。**当上述任一信号出现时立即调用本 skill**——流程由你（agent）主导编排：健康检查、状态重整、去重归档、reconcile 多源 memory、agent 记忆库巡检导入与 dream 综合，CLI 只提供检测报告与原子命令。基础用法见 `agenote-base`；会话中单次经验记录见 `agenote-review`。
---

# agenote-curator — 跨 agent 自动策展

定期对 agenote 记事本执行策展，保持健康度并优化检索权重；同时支持从多个 AI 编程工具抽取对话、跨 agent reconcile 多源 memory。

> 全部操作通过 `agenote` CLI（`~/.local/bin/agenote`）完成，MCP 接口已移除。默认操作 agenote 子库（`~/Documents/Org/agenote/`），`--domain human` 切到人类知识库根。

## 何时策展

- 周期性维护（每周/每次长会话后）
- 条目数明显增多（>50 张）
- 检索结果质量下降（噪音多、相关性差）
- 新增了对话源或更新了 XDG 环境变量
- 用户主动要求 `/agenote-curate`

## 策展流程（agent 主导）

**分工原则**：CLI 只提供检测报告（`health` / `gaps` / `deduplicate` / `list --unused-days` / `archive --stale`）与原子写命令（`update` / `archive` / `merge` / `touch` / `connect`）；**流程编排与去留决策由你执行**。所有状态转换前先看候选清单、核实内容，再显式落地——CLI 已无一键策展命令。

### Step 0 — 提取对话（可选）

```bash
# 底层走 agenote/extract/ 多源抽取器；输出固定到 conversations/<date>/（--output-dir 可覆盖）
agenote extract --source all --date $(date -d yesterday +%Y-%m-%d)
```

### Step 1 — 诊断

```bash
agenote health --quality --duplicates   # 健康度 + 质量扫描 + 重复检测
agenote stats                           # 状态分布
agenote gaps --stale-days 60            # 类别×类型矩阵 + 缺失组合 + 陈旧卡片
```

指标解读见下方「健康度指标解读」。

### Step 2 — 状态重整（agent 审查后显式执行）

```bash
agenote list --unused-days 30 --all     # 降级候选（>30 天未使用，只读）
agenote get <id>                        # 抽查候选内容，判断是否真过时
agenote update <id> --status stale      # 确认过时 → 降级
agenote update <id> --status stable     # done >30 天且质量合格 → 转 stable
agenote touch <id>                      # 仍有效 → 刷新 LAST_VERIFIED
agenote archive --stale                 # 归档候选清单（stale 且 >90 天未验证，只读）
agenote archive <id...> --reason "策展: >90天未验证"   # 审查后批量归档
```

**逐项判断，不要看天数就批量降级**——LAST_USED 超期只说明没人用，语义是否过时要读卡判断。

### Step 3 — type 聚拢（agent 判断）

type 收敛需要语义理解，CLI 只设门禁（add/update 新 type 需 `--force`），聚拢决策由你执行：

1. `agenote fields --type` 查看各 type 计数；对可疑 type 用
   `agenote list --type <t> --all` 核实**非归档**卡片数
2. **非归档卡片数 <10 的 type 列为聚拢候选**（≥10 说明该 type 是真实需求，如
   `note` 之于 `workflow`——一个是查到的知识，一个是做法总结，保留不动作）
3. 候选 type 逐张审查（`agenote get <id>`），语义可归入已有大 type 的：
   `agenote update <id> --type <目标type>`（自动同步属性/标签/文件名/索引，
   目标 type 必须已存在，否则需 `--force`）
4. 全部卡片归走后该 type 自然消亡；**确需保留** = 候选内卡片有独立语义、与所有
   已有 type 都不重合且预期会持续增长（如 mistake/ascended 虽少但语义独特，
   也可考虑并入 debug——以内容审查为准）

### Step 4 — 去重与合并

```bash
agenote deduplicate                    # 检测重复对（标题相似度 + category/tech 加权，只读）
agenote get <id_a> && agenote get <id_b>   # 核实是否真重复
agenote merge <primary> <secondary> --desc "合并原因"   # secondary 自动归档
```

### Step 5 — 矛盾调和与传播联动

见下方「矛盾调和规则」「传播联动规则」。

### Step 6 — memory 维护

```bash
agenote memory --stale                  # 列出 >30 天未更新记忆（只读）
agenote memory --archive-to-file <F###> # 审查后逐条归档 feedback 到 MEMORY-ARCHIVE.org
agenote memory --project <name>         # 项目记忆检索 + PATH/UPDATED 健康提示（只读）
agenote memory --project-touch <name>   # 确认活跃的项目刷新 LAST_ACTIVE
```

### Step 7 — 跨 agent reconcile + dream 综合

**reconcile**：按 `agenote/extract/__init__.py` 的 `_resolve_extractors()` 注册顺序跑全部 source，结果写到 `.reconcile/index.json`（**写入层自动过滤元消息噪声**——TodoWrite / system-reminder / checkpoint 等源自 harness 注入而非用户经验的内容，判据见 `agenote.core.is_noise_fact`）。首次跑新 source 先 `--dry-run` 试跑。

**dream 综合**（Agent 综合阶段，从 reconcile 事实提炼新 KB 卡片）：

`agenote dream` 返回 ≤`limit` 个候选（默认 5，可调）。每个候选含：

- `term`：触发候选的高频关键词
- `frequency`：该词在窗口内出现的次数（df）
- `score`：综合质量评分（IDF × √df × 形态学权重，越大越值得沉淀）
- `representative_title` + `representative_content`：代表事实（词密度最高那条）
- `source_trace`：溯源指针（= reconcile fact id，**调 `agenote trace` 用**）
- `suggested_category`：映射后的 kb category
- `source_facts`：贡献该词的事实 id 列表

**dream 不自动写 KB**——综合决策交给 agent（读候选 → 判断 → `agenote add`）：

1. 读 dream 候选的 `representative_content`（**索引层摘要，已截断**——见下方溯源）
2. **需深入判断时调 `agenote trace --id <candidate.source_trace>` 读完整原始对话**
   （含工具调用/推理/补丁，索引层这些都丢失了）。token 经济性：按需展开，不要无差别调
3. 判断该主题是否值得沉淀为 KB 卡片（应用下方"策展原则"的应记录/不应记录判据）
4. 值得 → `agenote add --title ... --entry note --category <suggested_category> --stdin`，正文引用 `source_facts` 中的 ID 以保留溯源
5. 已被现有 KB 卡片覆盖 → 跳过（或 `agenote touch <已有ID>` 标记复用）
6. 矛盾/已被取代 → 按"矛盾调和规则"处理旧卡

**为什么需要 trace**：reconcile 索引层 `content` 是 extractor 建索引时的**截断摘要**
（opencode/zcode：user 截 1000 字、assistant 截 2000 字、tool/patch 退化为 `[tool: name]` 标记）。
要判断一个 dream 候选是否真的有具体经验价值，必须读真实完整对话——这正是
`agenote trace` 的职责。dream 候选的 `score` 反映“统计上像经验词”，`agenote trace` 让你
确认"语义上确实是有用经验"。

**dream 参数**（与 `agenote dream` CLI 同源）：

- `window_days`：时间窗口（天），默认 90 覆盖 ~90% facts。0=不过滤。`0/7/30/90/180`
  窗口在真实数据上分别幸存 100%/7%/37%/93%/100% facts
- `offset`：跳过前 N 个候选（多轮抽取跳过噪声词；前 5 个没用就调 offset=5 继续）。
  **注意 offset 不稳定**：候选排序随 reconcile 索引更新漂移，`report.snapshot_hash`
  标识本次候选集指纹——两次调用指纹不同即说明排序已变，同一 offset 可能指向不同候选
- `limit`：本次最多返回 N 个候选（默认 5）
- 无 timestamp 的 fact（hermes 30 条）**默认保留**，不受窗口影响

**评分算法（IDF × √df × 形态学，TF 作 tie-breaker）**：

- IDF = log(total_facts / df)：稀有词天然高分
- √df：温和补偿高频好词，不至于被大量 df=MIN_TERM_FREQ 长尾淹没
- 形态学权重：含 `-`/`_` 的代码标识符 +2.0（CJK 二字虚词 ×0.4 降权）
- TF（tie-breaker，不进主评分）：score 并列时 TF 高的候选优先，消除随机排序。
  每个候选 dict 含 `tf_total` 字段供参考
- 实测 top 候选稳定为高质量项目标识符（guix-configs / self-improving / host-spawn 等），
  对话虚词（removed / understand / 让我）已被 IDF + 形态学 + 停用词表三重压制

**token 经济**：dream 候选 ≤limit 条，每条只暴露代表事实正文（截断版）；agent 不需读
全部 reconcile 事实。要深入按需调 trace。
**质量门槛**：score 高 ≠ 值得沉淀。优先综合 `score` 高**且** `representative_content`
含具体技术细节的候选；纯流程性/工具名词性的候选丢弃。

### Step 8 — agent 记忆库巡检与导入（scan-memories）

各 agent 的持久记忆库是**已提炼的高密度经验**，与 reconcile（读会话对话）互补，本步读记忆文件本体：

```bash
agenote scan-memories --json          # 全源只读扫描（含记忆全文，供逐条审查）
agenote scan-memories --source zcode  # 单源扫描
```

六个源，路径解析一律 env 优先 → config.toml `[memories.sources]` → XDG 占位符 → 默认目录：

| 源      | env                     | 默认根                        | 布局                          |
| ------ | ----------------------- | ----------------------------- | ----------------------------- |
| zcode  | `ZCODE_MEMORIES_DIR`    | `~/.zcode/cli/memories`       | projects/\<slug\>/memory/*.md |
| claude | `CLAUDE_CONFIG_DIR`     | `~/.claude`                   | projects/\<slug\>/memory/*.md |
| codex  | `CODEX_HOME`            | `~/.codex`                    | memories/**/*.md              |
| pi     | `PI_CODING_AGENT_DIR`   | `$XDG_CONFIG_HOME/omp`        | agent/memory/**/*.md          |
| reasonix | `REASONIX_HOME`       | `$XDG_DATA_HOME/reasonix`     | projects/\<slug\>/memory/*.md |
| hermes | `HERMES_HOME`           | `$XDG_DATA_HOME/hermes`       | memories/*.md（`§` 分节）     |

**逐条评估后显式导入，scan-memories 本身不写 KB**：

1. 读条目 `name` / `description` / `body`，按上方「策展原则」判断沉淀价值
2. 查重：`agenote search <关键词>`——KB 已覆盖 → `agenote touch <已有ID>` 标记复用，不重复导入
3. 分流导入（正文保留来源溯源：agent 名 + 原路径 + 原 name）：
   - 技术经验 / 踩坑 → `agenote add --title <结论式标题> --category <语义映射> --entry note --stdin`
   - 涉及错误教训的 → `--entry mistake`
   - agent 工作偏好 / 用户纠正（`type=feedback`）→ `agenote memory --add --type feedback --title ... --stdin`
4. **不导入**：单项目进度流水账（`type=project` 的状态类）、agent 人格画像（hermes 的 SOUL 类条目）、临时会话状态

**导入上限**：单轮策展导入 ≤10 条，超出触发 Andon（知识库膨胀）暂停待人工决策。

### Step 9 — 重整与提交

```bash
agenote reindex        # 重建索引；WEIGHT 随之按 usage/新鲜度公式重算（见「权重机制」）
agenote lint --fix     # 格式自动修复（语义问题报告，人工判断）
```

> **commit 是不可省略的收尾步骤**。用 `agenote commit` 封装 git add+commit。
> 历史教训：2026-07-07 之前多轮策展因 skill 写了不存在的 `agenote commit` 而从未真正提交，积累大量未跟踪/已修改文件。该子命令现已实现（D1 修复，2026-07-07），**默认精准 add 策展产物**（experiences/index.json/conversations/kb-viz.html），不会误吞无关文件（如其他仓库的同步改动）。

```bash
# 仓库根自动解析：agenote commit 内部用 git rev-parse --show-toplevel 找真实根
#   （ctx.root 如 ~/Documents/Org/agenote/ 是子目录，.git 在 ~/Documents/Org/）

# 1) 先看清楚本次策展涉及的改动
cd ~/Documents/Org && git status --short

# 2) 一键提交（默认只 add 策展产物，遵循仓库 commit.gpgsign 配置自动签名）
agenote commit -m "策展: (agenote) <一句话总结: 新增 K 张 / 更新 M 张 / reconcile N 条 / dream P 候选>"

#    cron 等无 pinentry 场景加 --no-gpg-sign；提交全部变更（含非策展文件）加 --all
#    agenote commit --no-gpg-sign -m "..."
#    agenote commit --all -m "..."
```

commit message 要求：以 `策展:` 前缀开头，50 字以内总结核心操作（新增 K 张 / 更新 M 张 / 晋升 P 条）。无变更时跳过（agenote commit 会提示"没有待提交的变更"）。

**预览路径**：`agenote commit -m "..." --dry-run` 先预览将提交的文件清单，去掉 `--dry-run` 真提交。

**阶段拆分规则**：若本轮同时有「遗留未提交改动」和「本次新策展产物」，应**分两个 commit**（先提交遗留，再提交本轮），不混在一起，保持历史可读。

### Step 10 — 输出报告

格式见下方「报告格式」。

## 策展原则

**应记录**：用户偏好和纠正、非显而易见的 bug（排查 >2 步）、环境特定陷阱、更好的方案。

**不应记录**：任务进度和 TODO、语法错误和拼写修正、纯流水账、已有经验完全覆盖的情况。

**维护理念**：过时卡片是负担不是资产、矛盾必须处理、卡片之间应有链接。

## 矛盾调和规则

| 矛盾类型           | 处理                              |
| ------------------ | --------------------------------- |
| 同条件相反结论     | 保留较新的，旧卡标注「已过时」    |
| 不同条件分别成立   | 互相引用，注明各自适用场景        |
| 旧方案被取代       | 旧卡标注「已被 \<新卡ID\> 取代」  |
| pattern 与卡片矛盾 | 修补 pattern，标注修订原因        |
| 歧义未决           | 双方标注 `(存疑)`，待下次策展验证 |

## 传播联动规则

- 新卡片推翻旧结论 → `agenote update` 在旧卡追加勘误
- 同类卡片 ≥3 张 → 晋升为 pattern
- 新卡片补充了已有 pattern → 修补 pattern
- 隐含关联的卡片 → `agenote connect` 建立双向链接
- pattern 引用的卡片已过时 → 标注 pattern 并引用新卡片

## Andon 机制（策展暂停）

后台策展过程中发现以下情况时，暂停自动流程并输出报告，等待人工决策：

- **严重矛盾** — 同一主题 ≥2 张卡片结论相反，且无法按调和规则自动处理
- **知识库膨胀** — 单次策展新增卡片 >10 张，怀疑提取策略过于激进
- **模式失效** — 已有 pattern 被 ≥2 张新卡片推翻
- **画像冲突** — 用户画像中的偏好与近期卡片记录严重不一致

暂停时输出：问题描述 + 涉及的卡片/模式 ID + 建议的三种处理方式。

## 报告格式

```
═══ 策展报告 ═══
日期：YYYY-MM-DD
提取对话：N 条
巡检 agent 记忆库：N 条（zcode x / claude y / codex z / pi w / reasonix v / hermes u，导入 i 条）
新增经验卡片：K 张（列出标题）
更新已有卡片：M 张（列出标题和更新原因）
晋升为 pattern：P 条（列出标题）
修补 pattern：Q 条
矛盾调和：C 个（列出涉及的卡片）
── MEMORY 统计 ──
feedback: X 条（Y stale）
project: X 个（Y 路径失效）
reference: X 条
deprecated: X 条
```

## 权重机制（spec §7）

检索时 `agenote search` 跨域扫描人类（weight 默认 1.5）+ agent（weight 默认 1.0）+ reconcile（weight 0.6-0.7）卡片，最终分数 = 原始相关度 × WEIGHT。

**WEIGHT 是派生值**：索引层（`_card_dict`）按以下公式计算，`reindex` / `touch` 等任何索引重建路径自动重算，卡片文件中的 WEIGHT 属性已废弃（新卡不再写入）：

```
WEIGHT = 基础权重 × 使用系数 × 新鲜度系数

基础权重:   人类=1.5, agent=1.0, reconcile=0.6-0.7
使用系数:   1 + 0.1 × min(USAGE_COUNT, 10)   # 常用的提升，上限 +1.0
新鲜度系数: last_used 超 STALE_DAYS(30天) → 0.8，否则 1.0
```

**reconcile 权重梯度**：

- hermes / pi：0.7（自家或信任度高）
- opencode / crush / codex / claude：0.6（外部源，略低避免淹没 KB）

## 数据源

抽取器实现在 `agenote/extract/`，源列表与路径以代码为准（当前 7 源：opencode/crush/codex/claude/pi/hermes/zcode）。新增源参考任一现有 extractor 的签名 `() -> tuple[list[ReconciledFact], list[str]]`，在 `agenote/extract/__init__.py` 的 `_resolve_extractors()` 注册即可。

**三重只读保护**（所有 extractor 共享）：SQLite `mode=ro` + `PRAGMA query_only=1` + 仅 SELECT；JSONL 仅读；文件缺失返回 `([], [msg])` 不抛异常。

**记忆库扫描源**（`agenote scan-memories`，实现于 `agenote/memscan.py`）：读各 agent 持久记忆文件（区别于上述会话源），6 源注册于 `memscan.SOURCES`，根目录解析走 `resolve_xdg_path` 同一套 env > config.toml `[memories.sources]` > XDG > 默认 优先级。目录式源跳过 MEMORY.md 索引；frontmatter 宽容解析；同样纯只读。

## 手动维护命令（CLI）

```bash
agenote health                            # KB 健康度
agenote deduplicate                       # 只检测重复
agenote list --unused-days 30 --all       # 降级候选（只读）
agenote archive --stale                   # 归档候选清单（只读）
agenote archive --list                    # 列出已归档
agenote restore <ID>                      # 恢复归档卡片
agenote reindex                           # 重建索引（WEIGHT 一并重算）
agenote memory --stale                    # 陈旧记忆清单（只读）
agenote scan-memories                     # 扫描各 agent 记忆库（只读）
agenote extract --source all --dry-run    # 抽取原始对话为 Org 文件（不落盘）
agenote extract --source claude --date 2026-06-29   # 指定源 + 日期
agenote trace --id opencode:ses_x:msg_y   # 回查 dream 候选的完整原始对话（不截断）
```

## 跨 agent 工作流（6 个子命令）

参考 MiMoCode Memory/Dream/Distill 体系。**纯只读/启发式，不调 LLM**。⚠ 与旧 MCP 接口不同：CLI 的 `extract`/`reconcile` **默认真写盘**，首次跑新 source 先加 `--dry-run` 试跑：

```bash
# 1. 抽取：从各 AI 工具的原始对话 → Org 文件（~/Documents/Org/conversations/<date>/）
agenote extract --source all --dry-run               # 全源 dry-run 预览
agenote extract --source opencode --date 2026-06-29  # 单源 + 指定日期
agenote extract --source all                         # 实际写入磁盘

# 2. reconcile：把外部 agent memory 索引到 .reconcile/index.json（让 agenote search 能搜到）
agenote reconcile --source hermes                    # 单源
agenote reconcile --source all --dry-run             # 全源 dry-run 预览
agenote reconcile --source all                       # 实际落盘

# 2.5 scan-memories：只读扫描各 agent 记忆库（不同于 reconcile 读会话，见 Step 8）
agenote scan-memories --json                         # 全源（含记忆全文）
agenote scan-memories --source zcode                 # 单源

# 3. dream：从 reconcile 事实启发式提炼候选新卡片（IDF × √df × 形态学评分）
agenote dream --window-days 90 --limit 5             # 默认参数
agenote dream --offset 5 --limit 5                   # 多轮抽取，跳过前 5
agenote dream --window-days 7                        # 聚焦近 7 天

# 4. trace：回查 dream 候选的原始完整对话（溯源，不截断）
agenote trace --id opencode:ses_xxx:msg_yyy          # 完整对话含 tool/patch
agenote trace --id pi:{uuid}:{msg_id}                # pi 同样支持
agenote trace --id hermes:23                         # 未实现则降级返回摘要

# 5. distill：把 KB 中反复使用的模式聚成候选工作流清单（纯只读；skill 草稿由 agent 撰写）
agenote distill
```

### dream 评估工作流

```bash
# 第一轮：取 top 候选
agenote dream --limit 5 --json
# 逐条看 score/term/frequency；score<30 直接跳过
# 需深入判断：agenote trace --id <source_trace> 读完整对话
# 决定：
#   写 KB      → agenote add ...（正文引用 source_facts 保留溯源）
#   复用已有   → agenote touch <现有卡片ID>
#   其余       → 跳过 / 矛盾调和

# 第二轮：前 5 个都不满意就跳页
agenote dream --offset 5 --limit 5
```

### reconcile 行为细节

- **三重只读**：每个 extractor 内部 SQLite mode=ro + query_only + 仅 SELECT
- **KB 优先**：reconcile 抽到的事实若与 KB 已有卡片同名，跳过不索引
- **去重**：同一事实可能从多个 DB 出现（如 crush 全局 + bind-mount 项目级），按 `id` 去重保留先出现的
- **0-fact 警告**：extractor 跑通但抽不到任何事实（数据未生成 / schema 漂移）→ 自动报 `[warn]`
- **不写回源**：绝不写回任何外部 agent 的原始数据（DB/JSONL 只读）
- **不污染 KB**：reconcile 事实进 `.reconcile/index.json`（独立目录），不进 `experiences/`

### dream / distill / trace 行为

- **dream**：找 reconcile 事实里高频出现、KB 未覆盖的主题 → 返回候选清单（含代表事实
  正文 + source_trace 溯源指针）。**绝不自动写 KB**；综合决策由 agent 主导（见 Step 7）。
  **零候选即成功**。参数：`--window-days`（默认 90）、`--offset`（默认 0）、`--limit`（默认 5）。
  评分：IDF × √df × 形态学权重。
- **trace**：按 `fact_id` 从原始 DB 回查完整对话（不截断），含工具调用/推理/补丁。
  三重只读保护。dream 候选的 `source_trace` 字段就是 `fact_id`。未实现 trace_session 的
  source（hermes/crush/codex/claude）降级返回索引层摘要。
- **distill**：把 KB 里 `type=ascended`/`usage_count≥2` 的卡片按 category+tech 聚类 →
  候选工作流清单（含源卡片 id，纯只读不落盘）。**零候选即成功**；值得沉淀的候选由
  agent 评估源卡片后自行撰写 SKILL.md。

## 健康度指标解读

| 指标           | 阈值              | 含义                                                                |
| -------------- | ----------------- | ------------------------------------------------------------------- |
| 孤立率         | <15% ✅ / <25% ⚠️ | 无 `[[file:]]` 链接的卡片占比                                       |
| 过时率         | <10% ✅ / <20% ⚠️ | stale 状态卡片占比                                                  |
| 类型偏斜       | <45% ✅           | 单一 type 占比过高                                                  |
| 薄弱类别       | ≥3 ✅             | 每个类别至少 3 张卡片                                               |
| by_source      | 各 agent 分布     | `by_source.counts`：每 agent 写卡数；`unknown` 列表暴露未登记 agent |
| 卡片增长率     | 月增 <20 张 ✅    | 超标的放慢提取频率                                                  |
| 矛盾未决数     | 0 ✅              | 必须处理，触发 Andon                                                |
| pattern 覆盖率 | ≥30% ✅           | 有对应 pattern 的经验占比                                           |

## 旧卡片审查清单

- 内容已过时 → 归档或标记过期
- 标题不包含结论 → 重命名
- 纯流水账 → 降级或删除
- 高度重复 → 合并保留更好的
- 与同类卡片矛盾 → 按调和规则处理
- 缺少时效性标注但涉及版本/API → 补充验证日期
- 无其他卡片/pattern 引用 → 评估补充关联或归档

## 何时不策展

- 单条经验刚写入 → 等 1 周积累 USAGE_COUNT 后再 curate
- 跨 agent 数据未跑通（schema 漂移）→ 先跑 `agenote extract --source <新源> --dry-run` 验证 schema
- KB 总卡片 < 30 → curate 收益小，可手动维护

## 策展完成检查清单

- [ ] `agenote reindex` 已执行
- [ ] `agenote lint --fix` 无残留错误
- [ ] **`agenote commit -m "策展: ..."` 已执行（强制，不可省略）**——封装 git add+commit，见「Step 9」
- [ ] 新增卡片元数据完整（category/tech/type/owner）
- [ ] 新增卡片含任务描述、执行过程、关键发现
- [ ] 代码块使用 Org mode 格式
- [ ] 仅提交本次策展涉及的文件

## 详细参考

- [策展工作流与质量标准](references/curation-guide.md) — 完整流程细节、矛盾调和、Andon 机制、报告格式
