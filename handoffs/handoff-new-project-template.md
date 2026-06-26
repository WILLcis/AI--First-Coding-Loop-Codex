# 接力包:把 AI-First Coding Loop 落地到任意新项目(完全通用模板)

> **给 Codex 的角色定位**:你是 harness 落地 agent。把 `https://github.com/WILLcis/AI--First-Coding-Loop-Codex` 提供的 AI-First 脚手架以**安全、可回滚、可自审**的方式合并进用户指定的 GitHub 仓库。
> 不破坏任何现有代码,**不直接合 main**——开分支 + draft PR,等用户 review 后再合。
>
> 本模板**完全通用**:项目身份与 LLM 选择都在检查点 0 由用户提供。

---

## 🛑 检查点 0(开场第一件事):问用户四件事

在做任何 git/clone/写文件操作**之前**,先问清楚:

1. **目标 GitHub 仓库 URL**(格式 `https://github.com/<org>/<repo>`)
2. **项目代号**(internal name,用于 PR 描述与 AGENTS.md 描述句)
3. **仓库现状**:
   - (A) 刚建好,基本空
   - (B) 已有代码——主要语言栈是什么?
4. **LLM 厂商 + 模型**:
   - 厂商:`anthropic` / `openai` / `deepseek` / `qwen` / `kimi` / `glm` / `baichuan` / `siliconflow` / `custom`
   - 全局默认模型字符串(例 `gpt-4o`、`deepseek-chat`、`claude-sonnet-4-6`)
   - (可选)旗舰模型,用于 verifier-security:例 `gpt-5.5`、`o1`、`deepseek-reasoner`、`claude-opus-4-6`

> ❗ 用户回答之前**不要**继续。这是反"自作主张"的第一道护栏。

---

## 1. 物料

- 这套脚手架仓库:`https://github.com/WILLcis/AI--First-Coding-Loop-Codex`
  - `core/` 项目无关 + 模型无关的核心(scripts/workflows/prompts/flags/state)
  - `codex/` skills + agents + AGENTS.md 模板
  - `tools/install.sh` 把 `core/` + `codex/` 装到目标仓
- 你可以 `git clone` 它到 `/tmp/aifcl`,然后跑 `bash /tmp/aifcl/tools/install.sh <用户的仓库目录>`

## 2. 任务流(带显式 🛑 检查点)

### 任务 A — 弄清新仓库的现状(读 + 不写)

1. `gh auth status` 确认登录,没登录 → 提示 `gh auth login`
2. `cd ~/projects`(或问用户偏好目录);`gh repo clone <用户给的 URL>`
3. 进入仓库,只读探查:
   ```bash
   ls -la
   cat README.md 2>/dev/null | head -100
   ls .github/workflows 2>/dev/null
   git log --oneline -20
   find . -maxdepth 3 \( -name 'package.json' -o -name 'go.mod' -o -name 'requirements*.txt' \
       -o -name 'pyproject.toml' -o -name 'Cargo.toml' -o -name 'pom.xml' \
       -o -name 'CMakeLists.txt' -o -name 'Gemfile' \) 2>/dev/null | head -20
   ```
4. 写"现状速写"(≤12 行):语言栈、目录骨架、是否有现成 CI、是否有 AGENTS.md。

**🛑 检查点 1 — 给用户看现状速写**,等用户确认或纠正。

### 任务 B — 决定合并策略

三选一,写理由:
- **策略 1:空仓** → 直接 `tools/install.sh` 铺到根
- **策略 2:有代码无 CI/AGENTS.md** → 主体直接装,**保留**用户原 README.md
- **策略 3:有现成 CI 或冲突** → 装到子目录 `.harness/`,渐进迁移;workflow 文件改名避撞

**🛑 检查点 2 — 给用户看策略 + 理由 + 期望影响**,等用户拍板。

### 任务 C — 在新分支上执行合并

```bash
git switch -c chore/bootstrap-ai-first-loop
git clone --depth 1 https://github.com/WILLcis/AI--First-Coding-Loop-Codex /tmp/aifcl
bash /tmp/aifcl/tools/install.sh "$PWD" --strategy <1|2|3>
```

**不要触碰**这些已有文件(若存在),除非用户明确同意:
- 现有 `README.md`(合并到末尾或单独 `HARNESS-README.md`)
- 现有 `.gitignore`(只 append 必要几行,不替换)
- 现有 `.github/workflows/*.yml`(改名避撞,告诉用户)
- 现有 `Makefile`、`Dockerfile`、`docker-compose.*`(同上)

### 任务 D — 项目化 + LLM 配置

#### D.1 改 `AGENTS.md`

替换占位:项目描述句、目录地图、本地起栈命令、安全禁区(用户检查点 0 给的项目代号 + README 内容)。

#### D.2 改 `ci.yml` 适配实际语言栈

把目标仓**实际不存在**的语言 job 注释掉(node-quality / go-quality / python-quality)。
若语言不在这三种里,新增一个 job,把 job 名加入 `ci-gate` 的 `needs`。

#### D.3 ⚠ 验证用户选的模型当前真的可用(关键)

按用户在检查点 0 选的厂商 + 模型,跑一次轻量列表 API 校验。
不可用 → 停下来问用户:用哪个替代?并在 PR 描述里**明确告诉**用户用了替代模型。

#### D.4 (可选)固定 Codex agent 模型

