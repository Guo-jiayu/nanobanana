# 电商详情图一键生成 — 设计文档

**日期**：2026-04-16
**目标产物**：项目根目录新增独立 HTML 文件 `电商详情图.html`

## 背景与目标

项目已有 `单张.html`（单图文生图/图生图）和 `批量一对一生图.html`（每张参考图独立生成一张结果图）。本次新增"电商详情图一键生成"场景：

- 用户上传 1 张或多张同一商品的图片（不同角度，或商品 + 赠品）
- 用户输入一段商品介绍文本
- 点一次按钮，由系统自动：
  1. 调多模态 LLM，让它基于图与文字规划 N 条文生图提示词
  2. 并发调用文生图 API，按规划生成所有详情图

成功标准：上传图与介绍后单击 → 一段时间后看到一组可下载的详情图，无需中间人工干预；失败的图能就地重试。

## 关键决策（来自 brainstorming）

| 决策点 | 选择 |
|---|---|
| UX 流程 | 全自动直通，无中间审核步骤；失败可单张重试 |
| LLM 与图像 API | 同 endpoint (`https://sparkcode.top/v1/chat/completions`)，**两个独立的 API Key**，模型名分别填 |
| LLM 输出格式 | 严格 JSON 数组：`[{ref_index, prompt, size?, aspect_ratio?}, ...]` |
| 生成数量 | UI 默认"自动"（LLM 在 3-8 之间决定）；用户可强制 1-10 |
| System Prompt | 写死在代码里，UI 不暴露 |
| 尺寸/宽高比 | UI 强制覆盖 > LLM 自定 > API 默认 |
| 页面位置 | 独立页面，不并入 `生图工作台.html` |
| 持久化 | 仅 localStorage 存配置；任务/图片留内存，刷新即清 |
| 结果展示 | 极简：只展示生成图 + 下载；失败显示占位 + 重试 |
| 并发与重试 | 默认并发 4，单张最多 5 次重试，仅 `CONTENT_FILTER` 自动重试 |

## §1 总体架构与数据流

**载体**：单个独立 HTML 文件 `电商详情图.html`，沿用项目现有约定（vanilla JS + 内联 style + 内联 script，无构建工具）。

**两阶段流水线**（点一次"一键生成"触发整条）：

```
  ┌───────────────────────────────┐
  │ 阶段 0: 收集输入             │
  │   • 1+ 商品图 (data URL)      │
  │   • 商品介绍文本              │
  │   • 数量 (自动 / 1-10)        │
  │   • 默认 size / aspect (可空) │
  └──────────────┬────────────────┘
                 ▼
  ┌───────────────────────────────┐
  │ 阶段 1: LLM 提示词生成 (1次)  │
  │   • 模型: 用户填的 LLM 模型名 │
  │   • Key:  LLM API Key         │
  │   • System Prompt: 写死       │
  │   • 输入: 全部图 + 商品介绍   │
  │            + "请生成 N 条/在  │
  │             8 条以内自动决定" │
  │   • 输出: 严格 JSON 数组      │
  │     [{ref_index, prompt,      │
  │       size?, aspect_ratio?}]  │
  └──────────────┬────────────────┘
                 ▼
  ┌───────────────────────────────┐
  │ 阶段 2: 并发文生图 (N 次)     │
  │   • Worker 池, 默认并发 4     │
  │   • 模型: 图像模型名          │
  │   • Key:  图像 API Key        │
  │   • 每条任务:                 │
  │     - 取 ref_images[idx-1]    │
  │     - prompt 文本             │
  │     - size/aspect: UI 强制    │
  │       覆盖 > LLM > 默认       │
  │   • 单图最多 5 次重试         │
  │     (仅 CONTENT_FILTER 自动)  │
  └──────────────┬────────────────┘
                 ▼
  ┌───────────────────────────────┐
  │ 极简结果网格                  │
  │   • 成功: 缩略 + 下载         │
  │   • 失败: 占位 + "重试这张"   │
  └───────────────────────────────┘
```

## §2 界面区块与组件

页面从上到下：标题栏 → 配置卡 → 商品图卡 → 商品介绍卡 → 结果卡 → 底部状态栏。

