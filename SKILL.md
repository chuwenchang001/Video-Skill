---
name: video-ad-generation
description: 端到端把一个商品生成一支可投放的广告短视频——自动理解商品并用五维量表决策唯一最优创意方向 → 承接方向产出 3–8 个分镜的分镜脚本 → 调火山方舟 Seedance 生成视频(15s 单段 / 30s 两段续写)+ 拼接 → 烧制字幕产出最终成片。供 BytePlus ModelArk / Dify 工作流导入使用。
pipeline: video
version: video-skill-v1
stages: 4
---

# Skill · 广告视频生成(video-ad-generation)

**一个端到端技能**:输入一个商品(名称、卖点、可选产品图、投放平台、视频类型、语种、时长),输出一支可直接投放的广告短视频 URL。技能内部按顺序走 **4 个阶段**,前两阶段是 LLM 推理,后两阶段是外部工具调用:

```
① 商品理解+创意决策 → ② 创意承接+分镜脚本 → ③ 视频生成(Seedance) → ④ 字幕烧制 → finalVideoUrl
     (LLM, t=0.4)         (LLM, t=0.6)          (工具·异步轮询)      (工具·异步轮询)
```

顺序执行,失败即停,已成功阶段可断点续跑。四个阶段是本技能的**内部环节**,不是四个独立技能。

## 何时使用

任何「为某商品在某平台、某视频类型下做一支广告视频」的任务。给定商品信息即可一站式产出成片;无需分别调用各阶段。

## 输入(表单 / run.input)

| 字段 | 类型 | 必填 | 默认 | 说明 |
|---|---|---|---|---|
| `product` | string | ✅ | — | 商品名称 |
| `features` | string[] | | — | 卖点/功能点 |
| `price` | string | | — | 价格 |
| `scenario` | string | | — | 使用场景 |
| `brandTone` | string | | — | 品牌调性/语气 |
| `productImageUrl` | string | | — | 产品图公网 URL(多模态 + 产品一致性) |
| `productImageUrls` | string[] | | — | 多张产品图 |
| `platform` | enum | | `douyin` | 投放平台 id |
| `videoType` | enum | | `spoken_grass` | 视频类型 id |
| `language` | enum | | `zh` | 成片台词/配音语种 id |
| `duration` | 15\|30 | | `15` | 视频总时长(秒),仅 15 或 30 |
| `aspectRatio` | enum | | `9:16` | 画幅 |

- 平台 id:`tiktok, instagram, facebook, shopee, lazada, youtube, amazon, douyin, kuaishou, shipinhao, xiaohongshu`
- 视频类型 id:`spoken_grass, narrative_short, funny_drama, product_review, unboxing, comparison_experiment, kol_recommend`
- 语种 id:`zh, en, id, th, vi, ja, ko, ms`
- 画幅:`9:16`(默认)`, 16:9, 1:1, 4:3, 3:4, 21:9`

## 输出

最终产物是 **`finalVideoUrl`**(带字幕成片;烧字失败时降级为无字幕成片)。技能同时保留各阶段中间产物(创意决策 JSON、分镜脚本 JSON、各视频段 URL),便于观测与断点续跑。

## 数据流

```
run.input
  └─①→ recommendedDirection / platformStrategy / sellingPoints …
        └─②→ scenes[](分镜脚本)
              └─③→ stitched.url / segments[0].url(成片)
                    └─④→ finalVideoUrl(带字幕最终成片)
```

---

## 阶段 ① 商品理解 + 创意方向决策(LLM,temperature = 0.4)

先把商品**自动归类**到内置类目,再结合投放平台与视频类型,用统一**五维量表**给多个候选创意方向打分,选出**唯一最优方向**。有产品图时走多模态。领域知识**内联注入** Prompt,非外部检索。

- System Prompt:[`prompts/1-product-understanding.system.txt`](prompts/1-product-understanding.system.txt)
- User Prompt:[`prompts/1-product-understanding.user.txt`](prompts/1-product-understanding.user.txt)
- 输出 Schema:[`schema/1-product-understanding.output.json`](schema/1-product-understanding.output.json)
- 知识库:[`kb/platforms.json`](kb/platforms.json) · [`kb/video-types.json`](kb/video-types.json) · [`kb/categories.json`](kb/categories.json) · [`kb/few-shot-samples.json`](kb/few-shot-samples.json)

