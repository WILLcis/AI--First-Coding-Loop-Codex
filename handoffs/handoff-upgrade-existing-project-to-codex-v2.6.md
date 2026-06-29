# 接力包:把进行中的项目升级到 Codex v2.6 harness

> **给 Codex 的角色定位**:你是 harness 升级 agent。你的任务是把一个**已经在开发中的真实项目**升级到 `https://github.com/WILLcis/AI--First-Coding-Loop-Codex` 的最新 Codex harness,同时保护现有业务代码、CI、发布节奏和团队工作流。
>
> 核心原则:**不直接合 main,不开大爆炸改造,不一次性强制所有新门禁。** 先做可回滚 draft PR,让用户 review 后再逐步启用。

---

## 检查点 0:开场先问 6 件事

在任何 clone / 写文件 / 改 workflow 之前,先问用户:

1. **目标仓库 URL 或本地路径**
2. **当前项目阶段**:开发中 / 已上线 / 有生产用户 / 有发布冻结期
3. **当前 harness 状态**:
   - A. 没装过 harness
   - B. 已装旧 Codex harness
   - C. 已装 CC / Claude Code 版 harness
   - D. 不确定
4. **当前 CI/CD 关键检查**:哪些 check 已经是 required?哪些 workflow 不能动?
5. **本次升级范围**:
   - 最小升级:只更新 skills / AGENTS / reusable workflow 修复
   - 标准升级:加 v2.4 coding discipline + PR 模板 + reusable/self-review 修复
   - 完整升级:再加入 optional gates(perf / image / secret),但先不设 required
   - local-first 升级:再加 `scripts/local_review.sh`,push 前本地评审,不配远端 API key
   - parallel 升级:再加 v2.6 并行编排 skills/agents/state,用于多子任务 fan-out/fan-in
6. **LLM 配置**:继续用现有 `LLM_PROVIDER` / `LLM_API_KEY`,还是要切厂商/模型?

> 用户回答前不要继续。已有项目最危险的不是少做一步,而是猜错现状。

---

## 1. 物料

- 新 harness 仓库:`https://github.com/WILLcis/AI--First-Coding-Loop-Codex`
- Codex 原生落点:
  - `codex/skills/` -> `.agents/skills/`
  - `codex/agents/` -> `.codex/agents/`
  - `codex/AGENTS.md.template` -> `AGENTS.md`
- v2.6 重点:
  - `.agents/skills/agent-coding-discipline/SKILL.md`
  - `.agents/skills/task-decomposer/SKILL.md`
  - `.agents/skills/parallel-orchestrator/SKILL.md`
  - `.codex/agents/subtask-implementer.toml`
  - `.codex/agents/merger.toml`
  - `state/orchestration/README.md`
  - `.github/pull_request_template.md`
  - reusable workflow 的 `LLM_API_KEY` 可选 + `github.token` 修复
  - optional gates:`perf-gate` / `image-scan` / `secret-scan`
  - `secret-scan` / `image-scan` 的 SARIF 上传 best-effort 修复
  - `scripts/local_review.sh` + `docs/local-vs-remote-review.md`

---

## 2. 任务流

### 任务 A — 只读盘点现状

进入目标仓库后只读探查:

```bash
git status --short --branch
git remote -v
git log --oneline -10
find . -maxdepth 3 \( -name AGENTS.md -o -name CLAUDE.md -o -path './.agents/skills/*' -o -path './.codex/agents/*' -o -path './.github/workflows/*' \) 2>/dev/null | sort
ls -la
ls .github/workflows 2>/dev/null || true
```

写一份"升级风险速写"(≤15 行):
- 现有 harness 版本和路径
- 现有 CI/workflow 名称
- 是否已有 `AGENTS.md` / `CLAUDE.md`
- 是否已有 PR 模板
- 哪些文件可能冲突
- 推荐升级范围

**检查点 1 — 给用户看速写,等确认。**

### 任务 B — 选升级策略

按现状三选一:

- **策略 1:就地标准升级**
  适合已有 Codex harness 或基本无冲突项目。更新 `.agents/skills/`、`.codex/agents/`、`AGENTS.md` 片段、workflow 模板。

- **策略 2:并存迁移**
  适合已有 CC 版 harness 或 CI 较复杂项目。保留旧文件,新增 Codex 路径,把 `CLAUDE.md` 内容迁到 `AGENTS.md`,旧路径只在 PR 描述里标注待清理。

- **策略 3:隔离安装到 `.harness/`**
  适合生产高风险或 workflow 冲突严重项目。先把 harness 放 `.harness/`,只接入 PR 模板和 AGENTS 更新,后续单独 PR 迁移 workflow。

