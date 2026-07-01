# Skill 4 · 字幕烧制(SUBTITLE_BURN)

Video Pipeline 末步。取 Skill 3 成片 URL,调用**外部字幕烧制 HTTP 服务**给成片烧入字幕,轮询到终态后把带字幕成片转存 TOS,作为 run 的**最终视频**(`finalVideoUrl`)。**工具型 Skill,纯 HTTP API 调用,无 LLM。**

| 项 | 值 |
|---|---|
| 类型 | 工具(外部 HTTP API + 轮询) |
| Base URL | env `SUBTITLE_API_BASE`,默认 `https://api.videodreamina.ai` |
| 提交 | `POST /api/subtitle/burn` → `{ task_id }` |
| 轮询 | `GET /api/subtitle/task/{task_id}`,3s/次,超时 600000ms(10min) |
| Style | env `SUBTITLE_STYLE`,默认 `default` |
| 烧字必需 | env `SUBTITLE_BURN_REQUIRED`,默认 `false` |
| 持久化 | 对象存储 TOS |
| 输入 Schema | [`schema/4-subtitle-burn.input.json`](../schema/4-subtitle-burn.input.json) |
| 输出 Schema | [`schema/4-subtitle-burn.output.json`](../schema/4-subtitle-burn.output.json) |

## 核心流程

1. **取源片**:从上游 `VIDEO_GENERATION` 取成片 URL —— 优先 `stitched.url`,否则 `segments[0].url`;缺失抛错。
2. **定语言**:用 `run.input.language`(在白名单 `zh,en,id,th,vi,ja,ko,ms` 内)透传;不在白名单则传 `auto` 由服务识别。字体由服务端按 language 选择(th→Noto Sans Thai｜zh→CJK TC｜ja→CJK JP｜ko→CJK KR｜其他→Noto Sans),本 Skill 只透传。
3. **提交**:`POST {SUBTITLE_API_BASE}/api/subtitle/burn`,body `{ video_url, language, style }` → 返回 `task_id`(非 2xx 抛错)。
4. **轮询**:`GET {SUBTITLE_API_BASE}/api/subtitle/task/{task_id}`,3s/次,超时 10min;响应文本含 `"success"`(取 `video_url`)/ `"failed"`(报错)为终态。
5. **持久化**:带字幕成片转存 TOS(`videos/<runId>/final_subtitled.mp4`);未配 TOS 则用服务返回 URL。
6. **设为最终成片**:`finalVideoUrl` 覆盖为带字幕成片。

## 失败降级

由 env `SUBTITLE_BURN_REQUIRED` 控制:

- **默认 `false`**:烧字失败 → 降级,返回 `burned=false` 并保留**无字幕成片**为 `finalVideoUrl`,run 仍完成。
- `true`:失败即抛错,由编排器 `failRun`。

## 输入

- 上游 `VIDEO_GENERATION`(**必须**)
- `language`(默认 `zh`)

## 输出(`SUBTITLE_BURN`)

`burned, language, taskId, sourceVideoUrl, subtitledUrl, persisted`(成功);或 `burned=false, language, sourceVideoUrl, error`(降级)。

## 外部 API 契约

```
提交  POST {SUBTITLE_API_BASE}/api/subtitle/burn
      body: { "video_url": "<成片URL>", "language": "<语种id|auto>", "style": "<SUBTITLE_STYLE>" }
      → { "task_id": "..." }        # 非 2xx(如 422 字段非法)抛错

轮询  GET  {SUBTITLE_API_BASE}/api/subtitle/task/{task_id}
      每 3s 一次,超时 600000ms
      终态: 响应文本含 "success"(取 video_url) / "failed"(报错)
```

## Dify 节点映射

- **取源片**:**Code 节点** —— 从 Skill 3 输出取 `stitched.url ?? segments[0].url`。
- **提交**:**HTTP 请求节点** —— `POST /api/subtitle/burn`,body 见上,拿 `task_id`。
- **轮询**:**迭代/循环节点** —— 每 3s `GET` 任务状态,直到 `success` / `failed` / 超时退出。
- **降级分支**:**条件分支** —— 失败时按 `SUBTITLE_BURN_REQUIRED` 决定:`false` 保留无字幕成片继续(`burned=false`),`true` 则整条工作流失败。
- **持久化**:**HTTP / 自定义工具节点**把带字幕成片上传 TOS,并把 `finalVideoUrl` 设为该成片。

在 Pipeline 中的位置:`PRODUCT_UNDERSTANDING → CREATIVE → VIDEO_GENERATION → [本步](产出最终成片)`。
