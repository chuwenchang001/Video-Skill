# 在 Dify 上搭建本 Skill 的操作说明

本文档教你把 `Video-Skill` 这套广告视频 Pipeline 在 **Dify** 里搭成一个可运行的 **Workflow**。整条链路是 4 个顺序步骤:

```
PRODUCT_UNDERSTANDING → CREATIVE → VIDEO_GENERATION → SUBTITLE_BURN
      LLM 节点            LLM 节点      HTTP+循环          HTTP+循环
```

仓库里各资产的对应关系:

| 你要填的东西 | 从哪个文件拿 |
|---|---|
| LLM 节点的 System / User Prompt | `prompts/*.system.txt` / `*.user.txt` |
| LLM 节点的结构化输出 Schema | `schema/*.output.json` |
| 每步的输入字段定义 | `schema/*.input.json` |
| Skill 1 要内联的知识库 | `kb/platforms.json` · `kb/video-types.json` · `kb/categories.json` · `kb/few-shot-samples.json` |
| 各服务地址/密钥 | `.env.example` |

---

## 0. 前置准备

1. **一个 Dify 项目**(SaaS 版 `cloud.dify.ai` 或自托管均可),新建一个 **Workflow**(不是 Chatflow,这条链没有多轮对话)。
2. **一个 OpenAI 兼容模型**(Skill 1/2 用)。如果用火山方舟 ModelArk,在 Dify「设置 → 模型供应商 → OpenAI-API-compatible」里填:
   - API Base：`https://ark.ap-southeast.bytepluses.com/api/v3`
   - API Key：你的 `OPENAI_API_KEY`
   - 模型名:`gpt-4o`(或 ModelArk 上对应的文本模型 ID)
3. **三个外部 HTTP 服务的地址与密钥**(Skill 3/4 用),对照 `.env.example`:
   - Seedance 视频生成(火山方舟异步任务,复用 `OPENAI_BASE_URL` / `OPENAI_API_KEY`)
   - 拼接服务 `STITCH_API_URL` / `STITCH_API_TOKEN`
   - 字幕烧制 `SUBTITLE_API_BASE`
   - 对象存储 `TOS_*`
   > Dify 里没有 `.env`,这些值填进各 HTTP 节点,或用 Dify 的**环境变量 / 会话变量**统一管理(Workflow 顶部「环境变量」面板)。

---

## 1. 开始节点(Start):定义表单输入

在 Start 节点加以下输入字段(对应 `schema/1-product-understanding.input.json` + 全局参数):

| 变量名 | 类型 | 必填 | 默认 | 说明 |
|---|---|---|---|---|
| `product` | string | ✅ | — | 商品名称 |
| `features` | string(多行) | | — | 卖点,一行一个(Dify 无数组输入,进 Prompt 前拆分) |
| `price` | string | | — | 价格 |
| `scenario` | string | | — | 使用场景 |
| `brandTone` | string | | — | 品牌调性 |
| `productImageUrl` | string | | — | 产品图公网 URL |
| `platform` | select | | `douyin` | 见下方枚举 |
| `videoType` | select | | `spoken_grass` | 见下方枚举 |
| `language` | select | | `zh` | `zh,en,id,th,vi,ja,ko,ms` |
| `duration` | number/select | | `15` | 仅 `15` 或 `30` |
| `aspectRatio` | select | | `9:16` | `9:16,16:9,1:1,4:3,3:4,21:9` |

- `platform` 枚举:`tiktok, instagram, facebook, shopee, lazada, youtube, amazon, douyin, kuaishou, shipinhao, xiaohongshu`
- `videoType` 枚举:`spoken_grass, narrative_short, funny_drama, product_review, unboxing, comparison_experiment, kol_recommend`

---

## 2. Skill 1 · PRODUCT_UNDERSTANDING(LLM 节点)

