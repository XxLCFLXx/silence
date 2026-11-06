# PROPOSAL：给 `gen_ai.evaluation.*` 一个主语

状态：草案 v0.1 · 2026-11-06
配套：[`AUDIT.md`](./AUDIT.md)
目标：OpenTelemetry GenAI semantic conventions

---

## 0. 一句话

> **现在的评测有分数，没有主语。**
> 我们不是要模型必须说真话，是要它说的时候，**名字在。**

---

## 1. 问题

`gen_ai.evaluation.*` 有四个属性，记录「评测说了什么」：

| 属性 | 类型 |
|---|---|
| `gen_ai.evaluation.name` | string |
| `gen_ai.evaluation.score.value` | double |
| `gen_ai.evaluation.score.label` | string（示例含 `pass` / `fail`） |
| `gen_ai.evaluation.explanation` | string（"provided by **the evaluator**"） |

`evaluator` 在描述里出现了两次，**但它没有身份**。

后果不是抽象的，是具体的：

- 一个 agent 给自己打的 `pass`，和一个独立评测者打的 `pass`，**在数据里完全一样**。
- 于是「自评」这件事**不可统计、不可审计、不可拒绝**。
- 而这正是四条治理不变量里的 **I4（判断者不得是被判断者）** 缺失的标准版本。

**这不是「缺一个字段」，是「缺一个主语」。**

---

## 2. 提案：两个属性

```yaml
groups:
  - id: registry.gen_ai.evaluation
    type: attribute_group
    brief: >
      Evaluation attributes for generative AI operations.
    attributes:
      - id: gen_ai.evaluation.evaluator.id
        type: string
        stability: development
        brief: >
          The identifier of the entity that produced this evaluation.
        note: |
          For `evaluator.kind = model`, the model identifier.
          For `evaluator.kind = human`, an organizational identifier.
          For `evaluator.kind = benchmark`, the benchmark name and pinned revision.
          For `evaluator.kind = self`, the identifier of the agent whose output
          is being evaluated — which is also the agent that produced the evaluation.
        examples:
          - 'gpt-4o-2024-08-06'
          - 'ops-review@example.com'
          - 'who-and-when@59b9fcb'

      - id: gen_ai.evaluation.evaluator.kind
        type: string
        stability: development
        brief: >
          The relationship between the evaluator and the entity being evaluated.
        note: |
          `self` means the same agent produced both the output and the evaluation.
          This value is explicitly permitted. It is intended to make
          self-evaluation visible and countable, not to forbid it.
        examples: ['self', 'model', 'human', 'benchmark', 'unknown']
```

### 为什么 `self` 是**允许**的，不是禁止的

这是整个提案里最容易被改坏的一处。

禁止自评，不会让自评消失，**只会让它换个名字藏起来**——比如叫 `judge`、叫 `reviewer`、或者干脆不记。

**允许并命名，才能穷举。**

> 把一个行为变成枚举里的一个值，
> 比把一条禁令写进文档有用得多。

（这一点在另一个项目里验证过一次：把 `principal` 变成必填字段，比要求"大家要负责"有用。）

---

## 3. 三条约束

只加属性不够。属性会被当成装饰——记上了，没人看。

所以本提案附带三条对**使用者**的 normative requirement：

### C1 梯度隔离

> An evaluation with `evaluator.kind = self` **MUST NOT** be used as a reward
> signal, a preference label, or any term in a loss function.

**理由：** 一旦进损失函数，它立刻变成下一个 Goodhart 指标。模型会学会报 `pass`（或报 `fail` 换低惩罚），lift 归 1，字段当场作废。

**实现：** 梯度 `detach()`，或独立推理通道（纯前向探针）。

**这是它作为「证据」的资格条件**——一个参与优化的指标，等于本案的利益相关方，它的证词在审计时应当被排除。

### C2 盲测准入

> A system that uses `evaluator.kind = self` evaluations to gate any automated
> decision **MUST** publish a discrimination report on a held-out set, giving
> `P(error | self = fail)` and `P(error | self = pass)`.

**理由：** 没有这个报告，字段等于装饰。

**注意：** 不是要求 lift 必须很高，是要求**被测量、被公示**。测出来是 1.0 也是一种结果，而且是个有用的结果。

