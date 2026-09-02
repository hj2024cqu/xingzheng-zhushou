# 文枢：面向高校行政服务的记忆增强与自进化文档问答系统

文枢是一个面向高校多部门行政场景的文档处理与智能问答系统。系统接入教务、财务、学生工作、人事、后勤和研究生培养等部门的制度文件，为用户提供跨部门问题路由、原文检索、答案生成和引用核验服务。

本项目关注的重点不是单次问答模型能力，而是如何让一个文档问答应用在持续运行中具备以下工程能力：

- 记住对后续任务有用的信息，同时不把非权威内容误当成制度事实；
- 复用历史会话、用户偏好、组织经验和已有执行流程，减少重复检索与重复推理；
- 从反馈和运行轨迹中发现高频问题，将有效处理方式沉淀为可执行策略；
- 在不阻塞在线问答的前提下完成回放、灰度、版本管理和回滚；
- 通过部门并行检索、上下文预算和异步任务提高应用侧吞吐与维护效率。

## 我的主要工作

我主要负责项目中的记忆系统和自进化系统，并完成它们与问答编排链路的集成。

### 1. 记忆系统

- 设计“独立事实平面 + 五类记忆平面”的分层架构，明确官方文档与模型记忆的边界；
- 实现统一的 `MemoryContextBuilder`，按用户、部门、角色、来源、时效和字符预算构建问答上下文；
- 实现工作记忆、情景记忆、用户语义记忆、组织知识记忆和程序性学习记忆；
- 为组织记忆增加文档版本校验、来源回查和失效传播，避免旧制度继续影响答案；
- 实现敏感信息过滤、TTL 清理、用户主动遗忘、使用记录和操作审计。

### 2. 自进化系统

- 实现 Execute、Observe、Reflect、Adapt、Deploy 五阶段闭环；
- 汇总显式反馈、隐式行为和 Verifier 自动信号，形成可追踪的 bad case；
- 基于历史 trace 聚类高频问题，生成可复用的 Skill 候选；
- 将处理经验拆分为 Skill、Hook 和 Rule 三类运行时资产；
- 实现候选策略的同题回放、基线对照、稳定灰度分桶、版本快照和劣化回滚；
- 支持人在环中、人在环上和人在环外三种治理阶段，使自动化范围能够逐步扩大。

这些工作共同服务于一个目标：把历史运行经验转化为下一次请求可以直接复用的上下文和执行计划，使应用随着使用逐步降低重复计算与人工维护成本。

## 系统结构

```text
用户 / 管理员
      │
      ▼
Next.js Web
      │ REST
      ▼
FastAPI Orchestrator（唯一控制平面）
      ├── MemoryContextBuilder
      ├── Intent / Query Rewrite
      ├── Skill / Hook / Rule
      ├── Hybrid Retrieval + Rerank
      ├── Answer + Verifier
      └── Trace / Feedback
              │
              ├── pi-agent：模型推理与受控工具调用
              ├── MongoDB：事实、长期记忆、策略版本与实验
              ├── Redis：会话状态、Stream 和异步作业
              └── 部门 Agent：按部门隔离并行执行
```

Python 后端负责权限、部门范围、事实选择、记忆治理和策略发布；`pi-agent` 只负责概率性推理和白名单工具调用。模型服务不可用时，编排层可以回退到本地 Python 实现，避免模型执行层获得数据权限或发布权限。

## 记忆板块

### 事实与记忆分离

行政问答最重要的问题不是“记住得更多”，而是“记住的内容是否有资格成为答案依据”。因此，系统把 active 官方文档及其 chunk 作为独立事实平面，任何记忆都不能直接替代官方原文。

```text
active documents / chunks（最高权威事实）
                  │
                  ▼
          MemoryContextBuilder
     ┌────────────┼────────────┐
     ▼            ▼            ▼
  会话状态      用户与组织记忆   Skills/Hooks/Rules
     │            │            │
     └────────────┴────────────┘
                  │ 选择、校验、裁剪
                  ▼
       Intent / Rewrite / Retrieval / Answer
```

