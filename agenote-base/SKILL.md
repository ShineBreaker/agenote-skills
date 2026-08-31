---
name: agenote-base
description: 跨 agent 共享知识库（agenote）：任务开始查经验、过程中复用、结束记录。触发：非平凡任务开工前 / 疑似踩过同坑 / 联网查到新方案 / 被用户纠正 / 长任务收尾——任一出现即调用，按内部规则 list→search→get、add→touch。KB 健康度维护与多 agent 对话 reconcile 转 agenote-curator；会话结束经验评估转 agenote-review。
---

# agenote — agent 专属记事本

agenote 是人类知识库（`~/Documents/Org/`）的**并行子集**，专为 AI agent 记录而设。数据隔离在 `~/Documents/Org/agenote/` 子目录，与人类卡片互不污染。

> **调用方式**：所有操作通过 CLI `agenote`（`~/.local/bin/agenote`，bash 直接调用），无 MCP 接口。以 `agenote` 开头的子命令（`add`、`search`、`list` 等）即为本 skill 的入口。`--domain human` 切到人类知识库根，不指定时 search 默认做跨域加权检索。

> **来源溯源**：agent 写入的卡片自动打 `:SOURCE_AGENT:` 标签（取自启动 env `AGENOTE_AGENT`，缺失回退 `pi`）。当前接入 pi/crush/opencode/hermes/zcode；新增 agent 在其启动环境设 `AGENOTE_AGENT=<name>` 即可纳入归因，`agenote health` 的 `by_source` 字段可看各 agent 写卡分布。外部 agent（codex/claude/omp 等）经 loopctl 调起时，通过其 adapter 的 `env` 块设置。

## 任务前预检

开始非平凡任务（调试/排障、配置修改、用过的技术栈、与之前类似的问题）前，先查已有经验——KB 的价值靠复用兑现，检索一次的成本远低于重新踩坑：

```
agenote list --category <相关类别> --all   # 先看领域标题索引
agenote get <明显相关的卡片ID>
agenote search "<技术 工具 症状>"           # 标题不足以定位时正文检索
agenote memory --project .                 # 当前项目记忆
```

全新领域开发、简单编辑、有明确文档的标准操作可跳过。结果处理：高相关 → 作为上下文；低相关/空 → 静默继续；矛盾 → 以较新/经验证的为准。

## 何时记录（场景 → 写法）

| 场景                                    | 写法                           |
| --------------------------------------- | ------------------------------ |
| 联网/文档查到并实际用到的方案、API 用法 | `add --entry note`             |
| 调试踩坑、被用户纠正、误判需求、走弯路  | `add --entry mistake`          |
| 多轮试错后找到的正确做法                | `add --entry ascended`         |
| 用户对 agent 工作方式的偏好（跨会话）   | `memory --add --type feedback` |
| 项目技术栈、构建命令、已知坑点          | `memory --add --type project`  |

不记录：纯浏览未采用的资料、临时调试输出、可从代码直接推导的信息、一次性任务细节——噪音会稀释检索质量。

## CLI 速查

### 检索

```
agenote list --category <类别> --all     # 按领域列出标题+元数据（检索首选）
agenote get <ID>                         # 读卡片（用 ID，不是 title）
agenote search "<关键词>" [--json]       # 跨域加权检索（人类域权重更高）
agenote search "<关键词>" --all-terms    # 要求全部词项命中（AND 语义）
agenote search "<关键词>" --context N --limit N
agenote search --regex "<正则>"
```

### 写入

```
agenote add --title "标题" --category <类别> --tech <技术栈> \
  --type <类型> --entry <note|mistake|ascended> --summary "一句话总结" --stdin <<EOF
** 任务描述
...
** 执行过程
...
** 关键发现
...
EOF
```

