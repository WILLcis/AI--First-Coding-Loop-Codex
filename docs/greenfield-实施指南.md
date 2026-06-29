# Greenfield 实施指南:从零开始,以 harness 为标准建项目

> 适用:全新项目、单一 TS/Node 栈精瘦起步、<15 人团队、你是创始人/CTO。
> 配套:本仓即"启动脚手架",跑 `./bootstrap.sh` 即可把它变成你的项目。

---

## 核心心法:倒过来建

迁移老项目时,你是"在跑着的车上换引擎";从零开始时,你有一个别人没有的奢侈——
**可以在写第一行业务代码之前,先把约束业务代码的系统建好。**

所以 greenfield 的实施顺序是**倒过来的**:

```
传统:   写功能 → 跑起来 → (以后再补)测试/CI/可观测
harness: 先建门禁/评审/开关/可观测 → 再让第一个功能"出生"就被它们约束
```

一句话:**你的第一个 commit 不是功能,是 harness。** 第一个功能是用来验证 harness 的。

这件事在 greenfield 比迁移**便宜得多**——没有历史包袱、没有要改的旧流程、没有要说服的存量习惯。错过这个窗口,等代码堆起来再补门禁,成本会翻很多倍。

---

## 里程碑总览(M0→M5)

| 里程碑 | 产出 | 验收(Definition of Done) |
|---|---|---|
| **M0** 决策与骨架 | monorepo 骨架 + 本地门禁绿 | `./bootstrap.sh` 跑完全绿 |
| **M1** 门禁强制化 | GitHub 分支保护 + 必需检查 | 一个故意失败的 PR 无法被合并 |
| **M2** AI 评审上线 | 三趟评审作为 PR 门禁 | 一个有问题的 PR 被 AI 评审 BLOCK |
| **M3** 部署流水线 | 六阶段 dev→prod + 熔断回滚 | 合并即自动部署到 dev,冒烟通过 |
| **M4** 自愈环 | 每日健康 + triage + 复检关单 | 制造一个错误能自动建单、修复后自动关单 |
| **M5** 第一个功能 | 走完整新功能路径上线 | 功能藏在开关后,灰度→全量,A/B 可读 |

下面逐个展开,每步都给"做什么 / 怎么验收"。

---

## M0 — 决策与骨架(半天)

**做什么**
1. 用本仓作为起点:`./bootstrap.sh your-project-name`。它会装依赖、构建、跑"本地版 CI 门禁"(typecheck + lint + 单测 + 集成测试),并生成 `config/.env.dev`。
2. 读 `AGENTS.md`,把 `[占位]` 换成你项目的真实信息(产品是什么、各包职责)。
3. 理解骨架里已经给你的东西:
   - `packages/contracts` 跨包类型的唯一事实来源
   - `packages/flags` 特性开关(env 驱动,Day 0 零外部依赖)
   - `services/api` 一个零运行时依赖的示例服务,**业务逻辑(`greet`)与 IO(`server`)分离,flags 用依赖注入**——这是让 agent 能离线测试的关键模式,你之后的服务都照这个套路
   - `Makefile` 是"契约层":`make check / test-unit / test-integration` 这些命令名永远不变,底层工具可换
   - `scripts/` 自愈环脚本、`prompts/` 评审与任务模板、`.github/workflows/` 全套流水线

**验收**:`./bootstrap.sh` 输出"所有门禁通过"。此刻你有一个**空但完全被武装**的项目。

---

## M1 — 把门禁变成"强制、不可绕过"(半天)

本地绿不算数,关键是**让 GitHub 强制执行**。这一步把"礼貌建议"变成"硬门禁"。

**做什么**
1. 推到 GitHub。
2. Settings → Branches → 给 `main` 加保护规则:
   - Require a pull request before merging(禁止直接 push main)
   - Require status checks to pass → 选 **`ci-gate`**(`ci.yml` 的汇总门禁)
   - Require branches to be up to date
   - (推荐)Require linear history + 开启 **Merge Queue**
3. 把 `ci.yml` 设为对 PR 和 merge_group 都触发(模板已配)。

**为什么这么设计**:`ci.yml` 里所有具体检查都汇总到一个 `ci-gate` job。你只需把这**一个** check 设为必需,新增检查时不用再去改分支保护——这是"统一门禁"的运维红利。

**验收**:开一个故意把类型写错的 PR,确认它**红且无法合并**;修好后变绿可合并。

---

## M2 — AI 评审上线(半天)

**做什么**
1. 仓库 secret 加 `CODEX_ACCESS_TOKEN`;可选 variable 加 `CODEX_MODEL`。
2. `ai-review.yml` 已配三趟并行评审(质量/安全/依赖),提示词在 `prompts/`。把汇总门禁 **`ai-review-gate`** 也加入分支保护的必需检查。
3. 明确分工:**AI 评审看逐行正确性/安全/依赖,人类评审只看战略风险。**

**验收**:开一个含明显问题(如未鉴权的新端点、或拼接用户输入到查询)的 PR,确认对应评审趟给出 `BLOCK` 并拦住合并。

> 调优:`prompts/*.md` 是版本化的、可迭代的。评审太吵就收紧 BLOCK 标准,漏报就补充关注点——**这就是 harness engineering:不满意时改提示词/门禁,而不是"让人更仔细"。**

---

## M3 — 部署流水线(1~2 天)

