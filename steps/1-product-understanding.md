# Skill 1 · 商品理解 + 创意方向决策(PRODUCT_UNDERSTANDING)

Video Pipeline 的第一步。给定商品信息(+可选产品图),先把商品**自动归类**到内置类目体系,再结合用户指定的**投放平台**与**视频类型**,用统一的**五维量表**为多个候选创意方向打分,选出**唯一最优方向**,供下游 Skill 2 严格承接。

- 决策类任务,**temperature = 0.4**,保证一致与可复现。
- 有产品图(`productImageUrl` / `productImageUrls`)时走**多模态**理解。
- 领域知识(平台 / 视频类型 / 类目 / 范例)**内联注入** Prompt,非外部检索。
- 输出严格 JSON,`response_format = json_object`。

| 项 | 值 |
|---|---|
| 类型 | LLM |
| System Prompt | [`prompts/1-product-understanding.system.txt`](../prompts/1-product-understanding.system.txt) |
| User Prompt 模板 | [`prompts/1-product-understanding.user.txt`](../prompts/1-product-understanding.user.txt) |
| 输入 Schema | [`schema/1-product-understanding.input.json`](../schema/1-product-understanding.input.json) |
| 输出 Schema | [`schema/1-product-understanding.output.json`](../schema/1-product-understanding.output.json) |
| 知识库 | [`kb/platforms.json`](../kb/platforms.json) · [`kb/video-types.json`](../kb/video-types.json) · [`kb/categories.json`](../kb/categories.json) · [`kb/few-shot-samples.json`](../kb/few-shot-samples.json) |
| 模型 | env `OPENAI_MODEL`(默认 `gpt-4o`),端点 `OPENAI_BASE_URL` / `OPENAI_API_KEY` |
| temperature | 0.4 |

## 输入(来自表单 run.input)

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `product` | string | ✅ | 商品名称 |
| `features` | string[] | | 卖点/功能点 |
| `price` | string | | 价格 |
| `scenario` | string | | 使用场景 |
| `brandTone` | string | | 品牌调性/语气 |
| `productImageUrl` | string | | 产品图公网 URL(多模态 + 下游产品一致性) |
| `productImageUrls` | string[] | | 多张产品图 |
| `platform` | enum | | 投放平台 id,默认 `douyin` |
| `videoType` | enum | | 视频类型 id,默认 `spoken_grass` |

- 平台 id:`tiktok, instagram, facebook, shopee, lazada, youtube, amazon, douyin, kuaishou, shipinhao, xiaohongshu`
- 视频类型 id:`spoken_grass, narrative_short, funny_drama, product_review, unboxing, comparison_experiment, kol_recommend`

> 容错:模型若把「字符串数组」返回成一整段字符串,按换行/逗号/顿号/分号切回数组。

## 输出

结构化 JSON:`classification`(自动归类)、`productSummary`、`targetAudience`(2–4)、`sellingPoints`(3–6)、`painPointsSolved`、`platformStrategy`、`videoTypeStrategy`、`creativeDirections`(2–4,按 `totalScore` 降序,首项即最佳)、`recommendedDirection`(唯一最优 + `whyBest`)。完整结构见输出 Schema。

**语言**:所有面向用户展示的可读字段一律**英文**;枚举 id 保持原样。

## 决策规则(按优先级)

1. **平台优先级(硬约束)**:创意先服从平台调性与算法偏好(抖音/TikTok 前 2–3 秒强 hook;快手/视频号信任人设;小红书第一人称真实测评;YouTube 信息价值;IG 审美;FB 利益点直给+社会证明;Shopee/Lazada 卖点+价格+优惠直给;Amazon 功能规格清晰、禁夸大)。
2. **类目—视频类型匹配度**:功能可量化→对比实验/测评;颜值新品礼赠→开箱;情感品牌向→剧情短片;美妆食品快消→口播种草/KOL;信任敏感(母婴/健康)→KOL/测评且克制。
3. **形态冲突处理**:用户已指定 `videoType`,须在该形态内最大化效果,不得改用其它形态;天然不契合时在 rationale 点明取舍并给最优解法。
4. **人群一致性**:`targetAudience` 与所选平台主力人群交集最大化。
5. **合规红线优先于创意张力**:类目 `riskNotes` 红线任何方向都不得触碰。
6. **Tie-break**:`totalScore` 差 ≤1 时优先 `feasibility` 高者,再相同取 `differentiation` 高者。

## 评估量表(每个候选方向按 1–5 分打五维)

`platformFit`(平台契合)、`audienceResonance`(人群共鸣)、`productFit`(卖点承载)、`feasibility`(可执行)、`differentiation`(差异化)。`totalScore` = 五维之和(5–25);`creativeDirections` 按 `totalScore` 降序;`recommended=true` 有且仅一个;全部偏低(最高<16)仍选相对最优并在 `whyBest` 说明局限。

## User Prompt 渲染

User Prompt 模板里的占位符,运行时用知识库渲染:

- `{{categories_rendered}}` ← `kb/categories.json`(每条:`id(label):创意倾向=…;红线=…`)
- `{{platform_kb_rendered}}` ← `kb/platforms.json` 里选中平台(时长/主力人群/算法偏好/内容惯例/限制)
- `{{video_type_kb_rendered}}` ← `kb/video-types.json` 里选中视频类型(擅长/适合/注意)
- `{{few_shot_rendered}}` ← `kb/few-shot-samples.json`(输入+输出范例)
- `{{product_info_json}}` ← 本次商品信息

## Dify 节点映射

- **LLM 节点**:System 填 `prompts/1-product-understanding.system.txt`,User 填 `prompts/1-product-understanding.user.txt`;开启**结构化输出**并绑定 `schema/1-product-understanding.output.json`;`temperature=0.4`;若指向 ModelArk,在模型配置里改 Base URL。
- **知识库注入**:`kb/*.json` 用 **Code 节点常量**或直接**内联进 User Prompt**渲染,替换上面 5 个占位符。**不要**做成 Dify 知识库(RAG)——这是配置型知识,不是检索型。
- **多模态**:有产品图时,把图片 URL 作为 LLM 节点的图片输入。
- **输出**:`recommendedDirection` 整个对象透传给下游 Skill 2 的 `{{PRODUCT_UNDERSTANDING}}`。

在 Pipeline 中的位置:`[本步] → CREATIVE → VIDEO_GENERATION → SUBTITLE_BURN`。下游 `CREATIVE` 严格承接本步的 `recommendedDirection`,不重新挑方向。