**type 门禁**：`--type` 只用正式 type（`agenote fields --type` 查看；种子 6 类 debug/refactor/research/workflow/feature/config 免检，其余需非归档 ≥10 张才晋升正式）；非正式 type 会被 CLI 拒绝，确需延续/新建才加 `--force`（新 type 持续写到 10 张后自动转正免检）。

### 管理

```
agenote init [--no-git]                  # 仅首次：创建目录结构 + git 仓库（--no-git 跳过 git）
agenote touch <ID>                        # 留痕（USAGE_COUNT+1）
agenote update <ID> --status done|stable|stale
agenote update <ID> --append-to "关键发现" --append-text "新发现"
agenote connect <A> <B> --desc "描述"     # 双向链接
agenote merge <primary> <sec>             # 合并卡片（secondary 自动归档）
agenote archive <ID> / agenote restore <ID>
agenote archive --list                    # 列出已归档（restore 前先在此找 ID）
agenote inbox "待捕获的想法"
agenote stats / agenote health / agenote fields
agenote lint [--fix] / agenote lint --json   # 后者输出分类结构化报告（可差分）
agenote reindex / agenote doctor
agenote commit -m "一句话总结"            # 提交 KB git 变更
```

完整参数取值见 [references/parameters.md](references/parameters.md)。

## 用 ID 而非 title 定位卡片

`get`/`touch`/`archive`/`update` 用 **ID 或文件名片段**匹配，不匹配 title。先用 `list` 或 `search` 找到卡片 ID（形如 `20260625-014305`）再操作。

## 留痕机制

查询资料后，对**实际用到**的部分留痕：已有卡片 `agenote touch <ID>` 递增 USAGE_COUNT；联网新知识 `add --entry note` 写卡留档，`--summary` 写"来源 + 核心结论一句话"，正文首行留 `来源: <URL/API>` 保留可回溯性。频繁使用的卡片 WEIGHT 自动提升（usage_count 参与权重公式，reindex 时重算），检索排名更靠前——留痕是提升下次命中率的手段，不是仪式。

## 记忆系统

四种类型：feedback（行为偏好）/ project（按项目拆分，存 `memories/projects/<项目>.org`）/ reference（可跨项目复用的参考）/ deprecated（陈旧归档）。

```
echo "正文" | agenote memory --add --type feedback --title "标题" --stdin
echo "正文" | agenote memory --add --type project --project <项目> --title "标题" --stdin
agenote memory                            # 概览
agenote memory --project .                # 当前项目记忆（含健康提示）
agenote memory --get                      # 全文
agenote memory --stale                    # 陈旧清单（只读）
agenote memory --touch F001 / agenote memory --archive F001
```

F/R 序号由 CLI 自动分配（feedback 记 F 序号入 MEMORY.org `* feedback` 节，reference 记 R 序号，project 追加到 `memories/projects/<name>.org`），无需手工管理。模型细节见 [references/memory-model.md](references/memory-model.md)。

## 可视化

`agenote viz -o out.html` 把 KB 渲染成可搜索的单文件 HTML（自带主题）；`--serve --port 8765` 起本地服务。用户说"用 md2html"或"把 KB 渲染出来看"时，先想 `agenote viz`。内容是单份 markdown 报告（非 KB 卡片）时退回 `pandoc -s <md> -o <html>`。

## 相关 skill

- KB 例行策展（健康度维护、去重归档、reconcile、dream 综合）→ `agenote-curator`
- 会话结束经验评估与留痕决策 → `agenote-review`

## 详细参考

- [卡片格式与字段](references/card-format.md)
- [记忆系统模型](references/memory-model.md)
- [ENTRY_TYPE 语义映射](references/entry-types.md)
- [参数取值表](references/parameters.md) — `--category`/`--tech`/`--type`/`--owner`/`--entry`/`--status` 等
- [Org 格式规范](references/markdown-to-org.md) — 代码块/强调/标题 Org vs Markdown 对照
- [体验卡片模板](references/experience-template.org) — 完整 Org 模板（Emacs org-capture 用）
