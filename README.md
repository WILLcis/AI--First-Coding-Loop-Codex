# AI-First Coding Loop

> 可复用的 AI-First 工程脚手架。**项目无关、模型无关**。
> 灵感:CREAO 的 harness engineering + Addy Osmani《Loop Engineering》。

把任何新项目从「人在按提示」改造成「系统在按提示,人做架构与判断」——这个仓库提供完整的"骨架"。

---

## 5 秒说清楚是什么

```
你的项目 + 这套骨架 = harness
       harness + 几个回路(daily-health / triage / ai-review / goal-loop)= AI-First 工程
       AI-First 工程 + 一个架构师 = 5 人公司做 100 人的活
```

---

## 仓库结构(按"可复用粒度"分层)

```
AI--First-Coding-Loop-Codex/
├── core/                 ★ 项目无关 + 模型无关 ★ 拷进任何仓库都能跑
│   ├── scripts/             Python/Bash 脚本(自愈环 / 评审 / loop / token / perf / 反认知投降)
│   ├── workflows/           GitHub Actions YAML 模板(默认门禁 + 可选门禁)
│   ├── prompts/             4 份评审 + 任务 prompt(中文)
│   ├── flags/               特性开关封装(Statsig + LocalProvider)
│   └── state/               外置记忆的目录约定
│
├── codex/               🤖 Codex 专属(SKILL + custom agent TOML)
│   ├── skills/              7 个按域拆分的 skill(安装到 .agents/skills/)
│   ├── agents/              7 个 Codex custom agent TOML(安装到 .codex/agents/)
│   └── AGENTS.md.template   Codex 根上下文模板(拷到你的项目根)
│
├── handoffs/             📋 接力包模板(给 Codex 等"代你执行"的助手)
│   ├── README.md
│   ├── handoff-new-project-template.md   完全通用,项目+LLM 都未指定
│   └── examples/
│       └── handoff-openai-gpt5.5.md       示例:已预填 OpenAI / GPT-5.5
│
├── docs/                 📖 方法论与落地指南(中文)
│   ├── 实施手册.md                  完整方法论 + 角色 + 4 周迁移路线
│   ├── greenfield-实施指南.md       从零开始的 Day0→DayN 节奏
│   ├── Week-0-实施清单.md          可勾选的 5 天落地清单
│   ├── v2-升级说明.md              7 项 Loop Engineering 升级
│   └── 多模型适配.md               8 厂商预设 + 推荐分层组合
│
└── tools/                🛠 装配工具
    ├── install.sh           把 core/ + codex/ 拷到目标仓库的正确位置
    └── verify.sh            5 项 sanity 检查目标仓配齐
```

---

## 三件事它能给你

1. **CI 门禁 + 三趟 AI 评审(模型无关)** — 一天部署多次也安全
2. **每日自愈反馈环** — 错误自动检测 → 聚类 → 打分 → 建工单 → 修复后自动关单
3. **反"认知投降"护栏** — 三项硬指标盯着架构师有没有真的在思考

---

## 怎么用(四种姿势)

### 姿势 0:**Local-only**(起步 / 单人开发,$0 远端 API)

不付远端 API 钱,push 前在本地用 Codex 跑三趟评审:

```bash
# 在你 push 前
bash <(curl -sSL https://raw.githubusercontent.com/WILLcis/AI--First-Coding-Loop-Codex/main/core/scripts/local_review.sh) --combined --copy
# 输出已自动复制到剪贴板,贴进 Codex 一次跑完三趟
```

何时用:**单人开发、纯测试、不需要 required check**。详见 [`docs/local-vs-remote-review.md`](docs/local-vs-remote-review.md) 的 A/B/C 路线决策表。

### 姿势 1:**拷贝**(全套自愈环,适合做主力 harness 的仓库)

