# Jev API 格式参考

完整官方文档：https://docs.typesafe.ai/api.md

## 端点

```
POST https://api.typesafe.ai/v1/systemone
Authorization: Bearer ${TYPESAFE_API_KEY}
Content-Type: application/json
```

## 请求体顶层

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `model` | string | 是 | 当前写 `"jev-latest"` |
| `state` | string \| object \| array | 是 | 评估的内容；本技能里填「背景 + 实际情况」拼接 |
| `questions` | map<string, Question> | 是 | 三元语三件套放这里，key 由你命名 |

`questions` 的 key 是给代码用的，不会发给模型。**为 jev-life 建议用带前缀的命名**：`choice_<slug>`、`noul_<slug>`、`score_<slug>`，便于在 `answers` 里快速对位。

## 三件套的 Question 形态

每件都有 `type` 和 `instructions`；`criteria` 形态各不相同。

```jsonc
// Choice — 从候选集合里选一个
{
  "type": "choice",
  "instructions": "Which strategy best fits this situation?",
  "criteria": {
    "<option_key>": "<rubric 描述这个选项>",
    "<option_key_2>": "<rubric 描述>",
    // 最多 255 个 option
  }
}
```

```jsonc
// Noul — 二元 yes/no
{
  "type": "noul",
  "instructions": "Will this strategy work in this context?",
  "criteria": {
    "true":  "<true 接近 1 时意味着什么>",
    "false": "<false 接近 0 时意味着什么>"
  }
}
```

```jsonc
// Score — 有序等级
{
  "type": "score",
  "instructions": "How would you rate the overall quality of this strategy?",
  "criteria": [
    "<最差档描述>",
    "<中间档描述>",
    "<最好档描述>"
    // 2–10 项
  ]
}
```

`instructions` 支持 string / object / array。如果问题需要拆成「问题 + 数据」，可用对象形式把数据字段放在一边，问题里用反引号引用字段名（参考官方文档「Structured instructions and criteria」）。

## 响应体

```json
{
  "model": "jev-latest",
  "answers": {
    "choice_<slug>": { /* Choice 答案 */ },
    "noul_<slug>":   { /* Noul 答案 */ },
    "score_<slug>":  { /* Score 答案 */ }
  },
  "usage": { /* token 计数 */ }
}
```

每个 `answers.<key>` 的内容（精确字段名以官方文档为准，下面给方向）：

- **Choice**：被选中的 option_key + 各 option 的概率分布
- **Noul**：0–1 的 yes 概率
- **Score**：在 `criteria` 各等级上的概率分布

`usage` 提供 token 用量，本技能目前不用它做限流控制，但若用户问起可以转给他。

## 错误

| Status | 含义 | 怎么处理 |
| --- | --- | --- |
| 401 / 403 | Key 无效或无权限 | 停止，告诉用户检查 `TYPESAFE_API_KEY` |
| 429 | 限流 | 告诉用户稍后重试 |
| 4xx（非上述） | 请求体不合法 | 检查 `questions` 结构、`model` 字段名 |
| 5xx / 网络错误 | 上游异常 | 把原文转给用户，建议重试 |

不要根据 4xx 响应里模糊的字段名「合理猜测」修复——直接告知用户报错原文。

## 三元语对齐示例

一次完整的 jev-life 请求大致长这样（场景：是否要换工作）：

```json
{
  "model": "jev-latest",
  "state": "背景：在当前公司 3 年，团队氛围一般但稳定。\\n\\n实际情况：收到一家创业公司的 offer，薪资涨 40% 但要 996。",
  "questions": {
    "choice_action": {
      "type": "choice",
      "instructions": "Which action best fits the situation?",
      "criteria": {
        "stay": "Stay at current job for stability",
        "negotiate": "Stay but negotiate a counter-offer",
        "switch": "Take the startup offer for growth and higher pay"
      }
    },
    "noul_success": {
      "type": "noul",
      "instructions": "Will the chosen action lead to better long-term outcome?",
      "criteria": {
        "true": "Action aligns with user's actual priorities",
        "false": "Action conflicts with unstated priorities"
      }
    },
    "score_quality": {
      "type": "score",
      "instructions": "Rate the overall quality of the chosen action.",
      "criteria": ["Poor", "Mediocre", "Good", "Excellent"]
    }
  }
}
```