1. 加一个 **LLM 节点**,模型选上面配好的文本模型,**temperature = 0.4**。
2. **System Prompt**:整段粘贴 `prompts/1-product-understanding.system.txt`。
3. **User Prompt**:粘贴 `prompts/1-product-understanding.user.txt`,把里面的占位符替换成 Dify 变量或内联知识库:
   - `{{categories_rendered}}` → 内联 `kb/categories.json` 渲染文本(见第 6 节)
   - `{{platform_kb_rendered}}` → 选中平台的 `kb/platforms.json` 条目
   - `{{video_type_kb_rendered}}` → 选中类型的 `kb/video-types.json` 条目
   - `{{few_shot_rendered}}` → `kb/few-shot-samples.json`
   - `{{product_info_json}}` → 由 Start 字段拼成的 JSON
   - `{{platform}}` / `{{videoType}}` → `{{#start.platform#}}` / `{{#start.videoType#}}`
4. **结构化输出**:开启 LLM 节点的「JSON Schema / 结构化输出」,粘贴 `schema/1-product-understanding.output.json`。若你的模型不支持强 Schema,退而用 `response_format=json_object` + 在 Prompt 末尾要求「只返回一个 JSON 对象」。
5. **多模态**:若 `productImageUrl` 非空,把它作为 LLM 节点的图片输入(需模型支持视觉)。

> 输出对象整体后面记作 `{{#skill1.output#}}`,其中 `recommendedDirection` 是下游要严格承接的方向。

---

## 3. Skill 2 · CREATIVE(LLM 节点)

1. 再加一个 **LLM 节点**,**temperature = 0.6**。
2. **System Prompt**:粘贴 `prompts/2-creative.system.txt`。
3. **User Prompt**:粘贴 `prompts/2-creative.user.txt`,占位符映射:
   - `{{PRODUCT_UNDERSTANDING_json}}` → `{{#skill1.output#}}`(Skill 1 整个输出)
   - `{{platform_label}}` / `{{platform_durationHint}}` / `{{platform_contentNorms}}` → 选中平台的 `kb/platforms.json` 字段
   - `{{videoType_label}}` / `{{videoType_strengths}}` → 选中类型的 `kb/video-types.json` 字段
   - `{{language_label}}` / `{{language_speak}}` → 语种表(见第 6 节语言映射)
   - `{{duration}}` → `{{#start.duration#}}`
   - `{{product_image_block}}` → 有图时放产品图说明,无图为空
   - `{{funny_drama_directive_if_applicable}}` → 见下
4. **搞笑短剧专项**:在这个 LLM 节点前加一个**条件分支(IF/ELSE)**判断 `videoType == funny_drama`;命中时把 `prompts/2-creative.funny_drama_directive.txt` 的内容拼到 User Prompt 的 `{{funny_drama_directive_if_applicable}}` 处(可用两套 Prompt 变体,或用变量插值)。
5. **结构化输出**:绑定 `schema/2-creative.output.json`。
6. **运行时覆盖(重要)**:LLM 之后接一个 **代码节点(Code)**:
   - 把 `output.durationSec` 强制改成 `start.duration`(15 或 30);
   - 把 `output.language` 回填成 `start.language`。
   > 因为下游分段逻辑严格依赖这个时长,不能信模型自填的值。

记该步最终输出为 `{{#skill2.output#}}`,核心是 `scenes` 分镜数组。

---

## 4. Skill 3 · VIDEO_GENERATION(Seedance 异步 + 拼接)

这一步没有 LLM,是一段**编排子流程**。建议顺序:

### 4.1 段提示词拼装(Code 节点)
输入 `{{#skill2.output#}}` 的 `scenes` + `start.language/duration/aspectRatio`,输出:
- `segments`:按分段规则算出的每段 prompt 文本
  - `duration ≤ 15` 或 `scenes < 2` → **单段**;
  - `duration == 30` → **两段**(前 15s 一段 + 余下一段);每段时长夹取到 `[5,10,15]` 里 ≤(总/2) 的最大档。
