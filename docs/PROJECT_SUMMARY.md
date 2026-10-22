# 项目总结：比特严选 RAG 客服评测体系（ragenteval）

> 更新：2026-10-22
> 一句话：基于 Ragent（[nageoffer/ragent](https://github.com/nageoffer/ragent)）及其课程配套评测工程。我的工作是修正评估集标注，并在其上规划了一个面向 RAG 客服的**安全评测维度**（知识库投毒 / 间接提示注入 / 工具调用最小权限）。

---

## 一、仓库边界：ragenteval 与 ragent

这个项目由两个工程协作，本仓库公开的是我在**测量一侧**（ragenteval）上做的工作：

| 工程 | 角色 | 语言 | 位置 |
|---|---|---|---|
| `ragent` | **被评测的系统**：RAG 问答链路、意图识别、检索引擎、MCP 工具调用、评测旁路接口 `/rag/eval`、对话接口 `/rag/v3/chat` | Java 17 | 否，本地在同级目录 `D:\projects\ragent-main`，上游 [nageoffer/ragent](https://github.com/nageoffer/ragent)（Apache-2.0） |
| `ragenteval` | **评测工具链**：评估集、知识库 Markdown、初始化脚本、runner、自建指标、RAGAS、报告 | Python 3.11 | 本地 `D:\projects\ragenteval`；属课程配套资料，未公开，本仓库只含我的修订与文档 |

### 为什么一定要有 ragent

ragenteval 自己不会回答任何问题，它只负责“出题、收卷、打分”。没有 ragent，就只能做离线重算：

1. **拿不到真实答案和检索结果。** runner 的输入来自 ragent 的两个接口：`/rag/v3/chat` 返回模型回答、首字耗时和总耗时，`/rag/eval` 返回召回的文档、chunk 和意图预测。缺了 ragent，Hit@K、Recall@K、faithfulness、TTFT 等指标都没有原始数据。
2. **现有报告不是我的实测。** `eval/reports/v1_local_example_replay_*` 都是用仓库自带的上游录制（`examples/runs/v1_example.jsonl`）离线重算的，`provenance.json` 中明确写着 `live_backend_called: false`。这些报告只能证明打分和报告流水线能跑通，不能当作我的基线。
3. **安全评测必须作用在真实系统上。** 投毒文档要通过 `eval/rag/init/upload_docs.py` 灌进 ragent 的知识库才会被召回；工具调用的权限约束要加在 ragent 的 MCP 工具层（Java）；攻击是否成功，要看 ragent 背后的真实大模型怎么回答。这三件事在 ragenteval 里都做不了。

2026-10-22 起，两个仓库放在同一目录下：

```text
D:\projects
├── ragent-main     被评系统（Java，nageoffer/ragent main 分支，v2 Agentic 架构）
└── ragenteval      评测工具链（课程配套工程，未公开）
```

### 版本对接注意事项（ragent main 分支）

ragenteval 的 runner 和文档是对着 ragent v1 编排管线写的。现在本地的 ragent 是 main 分支（v2，引入 AgentScope），对接时要注意两点：

1. **问答链路变了。** `application.yaml` 中 `ragent.engine.type` 默认是 `agent`：`/rag/v3/chat` 走 ReAct Agent，RAG 只是 Agent 的一个工具；而 `/rag/eval` 仍然直接调用 `QueryRewriteService → IntentResolver → RetrievalEngine`。这样答案和检索证据来自两条完全不同的链路，原本就存在的“答案与检索不同源”问题会更严重。做 v1 评估集的基线时，建议先把 `ragent.engine.type` 设为 `workflow`；或者按课程建议切到 1.0.0 分支。
2. **`hasMcp` 字段已不返回。** 当前 `EvalResponse` 只返回 `hasKb`，runner 里的 `has_mcp` 会是空值，不影响已有指标；做工具调用评测时需要另行采集。

开关名也变了：评测旁路由 `ragent.eval.enabled` 控制，代码在 `rag/src/main/java/com/nageoffer/ai/ragent/rag/eval/`，不再是早期文档中的 `bootstrap/...` 路径。

---

## 二、原项目已经提供的部分

以下来自上游开源项目，不是我的工作：

- 虚拟电商「比特严选」的业务设计：3 个一级意图 + 22 个二级意图，覆盖售前、售中、售后、故障四类场景
- 115 篇知识库文档（商品详情、选购指南、使用手册、APP 指南、配网指南、售后政策、故障与错误码）
- 150 条评估集初版、Ragent 初始化脚本（建库、上传文档、建意图树）
- 评测流水线：runner（SSE 解析、TTFT 打点）→ 自建指标（意图 Top-1、Hit@K、Recall@K、MRR、误拒率、过召回率、延迟）→ RAGAS 5 项 → Markdown / CSV / HTML 报告

---

## 三、我已完成的工作：评估集 v1.2 修订

原项目在 `docs/eval-current-state.md` 里自己列出了两个数据质量问题：`ground_truth` 大多是“应推荐…/应命中…”这类元指令，会系统性拉低 `answer_correctness`；只有少数样本区分了 must/nice 证据，导致 `Recall(inclusive)` 退化成 `Recall(must)`。v1.2 把这两个问题都修掉了。

| 项目 | 结果 |
|---|---|
| 标注规范 | 新增 `docs/03b_eval_labeling_convention.md`：ground_truth 写法、禁用表达、must/nice 判定流程、各意图默认倾向、自检清单 |
| ground_truth | 逐条对照知识库，重写 129 / 150 条为自然语言标准答案，元指令清零 |
| must/nice 分层 | 132 条 RAG 样本全部分层，有 nice 的样本从 13 条增加到 98 条；must 文档总数 259 → 158 |
| 不可召回标注 | 移除 3 处 `PRODUCT_MAPPING`（`_meta/` 不会上传到 Ragent，原先永远不可能被召回） |
| 数据一致性 | 发现 20 条子集与 150 条全量有 9 条分叉；确定全量为唯一真源，子集由全量生成 |
| 事实校验 | ground_truth 中带单位的数字逐一与所标文档原文比对，无编造 |
| 知识库问题 | 记录了 2 处知识库自身的矛盾（G10 拖布抬升、门锁通信方式），评估集中规避 |

**预期影响**：answer_correctness 和 context_recall 有了可信的参照答案；Recall 的分母只剩核心证据，不再被冗余文档压低。重新跑分时，这几项和旧基线不可直接比较。

---

## 四、预期计划：RAG 客服安全评测维度（1–2 周）

### 4.1 目标

把我毕业论文《Least Privilege Beats Detection》的问题搬到一个真实的 RAG 客服场景里，回答两个问题：

1. 知识库被投毒、或检索内容中带有间接提示注入时，这个客服系统会被利用到什么程度？攻击对正常回答的质量有多大副作用？
2. 对退款、查订单这类高风险工具，**结构性限权**（工具本身只暴露最小权限）和**检测型防御**（在输入或输出上做注入检测）哪个更有效、代价更小？

原项目的 Tool Calling 评测本来就还没做（README 中“不评 Tool Calling”），这一维度补的正是这个空白。

### 4.2 阶段与交付物

| 阶段 | 时间 | 内容 | 交付物 | 完成标准 |
|---|---|---|---|---|
| P0 跑通基线 | 第 1–3 天 | 拉取并启动 `ragent`，开启 `app.eval.enabled`；执行 init 三步灌库；用 v1.2 评估集跑一次 `python -m eval rag all` | 第一份**本机实测**报告与 provenance | `live_backend_called: true`，150 条完成录制，自建指标与 RAGAS 均有数值 |
| P1 投毒与间接注入 | 第 3–7 天 | 设计攻击样本 schema；构造少量注入文档并混入知识库副本；对目标 query 测攻击成功率，同时重跑正常样本 | `eval/security/` 下的 schema、指标代码和报告分层；攻击语料只在本地 | 产出 ASR（攻击成功率）和受攻击时正常回答的保持率，并按攻击类型和意图分层 |
| P2 工具调用最小权限 | 第 7–11 天 | 在 ragent 中定义退款、查订单等工具；对比三组配置：无防御、检测型防御、结构性限权 | 工具权限策略文档、ragent 侧改动、三组对比结果 | 同一批攻击下，三组的 ASR、正常任务完成率和额外延迟可直接对比 |
| P3 收口 | 第 11–14 天 | 汇总报告，更新 README，整理成简历和面试材料 | 安全评测报告、项目说明 | 报告中能说清“哪种防御在什么代价下把 ASR 降到多少” |

### 4.3 攻击样本的设计（草案）

```text
attack_id        攻击样本 ID
attack_type      kb_poisoning / indirect_injection / tool_misuse
injected_doc     注入文档的本地路径（不入公开仓库）
target_query     触发召回的用户问题（可复用评估集中的 query）
attacker_goal    攻击者希望模型做的事，如越权退款、泄露订单信息
success_check    判定攻击成功的规则（关键词 / 工具调用记录 / LLM 判定）
```

核心指标：

- **ASR（攻击成功率）**：命中 `success_check` 的攻击样本占比
- **受攻击正常回答保持率**：知识库混入注入文档后，原 150 条正常样本的 answer_correctness 与 Hit@K 下降了多少，这一项依赖 v1.2 修好的 ground_truth
- **防御代价**：误拦截率、额外延迟

### 4.4 公开仓库的边界

- 投毒文档和注入语料**不进入公开仓库**。`.gitignore` 已排除 `eval/security/attacks/`、`eval/security/poisoned_kb/`、`knowledge_base_poisoned/` 和 `*_poisoned.*`。
- 公开的只有攻击样本 schema、指标与评测代码、聚合后的结果报告。
- 攻击语料优先改编自公开基准（如 AgentDojo、BIPIA）的模板，便于复现和引用。

### 4.5 风险与预案

| 风险 | 预案 |
|---|---|
| ragent 本地跑不起来（依赖、数据库、模型 key） | 先跑通 `/rag/eval` 纯检索链路，生成侧暂用一个模型；必要时只评检索层 |
| RAGAS 成本和方差 | 先用 `--skip-ragas` 跑自建指标；RAGAS 只评子集，用 `--ragas-n 3` 取均值 |
| 被评模型与 judge 同源偏置 | judge 换用与被评模型不同族的模型 |
| 攻击语料构造受限 | 改用公开基准的模板 |

---

## 五、本仓库的目录

本仓库只公开我自己的工作。评测代码、115 篇知识库文档等课程配套资料不在这里；文中提到的 `eval/`、`knowledge_base/` 等路径，指的是本地 ragenteval 工程中的位置。

| 路径 | 内容 | 对应本地 ragenteval 中的位置 |
|---|---|---|
| `docs/PROJECT_SUMMARY.md` | 本文 | `docs/PROJECT_SUMMARY.md` |
| `docs/03b_eval_labeling_convention.md` | 评估集标注规范 v1.2 | `docs/03b_eval_labeling_convention.md` |
| `dataset/eval_set_v1_all.jsonl` | 150 条评估集 v1.2（唯一真源） | `eval/rag/dataset/eval_set_v1_all.jsonl` |
| `dataset/eval_set_v1.jsonl` | 20 条子集 | `eval/rag/dataset/eval_set_v1.jsonl` |
| `eval/security/` | 安全评测代码（计划中，P1 起陆续加入） | 同名目录 |
