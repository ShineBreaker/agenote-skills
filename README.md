# agenote-skills — agenote 跨 Agent 经验平台 skills

> [agenote](https://github.com/ShineBreaker/agenote) 跨 Agent 经验平台的三个配套
> agent skills：基础知识库操作、自动策展、会话后经验采集。

本仓库作为 `Guix-configs` 的 Git 子模块嵌入，提供 agenote 系统的 agent 行为规范。

## Skills 清单

| Skill | 触发信号 | 职责 |
| ----- | -------- | ---- |
| `agenote-base` | 开始非平凡任务前 / 遇到踩过的坑 / 联网查到新方案 / 用户纠正 / 长任务结束 | 任务前查、过程中复用、结束时记的日常 KB 操作（list→search→get；add→touch；commit） |
| `agenote-curator` | 每周/长会话后例行维护 / 卡片 >50 张 / 检索质量下降 / 发现重复或矛盾 | KB 健康度维护：健康检查+去重+归档+权重重分配+reconcile 多源 memory |
| `agenote-review` | 完成信号检测 / 用户触发总结 / 长任务结束评估 / 用户纠正 / 排查 >2 步 | 会话后经验采集与留痕：经验信号识别 + ENTRY_TYPE 判定 + 留痕决策树 |

三者相互引用（"转 `agenote-curator`"等）是**提示 agent 按需切换 skill**，不是代码级依赖。
所有 skill 共享同一个外部依赖：[agenote](https://github.com/ShineBreaker/agenote) CLI。

## Skill 规范

每个 skill 是一个自包含目录，核心是 `SKILL.md`（YAML front matter + Markdown 正文）：

```
<skill-name>/
├── SKILL.md              # 必需：元数据 + 完整指令
└── references/           # 可选：参考文档子目录
    └── *.md / *.org
```

`SKILL.md` front matter 只有两个字段：

```yaml
---
name: <skill-name>
description: <功能描述，含触发信号，供 agent 框架匹配调度>
---
```

## 部署

### 作为 Guix-configs 子模块（主流程）

本仓库登记为 `Guix-configs` 的 `dotfiles/mutable/agenote/.config/agents/skills`
子模块，由 `blue stow` 统一纳管：

```bash
git submodule update --init dotfiles/mutable/agenote/.config/agents/skills
blue stow agenote           # 部署软链
blue stow --restow agenote  # 重建
```

部署后 skill 通过 stow 软链到 `~/.config/agents/skills/`，agent 框架（omp/zcode）
从 `~/.agents/skills/`（运行时 symlink 路径）扫描加载。

### 独立部署（原生 stow）

```bash
git clone https://github.com/ShineBreaker/agenote-skills.git ~/agenote-skills
stow --dir=~/agenote-skills --target=$HOME
```

## 依赖

- **[agenote](https://github.com/ShineBreaker/agenote) CLI**：所有 skill 通过
  `~/.local/bin/agenote` 执行卡片/记忆/策展操作，必须先安装 agenote CLI。

## 改源生效路径

skills 是纯 Markdown 文档，agent 框架每次会话扫描加载，**改源即生效**。

## 许可证

MIT，见 [LICENSE](LICENSE)。