- 每段 prompt 末尾追加:该段台词(目标语种)+ 强语言指令「全片配音用目标语种,**不要任何字幕/文字/硬字幕**」+ flag `--ratio <aspectRatio> --duration <sec> --resolution <VIDEO_RESOLUTION>`。
- 从每段 `visual` 里解析 `@<url>` 标记,作为该段的 `reference_image` 列表。

### 4.2 提交生成(HTTP 请求节点)
- 调火山方舟 Seedance **创建异步任务**接口(`{{OPENAI_BASE_URL}}` 下的视频任务端点),`Authorization: Bearer {{OPENAI_API_KEY}}`。
- body 里带段 prompt、`reference_image`;续写段额外带上一段的 `video_url` 作 `reference_video`。
- 返回拿 `task_id`。

### 4.3 轮询到终态(迭代/循环节点)★难点
- 用 **迭代节点(Loop)**:每次 `GET` 任务状态,间隔 **5s**,直到状态为 `succeeded`(取 `video_url`)/ `failed`(报错)/ 累计超时 **10 分钟** 退出。
- Dify 循环里用「条件退出 + 计数上限」实现;拿到 `video_url` 后跳出。

### 4.4 两段续写与拼接(条件分支 + HTTP 节点)
- `duration == 30` 时:第一段完成后,把它的 `video_url` 作为 `reference_video` 提交第二段,重复 4.2–4.3。
- 两段都完成后,调**拼接服务**:`POST {{STITCH_API_URL}}`(带 `STITCH_API_TOKEN`)合成完整 mp4;失败不致命,回退第一段 URL。
- `duration == 15` 时直接用单段结果,跳过拼接。

### 4.5 持久化(HTTP / 自定义工具)
- 把成片上传 **TOS**(`TOS_*`);未配置则直接用 Seedance 返回的临时 URL(约 24h 过期)。

**本步输出**记为 `{{#skill3.output#}}`,成片 URL = `stitched.url`(多段)或 `segments[0].url`(单段)。

---

## 5. Skill 4 · SUBTITLE_BURN(字幕烧制,HTTP 异步)

### 5.1 取源片(Code 节点)
从 `{{#skill3.output#}}` 取 `stitched.url ?? segments[0].url`;缺失则报错。

### 5.2 定语言
用 `start.language`;若不在白名单 `zh,en,id,th,vi,ja,ko,ms` 内,则传 `auto` 由服务识别。

### 5.3 提交(HTTP 请求节点)
```
POST {{SUBTITLE_API_BASE}}/api/subtitle/burn
Content-Type: application/json
{ "video_url": "<成片URL>", "language": "<语种id|auto>", "style": "{{SUBTITLE_STYLE|default}}" }
→ { "task_id": "..." }        # 非 2xx(如 422)报错
```

### 5.4 轮询(迭代/循环节点)
```
GET {{SUBTITLE_API_BASE}}/api/subtitle/task/{task_id}
每 3s 一次,超时 10min
终态: 响应文本含 "success"(取 video_url) / "failed"(报错)
```

### 5.5 降级分支(条件分支)
- `SUBTITLE_BURN_REQUIRED = false`(默认):烧字失败 → 保留**无字幕成片**作为最终结果(`burned=false`),Workflow 仍成功。
- `= true`:失败即让整条 Workflow 失败。

### 5.6 持久化 + 最终成片
把带字幕成片上传 TOS(`videos/<runId>/final_subtitled.mp4`),把 **`finalVideoUrl` 设为这条带字幕成片**,作为 End 节点输出。

---

## 6. 知识库(KB)怎么进 Prompt

`kb/*.json` 是**配置型**知识,不是检索型 —— **不要**做成 Dify 知识库(RAG),直接内联进 Prompt:

