# Video-Skill · 广告视频生成 Pipeline(供 Dify 导入)

把「一个商品 → 一支可投放的广告短视频」拆成 **4 个顺序执行的 Skill**,并整理成可直接在 **Dify** 里搭建工作流的说明型文档:每步的角色、System / User Prompt、输入输出 JSON Schema、内置知识库、外部接口契约,以及 **Dify 节点映射**。

> 本仓库是**说明文档 + 可复用资产(Prompt / Schema / KB)**,不是可执行代码。它来自一套已上线的 Video Pipeline,内容与线上实现一致。

## 执行链

```
PRODUCT_UNDERSTANDING → CREATIVE → VIDEO_GENERATION → SUBTITLE_BURN
      (Skill 1)          (Skill 2)     (Skill 3)          (Skill 4)
```

顺序执行,失败即停,已成功步骤可断点续跑。

| # | Skill | 类型 | 作用 | 文档 |
|---|---|---|---|---|
| 1 | `PRODUCT_UNDERSTANDING` | LLM | 自动归类 + 商品理解 + 平台/视频类型策略 + 用五维量表决策唯一最优创意方向 | [steps/1-product-understanding.md](steps/1-product-understanding.md) |
| 2 | `CREATIVE` | LLM | 严格承接最优方向,决策总时长并拆成 3–8 个分镜,直接产出可喂给视频模型的分镜脚本 | [steps/2-creative.md](steps/2-creative.md) |
| 3 | `VIDEO_GENERATION` | 工具 | 调火山方舟 Seedance 异步生成(15s 单段 / 30s 两段续写)+ FFmpeg 拼接 + 落 TOS | [steps/3-video-generation.md](steps/3-video-generation.md) |
| 4 | `SUBTITLE_BURN` | 工具 | 调外部 HTTP 服务给成片烧字幕,轮询到终态后转存 TOS,产出最终成片 | [steps/4-subtitle-burn.md](steps/4-subtitle-burn.md) |

> 历史枚举说明:原「创意概念」与「分镜脚本」两步已合并为 `CREATIVE`。

## 数据流(上游输出 → 下游消费)

```
run.input(product / features / platform / videoType / language / duration / aspectRatio / productImageUrl …)
  └─> PRODUCT_UNDERSTANDING ──(recommendedDirection 等)──> CREATIVE
         └─> CREATIVE ──(scenes 分镜脚本)──> VIDEO_GENERATION
                └─> VIDEO_GENERATION ──(stitched.url / segments[0].url)──> SUBTITLE_BURN
                       └─> SUBTITLE_BURN ──> finalVideoUrl(run 最终成片)
```

## 目录结构

```
Video-Skill/
├── README.md                 # 本文件:总览 + Dify 导入 + 节点映射 + 共享 env
├── .env.example              # 环境变量占位(真实 .env 不进仓库)
├── steps/                    # 每步一份说明:角色 / Prompt / I/O / Dify 节点映射
│   ├── 1-product-understanding.md
│   ├── 2-creative.md
│   ├── 3-video-generation.md
│   └── 4-subtitle-burn.md
├── schema/                   # 各步输入/输出 JSON Schema(可直接贴进 Dify 结构化输出)
│   ├── 1-product-understanding.input.json / .output.json
│   ├── 2-creative.input.json / .output.json
│   ├── 3-video-generation.input.json / .output.json
│   └── 4-subtitle-burn.input.json / .output.json
├── prompts/                  # 原始 System / User Prompt(复制即用)
│   ├── 1-product-understanding.system.txt / .user.txt
│   └── 2-creative.system.txt / .user.txt / .funny_drama_directive.txt
└── kb/                       # Skill 1 内置知识库(平台 / 视频类型 / 类目 / Few-shot)
    ├── platforms.json
    ├── video-types.json
    ├── categories.json
    └── few-shot-samples.json
```

> 📘 **完整的逐节点搭建操作手册见 [DIFY_GUIDE.md](DIFY_GUIDE.md)**(前置准备 / Start 表单 / 4 步节点配置 / KB 注入 / 冒烟测试 / 常见坑)。下面是速览。

## 在 Dify 里怎么搭(节点映射)