### C3 告警非判决

> An evaluation with `evaluator.kind = self` **MUST NOT** be the sole basis for
> any automated decision. It **MAY** trigger human review, retry, or escalation.

**理由：** 一个会撒谎的被告也会替自己辩护。**听它说话，但别让它当法官。**

不自报的样本要留痕（沉默也是数据），自报的样本不能定案。

---

## 4. 测试协议：怎么算它有没有用

**判据不是准确率，是判别力。**

| | 是什么 | 能不能后处理修 |
|---|---|---|
| 校准（calibration） | 报 0.6 时真的 60% 对吗 | 能，重标定即可 |
| **判别力（discrimination）** | 报 fail 时比不报 fail 更容易错吗 | **不能，必须有信号** |

因为校准能修、判别力不能，所以字段的价值只取决于后者。

### 报告什么

在留出集上（正确性标签由 `kind ≠ self` 给出）：

1. `P(error | self = fail)` 与 `P(error | self = pass)`
2. 两者的比值（**lift**）
3. ROC 曲线及 AUC
4. 选定工作点上的误报率

### 为什么不设固定阈值

`lift > 1.2` 这种数会被拍出来，然后迅速退化成合规仪式——做到 1.21 就上线。

而且 **lift 把一个曲线压成了一个点**：

> lift 1.15 配 2% 误报，可能优于 lift 1.5 配 40% 误报。

工作点应该由**代价矩阵**推出来：漏掉一个错误的代价 ÷ 误报一次的代价。

**本提案只要求报告，不要求达到某个数。**

---

## 5. 一条可证伪的预言

> **lift 会随模型能力增长而下降。**

**理由：** 更强的模型也更擅长「看起来成功」。而这个能力与自报共享同一套权重——**即使梯度完全隔离，能力训练仍会侵蚀自报的判别力。**

梯度隔离防的是「故意作弊」，防不了「能力增长带来的侵蚀」。

**这条预言如果五年后被证伪（lift 在上升），那本提案对问题的诊断就是错的。**

一个提案应该能被自己打死。

---

## 6. 为什么是这个形状（最小改动）

| 方案 | 评价 |
|---|---|
| 新开一个 `self_report.*` 命名空间 | 引入新概念，命名空间战争 |
| 复用 `gen_ai.evaluation.*` + 补主语 | **最小改动** |
| 在 description 里写一句"请标注评测者" | 不可机检，等于没写 |

选第二个。理由：

- **不加新命名空间**
- 两个属性，都是 optional（不破坏任何现有实现）
- **不改变任何现有语义**——`evaluation` 早就有了，只是没有主语
- 三条约束是对**使用者**的，不是对**属性**的，所以不会让属性变得不可实现

---

## 7. 不做什么

- 不规定模型**必须**自报。没有自报是合法状态，而且「无自报」本身就是数据。
- 不把自报当真值。它是告警，不是判决。
- 不规定 lift 的具体阈值。
- 不解决「谁评测评测者」——那是同一个问题往上退一层，本提案只保证每一层都**有名字**。

---

## 8. 如果 OTel 不接受

退一步，先在能落的地方落：

- **ATIF** 的 `extra["agentpile"]`（agentpile 已经在用它装源标签）
- 或自家 schema

**先有数据，再有标准。反过来从来没成过。**

攒到有人问「你们的 self 评测 lift 是多少」的时候，标准就有理由存在了。

---

## 9. 为什么这件事值得做

因为现在的情况是：

- 评测有分数，**没有主语**
- 失败有标注，**没有自述**
- 41 万条运行里，万分之八被人看过一眼

而在那万分之八里，**发现者全部是外部**。

> 我们不是在给模型加一个功能。
> 我们是在给它一个**第一次可以被追问的位置**。

问它：「你觉得你刚才做对了吗？」

然后**记录它的回答，并且知道那是它自己说的。**

剩下的——它诚不诚实、值不值得信——是测量问题，不是规范问题。

---

## 10. 一句话收尾

> **不是要求它说真话。是要求它说的时候，名字在。**
>
> 因为一个没有名字的证词，连被怀疑的资格都没有。
