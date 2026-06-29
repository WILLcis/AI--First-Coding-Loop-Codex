# Reusable Workflow:在别人的仓库里"一行调用"Codex 评审

> **先看一眼**:你是不是真的需要远端跑?如果只是**单人测试 / 不要 required check**,
> 用 [`local_review.sh`](../core/scripts/local_review.sh) 在本地跑、吃本地 Codex 会员额度,
> 完全不需要 GitHub Actions secret。详见 [`local-vs-remote-review.md`](local-vs-remote-review.md)。
> 下面这套是给"多人协作 / 要 required check / 要审计 log"场景。

AI--First-Coding-Loop-Codex 暴露了一个**GitHub Actions 可复用工作流**(`workflow_call`):
任何 repo 在自己的 `.github/workflows/` 里写几行 `uses:` 就能拿到全套三趟 Codex 评审,
不需要拷贝文件,未来升级只需改 `ref`。

---

## 1. 调用方写法(最简版)

在你的目标仓 `<your-repo>/.github/workflows/pr-review.yml`:

```yaml
name: PR Review
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, ready_for_review]

jobs:
  review:
    uses: WILLcis/AI--First-Coding-Loop-Codex/.github/workflows/ai-review-reusable.yml@v2.6.2
    secrets:
      CODEX_ACCESS_TOKEN: ${{ secrets.CODEX_ACCESS_TOKEN }}
    with:
      ref: v2.6.2
```

**就这些**。下一个 PR 自动跑三趟评审。

> 调用方仓需要在 Settings → Actions → General → Workflow permissions 选 "Read and write permissions",否则 reusable workflow 无法评论 PR。

---

## 2. 完整参数

```yaml
jobs:
  review:
    uses: WILLcis/AI--First-Coding-Loop-Codex/.github/workflows/ai-review-reusable.yml@v2.6.2
    secrets:
      CODEX_ACCESS_TOKEN: ${{ secrets.CODEX_ACCESS_TOKEN }}
    with:
      ref: v2.6.2
      codex_model: ''             # 可选;留空用 Codex 默认模型
      codex_cli_version: latest   # 可选;可钉 @openai/codex npm 版本
```

`provider` / `base_url` / `model_*` / `LLM_API_KEY` 这些旧云上 LLM 参数仍被 workflow 接受,但已 deprecated 且不会参与评审。

---

## 3. 分支保护:把 reusable workflow 设为 required check

合并到 main 必须过 AI 评审:

1. 你 repo Settings → Branches → Branch protection rule for `main`
2. ✅ Require status checks → 加 **`review / ai-review-gate`**(注意是 `<job-name> / ai-review-gate`)
3. ✅ Do not allow bypassing the above settings

这样故意改坏的 PR 永远合不进 main。

---

## 4. reusable vs 拷贝版 选哪个

| 维度 | reusable(本文) | 拷贝版(`tools/install.sh`) |
|---|---|---|
| 上手速度 | 极快(几行) | 中(拷一堆文件) |
| 升级方式 | 改 `ref: vX.Y.Z` | 重新 install + diff merge |
| 离线/私有 GitHub Enterprise | 不行(需访问 WILLcis 仓) | 行 |
| 自定义 prompts/scripts | 不行(共享版) | 行 |
| state/ 持久化(triage 历史等) | 不能(只跑评审) | 行(全套自愈环) |
| 适合 | **只想要 Codex 评审** | 想完整 harness(自愈环、特性开关、reports) |

**最佳实践**:大部分仓用 reusable workflow 跑 AI 评审就够;少数核心仓用 install.sh 装全套自愈环。

---

## 5. 升级建议:**永远钉 ref 到 tag**,不要 `@main`

```yaml
# ❌ 危险:本仓更新可能在你的下一个 PR 突然改变评审行为
uses: WILLcis/AI--First-Coding-Loop-Codex/.github/workflows/ai-review-reusable.yml@main

# ✅ 推荐:钉到具体 tag,主动控制升级时机
uses: WILLcis/AI--First-Coding-Loop-Codex/.github/workflows/ai-review-reusable.yml@v2.6.2
```

每次本仓发新版后,你在目标仓发一个 PR 把 `ref:` 改成新 tag,review 看影响后再合。
这就是 harness 自己的"灰度发布"。

---

## 6. 故障排查

### "Missing CODEX_ACCESS_TOKEN"

调用方仓没有配置 Secret。去 Settings → Secrets and variables → Actions 添加:

```text
CODEX_ACCESS_TOKEN=<官方 Codex access token>
```

不要把浏览器登录态、ChatGPT cookie、或其他非官方 token 塞进来;workflow 调的是 `codex login --with-access-token`。

### "Resource not accessible by integration"

调用方仓的 `GITHUB_TOKEN` 权限不够。本 reusable workflow 已声明 `permissions: pull-requests: write`,
但调用方仓 Settings → Actions → General → Workflow permissions 必须设为 "Read and write"(默认是只读)。

### "Unable to checkout WILLcis/AI--First-Coding-Loop-Codex"

本仓是私有的话,调用方 GITHUB_TOKEN 没权访问。两个解法:
- (推荐)把本仓设为 public
- 在调用方仓配一个 PAT 通过 secret 传给 `actions/checkout` 的 `token` 输入(需要把 reusable workflow 改造接 secret)

### Codex CLI 安装失败

workflow 默认安装 `@openai/codex@latest`。如果你需要稳定复现,把 `codex_cli_version` 钉到一个已验证版本。
