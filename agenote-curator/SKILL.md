---
name: agenote-curator
description: agenote KB 例行策展与健康度维护。触发：每周/长会话后 / 卡片 >50 张 / 检索质量下降 / `/agenote-curate` / 发现重复或矛盾卡片 / 新增对话抽取源或记忆源——任一出现即调用。流程由 agent 主导编排（诊断、状态重整、去重归档、reconcile、agent 记忆库巡检导入、dream 综合），CLI 只提供检测报告与原子命令。基础用法见 agenote-base；会话中单次经验记录见 agenote-review。
---

# agenote-curator — 跨 agent 自动策展

定期对 agenote 记事本执行策展，保持健康度并优化检索权重；同时支持从多个 AI 编程工具抽取对话、跨 agent reconcile 多源 memory。

> 全部操作通过 `agenote` CLI（`~/.local/bin/agenote`）完成。默认操作 agenote 子库（`~/Documents/Org/agenote/`），`--domain human` 切到人类知识库根。

## 何时策展

- 周期性维护（每周/每次长会话后）
- 条目数明显增多（>50 张）
- 检索结果质量下降（噪音多、相关性差）
- 新增了对话源或更新了 XDG 环境变量
- 用户主动要求 `/agenote-curate`

## 策展流程（agent 主导）

**分工原则**：CLI 只提供检测报告与原子写命令（`health` / `gaps` / `deduplicate` / `update` / `archive` / `merge` / `touch` / `connect`）；**流程编排与去留决策由你执行**。所有状态转换前先看候选清单、核实内容，再显式落地——CLI 已无一键策展命令。

### Step 0 — 提取对话（可选）

```bash
# 底层走 agenote/extract/ 多源抽取器；输出固定到 conversations/<date>/（--output-dir 可覆盖）
agenote extract --source all --date $(date -d yesterday +%Y-%m-%d)
```

输入侧顺带检查 `~/Documents/Org/inbox.org`：如有点子/待办，评估是否转为正式卡片。

### Step 1 — 诊断

```bash
agenote health --quality --duplicates   # 健康度 + 质量扫描 + 重复检测
agenote stats                           # 状态分布
agenote gaps --stale-days 60            # 类别×类型矩阵 + 缺失组合 + 陈旧卡片
```

指标含义、阈值与红灯组合的解读见 [references/health-assessment.md](references/health-assessment.md)（含修复动作对照表）。

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

type 的正式性是动态的：**正式 type = 种子 6 类 ∪ 非归档卡片数达晋升阈值
（config `curation.type_promote_min`，默认 10）的 type**。CLI 门禁按同一口径实时
计算——非正式 type 写入（add/update `--type`）会被拒绝，`--force` 逃生；聚拢决策由你执行：

1. `agenote fields --type` 查看各 type 计数；对可疑 type 用 `agenote list --type <t> --all` 核实**非归档**卡片数
2. **非正式（非归档 <10 且非种子）的 type 列为聚拢候选**；≥10 已自动晋升正式（如 `note` 之于 `workflow`——一个是查到的知识，一个是做法总结），保留不动作
3. 候选 type 逐张审查（`agenote get <id>`），语义可归入已有大 type 的：`agenote update <id> --type <目标type>`（自动同步属性/标签/文件名/索引；目标 type 须为正式 type，否则需 `--force`）
4. 全部卡片归走后该 type 自动失去正式性（门禁随即拦截它）；**确需保留** = 候选内卡片有独立语义、预期会持续增长到晋升阈值以上（延续写入需逐张 `--force` 直到满 10 张自动转正——以内容审查为准）

### Step 4 — 去重与合并

```bash
agenote deduplicate                    # 检测重复对（标题相似度 + category/tech 加权，只读）
agenote get <id_a> && agenote get <id_b>   # 核实是否真重复
agenote merge <primary> <secondary> --desc "合并原因"   # secondary 自动归档
```

### Step 4.5 — KB→skill 晋升（WikiSkill 管道，agent 判断）