```
┌─────────────────────────────────────────────────────────────┐
│  电商详情图 · 一键生成              [🗑️ 清空] [📦 批量下载]  │
├─────────────────────────────────────────────────────────────┤
│ ⚙️ 配置                                                      │
│  ┌──────────────────────┬──────────────────────────────┐    │
│  │ LLM API Key  [____]  │ 图像 API Key  [____]         │    │
│  │ LLM 模型 [gemini-... ]│ 图像模型 [gemini-3.1-flash..]│    │
│  │ 数量 [自动▾]  并发 [4▾]                              │    │
│  │ 强制尺寸 [▾留空=LLM]  强制比例 [▾留空=LLM]          │    │
│  └──────────────────────┴──────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│ 🖼️ 商品图  (拖入或点击上传，可多张)                           │
│  ┌────┐┌────┐┌────┐┌────┐  + 上传                            │
│  │ #1 ││ #2 ││ #3 ││ #4 │                                   │
│  │ ×  ││ ×  ││ ×  ││ ×  │                                   │
│  └────┘└────┘└────┘└────┘                                   │
├─────────────────────────────────────────────────────────────┤
│ 📝 商品介绍                                                  │
│  ┌────────────────────────────────────────────────────┐     │
│  │ 这是一款 ... (textarea, 6 行)                      │     │
│  └────────────────────────────────────────────────────┘     │
├─────────────────────────────────────────────────────────────┤
│ 🎨 生成结果                                                  │
│  ┌────┐┌────┐┌────┐┌────┐┌────┐                             │
│  │图1 ││图2 ││失败││图4 ││图5 │                             │
│  │💾 ││💾 ││🔄 ││💾 ││💾 │                             │
│  └────┘└────┘└────┘└────┘└────┘                             │
├─────────────────────────────────────────────────────────────┤
│ 状态: LLM 生成提示词中... | 图: 3/5 完成                     │
│                                  [🚀 一键生成详情图]          │
└─────────────────────────────────────────────────────────────┘
```

**组件清单**：

| 单元 | 职责 | 输入 | 输出 |
|---|---|---|---|
| **ConfigPanel** | 渲染并读写所有配置；onchange→localStorage | localStorage | 发起请求时读取当前值 |
| **RefImageUploader** | 文件上传（input + 拖拽）；缩略图列；删除按钮；维护 `refImages: string[]` | File 事件 | data URL 数组 |
| **PromptInput** | textarea 商品介绍 | — | 文本字符串 |
| **Generator** | 协调阶段 1 + 阶段 2 的状态机；点"一键生成"入口 | 上面三者的当前值 | 写 `tasks: Task[]` 与全局 `phase` 状态 |
| **ResultGrid** | 渲染结果网格；监听 tasks 变化重渲染对应卡片 | tasks 数组 | DOM |
| **StatusBar** | 底部一行状态：当前阶段 + 进度 `成功/失败/总数` | phase + tasks | DOM |
| **ErrorBanner** | 顶部红/黄条：LLM 网络错 / JSON 解析错 + "重试 LLM" 按钮 | error 对象 | DOM |
| **ClearButton** | 清空 `refImages` / `productDescription` / `tasks` / `llmRawResponse`；**不动**配置；需二次确认 | — | state 重置 |
| **DownloadAllButton** | 仅下载所有 `status === 'done'` 的图，每张间隔 300ms 触发 `<a download>`（同批量页） | tasks | 浏览器下载 |

**核心状态对象**（页面级单例，不入库）：
```js
state = {
  refImages: ['data:image/...', ...],          // 上传图
  productDescription: '',
  phase: 'idle' | 'llm' | 'generating' | 'done' | 'llm_failed',
  llmRawResponse: null,                         // 解析失败时给用户看
  tasks: [{
    id, refIndex, prompt, size, aspect,         // LLM 生成的输入
    status: 'wait' | 'running' | 'done' | 'error',
    result: null, error: null, attempts: 0
  }, ...]
}
```

## §3 阶段 1 协议（LLM 调用 + JSON 解析）

