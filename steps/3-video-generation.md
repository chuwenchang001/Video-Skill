# Skill 3 · 视频生成(VIDEO_GENERATION)

Video Pipeline 第三步。消费 Skill 2 的分镜脚本,调用**火山方舟 ModelArk Seedance**(异步任务接口,非 OpenAI 兼容那套)生成视频。**这是确定性编排 + 模型调用的工具型 Skill,没有 LLM 提示词**——提示词由分镜脚本程序化拼装而成。

| 项 | 值 |
|---|---|
| 类型 | 工具(异步任务编排) |
| Provider | 火山方舟 ModelArk Seedance,异步任务(`create` + `poll`) |
| Base URL | env `OPENAI_BASE_URL`,默认 `https://ark.ap-southeast.bytepluses.com/api/v3` |
| Model | env `VIDEO_MODEL`,默认 `seedance-1-0-lite-t2v-250428` |
| API Key | env `OPENAI_API_KEY`(BYOM 账号用自带 Key) |
| Resolution | env `VIDEO_RESOLUTION`,默认 `720p`(480p/720p/1080p) |
| 轮询 | 5s/次,超时 600000ms(10min) |
| 输入 Schema | [`schema/3-video-generation.input.json`](../schema/3-video-generation.input.json) |
| 输出 Schema | [`schema/3-video-generation.output.json`](../schema/3-video-generation.output.json) |

## 核心流程

1. **分段决策**(由用户所选 `duration` 决定):
   - 总时长 ≤ **15s** 或 scenes < 2 → **单段**一次性生成;
   - **30s** → 拆 **两段**(前 15s 一段 + 余下一段),第二段在第一段视频上 **extension 续写**;
   - 每段时长**夹取**到模型可生成档 `[5, 10, 15]`(取 ≤ 请求的最大档)。
2. **段提示词**:把该段分镜浓缩为一段提示词;第二段加「延续上一段画面,继续」;末尾追加台词(目标语种)+ 强语言指令(全片配音用目标语种,**不要任何字幕/文字/硬字幕**)。
3. **产品参考图**:从分镜 `visual` 解析 `@<url>` 标记 → 作为 `role:reference_image` 附加,严格还原产品一致性;非视频安全格式(如 `.bmp`)先下载转 PNG 重新托管。
4. **生成与续写**:顺序生成;续写段把上一段 `video_url` 作 `role:reference_video` 传入。轮询 5s/次,超时 10min。
5. **拼接**:多段调外部 **FFmpeg 拼接服务**(env `STITCH_API_URL` / `STITCH_API_TOKEN`)合成完整 mp4(失败不致命,回退第一段)。
6. **持久化**:每段及成片转存 **TOS**;未配 TOS 则暂存 Ark 临时 URL(约 24h 过期)。

## 输入

- 上游 `CREATIVE`(**必须**;缺失抛错「缺少上游 Skill 2 的输出」)
- `language`(默认 `zh`)、`duration`(15 | 30)、`aspectRatio`(默认 `9:16`)

## 输出(`VIDEO_GENERATION`)

`model, byom, ratio, resolution, language, totalDurationSec, billedSeconds, billedTokens, segmentCount, extended, segments[ {index, taskId, durationSec, productRefs, persisted, tosKey, sourceUrl, url} ], stitched`。

成片 URL = `stitched.url`(多段)或 `segments[0].url`(单段),写入 run 的 `finalVideoUrl`。

## Seedance 参数(以文本 flag 附在 prompt 末尾)

`--ratio <aspectRatio>` `--duration <sec>` `--resolution <res>`。

## BYOM(自带模型)

被标记 BYOM 的账号必须用自带的 **Seedance Model ID + API Key**(仅用于本账号任务)。未配置 → 报错提示去 Account & Billing 配置(不消耗平台资源);仅支持 Seedance 模型。

## Dify 节点映射

这一步在 Dify 里是**工作流子流程**,不是单个 LLM 节点:

- **段提示词拼装**:**Code 节点** —— 输入 Skill 2 的 `scenes`,输出每段的 prompt 文本(含 `@<url>` 产品图、目标语种台词、`--ratio/--duration/--resolution` flag)。
- **分段条件**:**条件分支** —— `duration=15` 走单段;`duration=30` 走两段(第二段 `reference_video` = 第一段结果)。
- **提交生成**:**HTTP 请求节点** —— 调 Seedance `create` 异步任务,拿 `task_id`。
- **轮询**:**迭代/循环节点** —— 每 5s `GET` 查询任务状态,直到 `succeeded` / `failed` / 超时 10min 退出。**这是 Dify 里的主要实现难点**(异步任务必须用 loop + 条件退出)。
- **拼接**:多段时 **HTTP 请求节点**调 `STITCH_API_URL`(带 `STITCH_API_TOKEN`)合成完整 mp4;失败回退第一段 URL。
- **持久化**:**HTTP / 自定义工具节点**把成片上传 TOS。
- **产品参考图**:`@<url>` 需作为 Seedance 的 `reference_image` 传入 —— 在 Code 节点里从 `visual` 解析出 URL 列表。

在 Pipeline 中的位置:`PRODUCT_UNDERSTANDING → CREATIVE → [本步] → SUBTITLE_BURN`。
