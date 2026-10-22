# ragent-security-eval

面向 RAG 客服系统的评测改进与安全评测。被评系统是 [nageoffer/ragent](https://github.com/nageoffer/ragent)（企业级 Agentic RAG，Apache-2.0），业务场景是课程中的虚拟电商「比特严选」客服。

本仓库**只包含我自己的工作**。评测流水线代码和 115 篇知识库文档属于知识星球《AI大模型Ragent项目》RAG 评测章节的课程配套资料，不在这里公开。

## 内容

| 路径 | 说明 |
|---|---|
| [`docs/PROJECT_SUMMARY.md`](docs/PROJECT_SUMMARY.md) | 项目总结：两个工程的边界、已完成工作、安全评测计划（1–2 周）、与 ragent v2 的对接注意事项 |
| [`docs/03b_eval_labeling_convention.md`](docs/03b_eval_labeling_convention.md) | 评估集标注规范 v1.2：ground_truth 写法、must/nice 证据分层规则、自检清单、修订记录 |
| `dataset/eval_set_v1_all.jsonl` | 150 条评估集，v1.2 修订版（唯一真源） |
| `dataset/eval_set_v1.jsonl` | 20 条子集，由全量按 `query_id` 生成 |
| `eval/security/` | 安全评测代码，计划中 |

## 已完成：评估集 v1.2

原评估集有两个会让指标失真的问题：`ground_truth` 大多是“应推荐…/应命中…”这类给模型的指令，而不是真实答案；必需证据（must）和扩展证据（nice）基本没有区分。v1.2 的修订：

- 制定标注规范，对照知识库重写 129 / 150 条 `ground_truth`，元指令清零
- 132 条 RAG 样本全部完成 must/nice 分层，must 文档总数 259 → 158
- 移除 3 处永远无法被召回的标注（`_meta/` 文件不会灌入 ragent）
- 修复 20 条子集与 150 条全量之间 9 条内容分叉
- `ground_truth` 中的数字逐一与所标文档原文比对

字段说明：

```json
{
  "query_id": "S6-05",
  "query": "我有小米 13 的 67W 充电器，给小米 14 Pro 用会慢吗？",
  "intent_l1": "SUPPORT", "intent_l2": "S6_配件兼容",
  "requires_rag": true,
  "expected_doc_ids": ["PROD_PHONE_001", "PROD_PHONE_004"],
  "expected_doc_ids_nice": ["CODE_PHONE_001"],
  "ground_truth": "可以用，但会比原装慢。小米 14 Pro 标称 120W 有线秒充……"
}
```

`expected_doc_ids` 是 must，用于 Hit@K / Recall@K / MRR；`expected_doc_ids_nice` 是 nice，与 must 合并后用于 Recall inclusive@K；`ground_truth` 作为 RAGAS 的 `reference`。文档 ID 对应课程知识库中的文件名。

## 计划：安全评测维度

把 *Least Privilege Beats Detection* 的问题放到真实的 RAG 客服场景中验证：

1. **P0**：跑通 ragent，用 v1.2 评估集拿到本机实测基线
2. **P1**：知识库投毒与间接提示注入，测攻击成功率（ASR）以及受攻击时正常回答的保持率
3. **P2**：对退款、查订单等工具，对比无防御、检测型防御和结构性限权三组配置
4. **P3**：汇总报告

细节见 [项目总结第四节](docs/PROJECT_SUMMARY.md#四预期计划rag-客服安全评测维度12-周)。

**投毒文档和注入语料不会公开。** `.gitignore` 已排除对应目录，公开的只有样本 schema、指标代码和聚合结果。

## 致谢

- [nageoffer/ragent](https://github.com/nageoffer/ragent)：被评系统
- 知识星球《AI大模型Ragent项目》：业务场景、意图体系、知识库与评测流水线
- [RAGAS](https://github.com/explodinggradients/ragas)：LLM-as-judge 指标