审查 mistake/ascended 卡片时，挑出**同一主题证据 ≥3 次**的候选晋升为 skill（对应 `correction-funnel` Step 5.5；那边的触发端每次会话生效，本步是策展时的批量执行端）：

1. **先读审计簿** `~/.local/share/hermes/skills/skill-impact.md`——候选方向如有被拒记录（Rejected 条目），提案必须回应拒绝原因，不得原样重提
2. 每轮策展最多晋升 **1 个** skill（原子提案，防批量改动无法归因）；候选多于 1 个时挑 Evidence 最厚的
3. skill 本体按 `skill-authoring` §9 分类树放置（`<category>/<skill-name>/`，保证 skills_list 可发现、skill_view 可加载）；SKILL.md frontmatter 后加 `来源: agenote <卡片ID>` 反向链接
4. 用户过目拍板 → 落地 → 登记审计簿（Accepted/Rejected 一行，格式见簿头）
5. 源卡片 `--status done` + 正文补 `→ 已晋升为 skill <名字> (日期)`

零候选（无卡片达 3 次门槛）即跳过本步，属正常。

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

**reconcile**：按注册顺序跑全部 source，结果写到 `.reconcile/index.json`（写入层自动过滤 harness 注入的元消息噪声）。首次跑新 source 先 `--dry-run` 试跑。行为细节（三重只读、KB 优先、按 id 去重、0-fact 警告）见 [references/curation-guide.md](references/curation-guide.md)「reconcile 行为细节」。

```bash
agenote reconcile --source all [--dry-run]
```

**dream 综合**（Agent 综合阶段，从 reconcile 事实提炼新 KB 卡片）——`agenote dream` 返回 ≤limit 个候选，每个含 `term` / `frequency` / `score` / `representative_title` / `representative_content` / `source_trace` / `suggested_category` / `source_facts`。**dream 不自动写 KB**，综合决策流程：

1. 读 dream 候选的 `representative_content`（**索引层摘要，已截断**）
2. 需深入判断时调 `agenote trace --id <candidate.source_trace>` 读完整原始对话（含工具调用/推理/补丁；未实现 trace_session 的源——hermes/crush/codex/claude——降级返回索引层摘要，看到的内容与 dream 候选同级）。token 经济性：按需展开，不要无差别调——dream 的 `score` 反映"统计上像经验词"，trace 让你确认"语义上确实是有用经验"
3. 应用下方「策展原则」判断是否值得沉淀
4. 值得 → `agenote add --title ... --entry note --category <suggested_category> --stdin`，正文引用 `source_facts` 中的 ID 以保留溯源
5. 已被现有 KB 卡片覆盖 → 跳过（或 `agenote touch <已有ID>` 标记复用）
6. 矛盾/已被取代 → 按「矛盾调和规则」处理旧卡

评分算法（IDF × √df × 形态学）、参数语义（`--window-days` / `--offset` / `--limit` 与 offset 不稳定性）、两轮评估示例与质量门槛见 [references/curation-guide.md](references/curation-guide.md)「dream 综合机制」。**零候选即成功**。

```bash
agenote dream --window-days 90 --limit 5 [--json]
agenote trace --id "<source_trace>"
```

### Step 7.5 — distill：KB→候选工作流清单（纯只读）

`agenote distill` 把 KB 中 `type=ascended` / `usage_count≥2` 的卡片按 category+tech 聚类 → 输出候选工作流清单（含源卡片 id），纯只读不落盘。**零候选即成功**。值得沉淀的候选由你评估源卡片后自行撰写 SKILL.md（与 Step 4.5 的晋升管道衔接：distill 给候选，Step 4.5 管提案与审计）。

```bash
agenote distill
```

### Step 8 — agent 记忆库巡检与导入（scan-memories）

各 agent 的持久记忆库是**已提炼的高密度经验**，与 reconcile（读会话对话）互补，本步读记忆文件本体：

```bash
agenote scan-memories --json          # 全源只读扫描（含记忆全文，供逐条审查）
agenote scan-memories --source zcode  # 单源扫描
```

