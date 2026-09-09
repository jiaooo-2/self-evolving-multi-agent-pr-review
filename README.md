# 自进化多智能体 PR 风险审查与修复系统

> Self-Evolving Multi-Agent PR Risk Review and Repair System

EvoAgent 是一个面向 Pull Request 的多智能体代码审查平台。它以有界、可审计的 Agent 协作完成风险识别与修复建议，并通过评测门禁让审查策略和领域 Skill 在可验证的范围内持续演进。

## 核心能力

- **多智能体审查**：Lead 负责拆解、风险分级与最终综合；Security、Correctness/Reliability Worker 分别取证；Critic 独立质疑候选结论，降低误报。
- **有界执行**：每个角色都受调用次数、Token、时间和工具权限约束；普通任务单轮完成，高风险任务最多允许一轮返工。
- **证据驱动的结论**：高风险问题需要关联 AST、符号、扫描器、Git 上下文或测试输出等证据，且只报告本次改动引入的问题。
- **自动修复闭环**：支持 LLM 统一补丁、AST/CST 分析、修复前后测试对比；自动修复始终在独立分支生成 Draft PR，不直接改动原 PR 分支。
- **自进化审查策略**：从已确认的失败反馈中进行根因聚类，生成受限的结构化候选（提示词补充、示例、委派规则、工具选择策略与预算），而非直接修改生产源码。
- **可验证的版本激活**：候选策略必须在验证集达到最小提升，并在仓库隔离的隐藏 Holdout 集上通过受保护指标非退化门禁；所有候选、指标、激活决定均可追踪和回滚。
- **可演进的 Agent Skills**：审查领域以独立 `SKILL.md` 工件管理。Skill 候选可单独回放评测、门禁、激活或回滚，避免让所有领域规则进入每个任务上下文。
- **上下文与记忆管理**：按租户、仓库和任务隔离工作记忆与长期反馈记忆，支持检索、过期清理和任务结束后的经验沉淀。
- **生产化运行能力**：支持 SQLite/PostgreSQL、Redis Streams、任务 checkpoint/续跑、租约、指数退避、死信队列、RBAC、审计日志、Prometheus 和 OpenTelemetry。

## 架构概览

```text
GitHub Webhook / HTTP API
            │
            ▼
      ReviewService ─── Task Store (SQLite / PostgreSQL)
            │
            ▼
  Review Harness / Agent Runtime
  checkpoint · budget · trace · resume
            │
            ├── Diff / AST / Repository Tools / Scanners
            ├── Context Manager + scoped Memory
            └── Agentic Review
                  ├── Lead：拆解、风险分级、汇总
                  ├── Security Worker：输入、权限、敏感数据、危险调用链
                  ├── Correctness/Reliability Worker：状态、异常、并发、兼容性
                  └── Critic：独立质疑、反例与证据挑战
                            │
                            ▼
                    Finding / release gates
```

## 自进化流程

```text
已确认反馈 / 失败轨迹
          │
          ▼
根因聚类与结构化候选生成
          │  仅允许策略、示例、委派、工具选择与预算调整
          ▼
验证集回放：候选必须达到最小提升
          │
          ▼
隐藏 Holdout 回放：受保护指标不得退化
          │
          ├── 通过 → 持久化、激活，可灰度/影子验证
          └── 不通过或数据不足 → 保留为 deferred，不自动上线
```

系统同时支持两类独立演化对象：

1. `llm-review` 审查策略：基于失败轨迹生成并评估结构化候选；
2. Agent Skill：以 `SKILL.md` 为版本化工件，按领域独立评测、激活和回滚。

## 内置审查领域

| Skill | 关注点 |
|---|---|
| `security-review` | 鉴权、输入边界、敏感数据、危险调用链 |
| `correctness-review` | 状态转换、边界条件、异常处理和行为正确性 |
| `reliability-review` | 并发、重试、资源生命周期和故障恢复 |
| `performance-review` | 算法复杂度、热点路径和资源消耗 |
| `database-review` | 数据一致性、事务、查询和迁移风险 |
| `api-compatibility` | API 兼容性与契约变更 |
| `observability-review` | 日志、指标、追踪和可诊断性 |
| `test-quality` | 测试覆盖、断言质量与回归风险 |

