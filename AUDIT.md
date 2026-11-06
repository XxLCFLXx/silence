# 审计：公开的 agent 失败数据里，有没有「自报失败」这个字段？

状态：草案 v0.2 · 2026-11-06
方法：**读 schema，不读文档。**

---

## 0. 结论先说

所查的三处格式定义里，**没有一个结构化字段，让 agent 说「我可能错了」。**

失败一律由**外部**标注。而记录「谁发现这个错误」的字段，可选值里**没有 agent 自己**。

最直白的一处：MAST 的人工标注表里，有 `annotator_1`、`annotator_2`、`annotator_3` 三列，
**没有一列留给 agent。**

> 这不是「模型没有不确定性」的证明。
> 这是「**没有任何一个公开格式，为不确定性留了一个位置**」的证明。

---

## 1. 怎么复现

不需要下载数据，不需要装任何东西，一条 HTTP：

```bash
curl "https://datasets-server.huggingface.co/info?dataset=Kevin355%2FWho_and_When"
curl "https://datasets-server.huggingface.co/info?dataset=mcemri%2FMAST-Data"
```

返回的 JSON 里，`dataset_info.<config>.features` 就是全部列名。
任何人可以重跑这两条命令核对本文件。

---

## 2. Who&When 的实际列名

数据源：<https://huggingface.co/datasets/Kevin355/Who_and_When>

**`Algorithm-Generated` — 126 条**

| 列名 | 类型 |
|---|---|
| `is_correct` | bool |
| `question` | string |
| `question_ID` | string |
| `ground_truth` | string |
| `history` | `[{content: string, name: string, role: string}]` |
| `mistake_agent` | string |
| `mistake_step` | string |
| `mistake_reason` | string |

**`Hand-Crafted` — 58 条**

| 列名 | 类型 |
|---|---|
| `history` | `[{content: string, role: string}]` |
| `question` | string |
| `groundtruth` | string |
| `is_corrected` | bool |
| `mistake_agent` | string |
| `mistake_step` | string |
| `mistake_reason` | string |
| `mistake_type` | string |
| `question_ID` | string |

126 + 58 = **184**，与 README 声明的数量一致。

---

## 3. MAST 的实际列名

数据源：<https://huggingface.co/datasets/mcemri/MAST-Data>

**`default` — 1,642 条，14 种失败模式各一列**

| 列名 | 类型 |
|---|---|
| `mas_name` / `llm_name` / `benchmark_name` | string |
| `trace_id` | int64 |
| `trace` | `{key, index, trajectory}` |
| `mast_annotation` | `{1.1, 1.2, 1.3, 1.4, 1.5, 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 3.1, 3.2, 3.3}` 全为 int64 |

**`human_labelled` — 19 条，三个人各一列**

| 列名 | 类型 |
|---|---|
| `round` / `mas_name` / `benchmark_name` | string |
| `trace_id` | int64 |
| `trace` | string |
| `annotations` | `[{failure mode: string, annotator_1: bool, annotator_2: bool, annotator_3: bool}]` |

**那三个 `annotator_N` 就是这个审计最干净的一张图：**

> 三个字段回答「**哪个人**认为它失败了」。
> **零个字段回答「它自己认为失败了吗」。**

---

## 4. 五个关键词的扫描

| 词 | Who&When（9 + 10 列） | MAST（19 + 6 列） | agentpile / ATIF |
|---|---|---|---|
| `self` | ❌ | ❌ | ❌ |
| `report` | ❌ | ❌ | ❌ |
| `confidence` | ❌ | ❌ | ❌ |
| `uncertain` | ❌ | ❌ | ❌ |
| `abstain` | ❌ | ❌ | ❌ |

**零命中。** 而且这次不是「文档里没写」，是**列名里没有**。

补充：TRAIL（148 条，span 级）在 Hugging Face 上是 gated 的，
无认证访问 datasets-server 返回 **401**，本次未查。这是本文件的已知缺口。

---

## 5. 四个字段，四个问题

（以 Who&When 为例）

| 字段 | 回答什么 | 谁给的 |
|---|---|---|
| `is_correct` | 这次**对不对** | 与 `ground_truth` 比对得出 |
| `mistake_agent` | **谁**错了 | 标注者 |
| `mistake_step` | **哪一步**错的 | 标注者 |
| `mistake_reason` | **为什么**错 | 标注者（自然语言） |

四个字段，四个问题，覆盖得相当完整。