六个源（zcode / claude / codex / pi / reasonix / hermes），路径解析 env 优先 → config.toml → XDG → 默认目录，完整源表与布局见 [references/curation-guide.md](references/curation-guide.md)「数据源注册」。

**逐条评估后显式导入，scan-memories 本身不写 KB**：

1. 读条目 `name` / `description` / `body`，按「策展原则」判断沉淀价值
2. 查重：`agenote search <关键词>`——KB 已覆盖 → `agenote touch <已有ID>` 标记复用，不重复导入
3. 分流导入（正文保留来源溯源：agent 名 + 原路径 + 原 name）：
   - 技术经验 / 踩坑 → `agenote add --title <结论式标题> --category <语义映射> --entry note --stdin`
   - 涉及错误教训的 → `--entry mistake`
   - agent 工作偏好 / 用户纠正（`type=feedback`）→ `agenote memory --add --type feedback --title ... --stdin`
4. **不导入**：单项目进度流水账（`type=project` 的状态类）、agent 人格画像（hermes 的 SOUL 类条目）、临时会话状态

**导入上限**：单轮策展导入 ≤10 条，超出触发 Andon（知识库膨胀）暂停待人工决策。

### Step 9 — 重整与提交

```bash
agenote reindex        # 重建索引；WEIGHT 随之按 usage/新鲜度公式重算（公式见 references/curation-guide.md「权重机制」）
agenote lint --fix     # 格式自动修复（语义问题报告，人工判断）
```

> **commit 是不可省略的收尾步骤**——没有它，策展成果只存在于工作树，下次故障就丢。用 `agenote commit` 封装 git add+commit，默认精准 add 策展产物（experiences/index.json/conversations/kb-viz.html），不会误吞无关文件。

```bash
# 仓库根自动解析：agenote commit 内部用 git rev-parse --show-toplevel 找真实根
#   （ctx.root 如 ~/Documents/Org/agenote/ 是子目录，.git 在 ~/Documents/Org/）

# 1) 先看清楚本次策展涉及的改动
cd ~/Documents/Org && git status --short

# 2) 预览将提交的文件清单，去掉 --dry-run 真提交
agenote commit -m "策展: <一句话总结: 新增 K 张 / 更新 M 张 / reconcile N 条 / dream P 候选>" --dry-run

#    cron 等无 pinentry 场景加 --no-gpg-sign；提交全部变更（含非策展文件）加 --all
```

commit message 以 `策展:` 前缀开头，50 字以内总结核心操作。无变更时跳过（agenote commit 会提示）。

**阶段拆分**：若本轮同时有「遗留未提交改动」和「本次新策展产物」，分两个 commit（先提交遗留，再提交本轮），保持历史可读。

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
- pattern 证据 ≥3 次且跨任务可执行 → 走 Step 4.5 晋升为 skill
- 新卡片补充了已有 pattern → 修补 pattern
- 隐含关联的卡片 → `agenote connect` 建立双向链接
- pattern 引用的卡片已过时 → 标注 pattern 并引用新卡片

`patterns.org` 纯人工维护（无 CLI 子命令），检查/修补 pattern 时直接 grep 读取：`grep -A5 "<关键词>" ~/Documents/Org/patterns.org`。

## Andon 机制（策展暂停）

后台策展过程中发现以下情况时，暂停自动流程并输出报告，等待人工决策——这些情况说明自动流程的判断依据已失效，继续跑只会放大错误：

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
- [ ] `agenote commit -m "策展: ..."` 已执行（封装 git add+commit，见 Step 9）
- [ ] 新增卡片元数据完整（category/tech/type/owner）
- [ ] 新增卡片含任务描述、执行过程、关键发现
- [ ] 代码块使用 Org mode 格式
- [ ] 仅提交本次策展涉及的文件

## 详细参考

- [策展参考手册](references/curation-guide.md) — 卡片质量标准、dream/reconcile/权重机制细节、数据源注册
- [健康度评估范式](references/health-assessment.md) — 指标解读、红灯组合信号、体检报告骨架