## 快速开始

要求：Python 3.11+，以及一个 OpenAI Chat Completions 兼容模型。`agentic` 审查模式需要模型配置；未配置模型时服务可启动，但不能提交多智能体审查任务。

```powershell
python -m pip install -r requirements.txt

$bytes = New-Object byte[] 32
[Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
$env:EVOAGENT_AUTH_REQUIRED = 'true'
$env:EVOAGENT_AUTH_SECRET = [Convert]::ToBase64String($bytes)
$env:EVOAGENT_BOOTSTRAP_ADMIN_USERNAME = 'admin'
$env:EVOAGENT_BOOTSTRAP_ADMIN_PASSWORD = '<至少 10 个字符的密码>'

# 选择一个模型提供方，例如 DeepSeek
$env:EVOAGENT_LLM_PROVIDER = 'deepseek'
$env:EVOAGENT_DEEPSEEK_API_KEY = '<你的 API Key>'

python -m evoagent
```

服务默认运行于 `http://127.0.0.1:8080/`。登录后可通过 API 创建审查任务：

```powershell
$session = Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8080/v1/auth/login `
  -ContentType 'application/json' `
  -Body (@{username='admin'; password='<你的密码>'} | ConvertTo-Json)
$headers = @{Authorization="Bearer $($session.access_token)"}

Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8080/v1/reviews `
  -Headers $headers -ContentType 'application/json' `
  -Body (@{
    repository = 'demo/api'
    pull_request = 12
    mode = 'agentic'
    diff = '<unified diff>'
  } | ConvertTo-Json)
```

运行测试：

```powershell
python -m unittest discover -s tests -v
```

## GitHub 集成

系统通过 GitHub `pull_request` Webhook 接收 `opened`、`reopened` 和 `synchronize` 事件。GitHub 无法访问本机回环地址，因此本地调试时需使用 Cloudflare Tunnel、ngrok 或其他公网 HTTPS 转发服务，将公网 `/webhooks/github` 转发到本机服务。

建议使用 fine-grained Personal Access Token，并只为目标仓库授予所需最小权限：读取 PR Diff 需要 `Contents: Read` 和 `Pull requests: Read`；回写评论和创建修复分支时再授予对应写权限。

## 演化安全边界

- 候选不会直接产出或写入生产 Python 源码；
- 隐藏 Holdout 的案例内容不会通过 API 暴露；
- 验证/隐藏集不足、模型未配置或指标退化时，候选保持 `deferred`，不会自动激活；
- 修复提交始终落在独立 `evoagent/fix-pr-*` 分支，并以 Draft PR 形式交由人工审核；
- 密钥只从环境变量或被忽略的 `.env` 读取，绝不提交到仓库。

## 主要 API

| 方法 | 路径 | 用途 |
|---|---|---|
| `POST` | `/v1/reviews` | 创建审查任务 |
| `GET` | `/v1/tasks/{id}` | 获取任务状态、轨迹与报告 |
| `POST` | `/v1/tasks/{id}/fix` | 创建独立修复分支与 Draft PR |
| `POST` | `/v1/tasks/{id}/feedback` | 回流误报、漏报或坏修复反馈 |
| `POST` | `/v1/evolution/auto` | 由确认反馈生成策略候选并评测 |
| `GET` | `/v1/evolution/status` | 查看策略演化门禁与版本状态 |
| `POST` | `/v1/skill-evolution/auto` | 生成并评测 Agent Skill 候选 |
| `GET` | `/v1/skill-evolution/runs` | 查看 Skill 演化运行记录 |
| `POST` | `/v1/skills/{name}/versions/{version}/activate` | 激活或回滚审查策略版本 |
| `POST` | `/v1/skill-evolution/{name}/versions/{version}/activate` | 激活或回滚 Skill 版本 |

详细环境变量、Webhook 配置与 API 契约见 [`.env.example`](.env.example) 和源码中的 API 实现。
