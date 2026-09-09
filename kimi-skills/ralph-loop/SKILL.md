---
name: ralph-loop
description: Use when a development task should run unattended to completion over many iterations — the user mentions Ralph 模式/自主循环/无人值守开发, wants a PRD or requirement list implemented story-by-story without supervision, or asks to set up prd.json / scripts/ralph in a project.
---

# Ralph Loop（自主循环开发）

## Overview

Ralph 是一个 bash 循环：每轮 spawn 一个全新 `kimi -p` 实例实现一个 user story，状态只存于 git 历史、`prd.json`、`progress.txt`。工具本体已长期部署在 `D:\cursor_file\ralph`（已改造支持 `--tool kimi`，内置 `bin/jq.exe` 兜底），**不要重新克隆或改造**，直接复用。

## When to Use

- 需求可拆成 3+ 个独立小 story，且每个都有可机器检查的验收标准（语法检查/测试/lint）
- 用户明确要"挂着跑""不用盯着""自动做完"
- 项目是 git 仓库，工作区可以先清理干净

**不要用**：单个小改动（直接改更快）；无反馈 loop 的项目（没有任何可跑的检查，错误会跨迭代复利）；验收必须靠人眼看的任务（Ralph 无浏览器工具）。

## Workflow

```bash
# 1. 前置：项目根目录工作区必须干净（有未提交改动先按项目惯例提交）
cd /d/<project> && git status

# 2. 部署（jq 不用拷，ralph.sh 内置 fallback）
mkdir -p scripts/ralph
cp /d/cursor_file/ralph/ralph.sh /d/cursor_file/ralph/KIMI.md scripts/ralph/

# 3. 写 scripts/ralph/prd.json（完整示例见 D:\cursor_file\ralph\prd.json.example）
#    branchName 用 ralph/<feature>；骨架：
#    {"project":"<name>", "branchName":"ralph/<feature>", "description":"...",
#     "userStories":[{"id":"US-001","title":"...","description":"...",
#       "acceptanceCriteria":["可机器检查的条件","node --check xxx.js 通过"],
#       "priority":1,"passes":false,"notes":""}]}

# 4. 提交部署基线，然后【后台】启动（循环常跑几十分钟）
git add scripts/ralph && git commit -m "chore: 部署 Ralph 循环与 <feature> PRD"
./scripts/ralph/ralph.sh --tool kimi <N>   # N ≈ story 数 × 2；ralph.sh 仅两个参数：--tool amp|claude|kimi 和轮数
```

在 Kimi Code 里启动方式：Bash 工具 `run_in_background=true` + `disable_timeout=true`；在普通终端则用 `nohup ./scripts/ralph/ralph.sh --tool kimi <N> > scripts/ralph/ralph-run.out 2>&1 &`。

**可安全中断**：循环被 Ctrl-C / 休眠打断不会丢状态（状态全在 git + prd.json + progress.txt），直接重跑同一命令从未完成的 story 续跑；某 story 反复失败会把剩余轮次烧空并以退出码 1 结束，此时看 `scripts/ralph/last-run.log` 找失败原因。

## prd.json 关键约束

- **story 必须小到能在一个上下文窗口内完成**——"加字段/加组件/加筛选"可以，"建整个 dashboard"必须拆
- 验收标准必须可机器检查：静态网页用 `node --check xxx.js`，Python 用 `python -m py_compile`，有测试写测试；不要写"看起来正常"这类无法验证的条目
- UI story 接受"需人工浏览器验证"作为收尾，由 KIMI.md 要求 agent 写进 progress.txt

## 判断跑完（三信号叠加，别只信文本）

1. 循环退出码 0 且输出 `Ralph completed all tasks!`（退出码 1 = 达到轮数上限，直接重跑同一命令可从未完成 story 续跑）
2. `/d/cursor_file/ralph/bin/jq.exe '.userStories[].passes' scripts/ralph/prd.json` 全 true
3. `git log --oneline` 上每个 story 一条 `feat: [US-xxx] - ...` 提交；`scripts/ralph/progress.txt` 有逐 story 记录（含 learnings 和"需人工浏览器验证"标注）

## Common Mistakes

| 错误 | 正确做法 |
|------|---------|
| 不在项目根目录启动 | KIMI.md 路径是 `scripts/ralph/...` 相对项目根，cwd 必须在项目根 |
| 工作区不干净就开跑 | 先提交存量改动，否则首轮 commit 混入无关文件 |
| 给 `kimi -p` 加 `--yolo` | 互斥会报错；`-p` 自带 auto 权限。也因此在不可信目录不要跑 Ralph |
| 用 `tee /dev/stderr` 观测输出 | Git Bash 后台会报错；ralph.sh 已改为 tee 到 `scripts/ralph/last-run.log` |
| 跑完直接合并分支 | 合并回 main 必须经用户确认；UI 改动先人工浏览器验证 |

已知未加固项（dsh 互审 2026-08-30）：完成判定仅文本 grep（用上面三信号人工兜底）、无每轮超时。长期无人值守高频使用前先加固 `ralph.sh`。