**做什么**
1. 准备云资源(模板以 AWS 占位,可换 GCP/k8s):dev 与 prod 两套环境、容器服务、**部署熔断回滚**(指标恶化自动回退)。
2. 实现 `ops/` 下被 `deploy.yml` 调用的脚本:`build_and_push.sh`、`deploy.sh`、`run_tests.sh`、`healthcheck.sh`、`watch_metrics.sh`、`rollback.sh`、`notify.sh`(它们是薄封装,内容取决于你的云)。
3. 配 OIDC 部署角色 secret(`DEPLOY_ROLE_ARN`),避免长期密钥。
4. `deploy.yml` 已实现六阶段:Verify→DeployDev→TestDev→DeployProd→TestProd→Release,prod 后 5 分钟指标观察窗,恶化即触发 `rollback-prod`。

**验收**:合并一个 PR 到 main,确认自动部署到 dev 且冒烟通过;手动触发一次 prod 路径,确认 test-prod 失败时会自动回滚。

> 顺序提示:可以先只接 dev(阶段 2~3),验证顺畅后再接 prod(阶段 4~6)。不必一次到位。

---

## M4 — 自愈反馈环(1~2 天)

**做什么**
1. 让所有服务输出**结构化 JSON 日志**(`service/level/ts/requestId` 等)——示例服务 `server.ts` 已示范。这是 triage 能聚类的前提。
2. 接可观测后端(CloudWatch/Prometheus+Loki)与 Sentry;在 `scripts/_adapters.py` 选择 `OBSERVABILITY_BACKEND`。
3. 接工单系统(Linear/Jira/GitHub Issues):`scripts/_adapters.py` 的 `TrackerAdapter`。
4. 启用 `daily-health.yml`(每日 09:00 健康报告)与 `triage.yml`(10:00 聚类→九维打分→去重建单;部署后 verify 复检关单)。

**先用 mock 跑通逻辑**(无需任何凭证):
```
cd scripts
OBSERVABILITY_BACKEND=mock python3 health_report.py
OBSERVABILITY_BACKEND=mock TRACKER=github-dryrun python3 triage_engine.py
OBSERVABILITY_BACKEND=mock TRACKER=github-dryrun python3 verify_triage.py
```

**验收**:制造一个真实错误(或用 mock 的事故数据),确认①健康报告里出现它;②triage 自动建单含样本日志/受影响端点/建议路径;③推一个修复部署后,verify 复检把工单自动关闭。闭环成立。

---

## M5 — 第一个功能,走完整新功能路径(0.5~1 天)

现在 harness 齐了,让第一个真功能验证整条流水线。**仿照示例服务的模式**(逻辑/IO 分离、flags 注入)。

**做什么**(对照 `prompts/architect-task.md`)
1. **架构师**把功能写成结构化 prompt:目标、上下文(指明涉及哪些包/文件)、范围、可测的验收标准、约束、**指定特性开关名**。
2. **agent** 先出实现计划 + 风险,确认后写代码 + **自带单测/集成测试**,把功能藏在 `@app/flags` 开关后。
3. 开 PR → 三趟 AI 评审 + CI 门禁 → 人类只看战略风险 → 合并队列 → 六阶段部署。
4. 开关:先 `teamOnly` → `rolloutPct` 5→25→100 → 监控指标 → 全量或 kill。

**验收**:功能在开关后上线;能从团队灰度到全量;A/B 变体可在埋点中读到;必要时 `{"enabled":false}` 即时杀掉、无需部署。

---

## 时间盘点

一个专注的架构师(很可能就是你),**约 1 周**能把 M0→M4 全部立起来,M5 紧随其后。之后每个功能都自动享受这套系统。对比"先狂写功能、半年后补流程"的路径,这一周是你回报率最高的投资。

---

## 精瘦起步之后:如何加第二个栈 / 第二个服务

模板已为此设计,加东西**不需要重构流水线**:

- **加一个 TS 服务**:在 `services/` 下复制 `api` 的结构(逻辑/IO 分离、flags 注入、单测+集成测试),CI 的 `node` job 自动覆盖它。
- **加 Go / Python 服务**:在 `services/` 建目录,取消 `ci.yml` 里预留的 `go` / `python` job 注释即可。`Makefile` 的命令名不变。
- **加前端**:在 `apps/web` 放 workspace 包,复用 `@app/contracts`,E2E 即可覆盖端到端。
- **升级特性开关到 SaaS**:需要可视化看板/实时改灰度/托管 A/B 时,给 `@app/flags` 加一个外部 provider(参考 harness kit 的 `feature-flags.ts` 里的 Statsig 适配器),业务调用方式不变。

每加一样,问自己同一个 harness 问题:**这个东西对 agent 可见、可校验、可强制执行了吗?** 是,就继续;不是,先把它变成那样。

---

## 常见坑

1. **先写功能再补门禁**——greenfield 最大的浪费。门禁先行,成本最低。
2. **门禁不强制**:本地绿但 GitHub 没设必需检查 = 没有门禁。M1 必须做到"故意失败的 PR 合不进去"。
3. **日志不结构化**:自愈环直接失效。从第一个服务就用结构化 JSON 日志。
4. **快 AI 没有快校验**:别急着放大 agent 产能,先确保 M2(评审)+ 集成测试到位,否则是高速堆技术债。
5. **功能不藏开关后**:失去"当天 kill / 灰度 / A/B"的全部能力。强制每个功能都过 `@app/flags`。

---

*配套文件:`AGENTS.md`(agent 上下文)、`bootstrap.sh`(一键初始化)、`prompts/`(评审与任务模板)、`.github/workflows/`(全套流水线)、`scripts/`(自愈环)。完整方法论见 harness kit 的《实施手册》。*