**HTTP 请求体**：
```json
{
  "model": "<UI: LLM 模型名>",
  "messages": [
    { "role": "system", "content": "<写死的中文 system prompt>" },
    { "role": "user", "content": [
        { "type": "image_url", "image_url": { "url": "<refImages[0]>" } },
        { "type": "image_url", "image_url": { "url": "<refImages[1]>" } },
        { "type": "text", "text": "商品介绍：\n<用户输入>\n\n请生成 <N|不超过8> 条文生图提示词。" }
    ]}
  ],
  "max_tokens": 4096
}
```
- Header：`Authorization: Bearer <LLM API Key>`
- 调用 `https://sparkcode.top/v1/chat/completions`
- 这一次**不带** `extra_body.google.image_config`（那是图像生成参数）

**写死的 System Prompt（草稿，写代码时可润色）**：
> 你是电商详情图设计专家。基于用户提供的商品图与商品介绍，规划 N 张电商详情图（主图 / 卖点图 / 使用场景图 / 细节特写 / 规格说明 / 对比图 等中合理选择若干），为每张图生成一条独立、可直接送入文生图模型执行的中文提示词。
>
> 输出**只能**是一个 JSON 数组，不要包裹 markdown 代码块、不要任何额外文字。每个元素结构：
> ```
> { "ref_index": <从 1 开始的整数，指向用户上传的第几张商品图>,
>   "prompt": "<完整的、信息自包含的中文文生图提示词，描述构图、风格、光影、文字、布局>",
>   "size": "<可选: 512 | 1K | 2K | 4K>",
>   "aspect_ratio": "<可选: 1:1 | 16:9 | 9:16 | 4:3 | 3:4>" }
> ```
>
> - `ref_index` 必须在用户上传的图片范围内
> - 当用户在介绍中明确指定了尺寸/比例需求时，请填入 `size` / `aspect_ratio`
> - 未要求时省略这两个字段
> - 提示词需自包含商品的关键视觉信息，不要假设模型已经"看过"参考图
>
> 当用户指定了具体张数时严格按数量返回；未指定时返回 3-8 条之间的合理数量。

**响应解析（多层兜底，宽进严出）**：
```js
function parseLLMResponse(data) {
  const content = data.choices?.[0]?.message?.content;
  if (!content || typeof content !== 'string')
    throw new ParseError('LLM 未返回文本内容', data);

  // 尝试 1: 整体当 JSON
  // 尝试 2: 抓 ```json ... ``` 或 ``` ... ``` 包裹
  // 尝试 3: 抓首个 [ ... ] 平衡数组
  let arr;
  try { arr = JSON.parse(content); }
  catch {
    const fenced = content.match(/```(?:json)?\s*([\s\S]+?)```/);
    if (fenced) { try { arr = JSON.parse(fenced[1]); } catch {} }
  }
  if (!arr) {
    const m = content.match(/\[[\s\S]*\]/);
    if (m) { try { arr = JSON.parse(m[0]); } catch {} }
  }
  if (!Array.isArray(arr))
    throw new ParseError('未能解析为 JSON 数组', content);

  // 严格校验每一项
  return arr.map((item, i) => {
    if (typeof item.ref_index !== 'number' || item.ref_index < 1
        || item.ref_index > refImages.length)
      throw new ParseError(`第 ${i+1} 项 ref_index 越界: ${item.ref_index}`, content);
    if (typeof item.prompt !== 'string' || !item.prompt.trim())
      throw new ParseError(`第 ${i+1} 项 prompt 为空`, content);
    return {
      refIndex: item.ref_index,
      prompt: item.prompt.trim(),
      size: typeof item.size === 'string' ? item.size : null,
      aspect: typeof item.aspect_ratio === 'string' ? item.aspect_ratio : null
    };
  });
}
```

**失败兜底**（落到 ErrorBanner）：
- HTTP / 网络错 → 红条："LLM 调用失败：<msg>" + [重试 LLM]；`phase = 'llm_failed'`
- 解析失败（含空数组、`ref_index` 越界、`prompt` 空）→ 黄条："LLM 返回无法解析" + [展开看原文（折叠区显示 `llmRawResponse`）] + [重试 LLM]；`phase = 'llm_failed'`
- 重试 LLM = 重跑阶段 1（仅在 `phase === 'llm_failed'` 时可见）