**检查点 2 — 说明策略、会改哪些文件、不会动哪些文件,等用户拍板。**

### 任务 C — 开升级分支

```bash
git switch -c chore/upgrade-codex-harness-v2.6
rm -rf /tmp/aifcl
git clone --depth 1 --branch v2.6 https://github.com/WILLcis/AI--First-Coding-Loop-Codex /tmp/aifcl
```

如果工作区已有未提交改动,不要继续;先让用户决定 commit / stash / 换 worktree。

### 任务 D — 安装或合并文件

#### D.1 标准安装

```bash
bash /tmp/aifcl/tools/install.sh "$PWD" --strategy 2
```

安装器遇到已有且不同的文件会跳过并提示。**不要用 `cp -rf` 覆盖。**

#### D.2 手工合并重点文件

必须人工合并而不是盲覆盖:

- `AGENTS.md`:保留项目真实命令、目录、约束;追加 v2.4 coding discipline 与 v2.6 并行优先原则
- `.github/workflows/*.yml`:若已有同名 workflow,对比后决定 rename 或局部迁移
- `.github/pull_request_template.md`:若已有模板,把 pre-submit checklist 合进去
- `.gitignore`:只 append harness 必要项

#### D.2.1 v2.6 并行编排合并点

如果目标项目已经有旧 Codex harness,重点确认这些文件被新增或更新:

```text
.agents/skills/task-decomposer/SKILL.md
.agents/skills/parallel-orchestrator/SKILL.md
.codex/agents/subtask-implementer.toml
.codex/agents/merger.toml
state/orchestration/README.md
```

同时在 `AGENTS.md` 增加规则:

- 主 session 收到需求后先判断是否可并行分解
- 可并行任务先产出 `state/orchestration/<task-id>/decomposition.json`
- 并行子任务必须有 `scope` / `no_touch` / `verifies`
- `subtask-implementer` 不能递归 spawn 子 agent
- `merger` 只做 merge + 验证,遇到冲突或测试失败立即停

#### D.3 CC 版迁移到 Codex 版

如果发现旧路径:

```text
.claude/skills/ -> .agents/skills/
.claude/agents/ -> .codex/agents/
CLAUDE.md -> AGENTS.md
```

不要立刻删除旧路径,除非用户明确同意。默认在 PR 里列为"下一步清理"。

### 任务 E — 渐进启用新门禁

已有项目不要一次性把所有新 gate 设成 required。

本 PR 推荐状态:

| Gate | 本 PR 是否加入 | 是否 required |
|---|---|---|
| `ci-gate` | 保持现状 | 保持现状 |
| `ai-review-gate` | 可更新 / 可接入 | 若原本 required 才保持 required |
| `secret-scan-gate` | 建议加入 | 先不 required,跑绿后再单独启用 |
| `image-scan-gate` | 可加入 | 先不 required |
| `perf-gate` | 只有已有 k6 scenario 时加入 | 先不 required |

如果用户选择完整升级,可以添加:

```bash
mkdir -p scripts/perf-scenarios
cp /tmp/aifcl/core/scripts/perf-scenarios/example.js scripts/perf-scenarios/example.js
cp /tmp/aifcl/core/scripts/perf_check.py scripts/perf_check.py
```

但必须在 PR 描述里写清楚:`PERF_TARGET_URL` 未配置前不要把 `perf-gate` 设 required。

v2.6 里 `secret-scan` 与 `image-scan` 都应满足:

- SARIF 上传 `continue-on-error: true`
- 仓库没开 code scanning 时不误报"发现密钥/CVE"
- gate 只根据扫描器实际结果判定

### 任务 E.5 — 选择 Local-only 还是 Remote review

如果用户是单人开发、测试期、或暂时不需要 required check,优先启用 v2.6 local-first:

```bash
bash scripts/local_review.sh --combined --copy
```

让用户把生成的 prompt 贴进 Codex。
如果用户需要多人协作、required check 或审计 log,再配置 reusable workflow + `LLM_API_KEY`。
把选择记录到 PR 描述里,不要让用户同时付本地和远端两份 review 成本。

### 任务 F — 本地 sanity

先装依赖到临时 venv,避免污染系统 Python:

```bash
python3 -m venv /tmp/aifcl-verify-venv
/tmp/aifcl-verify-venv/bin/pip install -q -r scripts/requirements.txt
PATH=/tmp/aifcl-verify-venv/bin:$PATH bash /tmp/aifcl/tools/verify.sh
```

再做 targeted checks:

```bash
python3 - <<'PY'
import tomllib, glob
for f in sorted(glob.glob('.codex/agents/*.toml')):
    d=tomllib.load(open(f,'rb'))
    for key in ('name','description','developer_instructions'):
        assert d.get(key), f'{f}: missing {key}'
print('Codex agent TOML OK')
PY

python3 - <<'PY'
import yaml, glob
for f in sorted(glob.glob('.github/workflows/*.yml')):
    yaml.safe_load(open(f))
print('workflow YAML OK')
PY
```

如果加了 perf:

```bash
python3 - <<'PY'
import json
summary={'metrics':{
  'http_req_duration': {'values': {'p(95)': 120, 'p(99)': 180}},
  'http_reqs': {'values': {'rate': 10}},
  'http_req_failed': {'values': {'rate': 0}},
}}
open('/tmp/k6-summary.json','w').write(json.dumps(summary))
PY
python3 scripts/perf_check.py --scenario smoke --summary /tmp/k6-summary.json --update-baseline
git checkout -- state/perf-baseline.json 2>/dev/null || true
```

**检查点 3 — 给用户看 sanity 输出。红灯就停,不要继续 commit。**

### 任务 G — 配 GitHub Vars / Secrets(只补缺)

先看现有:

```bash
gh secret list
gh variable list
```

缺什么补什么:

```bash
gh secret set LLM_API_KEY
gh variable set LLM_PROVIDER -b"<provider>"
gh variable set LLM_MODEL -b"<model>"
```

不要在 PR 描述里贴 key。
不要擅自改分支保护;只在 PR 描述里建议用户后续启用哪些 required check。

**检查点 4 — 给用户看 secret/var 名称列表,不显示值。**

### 任务 H — Commit + draft PR

```bash
git status
git diff --stat
git add -A
git diff --staged | rg -i "(password|secret|api[_-]?key|token|sk-[A-Za-z0-9])" || true
git commit -m "chore: upgrade Codex harness to v2.6"
git push -u origin chore/upgrade-codex-harness-v2.6
gh pr create --draft \
  --title "[harness] upgrade Codex harness to v2.6" \
  --body-file /tmp/harness-upgrade-pr.md
```

PR body 必须包含:
- 升级策略
- 改动清单
- 没碰的文件/系统
- sanity 输出摘要
- 新增但未 required 的 gates
- 用户合并前必须看的风险点
- 合并后建议分步启用计划

**检查点 5 — 返回 PR URL,然后停止。不要继续改分支保护或合并。**

---

## 3. 推荐 PR 描述骨架

```md
## 这次升级做了什么
把当前项目升级到 Codex harness v2.6。

## 改动范围
- 新增/更新 `.agents/skills/`
- 新增/更新 `.codex/agents/`
- 更新 `AGENTS.md` 的 coding discipline 段落
- 更新/新增 PR 模板
- 更新 AI review reusable workflow 配置
- 新增 local-first review 脚本,可在 push 前本地生成三趟评审 prompt
- 新增 v2.6 并行编排:task-decomposer / parallel-orchestrator / subtask-implementer / merger / state/orchestration
- 可选:新增 perf/image/secret gates,但未设 required

## 没做什么
- 没改业务逻辑
- 没直接改 main
- 没开启新的 required check
- 没删除旧 harness 路径(如有)

## 验证
- `tools/verify.sh`: PASS
- Codex agent TOML: PASS
- workflow YAML: PASS
- perf_check smoke: PASS / 未启用

## 合并后建议
1. 观察 1-2 个 PR 的 self-review / ai-review 输出
2. 选一个可拆任务试跑 v2.6 decomposition,确认并行边界是否合适
3. secret-scan 连续绿后设为 required
4. image-scan 噪音处理完后设为 required
5. perf-gate 写真实 scenario + 建 baseline 后再设 required
6. 单独 PR 清理旧 `.claude/` / `CLAUDE.md` 路径(如有)
```

---

## 4. 反模式

- 不问现状就直接跑安装器
- 用 `cp -rf` 覆盖已有 workflow / AGENTS / README
- 在同一个 PR 里改业务代码
- 一次性把 perf/image/secret 全设 required
- 删除旧 `.claude/` 或 `CLAUDE.md` 而没有迁移确认
- 把 API key 写进文件、PR 描述或聊天记录
- sanity 红了还继续 commit

---

## 给用户的用法

把整份贴给 Codex,或让 Codex 读取此文件:

```text
请读取 handoffs/handoff-upgrade-existing-project-to-codex-v2.6.md,
按它从检查点 0 开始,把我这个进行中的项目升级到最新 Codex harness。
每个检查点必须停下来等我确认。
```
