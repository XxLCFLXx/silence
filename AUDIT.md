# 审计：公开的 agent 失败记录里，有没有「它自己说的话」？

状态：v0.3 · 2026-11-06
方法：**读 schema，不读文档。**

> v0.3 改了什么：查了 OpenTelemetry GenAI 语义约定，**原结论要修正**。
> 不是「没有位置」，是「**位置有，主体没有**」。

---

## 0. 结论

| 来源 | 有位置吗 | 有主体约束吗 |
|---|---|---|
| Who&When | ❌ | — |
| MAST | ❌ | — |
| agentpile / ATIF | ❌ | — |
| **OpenTelemetry GenAI** | **✅ `gen_ai.evaluation.*`** | **❌** |

准确的结论是两句：

> **有标准允许记录一次「pass / fail」的判定。**
> **但它不规定这个判定由谁做出。**

一个 agent 给自己打 `pass`，在规范上**完全合法**。

这不是「没有位置」，是「**位置存在，主体缺席**」——
和另一个项目里 `principal` 那个缺口，是同一种形状。

---

## 1. 怎么复现

全部不需要下载、不需要认证：

```bash
# Who&When / MAST 的列名
curl "https://datasets-server.huggingface.co/info?dataset=Kevin355%2FWho_and_When"
curl "https://datasets-server.huggingface.co/info?dataset=mcemri%2FMAST-Data"

# OTel GenAI 的全部属性（自动生成文件，列名即事实）
curl -s "https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/registry/attributes/gen-ai.md"
```

---

## 2. OpenTelemetry GenAI（v0.3 新增，最重要的一处）

文件：`docs/registry/attributes/gen-ai.md`（**自动生成**，所以它就是规范本身）

它有四个 `gen_ai.evaluation.*` 属性：

| 属性 | 类型 | 说明（原文） |
|---|---|---|
| `gen_ai.evaluation.name` | string | The name of the evaluation metric used for the GenAI response. |
| `gen_ai.evaluation.score.value` | double | The evaluation score returned by **the evaluator**. |
| `gen_ai.evaluation.score.label` | string | Human readable label for evaluation. 取值示例含 **`pass`**、**`fail`** |
| `gen_ai.evaluation.explanation` | string | A free-form explanation for the assigned score provided by **the evaluator**. |

**这是本审计里第一个「位置存在」的发现。**

然后看缺什么：

1. 在这份属性清单里，**没有 `gen_ai.evaluation.evaluator.*`**。
   「评测者说了什么」有字段，「评测者是谁」没有。
2. **没有任何条款要求 evaluator ≠ agent。**
3. `pass` / `fail` 是合法取值，而且没有限定谁能写。

所以：

> **规范定义了一个 `evaluator` 角色，却没有规定谁不能坐这个位置。**

顺带排除一个容易混的：`gen_ai.response.status` 也有 `failed` 这个值，
但它说的是**生成失败**（provider 层面），不是**任务判断失败**。两回事。

---

## 3. Who&When 的实际列名

<https://huggingface.co/datasets/Kevin355/Who_and_When>

**`Algorithm-Generated` — 126 条**

| 列名 | 类型 |
|---|---|
| `is_correct` | bool |
| `question` / `question_ID` / `ground_truth` | string |
| `history` | `[{content, name, role}]` |
| `mistake_agent` | string |
| `mistake_step` | string |
| `mistake_reason` | string |

**`Hand-Crafted` — 58 条**

| 列名 | 类型 |
|---|---|
| `history` | `[{content, role}]` |
| `question` / `groundtruth` / `question_ID` | string |
| `is_corrected` | bool |
| `mistake_agent` / `mistake_step` / `mistake_reason` / `mistake_type` | string |

126 + 58 = **184**，与 README 声明一致。

---

## 4. MAST 的实际列名

<https://huggingface.co/datasets/mcemri/MAST-Data>

**`default` — 1,642 条**：`mas_name` `llm_name` `benchmark_name` `trace_id` `trace` +
`mast_annotation{1.1 … 3.3}`（14 种失败模式，各一列 int64）

**`human_labelled` — 19 条**：

```
annotations: [{
    "failure mode": string,
    "annotator_1":   bool,
    "annotator_2":   bool,
    "annotator_3":   bool
}]
```

**三个字段回答「哪个人认为它失败了」。零个字段回答「它自己认为失败了吗」。**