- **推荐**:在 Skill 1 的 LLM 节点前放一个 **Code 节点**,`require`/内置这几个 JSON 常量,根据 `start.platform` / `start.videoType` 挑出对应条目,渲染成文本,输出成变量后插进 User Prompt 的占位符。
- **简化版**:若嫌麻烦,可把 `kb/categories.json`、`kb/few-shot-samples.json`(体量较大)整段贴进 System/User Prompt 固定部分;平台/类型只把**当前选中项**那一条贴进去,省 token。

**语言映射**(`language` id → label / speak,给 Prompt 里的 `{{language_label}}` / `{{language_speak}}` 用):

| id | label | speak(喂模型的配音语言) |
|---|---|---|
| zh | 中文 | 中文(普通话) |
| en | 英文 | 英语 |
| id | 印尼文 | 印尼语 |
| th | 泰文 | 泰语 |
| vi | 越南语 | 越南语 |
| ja | 日文 | 日语 |
| ko | 韩文 | 韩语 |
| ms | 马来文 | 马来语 |

---

## 7. 整体节点图

```
[Start 表单]
   │
   ▼
[LLM] Skill1 PRODUCT_UNDERSTANDING ──(recommendedDirection)
   │
   ▼
[IF videoType==funny_drama?] ──► [LLM] Skill2 CREATIVE ──► [Code 覆盖时长/语言]
   │                                                              │(scenes)
   ▼                                                              ▼
[Code 段提示词拼装] ──► [HTTP 提交 Seedance] ──► [Loop 轮询5s] ──┐
   │                                                             │
   ▼ (duration==30 再来一段续写)                                 │
[HTTP 拼接 STITCH] ──► [HTTP 上传 TOS] ──(成片URL)──────────────┘
   │
   ▼
[Code 取源片] ──► [HTTP 提交字幕burn] ──► [Loop 轮询3s]
   │
   ▼
[IF 成功? / SUBTITLE_BURN_REQUIRED] ──► [HTTP 上传 TOS] ──► [End: finalVideoUrl]
```

---

## 8. 冒烟测试

先用最小输入跑通 15s 单段,再测 30s 两段:

```json
{
  "product": "氨基酸温和洁面慕斯",
  "features": "氨基酸表活\n一泵出绵密泡\n敏感肌可用",
  "price": "79 元",
  "scenario": "早晚洁面 / 敏感肌换季",
  "platform": "douyin",
  "videoType": "spoken_grass",
  "language": "zh",
  "duration": 15,
  "aspectRatio": "9:16"
}
```

预期:Skill1 归类到 `beauty_personal_care` 并给出 `recommendedDirection` → Skill2 出 3–8 个分镜 → Skill3 出一段成片 URL → Skill4 烧字幕产出 `finalVideoUrl`。更多期望可参考 `kb/few-shot-samples.json` 里的范例。

---

## 9. 常见坑

| 现象 | 原因 / 解法 |
|---|---|
| Skill1/2 输出不是合法 JSON | 模型不支持强 Schema —— 用 `response_format=json_object`,并在 Prompt 末尾强调「只返回一个 JSON 对象,不要 markdown 围栏」 |
| 视频时长跟用户选的不一致 | 一定要有 Skill2 之后的 **Code 覆盖节点**,强制 `durationSec = start.duration` |
| Seedance/字幕一直 pending | 异步任务必须用 **Loop 轮询**,不能只调一次;注意设超时上限退出 |
| 成片带了字幕又叠了硬字幕 | Skill3 段提示词末尾要写「no subtitles / no captions / no on-screen text」,字幕只由 Skill4 烧制 |
| 产品长得不像 | 分镜 `visual` 里的 `@<url>` 必须作为 Seedance 的 `reference_image` 传入,别只当普通文本 |
| 30s 第二段画面接不上 | 第二段要把第一段 `video_url` 作 `reference_video` 传入做 extension 续写 |
| 成片 URL 隔天失效 | 没配 TOS 时用的是 Ark 临时 URL(约 24h);要长期保存必须上传 TOS |

---

各步更细的字段定义与规则见 [`README.md`](README.md) 与 `steps/1~4.md`。
