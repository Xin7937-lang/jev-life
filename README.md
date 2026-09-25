# jev-life

用 [typesafe Jev](https://docs.typesafe.ai/) 的 **Choice + Noul + Score** 三件套，为生活场景做概率化判断。不替你拍板——把每个选项的成功概率和质量档位一起摆出来，由你做最终决定。

> 「三元语」= Choice（从候选策略中选一个）+ Noul（成败概率）+ Score（质量等级）。三者共享同一份 `state`，一次调用、三个原始概率。

---

## 为什么

普通对话式建议的问题是「自信但不校准」——一句"建议你接受 offer"既可能是 90% 的好建议，也可能是 50/50 的赌博。Jev 是 [typesafe](https://docs.typesafe.ai/) 的 System One 模型，专门返回**校准过的概率分布**而不是文字结论。它不擅长创作，但擅长「在给定信息下，这件事有多大可能是真的」。

`jev-life` 把这种概率化判断用在**生活决策**上：换工作、要不要分手、是否接受创业 offer、要不要搬家。决策后果大、容错低，所以概率值多 5% 少 5% 都重要。

---

## 安装

### 前置

- Claude Code（`claude` CLI 可用）
- 一个 [typesafe](https://docs.typesafe.ai/) 账户 + API Key

### 步骤

**1. 把技能文件放到 Claude Code 的 skills 目录**

```bash
# macOS / Linux
mkdir -p ~/.claude/skills/jev-life/references
cp SKILL.md ~/.claude/skills/jev-life/
cp references/*.md ~/.claude/skills/jev-life/references/

# Windows PowerShell
$dest = "$env:USERPROFILE\.claude\skills\jev-life\references"
New-Item -ItemType Directory -Force -Path $dest
Copy-Item SKILL.md "$env:USERPROFILE\.claude\skills\jev-life\"
Copy-Item references\* $dest
```

**2. 配置 API Key**

```bash
# macOS / Linux
export TYPESAFE_API_KEY="<你的 typesafe API key>"

# Windows PowerShell
$env:TYPESAFE_API_KEY = "<你的 typesafe API key>"
```

也可以写到 `~/.claude/.env` 里让所有会话自动加载：

```
TYPESAFE_API_KEY=apikey_...
```

**3. 重启 Claude Code 让配置生效**

---

## 真实场景用例

### 场景 1 · 换工作

> "我同时拿到两个 offer，A 公司给 60 万但 996，B 公司给 45 万但完全 wlb。我该怎么选？"

技能会把候选收敛成 3–6 个具体选项（接 A、接 B、谈 counter、骑驴找马等），用三件套一次性问 Jev：哪个最合适 / 成功率多少 / 整体质量怎样。返回的不是"建议去 B"，而是"接 A 的概率 0.41、接 B 0.33、谈 counter 0.18... 选中策略的成功率 0.62，质量落在『良好』档概率 0.58"。

### 场景 2 · 50 万存款怎么投

> "我有 50 万存款，现在股市看起来要涨，要不要 All-in 一只科技股？"

技能把 All-in 视为「待评估的候选之一」而不是默认答案——它会拉出"All-in / 分批建仓 / 分散到指数 / 保留现金"等候选，让 Jev 用概率告诉你「All-in 的成功率只有 0.18，最稳妥的候选是『分散到指数』，选中概率 0.46」。

### 场景 3 · 室友冲突

> "我室友每周都带朋友回家开 party 到凌晨两三点，跟她说两次没用，要不要直接搬走？"

技能不会直接劝你"搬"或"忍"，而是把可能动作列成候选（直接搬 / 再谈一次 / 写书面规则 / 调自己预期...），让 Jev 评估每条路径的成功概率和你的生活质量得分。

### 场景 4 · 关系取舍

> "我妈疯狂催我相亲，28 岁了还没对象每天念叨。是硬着头皮去见她安排的，还是再拖一年？"

候选可能是「去一次 / 拖一年 / 直接跟妈摊牌 / 自己主动找对象」等，Jev 会基于你给的实际信息评估哪条路对你最不糟。

### 场景 5 · 健康决策

> "我爸刚体检查出肺部结节，医生给了两个方案：穿刺活检 vs 直接微创切除。怎么选？"

候选由你定义（保守 / 激进 / 转诊 / 等等），三件套同时评估，原始概率还给你。

---

## 怎么提问

技能的 description 锚定了 4 类典型触发：

- 「要不要 X」「该不该 X」—— 一个待定的决定
- 「是 A 还是 B」「我该怎么选」—— 二选一或多选一
- 单一动作：「要不要搬走」「要不要直接说」「要不要签字」
- 中文生活语境：工作、关系、家庭矛盾、健康、财务、搬家、怀孕、裁员

**不要用它做**：事实清单、未来预测（"明年房价会涨吗"）、产品推荐、开放式问题。

---

## 配置参数

技能本身只读 `TYPESAFE_API_KEY` 一个环境变量。其他参数（如 `model`、`runs_per_query`）走 typesafe 自己的默认值，目前没有技能级的可调参数。

---

## 文件结构

```
.
├── SKILL.md              # 技能主体：4 步工作流 + 完成判据 + 失败处理
├── references/
│   ├── api-format.md     # typesafe API 字段、错误码、三件套 JSON 示例
│   └── primitives.md     # Choice/Noul/Score 语义与构造反模式
├── evals.json            # 3 个功能评测（换工作 / 投资 / 室友冲突）
├── trigger_eval.json     # 20 个 description 触发评估 query
├── trigger_eval_review.html  # trigger eval 的交互式 review 界面
├── iteration-1/
│   ├── eval-{0,1,2}/      # 3 个功能评测的 with-skill / without-skill 输出
│   └── benchmark.json     # 评测结果摘要
└── review.html           # iteration-1 静态 viewer
```

---

## 失败处理

| 情况 | 技能表现 |
|---|---|
| `TYPESAFE_API_KEY` 未设置 | 拒绝运行，告知如何配置——不会用本地启发式冒充 |
| Key 无效（401/403）| 拒绝重试，提示检查 Key |
| API 5xx 或网络错误 | 把错误原文转给用户，不伪造判断 |
| 用户场景无法收敛到候选集合 | 停下来请用户聚焦，不强行构造 |

---

## 已知限制

- **没有 API Key 时**技能完全停摆——这是有意设计，宁可拒绝也不冒充
- **校准不等于准确**：Jev 返回的概率是基于训练分布的校准，不是预言
- **不替用户做最终决定**：技能给概率和解读，决策权重由用户掌握

---

## 评测

`iteration-1/` 目录下是 3 个功能评测（换工作 offer / 50 万 all-in / 室友洗碗）的完整对比：
- **with_skill/**：技能触发并走完工作流（本环境无 API Key，走的是干净的拒绝路径）
- **without_skill/**：基线 Claude 直接给出对话建议
- **benchmark.json**：对比摘要

`trigger_eval.json` 是技能 description 的触发评估 query 集（10 should-trigger + 10 should-not），用于 description 调优。

---

## License

本项目技能定义遵循 MIT。typesafe Jev 的使用受 [typesafe 服务条款](https://docs.typesafe.ai/)约束。