---
name: jev-life
description: Use when a user describes a concrete life situation they're stuck deciding — typical triggers are "要不要X"、"该不该X"、"是A还是B"、"我该怎么选" about jobs, relationships, family conflicts, health choices, finances, roommate conflict, pregnancy, layoffs. Works for binary choices (A vs B) or single decisive actions (move out, sign agreement, tell the boss). Returns a probability-weighted comparison via typesafe's Jev API using Choice+Noul+Score together — the distribution of outcomes, not a flat verdict. Skip for factual lists, future predictions, product picks, or open-ended questions without a defined decision. Requires TYPESAFE_API_KEY.
---

# jev-life

用 Jev 的三个 primitive（Choice / Noul / Score）共同评估一个生活情境。每次调用必三件套同时出现——这就是「三元语」。

## 前置条件

环境变量 `TYPESAFE_API_KEY` 必须存在。它作为 `Authorization: Bearer <TYPESAFE_API_KEY>` 的 key 传入 `https://api.typesafe.ai/v1/systemone`。缺失时**直接报错告知用户**，不要尝试降级到本地启发式——本技能的全部价值建立在 Jev 的判断上。

## 工作流

### Step 1 · 收集场景

向用户索取两个字段：

- **背景**：事情发生在什么环境里、涉及谁、约束是什么
- **实际情况**：已经发生或正在发生的事实

完成判据：两个字段都拿到用户的实际表述，而非模型自己补全。如果用户只给一项，只追问缺失项，不补全不编造。

### Step 2 · 生成候选策略并构造三元语

把候选策略构造为 Jev 的 Choice `criteria`（注意：**候选是你定义的，不是 Jev 生成的**）。同时为每条候选配上一个 Noul（成功概率）和一个 Score（质量等级），构成一次请求的三件套。

完成判据：

- `questions` 里同时存在且仅存在一个 Choice、一个 Noul、一个 Score
- Choice 的 `criteria` 是一个非空候选集合（一般 3–6 项，太少缺乏对比，太多概率分散无意义）
- 三件套共用同一份 `state`（背景 + 实际情况拼接成的字符串或对象）
- 三件套针对的是同一组候选，不是各问各的

如果场景太抽象无法收敛到一组候选（例如「我应该怎么活」），停下来告诉用户问题太宽，请他聚焦到一个具体决定。

### Step 3 · 调用 Jev

向 `POST https://api.typesafe.ai/v1/systemone` 发送请求，body 形如：

```json
{
  "model": "jev-latest",
  "state": "<背景>\\n\\n<实际情况>",
  "questions": {
    "choice_<slug>": { "type": "choice", "instructions": "...", "criteria": {...} },
    "noul_<slug>":   { "type": "noul",   "instructions": "...", "criteria": {"true": "...", "false": "..."} },
    "score_<slug>":  { "type": "score",  "instructions": "...", "criteria": ["...", "...", "..."] }
  }
}
```

请求体字段、`instructions`、`criteria` 的合法结构详见 [`references/api-format.md`](references/api-format.md)。

完成判据：收到 HTTP 2xx 且 `answers` 包含全部三个 question_id。

如果返回 4xx/5xx 或网络失败，**不要伪造判断**。把错误原文（含 status code、本文 key 缺哪一段）转给用户，告诉他无法得出最终结果，请他检查 `TYPESAFE_API_KEY` 是否有效、余额是否充足、或场景是否触发了上游拒绝。

### Step 4 · 呈现三元语结果

按 Choice / Noul / Score 顺序展示 Jev 返回的内容，**让用户看到原始概率而非被抹平的结论**。三元语结果最少包括：

- **choice**：被选中的策略 + 其他候选的概率分布（不只是赢家）
- **Noul**：0–1 的 yes 概率，以及 true/false 各自的语义
- **score**：在 Score 等级上的概率加权位置（例如落在「良好」档的概率 0.62、「一般」0.31、「差」0.07）

每个数值后用一句话点出「这意味着什么」，但不要替用户做最终行动决定——他比你更知道自己的偏好权重。

完成判据：三件套的答案都向用户呈现过，每个答案附了一句以上的人工解读。

## 失败处理

| 情况 | 行为 |
| --- | --- |
| `TYPESAFE_API_KEY` 未设置 | 拒绝运行，告知需在环境变量中配置 API Key |
| API 返回 401/403 | 告知 Key 无效或无权限，不要重试 |
| API 返回 429 | 告知限流，建议稍后重试 |
| API 返回 5xx 或网络错误 | 把错误原文转给用户，建议检查服务状态后重试 |
| `answers` 缺任意一个 | 视为本次判断不完整，告知缺失的 question_id，不补全 |
| 用户场景无法收敛为候选集合 | 停下来请用户聚焦，不要强行构造 |

## 约束

- **不编造** Jev API 的字段、状态码、错误格式。所有 API 知识见 [`references/api-format.md`](references/api-format.md)，如与官方文档冲突以官方为准（https://docs.typesafe.ai/api.md）。
- **不省略三件套**。三元语是 Choice+Noul+Score，少一个就不是 jev-life 在工作。
- **不替用户决策**。呈现概率和解读，让用户自己拍板。
- **不绕过缺失 Key**。没 Key 就明说，不要用启发式顶替。

## 参考

- [`references/api-format.md`](references/api-format.md) — 三件套完整的请求/响应示例、字段语义、错误码
- [`references/primitives.md`](references/primitives.md) — Choice/Noul/Score 三件套各自的语义与构造要点