**决策规则(按优先级)**:①平台优先(硬约束,踩中各平台调性与算法偏好)②类目—视频类型匹配 ③形态冲突时在指定 videoType 内最大化 ④人群一致性 ⑤合规红线优先于创意张力 ⑥Tie-break 先比 feasibility 再比 differentiation。

**五维量表(每维 1–5)**:`platformFit / audienceResonance / productFit / feasibility / differentiation`;`totalScore`=五维之和(5–25);候选按 totalScore 降序;`recommended=true` 有且仅一个。

**User Prompt 占位符**渲染:`{{categories_rendered}}`←`kb/categories.json`;`{{platform_kb_rendered}}`←选中平台条目;`{{video_type_kb_rendered}}`←选中类型条目;`{{few_shot_rendered}}`←`kb/few-shot-samples.json`;`{{product_info_json}}`←本次商品信息。

**产出**:`classification, productSummary, targetAudience, sellingPoints, painPointsSolved, platformStrategy, videoTypeStrategy, creativeDirections[], recommendedDirection`。下游严格承接 `recommendedDirection`,不重新挑方向。

## 阶段 ② 创意承接 + 分镜脚本(LLM,temperature = 0.6)

**不重新挑方向**,严格执行 ① 的 `recommendedDirection`,落成一支可喂给视频生成模型的**分镜脚本**,贴合平台/视频类型/语种/时长。

- System Prompt:[`prompts/2-creative.system.txt`](prompts/2-creative.system.txt)
- User Prompt:[`prompts/2-creative.user.txt`](prompts/2-creative.user.txt)
- 搞笑短剧专项指令:[`prompts/2-creative.funny_drama_directive.txt`](prompts/2-creative.funny_drama_directive.txt)
- 输出 Schema:[`schema/2-creative.output.json`](schema/2-creative.output.json)

**要点**:①直接采用上游方向;②`durationSec` 严格等于所选秒数;③拆 3–8 个分镜,各镜时长之和=总时长;④用 `platformStrategy.hookGuidance` 设计前 3 秒 hook;⑤`visual` 具体到可直接作视频提示词,成片不要任何屏幕字幕/花字;⑥守合规红线;⑦产品镜 `productShot=true` 且 `visual` 含 `@<产品图片URL>`;⑧`voiceover` 用目标语种,其它字段用中文。

**运行时覆盖(必须)**:LLM 输出后强制 `durationSec` = 用户所选 15/30、`language` 回填为所选语种 id —— 下游分段严格依赖它,不能信模型自填值。

**搞笑短剧专项**:`videoType==='funny_drama'` 时注入专项指令(笑点承载卖点,冲突→反转→解决,3 秒内入戏,结尾高潮带出卖点与 CTA)。

**产出(`CREATIVE`)**:`chosenDirection, title, durationSec, durationRationale, aspectRatio, hook, scenes[{index,durationSec,shotType,visual,productShot,cameraMovement,voiceover,bgm}], cta, taglines[], language`。

## 阶段 ③ 视频生成(工具 · 火山方舟 Seedance 异步任务)

消费分镜脚本,调 Seedance 生成视频。**无 LLM 提示词,是确定性编排 + 模型调用**;提示词由分镜脚本程序化拼装。

- 输入 Schema:[`schema/3-video-generation.input.json`](schema/3-video-generation.input.json) · 输出 Schema:[`schema/3-video-generation.output.json`](schema/3-video-generation.output.json)

**流程**:
1. **分段**:总时长 ≤15s 或 scenes<2 → 单段;30s → 两段(前 15s + 余下),第二段在第一段上 **extension 续写**;每段时长夹取到 `[5,10,15]` 中 ≤(总/2) 的最大档。
2. **段提示词**:该段分镜浓缩为提示词 + 台词(目标语种)+ 强语言指令(全片配音用目标语种,**不要任何字幕/文字/硬字幕**)+ flag `--ratio <aspectRatio> --duration <sec> --resolution <res>`。
3. **产品参考图**:解析分镜 `visual` 里的 `@<url>` → 作为 `role:reference_image`;非视频安全格式(如 `.bmp`)先转 PNG 重托管。
4. **生成/续写/轮询**:顺序生成,续写段把上一段 `video_url` 作 `role:reference_video`;轮询 5s/次,超时 10min。
5. **拼接**:多段调外部 **FFmpeg 拼接服务**(`STITCH_API_URL`)合成完整 mp4;失败不致命,回退第一段。
6. **持久化**:每段及成片转存 **TOS**;未配则暂存 Ark 临时 URL(约 24h 过期)。