## §4 阶段 2 协议（并发文生图 + 单张重试）

**任务结构**（阶段 1 解析后逐条变成 task）：
```js
{
  id: <自增>,
  refIndex: 2,                    // 用哪张参考图（1-based）
  prompt: '白底主图，...',
  size: '1K' | null,              // LLM 给的
  aspect: '16:9' | null,
  status: 'wait',
  result: null,                   // 成功后存 data URL
  error: null,
  attempts: 0
}
```

**配置快照**（关键）：在阶段 2 启动瞬间从 DOM 取一次值，传给所有 worker；批次中途用户改了配置不会影响当前批次。

```js
function snapshotImageConfig() {
  return {
    apiKey:   $('imageApiKey').value.trim(),
    model:    $('imageModel').value.trim(),
    uiSize:   $('forceSize').value,    // '' 表示未指定
    uiAspect: $('forceAspect').value
  };
}
```

**单张请求体生成**（参考批量页 `callGeminiAPI`，调整 size/aspect 优先级）：
```js
function buildImageRequest(task, snap) {
  // 优先级: UI 强制 > LLM 给的 > 不指定
  const size   = snap.uiSize   || task.size   || null;
  const aspect = snap.uiAspect || task.aspect || null;

  const body = {
    model: snap.model,
    messages: [{
      role: 'user',
      content: [
        { type: 'image_url', image_url: { url: refImages[task.refIndex - 1] } },
        { type: 'text', text: task.prompt }
      ]
    }],
    max_tokens: 4096
  };

  const cfg = {};
  if (size)   cfg.image_size = size;
  if (aspect) cfg.aspect_ratio = aspect;
  if (Object.keys(cfg).length) body.extra_body = { google: { image_config: cfg } };
  return body;
}
```
- Header：`Authorization: Bearer ${snap.apiKey}`
- 同 endpoint

**Worker 池**（直接复用批量页模式，简化版）：
```js
async function runImageStage() {
  const concurrency = parseInt($('concurrency').value);
  const snap = snapshotImageConfig();          // 一次快照，全批次共享
  const queue = state.tasks.filter(t => t.status === 'wait');
  const MAX_RETRIES = 5;

  const workers = Array(Math.min(concurrency, queue.length))
    .fill(0)
    .map(async () => {
      while (queue.length) {
        const task = queue.shift();
        await runOneTask(task, snap, MAX_RETRIES);
      }
    });
  await Promise.all(workers);
}

async function runOneTask(task, snap, maxRetries) {
  task.status = 'running'; renderResultCard(task);
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    task.attempts = attempt;
    try {
      const url = await callImageAPI(task, snap);  // 解析逻辑直接复用批量页
      task.result = url; task.status = 'done'; task.error = null;
      break;
    } catch (e) {
      const retryable = e.message.includes('CONTENT_FILTER');
      if (attempt < maxRetries && retryable) {
        await sleep(1000 + Math.random() * 1000);
        continue;
      }
      task.status = 'error';
      task.error = attempt > 1 ? `${attempt}次尝试均失败: ${e.message}` : e.message;
      break;
    }
  }
  renderResultCard(task);
  updateStatusBar();
}
```

**响应解析**：直接复用批量页 `callGeminiAPI` 中 content 解析分支（content 数组 / 字符串 / parts.inline_data / data[].b64_json / data[].url 五种兜底路径），只把"调用部分"换成读 task 字段。

**单张"重试这张"按钮**：
```js
async function retryOne(taskId) {
  const task = state.tasks.find(t => t.id === taskId);
  task.status = 'wait'; task.error = null; task.attempts = 0;
  await runOneTask(task, snapshotImageConfig(), 5);  // 重试时取当前 UI 配置（合理：用户改了再点重试就该用新值）
}
```
- 重试不抢占 worker 池，直接在按钮回调里跑（一次只重试一张；连击会同时跑多张，可以接受）。
- 与"启动时快照"语义不冲突：批次内是快照，重试是新一次"用户操作"，重新读当前 UI。

## §5 边界情况、错误处理与"测试"

**输入校验**（点"一键生成"瞬间检查，不通过直接 alert + return）：

