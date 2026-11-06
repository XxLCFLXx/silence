# silence

> **One spec defines an `evaluator` role for agent traces — and never says who cannot sit in it.**
> 有一份规范定义了「评测者」这个角色，却没有规定谁不能坐这个位置。

在已查的四份格式里，`self` / `report` / `confidence` / `uncertain` / `abstain`
五个词，**零命中**。

但位置不是零。

**一个 agent 给自己打 `pass`，在规范上完全合法。**

---

## 四份格式，两种缺口

| 来源 | 有位置吗 | 有主体约束吗 |
|---|---|---|
| Who&When（184 条） | ❌ | — |
| MAST（1,642 条） | ❌ | — |
| agentpile / ATIF | ❌ | — |
| **OpenTelemetry GenAI** | **✅ `gen_ai.evaluation.*`** | **❌** |

三份没有位置。一份有位置，但没有名字。

---

## 那份有位置的规范，长这样

`gen_ai.evaluation.*`，四个属性：

| 属性 | 说明（原文） |
|---|---|
| `gen_ai.evaluation.name` | The name of the evaluation metric. |
| `gen_ai.evaluation.score.value` | The evaluation score returned by **the evaluator**. |
| `gen_ai.evaluation.score.label` | Human readable label. 取值示例含 **`pass`**、**`fail`** |
| `gen_ai.evaluation.explanation` | Explanation provided by **the evaluator**. |

「评测者说了什么」有字段。

**「评测者是谁」没有字段。** 属性清单里没有 `gen_ai.evaluation.evaluator.*`。

**也没有任何条款要求 evaluator ≠ agent。**

> 规范定义了一个角色，却没说谁不能坐。

---

## 三份没有位置的，长这样

数据里有四个字段，四个问题：

| 字段 | 回答什么 | 谁给的 |
|---|---|---|
| `is_correct` | 这次**对不对** | 与 ground truth 比对 |
| `mistake_agent` | **谁**错了 | 标注者 |
| `mistake_step` | **哪一步**错的 | 标注者 |
| `mistake_reason` | **为什么**错 | 标注者 |

覆盖得相当完整。**唯独没有字段回答：它自己知不知道自己错了。**

---

## MAST 把这件事摆得最直白

```
annotations: [{
    "failure mode": string,
    "annotator_1":   bool,
    "annotator_2":   bool,
    "annotator_3":   bool
}]
```

**三个字段回答「哪个人认为它失败了」。
零个字段回答「它自己认为失败了吗」。**

四种设计，同一个缺席：

| 来源 | 「谁发现」怎么写 | agent 在里面吗 |
|---|---|---|
| Who&When | `mistake_agent`（标注得出） | ❌ |
| MAST | `annotator_1/2/3` | ❌ |
| agentpile | `label_source ∈ {expert, LLM judge, benchmark}` | ❌ |
| OTel | 没有这个字段，只有 `evaluator` 这个**角色** | ❌ |

---

## 唯一能放自报的地方，是散文

`history` / `trajectory` 是自由文本，agent 当然可以写「我不确定」。

但那不算字段：**不可查询、不可比较、不在 schema 里。**

> **结构化的自报不存在，只有散文。**

---

## 那个数

`agentpile` 覆盖 14 个数据集、约 **410,000 条运行**。
其中做过步级 / span 级错误定位标注的：

```
Who&When 184 + TRAIL 148 = 332
332 / 410,000 ≈ 0.08%
```

> **万分之八的运行，被定位到过具体哪一步出错。**
> 剩下的 99.92%，不是没有错，是**没有人看**。

（TRAIL 在 Hugging Face 上是 gated，数量取自 agentpile 的清单，未直接查 schema。）

---

## 复现

不需要下载数据，不需要认证：

```bash
curl "https://datasets-server.huggingface.co/info?dataset=Kevin355%2FWho_and_When"
curl "https://datasets-server.huggingface.co/info?dataset=mcemri%2FMAST-Data"
curl -s "https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/registry/attributes/gen-ai.md"
```

**任何人都能重跑，核对本仓库的每一句。**

---

## 这不是什么

- 不是「模型没有不确定性」的证明。**内部表征有没有，读 schema 看不出来。**
- 不是「模型从来不说我不确定」的证明。**它会说，只是说在散文里。**
- 不是完整普查。**只查了四处。**

准确的只有一句：

> **有位置的地方，没规定谁能坐；没位置的地方，什么也没有。**

---

## 下一步

`PROPOSAL.md`：**不是加字段，是给已有的位置加约束。**

给 OTel 提两个属性 + 三条硬规矩：

```
gen_ai.evaluation.evaluator.id      // 谁做的评测
gen_ai.evaluation.evaluator.kind    // human | model | benchmark | self
```

把 `self` 变成**枚举里的一个值**——不是禁止它，是让它**可被看见、可被统计、可被拒绝**。

和另一个项目是同一个动作：

| 项目 | 加什么 | 因为 |
|---|---|---|
| [`recourse`](https://github.com/XxLCFLXx/recourse) | `principal` | 没人记录「谁负责」 |
| `silence` | `evaluator.kind = self` | 没人记录「是不是它自己说的」 |

---

## 如果你是在搜这些

`agent failure detection` · `agent self-report` · `LLM calibration` ·
`abstention` · `OpenTelemetry GenAI` · `semantic conventions` ·
`failure attribution` · `agent observability`

上面那些多在做「**事后由人（或另一个模型）判断哪里错了**」。

这个仓库问的是另一件事：**系统有没有可能在被发现之前，自己说一声。**

---

## 文件

| 文件 | 作用 |
|---|---|
| **`AUDIT.md`** | 完整体检：四处列名、关键词扫描、可复现命令、明确写了没证明什么 |
| **`PROPOSAL.md`** | 给 `gen_ai.evaluation.*` 的两个属性 + 三条约束 + 一个 lift 测试 |

---

## 状态

v0.2，一个人，2026-11。MIT。

已知缺口：TRAIL（148 条）在 Hugging Face 上是 gated，无认证访问返回 401。

---

> **Who&When、MAST、agentpile 里没有这个位置。**
> **OTel 里有位置，但没有名字。**
