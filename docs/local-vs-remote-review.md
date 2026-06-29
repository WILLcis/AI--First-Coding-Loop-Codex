# Local vs Remote 评审:不付双倍钱的架构指南(v2.6.2)

> 一个常被忽略的问题:**reusable workflow 在 GitHub Actions 里跑,要不要再买一套云上 LLM API key?**
>
> 答:v2.6.2 起默认不走通用云上 LLM API key,而是用官方 `CODEX_ACCESS_TOKEN` 让 Codex CLI 在受信任的 GitHub Actions 里评审。若你只是单人本地用,仍然完全可以不配远端 secret。

---

## 架构事实(为什么远端"看起来"要 key)

```
你本地的 Codex / chat agent          GitHub Actions Runner
─────────────────────             ─────────────────────────
认证:  你的本地登录态 / 订阅        认证:  *它自己的* CODEX_ACCESS_TOKEN
                                       (它认不出你本机登录态)
触发:  你手动 prompt               触发:  *任何人* 开 PR / push
位置:  你的本地 fs                 位置:  GitHub-hosted runner
```

**核心约束**:GitHub Actions 跑在远端 runner 上,**不认识你本机 Codex session**。所以它要做评审,必须有自己能用的官方 Codex access token。

---

## 三条路线 + 决策表(选哪条)

| 路线 | 远端 API key | 自动化程度 | 月费(参考) | 适合 |
|---|---|---|---|---|
| **A. Local-only** | ❌ 不要 | 手动 / 半自动 | **$0**(吃订阅) | 单人开发、纯测试期、不要 required check |
| **B. Codex access token**(reusable workflow 默认) | ✅ `CODEX_ACCESS_TOKEN` | 全自动 | 看 Codex 会员/组织额度 | 多人协作、要 required check、要审计、跨设备 |
| **C. 平台原生托管评审** | ❌ 通常不要 harness key | 全自动 | 看平台/席位 | 已启用 Codex/GitHub 原生 review、接受平台绑定 |

**决策树**:

```
有别人(同事/外包/agent)会提 PR 吗?─── 是 ──► 远端,用 B 或 C
                                          │
                                          否
                                          ▼
要不要让 main 分支强制门禁(required check)?─── 是 ──► 远端,用 B 或 C
                                          │
                                          否
                                          ▼
                                       用 A:Local-only,完全不付远端钱
```

---

## A. Local-only(推荐起步)

**怎么做**:

```bash
# 你刚 commit 了一些代码,push 前想评审一下
cd <your-project>
git fetch origin main
bash <(curl -sSL https://raw.githubusercontent.com/WILLcis/AI--First-Coding-Loop-Codex/v2.6/core/scripts/local_review.sh)
# 它会打印三段 prompt(quality/security/dependency),
# 把每段贴进 Codex 一个会话,人工 review 完再 push
```

或者更紧凑的 one-shot 三趟:

```bash
bash <(curl -sSL https://raw.githubusercontent.com/WILLcis/AI--First-Coding-Loop-Codex/v2.6/core/scripts/local_review.sh) --combined
# 输出一段总 prompt,让 Codex 一次跑完三趟并给 VERDICT
```

**好处**:
- ✅ 零远端 API 账单——完全吃你本地 Codex / ChatGPT / chat agent 额度
- ✅ 零远端配置(无需 secret/var、无需 workflow permissions = write)
- ✅ 即改即看——push 前就能拦,不用等 GitHub Actions

**代价**:
- ❌ 你不在线时其他人提 PR 不会被评审
- ❌ 不能做 required check(GitHub 看不到本地 review 结果)
- ❌ 跨设备同事看不到一致策略

## B. Codex access token(reusable workflow 默认)

**怎么做**:见 [`reusable-workflows.md`](reusable-workflows.md) — 几行 `uses:` + 配 GitHub Secret:`CODEX_ACCESS_TOKEN`。

**好处**:全自动、required check 成立、跨设备一致、有审计 log。
**代价**:runner 会消耗 Codex 会员/组织额度;并且 secret 只适合受信任仓库/受信任 workflow。

## C. 平台原生托管评审(进阶)

如果你的团队已经启用 Codex / GitHub 原生 review,可以让平台直接在 PR 上评审,不再由本 harness 的 reusable workflow 调 LLM API。

**好处**:少配一套 key、全自动、和平台权限模型一致。
**代价**:
- 绑定平台能力与席位,不再是本 harness 的多厂商 `LLM_PROVIDER` 路线
- 评审策略、输出格式和可移植性受平台约束
- 和 `ai_review.py` / prompts 的三趟门禁不完全等价,接入前要先试跑

如果你**不需要多厂商灵活性**,这是最省配置的远端路线。否则继续用 B。

---

## 我现在该选哪个?(贴墙上)

| 你的处境 | 选 |
|---|---|
| 我刚开始,自己测试 harness 是否能用 | **A**(完全不配远端,用 `scripts/local_review.sh`) |
| 我一个人,但有 5 个分项目要管 | **A** + 写个脚本自动 review 各项目 |
| 我开始有同事提 PR / 想 required check | **B**(配 `CODEX_ACCESS_TOKEN`) |
| 我已经启用 Codex/GitHub 原生 review | **C**(平台托管,不重复配 harness key) |
| 我要切 DeepSeek / Qwen 省钱 | legacy 云 LLM 路径;默认 Codex workflow 不再走 `LLM_PROVIDER` |

---

## 反模式

- ❌ 没多人协作 / 没 required check 需求,却配了远端 token —— 本地跑就够
- ❌ 配了远端,但本地也手动跑一遍 —— 双倍消耗 token
- ❌ 把浏览器登录态 / ChatGPT cookie 当 `CODEX_ACCESS_TOKEN` 用 —— 协议不一样,会失败
- ❌ 觉得"local 不够正式" —— **正式与否取决于谁/什么时候用,不取决于跑在哪台机器**

---

## 升级路径(L → R)

从 Local-only 升到 Codex access token 不是回头路——是**渐进的**:

```
Day 0   纯本地评审,免费验证 harness 价值
Day N   团队增加到 2 人 / 你开始上 CI required check
       ↓
       开 PR 配 GitHub Secret:CODEX_ACCESS_TOKEN
       开 PR 加 .github/workflows/pr-review.yml
       合并后远端开始 take over,你不用再手动跑 local_review
```

A → B 的迁移只是**加 secret + 加 1 个 workflow 文件**,本地脚本可以继续保留作"push 前快速自检"用。
