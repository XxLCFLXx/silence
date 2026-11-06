# silence

> **No public agent trace format has a field for "I might be wrong."**
> 没有一种公开格式，允许 agent 说「我可能错了」。

在已查的三处格式定义里，`self` / `report` / `confidence` / `uncertain` / `abstain`
这五个词，**零命中**。

失败一律由**外部**标注——人是发现者，不是 agent。

---

## 数据里有四个字段，四个问题

| 字段 | 回答什么 | 谁给的 |
|---|---|---|
| `is_correct` | 这次**对不对** | 与 ground truth 比对 |
| `mistake_agent` | **谁**错了 | 标注者 |
| `mistake_step` | **哪一步**错的 | 标注者 |
| `mistake_reason` | **为什么**错 | 标注者 |

覆盖得相当完整。

**唯独没有字段回答：它自己知不知道自己错了。**

---

## 三个标注者，零个它自己

MAST 的人工标注表，schema 原文：

```
annotations: [{
    "failure mode": string,
    "annotator_1":   bool,
    "annotator_2":   bool,
    "annotator_3":   bool
}]
```

**三个字段回答「哪个人认为它失败了」。零个字段回答「它自己认为失败了吗」。**

换一套系统，同一个缺席：

| 来源 | 记录「谁发现错误」 | agent 在里面吗 |
|---|---|---|
| Who&When | `mistake_agent`（标注得出） | ❌ |
| MAST | `annotator_1/2/3` | ❌ |
| agentpile / ATIF | `label_source ∈ {expert, LLM judge, benchmark}` | ❌ |

两种完全不同的设计，同一个空缺。

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

这句话的措辞很重要：**不是「没有错误」，是「没有人看」。**

---

## 复现

不需要下载数据，不需要装任何东西：

```bash
curl "https://datasets-server.huggingface.co/info?dataset=Kevin355%2FWho_and_When"
curl "https://datasets-server.huggingface.co/info?dataset=mcemri%2FMAST-Data"
```

返回 JSON 里的 `features` 就是全部列名。**任何人都能重跑，核对本仓库的每一句。**

---

## 这不是什么

- 不是「模型没有不确定性」的证明。**内部表征有没有，读 schema 看不出来。**
- 不是「模型从来不说我不确定」的证明。**它会说，只是说在散文里。**
- 不是完整普查。**只查了三处，TRAIL 因 gated 未查。**

准确的说法只有一句：

> **没有任何一个公开格式，为「我不确定」留了一个位置。**

---

## 下一步

`PROPOSAL.md`：加一个字段。

```json
"self_report": {
  "claimed": "fail",      // pass | fail | unsure
  "confidence": 0.62,
  "at": "step:14"
}
```

和另一个项目是同一个动作：

| 项目 | 加什么字段 | 因为 |
|---|---|---|
| [`recourse`](https://github.com/XxLCFLXx/recourse) | `principal` | 没人记录「谁负责」 |
| `silence` | `self_report` | 没人记录「它自己怎么说」 |

---

## 如果你是在搜这些

`agent failure detection` · `agent self-report` · `LLM calibration` ·
`abstention` · `trace schema` · `agent observability` · `failure attribution`

上面那些多在做「**事后由人（或另一个模型）判断哪里错了**」。

这个仓库问的是另一件事：**系统有没有可能在被发现之前，自己说一声。**

---

## 文件

| 文件 | 作用 |
|---|---|
| **`AUDIT.md`** | 完整体检：三份列名表、关键词扫描、可复现命令、明确写了没证明什么 |
| `PROPOSAL.md` | 待写：`self_report` 字段定义 |

---

## 状态

v0.1，一个人，2026-11。MIT。

已知缺口：TRAIL（148 条）在 Hugging Face 上是 gated，无认证访问返回 401。

---

> 数据里有字段回答「对不对」「谁错了」「哪一步」「为什么」。
> 有字段回答「哪三个人认为它错了」。
> **没有字段回答「它自己知不知道」。**