组织 FAQ、流程提示和协调经验即使已经审核，也只能帮助检索。当它们被召回时，系统会重新检查 `doc_id`、`chunk_id` 和 `document_version`，读取仍处于 active 状态的原始 chunk，再交给 Answer 和 Verifier。文档被归档或版本替换后，关联组织记忆会被标记为 stale。

### 一个事实平面与五个记忆平面

| 平面 | 数据与存储 | 主要作用 | 应用侧价值 |
|---|---|---|---|
| 事实平面 | MongoDB `documents/chunks` | 保存 active 官方原文和版本关系 | 保证引用可追溯，避免记忆污染事实 |
| 工作记忆 | Redis，会话 TTL 默认 30 分钟 | 保存最近消息、实体、部门和引用 ID | 快速恢复短会话，避免每轮重建全部上下文 |
| 情景记忆 | `conversation_events/summaries` | 保存有序事件、滚动摘要和未解决问题 | 长对话用摘要代替完整历史，控制 prompt 长度 |
| 用户语义记忆 | `user_memory_items` | 保存用户明确提供的低敏偏好和资料 | 减少重复询问，支持个性化表达与检索 |
| 组织知识记忆 | `org_memory_items` | 保存带来源、部门、时效和审核状态的经验 | 复用高频 FAQ 与流程提示，同时回查官方来源 |
| 程序性学习记忆 | Skills/Hooks/Rules/Experiments | 保存“如何处理一类问题”的执行经验 | 复用检索扩展、回答模板和约束，减少重复规划 |

### 统一上下文构建

`backend/app/memory/context_builder.py` 是记忆的统一读取入口。一次请求的上下文构建过程如下：

1. 从 Redis 读取最近消息、已解析实体和活动部门；
2. 从情景记忆恢复会话摘要与尚未解决的问题；
3. 只召回当前用户可见且未过期的用户语义记忆；
4. 按部门、角色、时效和权威等级筛选组织记忆；
5. 回查组织记忆绑定的 active 文档 chunk；
6. 读取当前部门可执行的 Skill、Hook 和 Rule；
7. 按字符预算裁剪低优先级内容，并记录本次实际使用的记忆 ID。

默认上下文预算为 6000 字符，最近消息上限为 10 条，用户记忆和组织记忆默认各召回 8 条。预算不是简单截断字符串，而是按组织记忆、用户记忆、旧消息的顺序逐级裁剪，优先保留更直接影响当前任务的信息。

### 生命周期、隐私与审计

- 工作记忆只保存必要状态和 chunk ID，不长期复制完整检索正文；
- 会话事件默认保留 90 天，摘要和低敏用户记忆默认保留 180 天；
- 身份证、密码、健康、心理测评、处分和财务明细等敏感内容禁止进入长期记忆；
- 系统推断只进入 `memory_candidates`，未获用户确认时不会成为 active 用户记忆；
- 用户可通过 `/api/v1/memory/me` 查看、写入和删除自己的记忆；
- `memory_usage` 记录哪些记忆真正进入了某次 trace，`memory_audit` 记录写入、删除和治理操作。

这一设计使记忆不仅可用，而且可解释、可删除、可过期、可追责。

## 自进化板块

自进化不是定时重新训练模型，而是让系统把运行反馈转化为受治理的应用策略。它修改的是检索查询、部门范围、top-k、回答模板和约束规则，而不是绕过审核直接修改官方知识。

### 五阶段闭环

```text
Execute → Observe → Reflect → Adapt → Deploy
   ▲                                      │
   └──────── 下一轮请求使用新策略 ────────┘
```

| 阶段 | 输入与处理 | 输出 |
|---|---|---|
| Execute | 按当前 Skill、Hook、Rule 执行问答并记录 trace | 查询、检索、答案、引用、耗时和策略命中记录 |
| Observe | 汇总点赞/点踩/纠错、追问等行为和 Verifier 信号 | 未消费反馈与 bad case 集合 |
| Reflect | 使用 pi Runtime 或确定性规则分析失败原因 | 检索、生成、知识缺口等根因和候选建议 |
| Adapt | 聚类高频 trace，生成或更新策略资产 | Skill、Hook、Rule 候选及新版本 |
| Deploy | 回放、审核、灰度和质量门禁 | 激活、继续观察或回滚 |

