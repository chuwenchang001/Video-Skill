# Skill 2 · 创意承接 + 分镜脚本(CREATIVE)

Video Pipeline 第二步(**合并版**:原「创意概念」与「分镜脚本」两步合并)。它**不重新挑选创意方向**,而是严格执行 Skill 1 已决策的 `recommendedDirection`,把它落成一支可拍摄、可喂给视频生成模型的**分镜脚本**,完全贴合指定的投放平台、视频类型、目标语种与时长。

- **temperature = 0.6**。
- `visual` 要具体到能直接作为视频生成模型的提示词。
- 时长是强约束:模型输出后,运行时**强制覆盖** `durationSec` 为用户所选 15 或 30 秒。

| 项 | 值 |
|---|---|
| 类型 | LLM |
| System Prompt | [`prompts/2-creative.system.txt`](../prompts/2-creative.system.txt) |
| User Prompt 模板 | [`prompts/2-creative.user.txt`](../prompts/2-creative.user.txt) |
| 搞笑短剧专项指令 | [`prompts/2-creative.funny_drama_directive.txt`](../prompts/2-creative.funny_drama_directive.txt) |
| 输入 Schema | [`schema/2-creative.input.json`](../schema/2-creative.input.json) |
| 输出 Schema | [`schema/2-creative.output.json`](../schema/2-creative.output.json) |
| 模型 | env `OPENAI_MODEL`(默认 `gpt-4o`) |
| temperature | 0.6 |

## 输入

- 上游 `PRODUCT_UNDERSTANDING`(**必须**;缺失则抛错「缺少上游 Skill 1 的输出」)
- `platform`(默认 `douyin`)、`videoType`(默认 `spoken_grass`)
- `language`(默认 `zh`)、`duration`(15 | 30)、`productImageUrl`(可选)

## 输出(`CREATIVE`)

```
chosenDirection, title, durationSec, durationRationale, aspectRatio, hook,
scenes[ {index, durationSec, shotType, visual, productShot, cameraMovement, voiceover, bgm} ],
cta, taglines[], language
```

- `scenes`:3–8 个分镜,各镜 `durationSec` 之和应等于总 `durationSec`。
- 产品展示镜:`productShot=true` 且 `visual` 含 `@<产品图片URL>`;无图时全部 `productShot=false`。

## 语言规则

每个分镜的 `voiceover`(台词/旁白/口播)必须用**目标语种**撰写;成片不需要屏幕字幕/花字;其余说明性字段(`visual`、`shotType`、`cameraMovement`、`durationRationale` 等)一律用中文,便于运营查看。

## 运行时覆盖与专项指令

- **时长覆盖**:输出 `durationSec` 强制 = 用户所选 15/30(下游据此决定单段/两段)。
- **language 回填**:把用户所选语种 id 写入 `output.language`。
- **30s 分段引导**:安排成「前约 15 秒 + 后约 15 秒」两部分,便于分两段生成;15s 单段叙事。
- **搞笑短剧专项(`videoType==='funny_drama'`,最高优先级)**:注入 [`prompts/2-creative.funny_drama_directive.txt`](../prompts/2-creative.funny_drama_directive.txt) —— 笑点必须承载卖点,从核心卖点挑 1 个最有戏剧张力的点设计「冲突→反转→解决」,3 秒内入戏,结尾在笑点高潮自然带出卖点与 CTA。

## User Prompt 渲染要点

运行时注入:上游决策 JSON、平台约束(时长建议/内容惯例)、视频类型、目标语种(label + speak)、时长、产品参考图块。核心要求:

1. 直接采用上游 `recommendedDirection` 作为本片方向,不另选;
2. `durationSec` 严格等于用户所选秒数;
3. 拆成 3–8 个分镜,各镜时长之和等于总时长;
4. 用 `platformStrategy.hookGuidance` 设计前 3 秒 hook;应用 `videoTypeStrategy.formatNotes` 与 `executionHints`;
5. `visual` 可直接作视频生成提示词,标注运镜/旁白/音乐,成片不要任何屏幕字幕/花字;
6. 遵守上游归类的合规红线;
7. 产品镜 `productShot=true` 且 `visual` 含 `@<URL>`;
8. `voiceover` 用目标语种,其它字段用中文。

## Dify 节点映射

- **LLM 节点**:System 填 `prompts/2-creative.system.txt`,User 填 `prompts/2-creative.user.txt`;开启**结构化输出**绑定 `schema/2-creative.output.json`;`temperature=0.6`。
- **上游变量**:User Prompt 里 `{{PRODUCT_UNDERSTANDING_json}}` 引用 Skill 1 节点输出的整个对象。
- **搞笑短剧条件**:用**条件分支**判断 `videoType==='funny_drama'`,命中则把专项指令拼接到 User Prompt 的 `{{funny_drama_directive_if_applicable}}` 位置。
- **时长/语言覆盖**:LLM 之后接一个 **Code 节点**,把 `durationSec` 强制改成表单所选 15/30、把 `language` 回填为所选语种 id,再传给下游。

在 Pipeline 中的位置:`PRODUCT_UNDERSTANDING → [本步] → VIDEO_GENERATION → SUBTITLE_BURN`。