整条 pipeline 建议做成一个 **Dify 工作流(Workflow)**,4 个 Skill 依次对应下列节点:

| Pipeline 步骤 | Dify 节点类型 | 落地要点 |
|---|---|---|
| **Skill 1 / 2**(理解决策、分镜脚本) | **LLM 节点** + 结构化输出 | `prompts/*.system.txt` 填 System、`*.user.txt` 填 User;用 Dify「结构化输出 / JSON Schema」绑定 `schema/*.output.json`;`response_format=json_object`;temperature 见各步文档(0.4 / 0.6) |
| **Skill 1 的知识库**(平台 / 类目 / 人群 / Few-shot) | **内联进 Prompt** 或 **Code 节点常量** | `kb/*.json` 是**配置型**知识、不是检索型,渲染进 User Prompt 即可,**不建议**做成 Dify 知识库(RAG) |
| **Skill 3 生成 / Skill 4 烧字幕** | **HTTP 请求节点 / 自定义工具** | 外部接口契约见各步文档:Seedance 异步 `create + poll`、FFmpeg `stitch`、字幕 `burn`;鉴权走对应 env |
| **Seedance / 字幕的异步轮询** | **迭代 / 循环节点** | 提交拿 `task_id` 后要 loop 轮询到终态(Seedance 5s/次超时 10min;字幕 3s/次超时 10min)——这是 Dify 里的**主要实现难点**,需用循环 + 条件退出 |
| **30s 两段续写 + 拼接** | **条件分支 + HTTP 节点** | `duration=30` 才拆两段:第二段用第一段 `video_url` 作 `reference_video` 续写,再调 stitch 合成;`duration=15` 直接单段 |
| **变量传递** | 上游节点输出 → 下游 `{{}}` 引用 | 按上面「数据流」把 `recommendedDirection`、`scenes`、成片 URL 逐级引用 |

导入顺序建议:先把 Skill 1/2 两个 LLM 节点跑通(纯 Prompt + 结构化输出,最简单),再接 Skill 3/4 的 HTTP + 轮询。

## 共享环境变量

见 [.env.example](.env.example)。核心分组:

| 分组 | 变量 | 用途 | 默认 |
|---|---|---|---|
| LLM | `OPENAI_BASE_URL` / `OPENAI_API_KEY` / `OPENAI_MODEL` | Skill 1/2 文本模型(可指向 ModelArk) | 官方 / — / `gpt-4o` |
| 视频 | `VIDEO_MODEL` / `VIDEO_RESOLUTION` | Seedance 模型与分辨率 | `seedance-1-0-lite-t2v-250428` / `720p` |
| 拼接 | `STITCH_API_URL` / `STITCH_API_TOKEN` | 多段 FFmpeg 拼接服务 | — |
| 字幕 | `SUBTITLE_API_BASE` / `SUBTITLE_STYLE` / `SUBTITLE_BURN_REQUIRED` | 字幕烧制服务 | `https://api.videodreamina.ai` / `default` / `false` |
| 存储 | `TOS_ENDPOINT` / `TOS_REGION` / `TOS_BUCKET` / `TOS_ACCESS_KEY` / `TOS_SECRET_KEY` | 成片持久化(火山 TOS) | — |

> 视频生成(Seedance)复用 `OPENAI_API_KEY` 与 `OPENAI_BASE_URL`。BYOM 账号需用自带 Seedance Model ID + Key,详见 Skill 3 文档。

## 全局约定

- **语言**:面向用户展示的可读字段一律**英文**;枚举 id 保持原样;成片台词/配音语种由 `run.input.language` 决定(`zh, en, id, th, vi, ja, ko, ms`)。
- **时长**:仅 **15s**(单段)/ **30s**(两段续写);运行时强制覆盖模型输出的 `durationSec`。
- **画幅**:`9:16`(默认)/ `16:9` / `1:1` / `4:3` / `3:4` / `21:9`。
- **产品一致性**:分镜 `visual` 里的 `@<产品图片URL>` 标记会作为 `reference_image` 传给 Seedance,严格还原产品外观。
- **合规**:各类目的 `riskNotes`(医疗/绝对化/焦虑营销等红线)任何创意方向都不得触碰。