| 检查 | 失败信息 |
|---|---|
| LLM API Key 为空 | "请填写 LLM API Key" |
| 图像 API Key 为空 | "请填写 图像 API Key" |
| `refImages.length === 0` | "请至少上传一张商品图" |
| `productDescription.trim() === ''` | "请填写商品介绍" |
| LLM 模型/图像模型为空 | "请填写模型名" |

**运行时边界**：

| 场景 | 处理 |
|---|---|
| 阶段 1 LLM 网络错 / HTTP 4xx-5xx | ErrorBanner 红条 + [重试 LLM]；`phase = 'llm_failed'` |
| 阶段 1 JSON 解析失败 | ErrorBanner 黄条 + 折叠的原文 + [重试 LLM]；`state.llmRawResponse` 保留 |
| 阶段 1 返回空数组 `[]` | 同"解析失败"路径，提示"LLM 未生成任何提示词" |
| 阶段 1 返回的 `ref_index` 越界 | 解析阶段直接 `ParseError` 拦下，黄条提示 |
| 阶段 2 任意单图失败 | 该卡片显示"生成失败"+ 错误信息（截断） + [🔄 重试这张] |
| 阶段 2 全部失败 | 状态栏 `0/N`，不弹全局错；用户自己看卡片或重跑 |
| 用户在生成中点[一键生成] | 按钮禁用（`disabled` 由 phase 控制） |
| 用户在生成中改配置 | 配置 onchange 仍写 localStorage；当前批次**不受影响**（`runImageStage` 启动时已 `snapshotImageConfig()`，全批次共享）；只对下一次"一键生成"或"重试这张"生效 |
| 极大商品图（>4MB base64） | 不主动压缩，按原样发；过大被服务端 413 时落到普通错误路径 |
| 用户上传非图片文件 | `<input accept="image/*">` 已限；MIME 误识别就让 FileReader 跑出来，能 readAsDataURL 就当图，不额外校验 |

**localStorage Schema**（`'detail_image_config'` 一个 key，JSON）：
```json
{
  "llmApiKey": "...",
  "imageApiKey": "...",
  "llmModel": "gemini-2.5-pro",
  "imageModel": "gemini-3.1-flash-image-preview",
  "count": "auto" | "1".."10",
  "concurrency": "1" | "2" | "4" | "8",
  "forceSize": "" | "512" | "1K" | "2K" | "4K",
  "forceAspect": "" | "1:1" | "16:9" | "9:16" | "4:3" | "3:4"
}
```
- 不存任何任务/图片数据
- 字段缺失时各自有兜底默认（与现有页面一致）

**"测试"策略**（vanilla 单文件 HTML，无单元测试框架；按现有项目风格走人工冒烟）：

| 场景 | 期望行为 |
|---|---|
| 不填 Key 点生成 | alert + 阻断 |
| 填错 LLM Key（401） | 红条 + 可重试 |
| LLM 返回纯文本"好的，这是…" | 黄条 + 显示原文 + 可重试 |
| LLM 返回 ```json ... ``` 包裹 | 解析成功 |
| LLM 返回 `ref_index: 99` | 黄条提示越界 |
| 上传 1 张图 + 简单介绍 + 数量 3 | 3 张结果，全在首张参考图上变化 |
| 上传 3 张图 + 数量"自动" | LLM 自由发挥 3-8 张，分散到各 ref |
| 故意触发图像 CONTENT_FILTER | 自动重试，状态栏 attempts 计数 |
| 关掉网络再点重试这张 | 该卡片 error，其他不影响 |

## 不在范围内（YAGNI）

为避免过度设计，以下显式排除：
- 中间提示词审核 / 编辑环节（用户选了全自动直通）
- LLM 模型名 = 图像模型名时的复用（明确两个 Key 两个模型，不做合并）
- 历史项目持久化（任务和图都不入库）
- 提示词文案 / 参考图来源在结果卡片上的可见展示（极简 UI）
- 集成进 `生图工作台.html`（独立页）
- 拖拽上传是 nice-to-have，写代码时若简单就加，不强制
- 自动压缩大图、自动检测违规图等"贴心但越界"的功能