Loop 通过 Redis Stream 和持久化 job 异步执行。管理端触发后轮询 `queued → running → completed/failed`，在线问答无需等待聚类、回放和策略发布完成。

### 三类可进化资产

| 资产 | 回答的问题 | 示例 | 对应用执行的影响 |
|---|---|---|---|
| Skill | 一类问题应该怎样处理 | 学术截止日期核验 | 扩展 query、调整 top-k、增加校历核验和回答模板 |
| Hook | 什么事件发生时触发什么动作 | 同时出现“选课”和“缴费” | 扩展到教务与财务部门并行检索 |
| Rule | 所有回答必须遵守什么约束 | 无来源不猜测、关键结论必须引用 | 运行时注入 Answer 与 Verifier |

`SkillMiner` 对近期 trace embedding 进行 DBSCAN 聚类；当同类问题达到配置阈值后，系统才生成 Skill 草稿。聚类或模型调用失败时会回退到关键词分组和确定性建议，保证 Loop 能够降级运行。

### 回放、灰度与回滚

候选 Skill 不直接替换线上策略。`StrategyEvaluator` 会在同一批历史问题上分别执行基线和候选路径，对比 Verifier 分数、通过率和检索数量。只有候选不低于基线且满足质量门禁时，才允许进入审核或灰度阶段。

灰度分桶使用 `user_id + session_id + skill_id + version` 的稳定哈希，同一用户不会在 treatment/control 之间随机跳动。系统分别保存：

- `strategy_versions`：策略快照和版本关系；
- `experiments`：灰度比例、状态和回放结果；
- `strategy_executions`：每次命中、分桶、执行结果和 outcome；
- `strategy_proposals`：待审核的变更建议。

运行指标显示候选劣化时，系统可以停用新版本并恢复上一策略快照。

### 人机协同边界

| 阶段 | 自动化范围 |
|---|---|
| `human_in_loop` | 所有候选必须人工审核后生效 |
| `human_on_loop` | 回放通过且置信度达到阈值的候选可自动灰度，其余进入审核 |
| `human_out_of_loop` | 仅在预先圈定范围内自动发布，并持续监控与回滚 |

新文档还会触发“自动出题—系统作答—部门审核—抽样复查”流程。只有正确率和样本量达到阈值后，对应部门才会逐步减少人工审核；后续抽检失败会自动退回更严格的阶段。

## 面向应用的加速设计

本项目所说的加速主要是应用侧加速，而不是单一算子或模型参数层面的硬件加速。目标是在不牺牲事实可靠性和治理能力的情况下，缩短有效答案路径并降低重复工作。

### 在线请求加速

- Redis 工作记忆直接恢复最近实体和部门，减少重复意图解析；
- 情景摘要代替完整历史消息，限制上下文随对话长度线性增长；
- 用户与组织记忆先提供检索线索，再回查官方 chunk，减少无目标的全库检索；
- Skill 复用已经验证的 query expansion、top-k 和模板，减少同类问题的重复规划；
- 部门 Agent 支持并行调用，跨部门问题允许部分成功，不被单个部门阻塞；
- `pi-agent` 分阶段设置 Intent、Rewrite、Answer、Verify 超时，失败时自动回退。

### 离线演进加速

- Trace 聚类把零散 bad case 合并为问题模式，减少逐条人工分析；
- 同题回放在发布前复用历史请求完成自动对照；
- 稳定灰度分桶让策略验证可以持续在线进行；
- Loop 使用异步作业，不占用在线问答请求；
- K8s 可按部门独立扩缩容，使热点部门不会拖慢全局服务。

系统通过 trace、`memory_usage`、策略执行记录和实验数据观察上下文长度、检索数量、Verifier 分数、端到端延迟、策略命中率和反馈变化。首 token、P99 和单轮成本目前属于工程优化目标，不能仅凭架构设计视为已经达到的实测结论。

## 关键代码位置

