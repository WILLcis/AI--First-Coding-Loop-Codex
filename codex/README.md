# codex/ — Codex 专属

> 这里的东西**只有当你使用 Codex CLI / IDE / app 这类支持 Skill 与 custom agent 的 Codex 客户端时才有用**。
> 不用 Codex 也能玩 `core/`,只是少了 skill 自动发现 + subagent 角色分发的便利。

## 文件去向

把整个目录的子文件夹按下面位置拷到你的项目仓库根:

| 来源 | 放到目标仓的 | 客户端 |
|---|---|---|
| `skills/` | `.agents/skills/` | Codex repo-scoped skills |
| `agents/` | `.codex/agents/` | Codex project-scoped custom agents |
| `AGENTS.md.template` | `AGENTS.md` | Codex project instructions |

`tools/install.sh` 会自动安装到上面的 Codex 原生路径。

## skills(9 个)

每个都是 `<name>/SKILL.md`,顶部 YAML front-matter 让 agent 自动按描述触发:

- `task-decomposer` — 主 session 收到需求后先判断是否可并行分解,输出 DAG
- `parallel-orchestrator` — 拿 DAG 做 fan-out/fan-in,调度并行 sub-agent 与 merger
- `agent-coding-discipline` — 写码 agent 的行为纪律:读再写、最小改动、fail-first 测试、依赖克制
- `architect-task-writer` — 把模糊想法变成结构化任务 prompt
- `pr-investigator` — 对自动建的工单做根因调查
- `feature-flag-setup` — 加新功能必走的开关流程
- `api-endpoint-creator` — 加新 HTTP 端点的标准范式
- `triage-severity-scorer` — 九维打分规则
- `weekly-comprehension-check` — **写给人的反认知投降仪式**(agent 只能提醒,不能代做)

## agents(9 个)

| name | 模型分层定位 | Codex 配置 |
|---|---|---|
| explorer | 轻量只读探查 | `model_reasoning_effort = "low"` |
| implementer | 写码主力 | `model_reasoning_effort = "medium"` |
| subtask-implementer | 并行子任务写码 | `model_reasoning_effort = "medium"` |
| merger | 整合并行子任务产出 | `model_reasoning_effort = "medium"` |
| verifier-quality | 质量评审 | `model_reasoning_effort = "medium"` |
| verifier-security | **安全评审,这里别省** | `model_reasoning_effort = "high"` |
| verifier-dependency | 依赖/许可证扫描 | `model_reasoning_effort = "low"` |
| triage-scorer | 错误簇九维打分 | `model_reasoning_effort = "medium"` |
| checker | 与 implementer 分离的 done 判定 | `model_reasoning_effort = "medium"` |

模型默认继承你的 Codex 会话配置;如需固定某个 agent 的模型,可在对应 TOML 里添加 `model = "..."`。

## SKILL.md 的写法

```yaml
---
name: <kebab-case>
description: <一段紧凑、无聊、可被 LLM 解析的功能描述>
when_to_use: <具体触发条件>
when_NOT_to_use: <反触发条件,防越界>
---
```

description 要**无聊**——Addy 反复强调"一段紧凑无聊的 description 比聪明的 description 更容易被准确触发"。