**模型/环境**:Base URL `OPENAI_BASE_URL`(默认 ModelArk 端点),Model `VIDEO_MODEL`(默认 `seedance-1-0-lite-t2v-250428`),Key `OPENAI_API_KEY`,分辨率 `VIDEO_RESOLUTION`(默认 720p)。**BYOM 账号**须用自带 Seedance Model ID + Key。

**产出(`VIDEO_GENERATION`)**:`model, byom, ratio, resolution, language, totalDurationSec, billedSeconds, segmentCount, extended, segments[], stitched`。成片 URL = `stitched.url`(多段)或 `segments[0].url`(单段)。

## 阶段 ④ 字幕烧制(工具 · 外部 HTTP 服务)

取 ③ 成片 URL,调外部字幕烧制服务烧入字幕,轮询到终态后转存 TOS,作为最终成片。**纯 HTTP 调用,无 LLM。**

- 输入 Schema:[`schema/4-subtitle-burn.input.json`](schema/4-subtitle-burn.input.json) · 输出 Schema:[`schema/4-subtitle-burn.output.json`](schema/4-subtitle-burn.output.json)

**接口契约**:
```
提交  POST {SUBTITLE_API_BASE}/api/subtitle/burn
      { "video_url":"<成片URL>", "language":"<语种id|auto>", "style":"<SUBTITLE_STYLE>" } → { task_id }
轮询  GET  {SUBTITLE_API_BASE}/api/subtitle/task/{task_id}   每 3s,超时 10min
      终态:响应含 "success"(取 video_url) / "failed"(报错)
```
语言:用 `run.input.language`(白名单内)透传,否则传 `auto`;字体由服务端按 language 选择。

**失败降级**(env `SUBTITLE_BURN_REQUIRED`,默认 `false`):失败 → 保留无字幕成片为 `finalVideoUrl`,技能仍完成;设 `true` 则失败即终止。

**产出**:`finalVideoUrl` 覆盖为带字幕成片。

---

## 全局约定

- **语言**:面向用户展示的可读字段一律**英文**;枚举 id 保持原样;成片台词/配音语种由 `language` 决定。
- **时长**:仅 15s(单段)/ 30s(两段续写);运行时强制覆盖模型输出的 `durationSec`。
- **产品一致性**:分镜 `visual` 里的 `@<产品图片URL>` 会作为 `reference_image` 传给 Seedance,严格还原外观。
- **合规**:各类目 `riskNotes`(医疗/绝对化/焦虑营销等红线)任何创意方向都不得触碰。

## 附属资产

| 目录 | 内容 |
|---|---|
| [`prompts/`](prompts/) | 阶段 ①② 的原始 System / User Prompt + 搞笑短剧专项指令 |
| [`schema/`](schema/) | 各阶段输入/输出 JSON Schema(可直接绑 Dify 结构化输出) |
| [`kb/`](kb/) | 阶段 ① 内置知识库:平台 / 视频类型 / 类目 / Few-shot |

## 环境变量

见 [`.env.example`](.env.example)。分组:LLM(`OPENAI_BASE_URL/API_KEY/MODEL`)、视频(`VIDEO_MODEL/VIDEO_RESOLUTION`)、拼接(`STITCH_API_URL/TOKEN`)、字幕(`SUBTITLE_API_BASE/STYLE/BURN_REQUIRED`)、存储(`TOS_*`)。视频生成复用 `OPENAI_API_KEY/BASE_URL`。

## 在 Dify 里落地

本技能在 Dify 里对应**一个 Workflow**:阶段 ①② 是 **LLM 节点 + 结构化输出**,阶段 ③④ 是 **HTTP 节点 + 循环轮询**(异步任务的主要难点),知识库**内联进 Prompt**而非做 RAG。**逐节点搭建操作手册见 [`DIFY_GUIDE.md`](DIFY_GUIDE.md)。**