默认让 `.codex/agents/*.toml` 继承当前 Codex 会话模型,只保留 `model_reasoning_effort`。
如果用户明确要求把模型写入仓库历史,在对应 agent TOML 里添加 `model = "<用户选的模型>"`。

#### D.5 跑 5 项本地 sanity

```bash
python3 scripts/check_env_parity.py config/.env.dev.example config/.env.prod.example 2>&1 | tail -3
OBSERVABILITY_BACKEND=mock TRACKER=github-dryrun python3 scripts/triage_engine.py 2>&1 | tail -4
for p in quality security dependency; do
  python3 scripts/ai_review.py --pass $p --mock 2>&1 | head -2
done
LLM_PROVIDER=<用户选的> LLM_MODEL=<用户选的> OBSERVABILITY_BACKEND=mock python3 scripts/health_report.py >/dev/null 2>&1
python3 -c "
import json
r=json.loads(open('state/token-usage.jsonl').readlines()[-1])
assert r['provider']=='<用户选的厂商>', f'unexpected: {r}'
print('✓ token-usage provider=', r['provider'], 'model=', r['model'])
"
python3 -c "import yaml,glob; [print('OK',f) for f in sorted(glob.glob('.github/workflows/*.yml')) if yaml.safe_load(open(f))]"
```

**🛑 检查点 3 — 给用户看 5 项 sanity 的尾部输出**。全绿才往下。

### 任务 E — 用 gh CLI 配 GitHub Vars/Secrets

```bash
gh repo set-default <用户的仓库 URL>

read -sp "LLM API key (sk-...): " KEY; echo
gh secret set LLM_API_KEY -b"$KEY"

gh variable set LLM_PROVIDER -b"<用户选的厂商>"
gh variable set LLM_MODEL    -b"<用户选的默认模型>"

# per-role(可选,根据用户选择填)
gh variable set LLM_MODEL_EXPLORER             -b"<廉价档>"
gh variable set LLM_MODEL_IMPLEMENTER          -b"<主力档>"
gh variable set LLM_MODEL_VERIFIER_QUALITY     -b"<主力档>"
gh variable set LLM_MODEL_VERIFIER_SECURITY    -b"<旗舰档>"
gh variable set LLM_MODEL_VERIFIER_DEPENDENCY  -b"<廉价档>"
gh variable set LLM_MODEL_TRIAGE_SCORER        -b"<廉价档>"
gh variable set LLM_MODEL_CHECKER              -b"<与 IMPLEMENTER 不同>"
gh variable set MONTHLY_TOKEN_BUDGET           -b"50000000"

gh secret list
gh variable list
```

**🛑 检查点 4 — 给用户看 secret/variable list 输出**,确认完整。

### 任务 F — Commit + Push + 开 draft PR(不要合)

```bash
git add -A
git status
git commit -m "chore: bootstrap AI-First Coding Loop (harness)

引入 https://github.com/WILLcis/AI--First-Coding-Loop-Codex:
- core/ 拷到 scripts/ / .github/workflows/ / prompts/ / flags/ / state/
- codex/ skills 拷到 .agents/skills/,agents 拷到 .codex/agents/
- LLM: <用户选的厂商> / <用户选的模型>(切厂商只改 GitHub vars/secrets)
"
git push -u origin chore/bootstrap-ai-first-loop
gh pr create --draft \
  --title "[harness] bootstrap AI-First Coding Loop" \
  --body "<结构化描述,含改动清单 + LLM 配置 + 自审清单 + 怎么继续>"
```

**🛑 检查点 5 — 给用户:**
- PR URL
- 改了哪些文件、加了哪些、**没**碰哪些
- 5 项 sanity 输出
- GitHub Secrets/Vars 配置确认
- 建议用户哪些地方亲自看一眼(战略风险点)

### 任务 G — 报告回来

200 字内,先一句话定性,再列细节。**不要继续做 Day 1/Day 2**,等用户在 PR 上 review 后单独说"继续"。

---

## 3. 检查点总表

| # | 时机 | 给用户看 | 等用户 |
|---|---|---|---|
| **0** | 开场第一件事 | 4 个问题清单 | 必答 |
| 1 | 任务 A 完 | 现状速写 ≤12 行 | 确认/纠正 |
| 2 | 任务 B 完 | 合并策略 + 理由 | 拍板 |
| 3 | 任务 D.5 完 | 5 项 sanity 输出 | 全绿继续 |
| 4 | 任务 E 完 | secret/var 列表 | 确认完整 |
| 5 | 任务 F 完 | PR URL + 总结 | 用户自审 |

## 4. 反模式(绝对不做)

- ❌ 跳过检查点 0,自己猜项目 / 自己选 LLM
- ❌ 直接 push 到 main
- ❌ 覆盖用户已有的 README/Dockerfile/Makefile/CI
- ❌ 自作主张配 AWS / 合 PR / 改用户的分支保护
- ❌ `git push --force` 任何分支
- ❌ 让 `AGENTS.md` 留 `[占位]`
- ❌ 模型不可用时**默默回退**——必须显式告诉用户
- ❌ 把 `LLM_API_KEY` 写进代码、commit 进仓库、或贴在 PR 描述里

---

## 给用户的提示

把整份贴进 Codex 第一条消息,它会从 4 个项目身份问题开始问你,然后经过 5 个检查点把整套合到一个 draft PR,**最后让你自己 review 决定合不合**。