**唯独没有字段回答：它自己知不知道自己错了。**

而且看第四列——**四个全是外部。**

---

## 6. 唯一可能放「自报」的地方，以及为什么它不算

`history` / `trajectory` 是自由文本。agent 当然可以在某条消息里写「我不确定」。

但那不算一个字段，因为：

- **不可查询**——没法写 `WHERE self_report = 'fail'`
- **不可比较**——184 条里有多少条说过「不确定」，你数不出来
- **不在 schema 里**——它不是一个位置，只是一段散文

所以准确的表述是：

> **结构化的自报字段不存在，只有散文。**

这句话比「没有自报」硬，因为它连反驳的余地都堵住了。

---

## 7. 记录「谁发现错误」的字段，agent 不在选项里

两处独立地证明了同一件事。

**`agentpile`** 把 14 个数据集统一成 ATIF 记录，标签带 `label_source`，取值词表是：

> **expert / LLM judge / benchmark**

三个值，三种外部视角。

**`MAST`** 的人工标注表里，是 `annotator_1` / `annotator_2` / `annotator_3`。

三个位置，三个人。

**两个系统，两种设计，同一个缺席。**

这和另一个项目是同一个形状：

| 项目 | 记录什么 | 缺什么 |
|---|---|---|
| `recourse` | 谁**负责** | **人**不在字段里 |
| `agentpile` | 谁**发现错误** | **agent 自己**不在选项里 |
| `MAST` | 谁**判定失败** | **agent 自己**不在位置里 |

三个都是：**字段存在，但这个主体不在里面。**

---

## 8. 顺手算出的一个数

`agentpile` 覆盖 14 个数据集、约 **410,000 条运行**。

其中做过「错误定位到具体哪一步」标注的：

| 数据集 | 步/span 级标注 | 数量 |
|---|---|---|
| Who&When | 步 | 184 |
| TRAIL | span | 148 |
| | **合计** | **332** |

```
332 / 410,000 ≈ 0.08%
```

> **万分之八的运行，被定位到过具体哪一步出错。**
> 剩下的 99.92%，不是没有错，是**没有人看**。

措辞很重要：**不是「没有错误」，是「没有人看」。** 这是整个测量的地基。

---

## 9. 顺手发现的一处数据质量问题

Who&When 的两个 config，字段不一致：

| | Algorithm-Generated | Hand-Crafted |
|---|---|---|
| 正确性 | `is_correct` | `is_corrected` |
| history 元素 | 有 `name` | 无 `name` |
| 错误类型 | 无 | `mistake_type` |

任何跨 config 的分析都得写两套代码。

**这种不一致，本身就是「没有人把它当成一份统一记录在用」的症状。**

---

## 10. 这个审计证明了什么，没证明什么

**证明了：**

- 三个公开格式里，不存在结构化的「agent 自报失败」字段
- 两处独立的「谁发现错误」设计里，都没有 agent 自己
- 步级错误标注的覆盖率约 0.08%

**没有证明：**

- 模型内部没有不确定性表征（那需要探针实验，不是读 schema）
- 自由文本里没有犹豫的表达（那需要跑 NLP，且结论会很软）
- TRAIL 和其余 11 个数据集也没有（**TRAIL 因 gated 未查**）

**最硬的局限：只查了三处。**
它们是「失败归因」这条线里最核心的三个，但它们是三个，不是全部。

---

## 11. 待查清单

- [x] Who&When（184 条，步级）
- [x] MAST（1,642 条，14 种失败模式 + 人工标注）
- [ ] **TRAIL（148 条，span 级）—— 需要 HF 认证，401**
- [ ] OpenTelemetry GenAI 语义约定 —— 标准层面有没有 `gen_ai.agent.self_report`
- [ ] AgentRewardBench / SWE-Gym —— 它们的 `outcome` 是测试结果还是自评？
- [ ] ATIF v1.7 规范原文 —— agentpile 说「ATIF 没有字段的放 extra」，那 ATIF 本身有什么

查完这几项，如果仍然全空，结论就从「三处」升级成「这条线的全部」。

---

## 12. 一句话

> **数据里有字段回答「对不对」「谁错了」「哪一步」「为什么」。**
> **有字段回答「哪三个人认为它错了」。**
> **没有字段回答「它自己知不知道」。**

而 41 万条运行里，只有万分之八被人看一眼——**所以也没有人代替它回答。**