```bash
# 在你的目标仓库根目录
TARGET=$(pwd)
git clone https://github.com/WILLcis/AI--First-Coding-Loop-Codex /tmp/aifcl
bash /tmp/aifcl/tools/install.sh "$TARGET"
```

`install.sh` 会把 `core/` 内容铺到目标仓的对应位置:
- `core/scripts/` → `scripts/`
- `core/workflows/` → `.github/workflows/`
- `core/prompts/` → `prompts/`
- `core/flags/` → `flags/`
- `core/state/` → `state/`
- `codex/skills/` → `.agents/skills/`(Codex repo-scoped skills)
- `codex/agents/` → `.codex/agents/`(Codex project-scoped custom agents)
- `codex/AGENTS.md.template` → `AGENTS.md`(若不存在)

### 姿势 2:**git subtree**(想跟上游同步更新)

```bash
git subtree add --prefix=.harness https://github.com/WILLcis/AI--First-Coding-Loop-Codex main --squash
# 之后拉更新:
git subtree pull --prefix=.harness https://github.com/WILLcis/AI--First-Coding-Loop-Codex main --squash
```

### 姿势 3:**接力包**(交给 Codex 全自动落地)

把 `handoffs/handoff-new-project-template.md` 整份贴进 Codex 第一条消息(或 `handoffs/examples/handoff-openai-gpt5.5.md` 如果你想要 OpenAI/GPT-5.5 预设)。
Codex 会从"问你三个项目身份问题"开始,经过 5 个检查点把整套合并成一个 draft PR。

### 姿势 4:**Reusable Workflow**(★ 最快——4 行,零拷贝)

只想要 AI 评审、不要全套自愈环?在目标仓 `.github/workflows/pr-review.yml` 写:

```yaml
on: { pull_request: { branches: [main] } }
jobs:
  review:
    uses: WILLcis/AI--First-Coding-Loop-Codex/.github/workflows/ai-review-reusable.yml@main
    secrets: { LLM_API_KEY: ${{ secrets.LLM_API_KEY }} }
    with:   { provider: openai, model_default: gpt-5.5 }
```

**就这些**——下一个 PR 自动跑三趟评审,无需拷贝任何文件,本仓升级你 `@main` 改成 `@v2.5` 就跟上。
> 调用方仓需要在 Settings → Actions → General → Workflow permissions 选 "Read and write permissions",否则 reusable workflow 无法评论 PR。
完整参数、各厂商示例、与 install.sh 拷贝版的取舍,见 [`docs/reusable-workflows.md`](docs/reusable-workflows.md)。

---

## 切换 LLM 厂商(任意时刻,无需改代码)

只改 GitHub Repo Variables / Secrets:

```
vars.LLM_PROVIDER     = openai | anthropic | deepseek | qwen | kimi | glm | baichuan | siliconflow | custom
secrets.LLM_API_KEY   = 该厂商的 key
(可选)
vars.LLM_BASE_URL                 私有部署 / 未预设的厂商
vars.LLM_MODEL                     全局默认模型
vars.LLM_MODEL_VERIFIER_SECURITY   per-role 模型覆盖(其余 role 同理)
```

详见 [`docs/多模型适配.md`](docs/多模型适配.md)。

---

## 最重要的两件事(贴墙上)

1. **门禁不强制 = 没有门禁**。把 `ci-gate` 与 `ai-review-gate` 设为分支保护的 required check,且勾上 "Do not allow bypassing the above settings"——**连 admin 也不能绕过**。
2. **快 AI 没有快校验 = 高速堆积技术债**。先建测试与评审脚手架,再放大 agent 产能。

---

## 致谢

- CREAO 创始人/CTO 的 [AI-First 工程方法论原文](https://www.businessinsider.com)
- Addy Osmani《Loop Engineering》: <https://addyosmani.com/blog/loop-engineering/>
- Anthropic Claude / OpenAI / DeepSeek 等所有把"脚手架引擎化"变可能的模型厂商

License:MIT(添加 LICENSE 文件前请你决定)