---

## 5. 五个关键词的扫描

| 词 | Who&When | MAST | OTel GenAI | agentpile / ATIF |
|---|---|---|---|---|
| `self` | ❌ | ❌ | ❌ | ❌ |
| `report` | ❌ | ❌ | ❌ | ❌ |
| `confidence` | ❌ | ❌ | ❌ | ❌ |
| `uncertain` | ❌ | ❌ | ❌ | ❌ |
| `abstain` | ❌ | ❌ | ❌ | ❌ |

**五个词，零命中。**

但要读准：**词没有，位置有一个**（OTel 的 `evaluation.*`）。
所以真正的缺口不是「没有字段」，是**没有「谁在说」**。

---

## 6. 四个字段，四个问题（以 Who&When 为例）

| 字段 | 回答什么 | 谁给的 |
|---|---|---|
| `is_correct` | 这次**对不对** | 与 ground truth 比对 |
| `mistake_agent` | **谁**错了 | 标注者 |
| `mistake_step` | **哪一步**错的 | 标注者 |
| `mistake_reason` | **为什么**错 | 标注者 |

四个字段，四个问题，覆盖得相当完整。

**唯独没有字段回答：它自己知不知道自己错了。**

---

## 7. 唯一能放「自报」的地方，是散文

`history` / `trajectory` 是自由文本，agent 当然可以写「我不确定」。

但那不算一个字段：**不可查询、不可比较、不在 schema 里。**

> **结构化的自报不存在，只有散文。**

---

## 8. 记录「谁发现错误」的地方，agent 都不在

四种设计，同一个缺席：

| 来源 | 「谁发现」怎么写 | agent 在里面吗 |
|---|---|---|
| Who&When | `mistake_agent`（标注得出） | ❌ |
| MAST | `annotator_1/2/3` | ❌ |
| agentpile | `label_source ∈ {expert, LLM judge, benchmark}` | ❌ |
| **OTel** | **没有这个字段**（只有 `evaluator` 这个**角色**，没有身份） | ❌ |

---

## 9. 那个数

`agentpile` 覆盖 14 个数据集、约 **410,000 条运行**。
其中做过步级 / span 级错误定位标注的：

```
Who&When 184 + TRAIL 148 = 332
332 / 410,000 ≈ 0.08%
```

> **万分之八的运行，被定位到过具体哪一步出错。**
> 剩下的 99.92%，不是没有错，是**没有人看**。

措辞很重要：**不是「没有错误」，是「没有人看」。**

（TRAIL 在 Hugging Face 上是 gated，本次未直接查 schema，数量取自 agentpile 的清单。）

---

## 10. 顺手发现的一处数据质量问题

Who&When 两个 config 字段不一致：

| | Algorithm-Generated | Hand-Crafted |
|---|---|---|
| 正确性 | `is_correct` | `is_corrected` |
| history 元素 | 有 `name` | 无 `name` |
| 错误类型 | 无 | `mistake_type` |

任何跨 config 的分析都得写两套代码。

**这种不一致，本身就是「没有人把它当成一份统一记录在用」的症状。**

---

## 11. 这个审计证明了什么，没证明什么

**证明了：**

- 四处里，三处没有给「自报」留位置，一处有位置但没有主体约束
- 五处「谁发现错误」的设计里，agent 都不在其中
- 步级错误标注的覆盖率约 0.08%

**没有证明：**

- 模型内部没有不确定性表征（那需要探针实验，不是读 schema）
- 自由文本里没有犹豫的表达（那需要跑 NLP，且结论会很软）
- 其余 11 个数据集也没有

**最硬的局限：只查了四处。**

---

## 12. 待查

- [x] Who&When（184 条，步级）
- [x] MAST（1,642 条，14 种失败模式）
- [x] OpenTelemetry GenAI 语义约定
- [ ] TRAIL（148 条，span 级）—— **gated，401**
- [ ] AgentRewardBench / SWE-Gym —— `outcome` 是测试结果还是自评？
- [ ] ATIF v1.7 规范原文

---

## 13. 一句话

> **Who&When、MAST、agentpile 里没有这个位置。**
> **OTel 里有位置，但没有名字。**
>
> 一个 agent 给自己打 `pass`，在规范上完全合法——**因为规范没说那个位置不能是它自己坐。**
