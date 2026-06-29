# Week 0 实施清单(Codex 版)

> 适用:从空仓起步 · GitHub + Codex + OpenAI-compatible LLM + Statsig + AWS + Linear · 创始人/CTO 单人推动
> 目标:**5 个工作日内**让 harness 完全跑通,从此每一行业务代码"出生"就被它约束。
> 用法:按顺序勾选;每个动作下都有"验收"——验收不过就别往下走,卡住时跳到末尾"常见坑"。

---

## 0. 前置(开工前 1 小时,把账号和信息凑齐)

不齐就一定会卡。下面这些**先全部备好**,后面 Day 0~5 全程不用再停下来等开通。

- [ ] **GitHub 账号 + 组织**(建议建一个 org,不要用个人账号——后面接 Apps、OIDC、Enterprise 时会省心)
- [ ] **Claude Code CI 凭证**:准备好 `CLAUDE_CODE_OAUTH_TOKEN`;远端 PR 评审会用它跑 Claude Code Action
- [ ] **AWS 账号**(单独的根账号,后续可拆 dev/prod 子账号)+ 一张能扣费的卡;选好 region(下文以 `ap-northeast-1` 为例)
- [ ] **Statsig 账号**:[https://console.statsig.com](https://console.statsig.com),建一个 project,记下 Server Secret Key
- [ ] **Linear 账号**:[https://linear.app](https://linear.app),建一个 workspace,API Key 在 Settings → API
- [ ] **Sentry 账号**:[https://sentry.io](https://sentry.io),建一个 org + project
- [ ] **Microsoft Teams 频道**(或 Slack,任选)+ 一个 Incoming Webhook URL
- [ ] **本地机**:Node ≥ 20、pnpm 9、Docker、Python 3.12、git、AWS CLI v2
- [ ] **域名**(可选,但 prod 部署时要):一个能改 DNS 的域

> 预算预估(月费,完全可控):LLM ~$50~$200(看 PR 数量和模型)/ AWS dev+prod ~$80~$300 / Statsig 个人免费档 / Linear $8/人 / Sentry 个人免费档 / GitHub Team $4/人。**Week 0 总开销通常 < $300**。

---

## Day 0(今天,3~4 小时):创建仓库 + 提交 harness + 本地门禁绿

**目标**:有一个空仓,里面只有 harness,本地能跑通 CI 等价的全套校验。

### 上午(约 1.5 小时)

- [ ] **决策 5 分钟**:确定项目名(后面会作为 GitHub repo 名 + AWS 资源前缀);确定架构师(很可能就是你);列出未来 3 个月可能涉及的语言(影响 ci.yml 里要不要取消那些 Go/Python 注释)
- [ ] 在 GitHub org 下建私有空仓 `your-org/your-project`,**不要勾选 README / .gitignore / license**(我们要手动 push)
- [ ] 克隆 Codex 版 harness 并安装到空仓:
  ```bash
  git clone --depth 1 https://github.com/WILLcis/AI--First-Coding-Loop-Codex /tmp/aifcl
  bash /tmp/aifcl/tools/install.sh "$PWD" --strategy 1
  ```
- [ ] 进入项目目录,初始化 git:
  ```bash
  cd your-project
  git init -b main
  git add .
  git commit -m "chore: bootstrap harness (六阶段流水线 + 三趟评审 + 自愈环)"
  git remote add origin git@github.com:your-org/your-project.git
  git push -u origin main
  ```

**验收**:仓库上能看到 `.github/workflows/`、`scripts/`、`prompts/`、`flags/`、`AGENTS.md`,且 push 后 GitHub Actions 标签页里 CI 已开始跑(此时大概率红——因为还没装依赖逻辑,不要慌)。

### 下午(约 2 小时)

- [ ] **填好 `AGENTS.md`**:把 `[占位]` 都替换成你的产品描述、目录约定、本地起栈命令。**这一步是 ROI 最高的**——后面所有 agent 都靠它工作。
- [ ] **决定你的语言起点**:精瘦起步建议先只接一个语言(参考另一份 greenfield-starter)。但既然你选了 harness kit + 空仓,这通常意味着你打算多语言齐上——确保 `services/` 至少有一个能跑的 hello service(可以参考 greenfield-starter 里 `services/api` 的范式抄过来,先让 monorepo 有"活的代码")。
- [ ] 配 `config/.env.dev.example` 和 `.env.prod.example`,key 集合务必一致(`scripts/check_env_parity.py` 会校验)
- [ ] 本地跑一次"CI 等价"全套:
  ```bash
  make bootstrap
  make check
  make test-unit
  make test-integration
  python3 scripts/check_env_parity.py config/.env.dev.example config/.env.prod.example
  ```

**验收**:本地全套**绿**。再 push 一次,GitHub 上 CI 也变绿。

---

## Day 1(半天):让门禁"强制、不可绕过"

**目标**:从此 main 分支只能通过 PR + 绿门禁进入,任何人(包括你自己)都无法绕过。

- [ ] GitHub repo → **Settings → Branches** → 给 `main` 加保护规则:
  - ✅ Require a pull request before merging(`Required approvals` 至少 0,我们靠 AI 评审,人审看战略风险)
  - ✅ Require status checks to pass before merging
  - 必需检查:**`ci-gate`**(此刻先加这一个,`ai-review-gate` 在 Day 2 加)
  - ✅ Require branches to be up to date before merging
  - ✅ Require conversation resolution
  - ✅ Do not allow bypassing the above settings(**这条最关键**:连 admin 也不能绕过)
  - ❌ Allow force pushes(关闭)
  - ❌ Allow deletions(关闭)
- [ ] 仓库设置 → General → 关闭不需要的:Wiki / Projects(我们用 Linear)/ Discussions(看团队需要)
- [ ] 开启 **Merge queue**(beta;或装 Graphite App 替代)
- [ ] **冒烟测试**:开一个 PR,故意改坏一行类型(比如 `const x: number = "abc"`),确认 PR **红且无法合并**;改回来变绿可合并

**验收**:故意失败的 PR 被拦住、修好后能合。**门禁此刻已是真门禁**。

---

## Day 2(1 天):接通三方服务

**目标**:LLM provider、Statsig、Linear、Sentry、Teams 五个外部服务全部接进 GitHub Secrets/Vars,工作流第一次能跨服务调通。

按顺序做(每个 ~30 分钟):

### 2.1 Claude Code token + 三趟 AI 评审

- [ ] 仓库 → **Settings → Secrets and variables → Actions → New repository secret**
  - `CLAUDE_CODE_OAUTH_TOKEN` = Claude Code OAuth token
  - Variable:`CLAUDE_MODEL` = 可选;留空则使用 Claude Code 默认模型
- [ ] 开一个测试 PR,确认 `.github/workflows/ai-review.yml` 的三个 job(quality / security / dependency)被触发并完成
- [ ] **把 `ai-review-gate` 加进 main 分支保护的必需检查**

**验收**:开一个"故意有安全问题"的 PR(比如新增端点没鉴权),security 评审给出 `BLOCK`,PR 合不进去。

### 2.2 Statsig 特性开关

- [ ] Secret:`STATSIG_SERVER_SECRET` = Statsig project 的 Server Secret Key
- [ ] Statsig 控制台先建一个 flag:`new_greeting`(或你打算做的第一个功能的 flag 名),默认 disabled
- [ ] `flags/feature-flags.ts` 已默认在有 `STATSIG_SERVER_SECRET` 时用 Statsig provider,不用改代码

**验收**:写一段 5 行测试代码 `await flags.isEnabled(FLAGS.NEW_CHECKOUT_FLOW, { isTeamMember: true })`,在本地跑应返回 `true`(团队成员)或 `false`(非团队成员且未开灰度)。

### 2.3 Linear 工单

- [ ] Secrets:`LINEAR_API_KEY`
- [ ] Vars:`TRACKER` = `linear`;`LINEAR_TEAM_ID`(在 Linear → Settings → API → Team IDs 里看)
- [ ] 本地 dry-run 测一下脚本:
  ```bash
  OBSERVABILITY_BACKEND=mock TRACKER=linear LINEAR_API_KEY=... LINEAR_TEAM_ID=... \
    python3 scripts/triage_engine.py
  ```

**验收**:Linear 工作区里出现一张自动建的工单,标题含 `[fp:xxxxxx]`,内容含样本日志和九维打分。

### 2.4 Sentry 异常上报

- [ ] 在 `services/` 各服务里接 Sentry SDK,把 DSN 配进 `.env.*`
- [ ] Secret:`SENTRY_AUTH_TOKEN`;Vars:`SENTRY_ORG`、`SENTRY_PROJECT`
- [ ] 在某个服务里故意 throw 一个未捕获异常,部署后 Sentry 应能收到

**验收**:Sentry 项目里能看到至少一条事件。

### 2.5 Teams/Slack 通知

- [ ] Secret:`NOTIFY_WEBHOOK_URL`
- [ ] 手动触发 `daily-health.yml`:Actions 标签 → Daily Health Report → Run workflow

**验收**:Teams/Slack 频道收到一条由 AI 生成的健康摘要。

---

## Day 3(1 天):AWS 部署链路联通(只接 dev)

**目标**:合并 PR 后自动构建并部署到 dev 环境,test-dev 阶段冒烟通过。**prod 这天先不接**——稳了再说。

### 3.1 AWS 准备(上午)

- [ ] 决定容器服务方案:**ECS Fargate**(简单)或 **EKS**(扩展性强)。Week 0 推荐 ECS Fargate。
- [ ] 在 dev account 建:VPC、ECS cluster `your-project-dev`、ECR repo、ALB、Log Groups
- [ ] **重点:OIDC 信任 GitHub**——避免在 GitHub 存 AWS 长期密钥:
  - IAM → Identity providers → Add → OpenID Connect → Provider URL `https://token.actions.githubusercontent.com`,audience `sts.amazonaws.com`
  - 建一个 IAM Role `gha-deploy-dev`,trust policy 限制到你的 repo(`token.actions.githubusercontent.com:sub` = `repo:your-org/your-project:ref:refs/heads/main`),attach 部署所需权限(ECR push、ECS update-service、CloudWatch logs)
- [ ] Secret:`DEPLOY_ROLE_ARN` = 上面那个 role 的 ARN
- [ ] Vars:`AWS_REGION` = `ap-northeast-1`(或你选的)

### 3.2 写 `ops/` 下的薄封装脚本(下午)

`deploy.yml` 调用了这些脚本——它们是和云强耦合的部分,你要按你的 ECS 实现:

- [ ] `ops/build_and_push.sh <image-tag>`:`docker build` 各服务 → `aws ecr get-login-password | docker login` → `docker push`
- [ ] `ops/deploy.sh <env> <image-tag> [--circuit-breaker]`:`aws ecs update-service --force-new-deployment` + 启用 deployment circuit breaker(`--deployment-configuration "deploymentCircuitBreaker={enable=true,rollback=true}"`)
- [ ] `ops/run_tests.sh <env> [--smoke]`:对该环境跑 health + e2e 冒烟
- [ ] `ops/healthcheck.sh <env>`:轮询健康端点
- [ ] `ops/watch_metrics.sh <env> --window 300`:从 CloudWatch 拉 5 分钟错误率/延迟,超阈值则非零退出(触发回滚)
- [ ] `ops/rollback.sh <env>`:`aws ecs update-service` 回到上一个 task definition
- [ ] `ops/notify.sh <text>`:`curl -X POST $NOTIFY_WEBHOOK_URL ...`

### 3.3 联调(下午晚些时候)

- [ ] PR 合到 main → 看 Actions 里 `deploy.yml` 跑通 stage 2~3(`build-and-deploy-dev` 与 `test-dev`)
- [ ] 故意让 test-dev 失败一次,确认会触发 `rollback-prod`(因为我们还没接 prod,这里只验证 rollback 逻辑被触发)

**验收**:合并 PR 自动部署到 dev、冒烟通过、Sentry 没新错误、Teams 收到部署通知。**先到这里**,prod 接通建议放到 Week 1。

---

## Day 4(半天):自愈反馈环上线

**目标**:每日健康报告 + triage 引擎按计划自动跑,Linear 自动建单。

- [ ] 把 `daily-health.yml` 的 cron `0 9 * * *` 调整到适合你团队的时间(默认 UTC 09:00 = 北京 17:00)
- [ ] `triage.yml` 默认晚一小时 cron `0 10 * * *`
- [ ] Vars:`OBSERVABILITY_BACKEND` = `cloudwatch`;Secret:`OBSERVABILITY_READ_ROLE_ARN`(只读 CloudWatch 的 IAM role)
- [ ] CloudWatch Logs 至少配 1 个 Log Group `/your-project/dev`,服务输出结构化 JSON 日志(`server.ts` 已示范)
- [ ] **冒烟**:Actions 手动触发 daily-health → Teams 收到报告;手动触发 triage → 看 Linear 是否建单
- [ ] **复检冒烟**:Actions → Triage Engine → Run workflow → mode `verify` → 已"消失"的错误工单应被自动关闭

**验收**:让一个服务每分钟故意打几条 error 日志,等到下一个 triage 周期跑,Linear 出现含九维打分的工单;然后停止打错,触发 verify,工单自动关闭。**闭环成立。**

---

## Day 5(1 天):跑通第一个真实功能,验证整条流水线

**目标**:让"架构师 + agent + AI 评审 + CI + 部署 + 灰度 + kill switch"这条流水线被一个**真功能**走通。这一天的目的不是产功能,而是**验证流水线**。

- [ ] **架构师写任务**:复制 `prompts/architect-task.md`,填一个小但完整的功能(例:在 api 上加一个 `/v1/users/me` 端点)
  - 目标、上下文、范围、可测验收标准、约束(必须藏在 `new_user_me` flag 后)、灰度计划
- [ ] **agent 实现**:在 PR 描述里贴架构师的任务模板,触发实现(可用 `@claude` mention,或开一个新分支让 Codex 来跑)
  - 要求 agent 输出**实现计划 + 风险**,你确认后再写码
  - 自带测试
  - 功能包在 Statsig flag 后
- [ ] **PR 开**:三趟 AI 评审 + CI 跑;你**只看战略风险**(失败模式、安全边界、未来债务)
- [ ] **合并队列合并** → 自动部署 dev → 冒烟过
- [ ] **Statsig 灰度**:先 `teamOnly`,自己访问验证;再 5% → 25% → 100%
- [ ] **触发 kill**:故意让 flag 关闭(`{"enabled":false}` 或 Statsig kill switch),验证功能即时关闭、无需部署

**验收**:这个小功能从架构师写任务到 100% 上线 < 4 小时,kill switch 测试通过。

---

## ✅ Week 0 完成的判据(7 条)

走完上面后,以下 7 条**全部**为真,Week 0 才算合格:

1. main 分支被保护,直接 push 失败;故意失败的 PR 合不进去
2. 三趟 AI 评审作为门禁运行,且能 BLOCK 一个有问题的 PR
3. 一个 PR 合并后能自动部署到 AWS dev 并冒烟通过
4. 每日健康报告自动跑、推到 Teams/Slack
5. triage 引擎自动建 Linear 单并能在 verify 后自动关闭
6. 一个真功能藏在 Statsig flag 后,能灰度、能 kill switch 即时关闭
7. `AGENTS.md` 是真实的、不再有 `[占位]`,新 agent 只读它就能开始干活

---

## 💰 Week 0 预算清算(把账理清)

| 项目 | 一次性 | 月固定 | 备注 |
|---|---|---|---|
| Anthropic API | — | $50~$200 | 取决于 PR 数;Week 0 通常 < $30 |
| AWS dev(ECS Fargate + ALB + CloudWatch) | — | $80~$150 | 1~2 个小 service |
| AWS prod(Week 1 才接) | — | (暂 $0) | — |
| Statsig | — | $0 | 个人免费档够用 |
| Linear | — | $8/人 | 5 人 $40 |
| Sentry | — | $0 | 5K errors/月免费 |
| Teams/Slack webhook | — | $0 | — |
| GitHub Team | — | $4/人 | 5 人 $20 |
| **Week 0 合计** | $0 | **$200~$420** | 完全可控 |

---

## 🪤 常见坑(卡住时翻这里)

1. **CI 一直红,本地是绿的** → 99% 是依赖版本不一致。确认 GitHub Actions 用的是 Node 20、pnpm 9、Python 3.12;`pnpm install --frozen-lockfile` 必须有 `pnpm-lock.yaml` 一起提交。
2. **三趟 AI 评审超时** → 大 PR 容易触发。先缩 PR 范围;长期解决是配 `model: claude-sonnet-4-6` 给非关键趟(质量与依赖),保留 opus 给安全。
3. **OIDC 部署一直 `AccessDenied`** → 检查 IAM Role trust policy 里的 `sub` claim,它**必须**精确匹配 `repo:org/repo:ref:refs/heads/main`(不是 `*`);否则 main 分支以外的 push 都会失败,这是设计。
4. **Statsig flag 不生效** → 你 `await` 了吗?所有 `flags.isEnabled` 调用都是 async,常见 bug 是忘了 await。
5. **Linear 自动建单太吵** → 把 `TRIAGE_THRESHOLD` 环境变量从 0.35 调到 0.5,只让真正的事故进 Linear。
6. **Sentry/CloudWatch 看不到错误** → 你的日志结构化了吗?九成情况是日志没 JSON 化或缺关键字段(`service`/`level`)。
7. **熔断回滚没触发** → ECS 的 `deploymentCircuitBreaker.rollback` 必须显式开;它默认是 `false`。
8. **整周做完发现 agent 写不动东西** → AGENTS.md 太空。回去把"如何起栈/如何测试/编码规范/禁区"写细。

---

## Week 1 预告(走完 Week 0 后立刻接)

- 接 prod 部署(deploy.yml 的 stage 4~6)与熔断回滚
- 把 ci.yml 里 Go / Python 的预留 job 取消注释(如有相应服务)
- 开始正经做产品功能;每个功能强制走 `prompts/architect-task.md`
- 把"AI-native"外溢到产品/市场(每日数据摘要、发布说明 AI 化)

---

> 这份清单**没有形容词、没有口号**——它就是你这一周要打的勾。
> 卡哪一步就停在哪一步,**别跳**;harness 的整个价值在于"绝不可绕过"。