| 模块 | 位置 |
|---|---|
| 统一记忆上下文 | `backend/app/memory/context_builder.py` |
| 工作/情景/用户/组织记忆 | `backend/app/memory/working.py`、`episodic.py`、`user_semantic.py`、`organization.py` |
| 记忆策略与生命周期 | `backend/app/memory/policy.py`、`retention.py` |
| 自进化主循环 | `backend/app/loop/loop_engine.py` |
| Skill 挖掘与执行 | `backend/app/loop/skill_miner.py`、`skill_executor.py` |
| Hook 与 Rule | `backend/app/loop/hook_engine.py`、`rule_engine.py` |
| 策略回放 | `backend/app/loop/strategy_evaluator.py` |
| 问答集成 | `backend/app/harness/orchestrator.py` |
| 管理端可视化 | `web/src/components/admin/LoopPanel.tsx`、`InsightsPanel.tsx` |

## 技术栈

- 后端：Python 3.11、FastAPI、MongoDB、Redis Stream
- 前端：Next.js 15、React 19、TypeScript
- 检索：BM25、向量检索、Embedding、Reranker
- Agent 执行：pi-agent、DeepSeek、受控 tool calling
- 工程化：Docker Compose、Kubernetes、Helm、Prometheus

## 快速启动

### Docker Compose

```bash
cp .env.example .env
# 编辑 .env，填写模型服务密钥和生产安全配置
docker compose up --build -d
docker compose ps
```

启动后可访问：

| 服务 | 地址 |
|---|---|
| Web | http://localhost:8080 |
| FastAPI / OpenAPI | http://localhost:8000/docs |
| pi-agent health | http://localhost:8100/health |

导入演示数据与部门文档：

```bash
docker compose exec backend python -m scripts.seed_data
docker compose exec backend python -m scripts.ingest_department_files --base /app/department_files
```

### 本地开发

```bash
# backend
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
STORAGE_MODE=memory uvicorn app.main:app --reload --port 8000

# pi-agent
cd services/pi-agent
npm install
npm run dev

# web
cd web
npm install
BACKEND_URL=http://localhost:8000 npm run dev
```

## 关键配置

| 变量 | 含义 | 默认值 |
|---|---|---|
| `MEMORY_SESSION_TTL_SECONDS` | 工作记忆 TTL | `1800` |
| `MEMORY_EVENT_RETENTION_DAYS` | 会话事件保留时间 | `90` |
| `MEMORY_SUMMARY_RETENTION_DAYS` | 摘要保留时间 | `180` |
| `SKILL_MIN_CLUSTER` | 生成 Skill 的最小高频问题簇 | `20` |
| `SKILL_SANDBOX_MIN_SUCCESS` | Skill 沙箱成功率门禁 | `0.85` |
| `HOOK_HIGH_CONFIDENCE` | 自动激活 Hook 的置信度阈值 | `0.9` |
| `LOOP_PHASE` | 自进化治理阶段 | `human_in_loop` |
| `PI_AGENT_ENABLED` | 是否启用 pi-agent 执行平面 | `true` |
| `STORAGE_MODE` | MongoDB 或内存存储 | `mongo` |

生产环境必须修改 `AUTH_SECRET`、`INTERNAL_API_TOKEN`、MongoDB 密码和 Redis 密码，并关闭演示用户初始化。

## 验证

```bash
cd backend && .venv/bin/pytest -q
cd web && npm run build
cd services/pi-agent && npm run build
```

真实文档评测集位于 `backend/evaluation/real_document_qa.json`，可以运行：

```bash
cd backend
python -m scripts.evaluate_rag
```

评测输出包含 Recall@5、MRR、引用正确率和答案一致性。部门 Agent 的 1→20 副本负载测试见 `loadtest/README.md`。

## 进一步文档

- `backend/app/memory/README.md`：记忆边界、权威顺序和生命周期
- `docs/loop-engineering.md`：自进化流程、审核与灰度机制
- `docs/architecture.md`：服务拓扑与数据流
- `docs/api.md`：接口说明
- `docs/deployment.md`：Docker、K8s 与 Helm 部署
