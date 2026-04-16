# 电商详情图一键生成 — 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在项目根目录新增独立 HTML 文件 `电商详情图.html`，实现"上传商品图 + 商品介绍 → LLM 自动规划详情图提示词 → 并发文生图"的全自动直通流水线。

**Architecture:** 单文件 vanilla JS，沿用 `单张.html` / `批量一对一生图.html` 的风格（内联 style + script，无构建工具）。两阶段流水线：阶段 1 调多模态 LLM 一次返回 JSON 数组；阶段 2 用 worker 池并发调文生图 API。仅 localStorage 存配置，任务/图片留内存。

**Tech Stack:** HTML5 / 原生 JS（async/await）/ Fetch API / localStorage / SparkCode OpenAI 兼容 API（`https://sparkcode.top/v1/chat/completions`）

**项目无单元测试框架**：每个任务的"测试"步骤是用浏览器打开文件做指定的人工冒烟检查；最终 Task 12 走完 spec §5 末尾的完整验收表。

---

## 文件结构

| 路径 | 创建/修改 | 职责 |
|---|---|---|
| `电商详情图.html` | **创建** | 整个页面：HTML 骨架、CSS、所有 JS 逻辑 |
| `docs/superpowers/specs/2026-04-16-ecommerce-detail-images-design.md` | 已存在 | 设计文档（参考用） |

整个特性都在一个文件里，沿用项目现有约定。文件最终会有 ~700 行，按职责区分为这些 JS 区块（在 `<script>` 内顺序排列）：

```
<script>
  // 1. Constants & State
  // 2. Helpers ($, sleep, escapeHtml, downloadDataUrl)
  // 3. Config (saveConfig / loadConfig)
  // 4. Reference Images (addRef / removeRef / renderRefList)
  // 5. Status Bar & Error Banner (updateStatusBar / showError / clearError)
  // 6. Stage 1: LLM (SYSTEM_PROMPT, callLLM, parseLLMResponse, ParseError)
  // 7. Stage 2: Image API (callImageAPI, snapshotImageConfig, buildImageRequest)
  // 8. Worker Pool (runImageStage, runOneTask)
  // 9. Result Grid (renderResultCard, renderResultGrid)
  // 10. Generator entry (generate, validateInputs, retryLLM, retryOne)
  // 11. Clear / Download All
  // 12. DOMContentLoaded init wiring
</script>
```

---

### Task 1: HTML 骨架 + CSS + 全局 state

**Files:**
- Create: `电商详情图.html`

**目标：** 一张静态可打开的页面，所有卡片为空壳，按钮存在但点了无反应。建立全局 `state` 与 helpers，后续任务往里填东西。

- [ ] **Step 1: 创建文件 `电商详情图.html`，内容如下**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>电商详情图 · 一键生成</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background: #f0f2f5; color: #1e293b; min-height: 100vh; padding-bottom: 80px;
  }
  .container { max-width: 1200px; margin: 0 auto; padding: 20px; }
  .header {
    display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;
  }
  .header h1 { font-size: 22px; font-weight: 700; }
  .header-actions { display: flex; gap: 10px; }

  .card {
    background: #fff; border-radius: 12px; padding: 20px; margin-bottom: 16px;
    box-shadow: 0 2px 6px rgba(0,0,0,0.06); border: 1px solid #e2e8f0;
  }
  .card-title {
    font-size: 16px; font-weight: 700; margin-bottom: 14px;
    border-bottom: 1px solid #eee; padding-bottom: 8px;
  }

  .form-grid {
    display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 14px;
  }
  .form-group { display: flex; flex-direction: column; gap: 4px; }
  .form-group.span2 { grid-column: span 2; }
  label { font-size: 13px; font-weight: 600; color: #64748b; }
  input, select, textarea {
    padding: 8px 12px; border: 1px solid #cbd5e1; border-radius: 8px;
    font-size: 14px; outline: none; font-family: inherit;
    transition: border-color 0.15s;
  }
  input:focus, select:focus, textarea:focus {
    border-color: #4f8cff; box-shadow: 0 0 0 3px rgba(79,140,255,0.12);
  }
  textarea { resize: vertical; min-height: 100px; }

  .btn {
    padding: 8px 18px; border: none; border-radius: 8px; cursor: pointer;
    font-size: 14px; font-weight: 600; transition: background 0.15s;
  }
  .btn-primary { background: #4f8cff; color: #fff; }
  .btn-primary:hover { background: #3a6fd8; }
  .btn-primary:disabled { background: #94a3b8; cursor: not-allowed; }
  .btn-ghost { background: #f1f5f9; color: #475569; }
  .btn-ghost:hover { background: #e2e8f0; }
  .btn-danger-ghost { background: #fee2e2; color: #b91c1c; }

  .bottom-bar {
    position: fixed; bottom: 0; left: 0; right: 0; background: #fff;
    border-top: 1px solid #e2e8f0; padding: 12px 24px;
    display: flex; justify-content: space-between; align-items: center;
    z-index: 100; box-shadow: 0 -2px 6px rgba(0,0,0,0.04);
  }
  #statusText { font-size: 14px; color: #475569; }

  /* Banner */
  .banner {
    padding: 12px 16px; border-radius: 8px; margin-bottom: 14px;
    display: flex; flex-direction: column; gap: 8px;
  }
  .banner-row { display: flex; justify-content: space-between; align-items: center; gap: 12px; }
  .banner-red { background: #fee2e2; color: #991b1b; border: 1px solid #fca5a5; }
  .banner-yellow { background: #fef3c7; color: #92400e; border: 1px solid #fcd34d; }
  .banner pre {
    margin-top: 8px; background: rgba(0,0,0,0.04); padding: 10px; border-radius: 6px;
    font-size: 12px; max-height: 200px; overflow: auto; white-space: pre-wrap;
    word-break: break-all; font-family: ui-monospace, monospace;
  }
  .banner-btn {
    padding: 4px 12px; font-size: 12px; background: #fff; border: 1px solid currentColor;
    border-radius: 6px; cursor: pointer; color: inherit; font-weight: 600;
  }

  /* Reference image thumbs */
  .ref-list { display: flex; flex-wrap: wrap; gap: 10px; margin-top: 10px; }
  .ref-thumb {
    position: relative; width: 90px; height: 90px; border-radius: 8px; overflow: hidden;
    border: 2px solid #cbd5e1;
  }
  .ref-thumb img { width: 100%; height: 100%; object-fit: cover; }
  .ref-thumb .label {
    position: absolute; bottom: 0; left: 0; right: 0; background: rgba(0,0,0,0.6);
    color: #fff; font-size: 11px; text-align: center; padding: 1px 0;
  }
  .ref-thumb .remove {
    position: absolute; top: 2px; right: 2px; width: 20px; height: 20px;
    background: rgba(0,0,0,0.55); color: #fff; border: none; border-radius: 50%;
    font-size: 14px; line-height: 18px; cursor: pointer;
  }
  .upload-btn {
    width: 90px; height: 90px; border: 2px dashed #94a3b8; border-radius: 8px;
    background: #f8fafc; color: #475569; font-size: 13px; cursor: pointer;
    display: flex; align-items: center; justify-content: center;
  }
  .upload-btn:hover { border-color: #4f8cff; color: #4f8cff; }

  /* Result grid */
  .result-grid {
    display: grid; grid-template-columns: repeat(auto-fill, minmax(180px, 1fr)); gap: 14px;
  }
  .result-card {
    background: #f8fafc; border: 1px solid #e2e8f0; border-radius: 8px;
    overflow: hidden; display: flex; flex-direction: column;
  }
  .result-card .media {
    aspect-ratio: 1; background: #e2e8f0; display: flex; align-items: center;
    justify-content: center; color: #94a3b8; font-size: 13px; position: relative;
  }
  .result-card .media img { width: 100%; height: 100%; object-fit: cover; cursor: zoom-in; }
  .result-card .media .err-text {
    padding: 8px; font-size: 11px; color: #b91c1c; word-break: break-all;
  }
  .result-card .actions { display: flex; gap: 6px; padding: 8px; }
  .result-card .actions .btn { flex: 1; padding: 5px 8px; font-size: 12px; }
  .spinner {
    width: 24px; height: 24px; border: 3px solid #cbd5e1;
    border-top-color: #4f8cff; border-radius: 50%; animation: spin 0.8s linear infinite;
  }
  @keyframes spin { to { transform: rotate(360deg); } }

  /* Modal for zoom */
  .modal {
    display: none; position: fixed; inset: 0; background: rgba(0,0,0,0.85);
    z-index: 200; cursor: zoom-out; padding: 40px;
  }
  .modal.show { display: flex; align-items: center; justify-content: center; }
  .modal img { max-width: 100%; max-height: 100%; border-radius: 4px; }
</style>
</head>
<body>
<div class="container">
  <div class="header">
    <h1>🛍️ 电商详情图 · 一键生成</h1>
    <div class="header-actions">
      <button class="btn btn-ghost" id="clearBtn">🗑️ 清空</button>
      <button class="btn btn-ghost" id="downloadAllBtn">📦 批量下载</button>
    </div>
  </div>

  <div id="errorBanner"></div>

  <div class="card" id="configCard">
    <div class="card-title">⚙️ 配置</div>
    <!-- Task 2 fills here -->
  </div>

  <div class="card" id="refCard">
    <div class="card-title">🖼️ 商品图（可上传多张同一商品的不同角度）</div>
    <!-- Task 3 fills here -->
  </div>

  <div class="card" id="descCard">
    <div class="card-title">📝 商品介绍</div>
    <!-- Task 4 fills here -->
  </div>

  <div class="card" id="resultCard">
    <div class="card-title">🎨 生成结果</div>
    <div class="result-grid" id="resultGrid"></div>
  </div>
</div>

<div class="bottom-bar">
  <div id="statusText">就绪</div>
  <button class="btn btn-primary" id="generateBtn">🚀 一键生成详情图</button>
</div>

<div class="modal" id="zoomModal"><img id="zoomImg" src=""></div>

<script>
  // ===== Constants =====
  const API_URL = 'https://sparkcode.top/v1/chat/completions';
  const STORAGE_KEY = 'detail_image_config';
  const MAX_TOKENS = 4096;
  const MAX_RETRIES = 5;

  // ===== State =====
  const state = {
    refImages: [],            // data URL strings
    productDescription: '',
    phase: 'idle',            // 'idle' | 'llm' | 'generating' | 'done' | 'llm_failed'
    llmRawResponse: null,
    tasks: []                 // see Task 6 for shape
  };

  // ===== Helpers =====
  const $ = id => document.getElementById(id);
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  function escapeHtml(str) {
    const div = document.createElement('div');
    div.textContent = String(str);
    return div.innerHTML;
  }
  function downloadDataUrl(url, name) {
    const a = document.createElement('a');
    a.href = url;
    a.download = name;
    document.body.appendChild(a);
    a.click();
    a.remove();
  }
  function zoomImg(src) {
    $('zoomImg').src = src;
    $('zoomModal').classList.add('show');
  }
  $('zoomModal').addEventListener('click', () => $('zoomModal').classList.remove('show'));

  // ===== Init (filled by later tasks) =====
  window.addEventListener('DOMContentLoaded', () => {
    // Tasks 2+ register their wiring here
  });
</script>
</body>
</html>
```

- [ ] **Step 2: 在浏览器打开 `电商详情图.html`**

期望：能看到标题"🛍️ 电商详情图 · 一键生成"，右上角两个按钮，4 个空白卡片标题（配置/商品图/商品介绍/生成结果），底部状态栏写"就绪"，右侧有"🚀 一键生成详情图"按钮。点按钮无反应（正常）。F12 控制台无 JS 报错。

- [ ] **Step 3: 提交**

```bash
cd /e/python_code/picture
git add 电商详情图.html
git commit -m "feat(detail): 页面骨架与全局 state"
```

---

### Task 2: 配置面板 + localStorage 持久化

**Files:**
- Modify: `电商详情图.html`（在 `<!-- Task 2 fills here -->` 处填 HTML；在 `<script>` 顶部 helpers 之后添加 saveConfig/loadConfig）

**目标：** 配置卡片渲染所有字段；onchange 写 localStorage；DOMContentLoaded 时读回填充。

- [ ] **Step 1: 替换配置卡内的 placeholder**

把配置卡里 `<!-- Task 2 fills here -->` 那一行替换为：

```html
<div class="form-grid">
  <div class="form-group span2">
    <label>LLM API Key（用于生成提示词）</label>
    <input id="llmApiKey" type="password" placeholder="sk-...">
  </div>
  <div class="form-group span2">
    <label>图像 API Key（用于文生图）</label>
    <input id="imageApiKey" type="password" placeholder="sk-...">
  </div>
  <div class="form-group">
    <label>LLM 模型</label>
    <input id="llmModel" placeholder="gemini-2.5-pro">
  </div>
  <div class="form-group">
    <label>图像模型</label>
    <input id="imageModel" value="gemini-3.1-flash-image-preview">
  </div>
  <div class="form-group">
    <label>数量</label>
    <select id="count">
      <option value="auto">自动（3-8 张）</option>
      <option value="1">1</option>
      <option value="2">2</option>
      <option value="3">3</option>
      <option value="4">4</option>
      <option value="5">5</option>
      <option value="6">6</option>
      <option value="7">7</option>
      <option value="8">8</option>
      <option value="9">9</option>
      <option value="10">10</option>
    </select>
  </div>
  <div class="form-group">
    <label>并发</label>
    <select id="concurrency">
      <option value="1">1</option>
      <option value="2">2</option>
      <option value="4" selected>4</option>
      <option value="8">8</option>
    </select>
  </div>
  <div class="form-group">
    <label>强制尺寸（留空 = LLM 决定）</label>
    <select id="forceSize">
      <option value="">— 留空 —</option>
      <option value="512">512</option>
      <option value="1K">1K</option>
      <option value="2K">2K</option>
      <option value="4K">4K</option>
    </select>
  </div>
  <div class="form-group">
    <label>强制比例（留空 = LLM 决定）</label>
    <select id="forceAspect">
      <option value="">— 留空 —</option>
      <option value="1:1">1:1</option>
      <option value="16:9">16:9</option>
      <option value="9:16">9:16</option>
      <option value="4:3">4:3</option>
      <option value="3:4">3:4</option>
    </select>
  </div>
</div>
```

- [ ] **Step 2: 在 `<script>` 内 `// ===== Init` 这行**之前**插入 Config 区块**

```js
  // ===== Config =====
  const CONFIG_FIELDS = [
    'llmApiKey', 'imageApiKey', 'llmModel', 'imageModel',
    'count', 'concurrency', 'forceSize', 'forceAspect'
  ];

  function saveConfig() {
    const cfg = {};
    CONFIG_FIELDS.forEach(id => { cfg[id] = $(id).value; });
    localStorage.setItem(STORAGE_KEY, JSON.stringify(cfg));
  }

  function loadConfig() {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return;
    try {
      const cfg = JSON.parse(raw);
      CONFIG_FIELDS.forEach(id => {
        if (cfg[id] !== undefined && cfg[id] !== '') $(id).value = cfg[id];
      });
    } catch (e) {
      console.warn('localStorage config parse failed:', e);
    }
  }

  function bindConfigPersistence() {
    CONFIG_FIELDS.forEach(id => {
      $(id).addEventListener('change', saveConfig);
      $(id).addEventListener('input', saveConfig);
    });
  }
```

- [ ] **Step 3: 在 DOMContentLoaded 回调里加初始化调用**

把 `// Tasks 2+ register their wiring here` 那行下面紧接着加：

```js
    loadConfig();
    bindConfigPersistence();
```

- [ ] **Step 4: 浏览器冒烟测试**

打开页面 → 配置卡能看到 8 个字段（两个 Key、两个模型、数量/并发/强制尺寸/强制比例）。在 LLM API Key 输入 `test-key-123`，刷新页面，期望该值仍在。F12 控制台 `localStorage.getItem('detail_image_config')` 能看到 JSON。

- [ ] **Step 5: 提交**

```bash
git add 电商详情图.html
git commit -m "feat(detail): 配置面板 + localStorage 持久化"
```

---

### Task 3: 商品图上传 + 缩略图列表

**Files:**
- Modify: `电商详情图.html`

**目标：** 用户能上传多张图，每张生成 90×90 缩略图带"#1"标号和删除按钮；底层维护 `state.refImages`。

- [ ] **Step 1: 替换商品图卡里的 placeholder**

```html
<div class="ref-list" id="refList">
  <button class="upload-btn" id="uploadBtn">+ 上传</button>
  <input type="file" id="refFileInput" accept="image/*" multiple hidden>
</div>
```

- [ ] **Step 2: 在 `<script>` 内 Config 区块下方插入 Reference Images 区块**

```js
  // ===== Reference Images =====
  function readFileAsDataURL(file) {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = () => resolve(reader.result);
      reader.onerror = reject;
      reader.readAsDataURL(file);
    });
  }

  async function handleRefUpload(event) {
    const files = Array.from(event.target.files);
    for (const f of files) {
      try {
        const url = await readFileAsDataURL(f);
        state.refImages.push(url);
      } catch (e) {
        console.warn('read file failed:', e);
      }
    }
    event.target.value = '';
    renderRefList();
  }

  function removeRef(idx) {
    state.refImages.splice(idx, 1);
    renderRefList();
  }

  function renderRefList() {
    const list = $('refList');
    // remove existing thumbs but keep upload button & file input
    list.querySelectorAll('.ref-thumb').forEach(el => el.remove());
    const uploadBtn = $('uploadBtn');
    state.refImages.forEach((url, i) => {
      const div = document.createElement('div');
      div.className = 'ref-thumb';
      div.innerHTML = `
        <img src="${url}" onclick="zoomImg('${url.replace(/'/g, "\\'")}')">
        <div class="label">#${i + 1}</div>
        <button class="remove" data-idx="${i}">&times;</button>
      `;
      div.querySelector('.remove').addEventListener('click', () => removeRef(i));
      list.insertBefore(div, uploadBtn);
    });
  }

  function bindRefUploader() {
    $('uploadBtn').addEventListener('click', () => $('refFileInput').click());
    $('refFileInput').addEventListener('change', handleRefUpload);
  }
```

注意：`zoomImg` 在 Task 1 里已定义。`onclick` 中用模板嵌 url 不安全（data url 内可能含单引号？通常是 base64 不会，但稳妥起见做 escape）；这里用 `data-` 改造也可以，但项目其他页面就是这样写的，保持一致。

- [ ] **Step 3: DOMContentLoaded 里追加 `bindRefUploader();`**

更新 init 块为：

```js
  window.addEventListener('DOMContentLoaded', () => {
    loadConfig();
    bindConfigPersistence();
    bindRefUploader();
  });
```

- [ ] **Step 4: 浏览器冒烟测试**

打开页面 → 商品图卡里看到 90×90 虚线"+ 上传"按钮 → 点击选 2-3 张图 → 看到带"#1/#2/#3"标号的缩略图，每张右上角有 × 按钮 → 点 × 那张消失，编号重排 → 点缩略图，全屏遮罩弹出大图，再点遮罩关闭。

- [ ] **Step 5: 提交**

```bash
git add 电商详情图.html
git commit -m "feat(detail): 商品图上传与缩略图列表"
```

---

### Task 4: 商品介绍 textarea + 清空 + 批量下载按钮

**Files:**
- Modify: `电商详情图.html`

**目标：** textarea 实时同步到 `state.productDescription`；清空按钮重置 state（不动配置）；批量下载按钮下载所有 `done` 任务（暂无任务，按钮按了不报错即可）。

- [ ] **Step 1: 替换商品介绍卡里的 placeholder**

```html
<textarea id="productDescription" rows="6" placeholder="例如：这是一款专为户外露营设计的便携式折叠桌，铝合金材质，承重 30kg，10 秒展开，附带收纳袋……"></textarea>
```

- [ ] **Step 2: 在 `<script>` 里 Reference Images 区块下方插入 Description / Clear / DownloadAll 区块**

```js
  // ===== Description =====
  function bindDescription() {
    $('productDescription').addEventListener('input', e => {
      state.productDescription = e.target.value;
    });
  }

  // ===== Clear =====
  function clearAll() {
    if (!confirm('确认清空所有上传图、商品介绍和生成结果？（配置不会清）')) return;
    state.refImages = [];
    state.productDescription = '';
    state.tasks = [];
    state.llmRawResponse = null;
    state.phase = 'idle';
    $('productDescription').value = '';
    renderRefList();
    renderResultGrid();    // defined in Task 9
    clearError();          // defined in Task 5
    updateStatusBar();     // defined in Task 5
  }

  // ===== Download All =====
  function downloadAll() {
    const done = state.tasks.filter(t => t.status === 'done' && t.result);
    if (done.length === 0) {
      alert('没有可下载的图片');
      return;
    }
    done.forEach((t, i) => {
      setTimeout(() => downloadDataUrl(t.result, `detail_${i + 1}.png`), i * 300);
    });
  }
```

注意：`renderResultGrid` / `clearError` / `updateStatusBar` 在后续任务定义；本任务只调用，不会因为它们还不存在就报错——因为 `clearAll` 只在用户点击时执行，本任务的冒烟里不点清空。

- [ ] **Step 3: DOMContentLoaded 里追加按钮绑定**

```js
  window.addEventListener('DOMContentLoaded', () => {
    loadConfig();
    bindConfigPersistence();
    bindRefUploader();
    bindDescription();
    $('clearBtn').addEventListener('click', clearAll);
    $('downloadAllBtn').addEventListener('click', downloadAll);
  });
```

- [ ] **Step 4: 浏览器冒烟测试**

打开页面 → 商品介绍 textarea 可输入；F12 控制台 `state.productDescription` 与输入同步 → 点"📦 批量下载"，弹出"没有可下载的图片"。**先不要点"🗑️ 清空"**（其依赖的 render/update 函数还没写）。

- [ ] **Step 5: 提交**

```bash
git add 电商详情图.html
git commit -m "feat(detail): 商品介绍输入与清空/下载入口"
```

---

### Task 5: 状态栏 + ErrorBanner

**Files:**
- Modify: `电商详情图.html`

**目标：** 实现 `updateStatusBar()` 与 `showError()` / `clearError()`，控制底部状态文本与顶部错误条。

- [ ] **Step 1: 在 `<script>` 里 Description 区块下方插入 Status & Error 区块**

```js
  // ===== Status Bar =====
  const PHASE_LABEL = {
    idle: '就绪',
    llm: 'LLM 生成提示词中...',
    generating: '并发生图中',
    done: '完成',
    llm_failed: 'LLM 阶段失败'
  };

  function updateStatusBar() {
    const total = state.tasks.length;
    const ok = state.tasks.filter(t => t.status === 'done').length;
    const err = state.tasks.filter(t => t.status === 'error').length;
    let text = PHASE_LABEL[state.phase] || state.phase;
    if (total > 0) text += ` ｜ 成功 ${ok} / 失败 ${err} / 总数 ${total}`;
    $('statusText').textContent = text;

    const busy = state.phase === 'llm' || state.phase === 'generating';
    $('generateBtn').disabled = busy;
  }

  // ===== Error Banner =====
  function showError(level, message, rawText) {
    // level: 'red' (network/HTTP) | 'yellow' (parse)
    const cls = level === 'yellow' ? 'banner-yellow' : 'banner-red';
    const rawBlock = rawText
      ? `<details><summary style="cursor:pointer;font-size:13px;">展开 LLM 原文</summary><pre>${escapeHtml(rawText)}</pre></details>`
      : '';
    $('errorBanner').innerHTML = `
      <div class="banner ${cls}">
        <div class="banner-row">
          <div>${escapeHtml(message)}</div>
          <button class="banner-btn" id="retryLlmBtn">🔁 重试 LLM</button>
        </div>
        ${rawBlock}
      </div>
    `;
    $('retryLlmBtn').addEventListener('click', retryLLM);   // defined in Task 10
  }

  function clearError() {
    $('errorBanner').innerHTML = '';
  }
```

- [ ] **Step 2: DOMContentLoaded 里追加状态栏初始化**

```js
  window.addEventListener('DOMContentLoaded', () => {
    loadConfig();
    bindConfigPersistence();
    bindRefUploader();
    bindDescription();
    $('clearBtn').addEventListener('click', clearAll);
    $('downloadAllBtn').addEventListener('click', downloadAll);
    updateStatusBar();
  });
```

- [ ] **Step 3: 浏览器冒烟测试**

打开页面 → 底部状态栏显示"就绪" → F12 控制台跑：
```js
showError('red', '测试网络错: connection refused');
```
顶部出现红条 + "🔁 重试 LLM" 按钮（点了会报 `retryLLM is not defined`，正常，下面任务会写）。再跑 `showError('yellow', '解析失败', '这是 LLM 原始输出文本'); ` 出现黄条 + 折叠"展开 LLM 原文"。再跑 `clearError();` 消失。

- [ ] **Step 4: 提交**

```bash
git add 电商详情图.html
git commit -m "feat(detail): 状态栏与错误条组件"
```

---

### Task 6: 阶段 1 — LLM 调用 + JSON 三层兜底解析

**Files:**
- Modify: `电商详情图.html`

**目标：** 实现 `callLLM()`（HTTP 请求）+ `parseLLMResponse()`（三层兜底 + 严格校验）+ 写死的 `SYSTEM_PROMPT`。

- [ ] **Step 1: 在 `<script>` 里 Status / Error 区块下方插入 Stage 1 区块**

```js
  // ===== Stage 1: LLM =====
  const SYSTEM_PROMPT =
`你是电商详情图设计专家。基于用户提供的商品图与商品介绍，规划 N 张电商详情图（主图 / 卖点图 / 使用场景图 / 细节特写 / 规格说明 / 对比图 等中合理选择若干），为每张图生成一条独立、可直接送入文生图模型执行的中文提示词。

输出**只能**是一个 JSON 数组，不要包裹 markdown 代码块、不要任何额外文字。每个元素结构：
{ "ref_index": <从 1 开始的整数，指向用户上传的第几张商品图>,
  "prompt": "<完整的、信息自包含的中文文生图提示词，描述构图、风格、光影、文字、布局>",
  "size": "<可选: 512 | 1K | 2K | 4K>",
  "aspect_ratio": "<可选: 1:1 | 16:9 | 9:16 | 4:3 | 3:4>" }

要求：
- ref_index 必须在用户上传的图片范围内
- 当用户在介绍中明确指定了尺寸/比例需求时，请填入 size / aspect_ratio
- 未要求时省略这两个字段
- 提示词需自包含商品的关键视觉信息，不要假设模型已经"看过"参考图
- 当用户指定了具体张数时严格按数量返回；未指定时返回 3-8 条之间的合理数量`;

  class ParseError extends Error {
    constructor(message, raw) { super(message); this.raw = raw; }
  }

  function buildLLMRequest(apiKey, model, count) {
    const userText =
      `商品介绍：\n${state.productDescription}\n\n` +
      (count === 'auto'
        ? '请生成 3-8 条之间合理数量的文生图提示词。'
        : `请生成正好 ${count} 条文生图提示词。`);

    const content = state.refImages.map(url => ({
      type: 'image_url', image_url: { url }
    }));
    content.push({ type: 'text', text: userText });

    return {
      model,
      messages: [
        { role: 'system', content: SYSTEM_PROMPT },
        { role: 'user', content }
      ],
      max_tokens: MAX_TOKENS
    };
  }

  async function callLLM(apiKey, model, count) {
    const body = buildLLMRequest(apiKey, model, count);
    const resp = await fetch(API_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer ' + apiKey
      },
      body: JSON.stringify(body)
    });
    if (!resp.ok) {
      const text = await resp.text();
      throw new Error(`HTTP ${resp.status}: ${text.substring(0, 300)}`);
    }
    const data = await resp.json();
    if (data.error) throw new Error(data.error.message || JSON.stringify(data.error));
    return data;
  }

  function parseLLMResponse(data) {
    const content = data.choices?.[0]?.message?.content;
    if (!content || typeof content !== 'string') {
      throw new ParseError('LLM 未返回文本内容', JSON.stringify(data).substring(0, 500));
    }

    let arr;
    // 尝试 1: 整体当 JSON
    try { arr = JSON.parse(content); }
    catch {
      // 尝试 2: 抓 ```json ... ``` 包裹
      const fenced = content.match(/```(?:json)?\s*([\s\S]+?)```/);
      if (fenced) {
        try { arr = JSON.parse(fenced[1]); } catch {}
      }
    }
    // 尝试 3: 抓首个 [ ... ]
    if (!arr) {
      const m = content.match(/\[[\s\S]*\]/);
      if (m) {
        try { arr = JSON.parse(m[0]); } catch {}
      }
    }

    if (!Array.isArray(arr)) {
      throw new ParseError('未能解析为 JSON 数组', content);
    }
    if (arr.length === 0) {
      throw new ParseError('LLM 返回了空数组（未生成任何提示词）', content);
    }

    return arr.map((item, i) => {
      if (typeof item.ref_index !== 'number'
          || item.ref_index < 1
          || item.ref_index > state.refImages.length) {
        throw new ParseError(
          `第 ${i + 1} 项 ref_index 越界: ${item.ref_index}（共上传 ${state.refImages.length} 张）`,
          content
        );
      }
      if (typeof item.prompt !== 'string' || !item.prompt.trim()) {
        throw new ParseError(`第 ${i + 1} 项 prompt 为空`, content);
      }
      return {
        refIndex: item.ref_index,
        prompt: item.prompt.trim(),
        size: typeof item.size === 'string' ? item.size : null,
        aspect: typeof item.aspect_ratio === 'string' ? item.aspect_ratio : null
      };
    });
  }
```

- [ ] **Step 2: 浏览器冒烟（解析器单元测试）**

打开页面 → F12 控制台逐条跑（不打 LLM，只测解析器）：

```js
// 准备假的 refImages 让 ref_index 校验有上下文
state.refImages = ['fake1', 'fake2'];

// 1. 整体 JSON
parseLLMResponse({choices:[{message:{content:'[{"ref_index":1,"prompt":"白底主图"}]'}}]});
// 期望: 返回 [{refIndex:1, prompt:"白底主图", size:null, aspect:null}]

// 2. ```json 包裹
parseLLMResponse({choices:[{message:{content:'```json\n[{"ref_index":2,"prompt":"场景图"}]\n```'}}]});
// 期望: 返回长度 1，refIndex=2

// 3. 文字+数组
parseLLMResponse({choices:[{message:{content:'好的，这是结果：[{"ref_index":1,"prompt":"x"}] 完毕'}}]});
// 期望: 返回长度 1

// 4. ref_index 越界 → ParseError
try { parseLLMResponse({choices:[{message:{content:'[{"ref_index":99,"prompt":"x"}]'}}]}); }
catch (e) { console.log('caught:', e.message, '| raw:', e.raw); }
// 期望: caught: 第 1 项 ref_index 越界: 99（...）

// 5. 空数组 → ParseError
try { parseLLMResponse({choices:[{message:{content:'[]'}}]}); }
catch (e) { console.log('caught:', e.message); }
// 期望: caught: LLM 返回了空数组...

// 6. 完全非 JSON → ParseError
try { parseLLMResponse({choices:[{message:{content:'今天天气不错'}}]}); }
catch (e) { console.log('caught:', e.message); }
// 期望: caught: 未能解析为 JSON 数组
```

全部按期望表现即通过。

- [ ] **Step 3: 提交**

```bash
git add 电商详情图.html
git commit -m "feat(detail): 阶段1 LLM 调用与三层兜底解析"
```

---

### Task 7: 阶段 2 — 单图调用 + 配置快照 + 响应解析

**Files:**
- Modify: `电商详情图.html`

**目标：** 实现 `snapshotImageConfig()` / `buildImageRequest(task, snap)` / `callImageAPI(task, snap)`。响应解析复用批量页 5 路径兜底。

- [ ] **Step 1: 在 `<script>` 里 Stage 1 区块下方插入 Stage 2 单调用区块**

```js
  // ===== Stage 2: Image API (single call) =====
  function snapshotImageConfig() {
    return {
      apiKey:   $('imageApiKey').value.trim(),
      model:    $('imageModel').value.trim(),
      uiSize:   $('forceSize').value,
      uiAspect: $('forceAspect').value
    };
  }

  function buildImageRequest(task, snap) {
    const size   = snap.uiSize   || task.size   || null;
    const aspect = snap.uiAspect || task.aspect || null;
    const refUrl = state.refImages[task.refIndex - 1];

    const body = {
      model: snap.model,
      messages: [{
        role: 'user',
        content: [
          { type: 'image_url', image_url: { url: refUrl } },
          { type: 'text', text: task.prompt }
        ]
      }],
      max_tokens: MAX_TOKENS
    };

    const cfg = {};
    if (size)   cfg.image_size = size;
    if (aspect) cfg.aspect_ratio = aspect;
    if (Object.keys(cfg).length) {
      body.extra_body = { google: { image_config: cfg } };
    }
    return body;
  }

  function extractImageFromResponse(data) {
    const message = data.choices?.[0]?.message;
    if (!message) throw new Error('API 响应中没有 message 字段');

    const finishReason = data.choices?.[0]?.finish_reason;
    if (finishReason === 'content_filter') {
      throw new Error('CONTENT_FILTER: 被内容安全过滤器拦截，将自动重试');
    }

    const content = message.content;

    // 路径 1: content 是数组 (多模态 part 列表)
    if (Array.isArray(content)) {
      for (const part of content) {
        if (part.type === 'image_url' && part.image_url?.url) return part.image_url.url;
        if (part.inline_data) {
          return `data:${part.inline_data.mime_type || 'image/png'};base64,${part.inline_data.data}`;
        }
        if (typeof part.text === 'string') {
          const m = part.text.match(/data:image\/[a-zA-Z+]+;base64,[A-Za-z0-9+/=]+/);
          if (m) return m[0];
        }
        if (typeof part === 'string') {
          const m = part.match(/data:image\/[a-zA-Z+]+;base64,[A-Za-z0-9+/=]+/);
          if (m) return m[0];
        }
      }
    }

    // 路径 2: content 是字符串，里面嵌 data url
    if (typeof content === 'string' && content.length > 0) {
      const m = content.match(/data:image\/[a-zA-Z+]+;base64,[A-Za-z0-9+/=]+/);
      if (m) return m[0];
    }

    // 路径 3: message.parts[].inline_data
    const parts = message.parts || [];
    for (const p of parts) {
      if (p.inline_data) {
        return `data:${p.inline_data.mime_type || 'image/png'};base64,${p.inline_data.data}`;
      }
    }

    // 路径 4 / 5: data[0].b64_json or data[0].url
    if (data.data?.[0]?.b64_json) return `data:image/png;base64,${data.data[0].b64_json}`;
    if (data.data?.[0]?.url)      return data.data[0].url;

    const preview = typeof content === 'string'
      ? content.substring(0, 200)
      : JSON.stringify(content)?.substring(0, 200);
    throw new Error(`解析失败(content type=${typeof content}). 前 200 字: ${preview}`);
  }

  async function callImageAPI(task, snap) {
    const body = buildImageRequest(task, snap);
    const resp = await fetch(API_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer ' + snap.apiKey
      },
      body: JSON.stringify(body)
    });
    if (!resp.ok) {
      const text = await resp.text();
      throw new Error(`HTTP ${resp.status}: ${text.substring(0, 200)}`);
    }
    const data = await resp.json();
    if (data.error) throw new Error(data.error.message || JSON.stringify(data.error));
    return extractImageFromResponse(data);
  }
```

- [ ] **Step 2: 浏览器冒烟（不联网，校验函数存在与签名）**

F12 控制台跑：

```js
typeof callImageAPI;          // 'function'
typeof snapshotImageConfig;   // 'function'

// 校验快照读 DOM
$('imageApiKey').value = 'sk-test';
$('imageModel').value = 'gemini-3.1-flash-image-preview';
$('forceSize').value = '1K';
snapshotImageConfig();
// 期望: {apiKey:'sk-test', model:'gemini-3.1-flash-image-preview', uiSize:'1K', uiAspect:''}

// 校验请求体生成（UI 强制覆盖 LLM）
state.refImages = ['data:image/png;base64,xxx'];
buildImageRequest(
  { refIndex: 1, prompt: 'p', size: '512', aspect: '16:9' },
  { apiKey: 'k', model: 'm', uiSize: '1K', uiAspect: '' }
);
// 期望: extra_body.google.image_config = {image_size:'1K', aspect_ratio:'16:9'}
//       —— 注意 size 被 UI 覆盖成 1K，aspect UI 空所以走 LLM 的 16:9
```

- [ ] **Step 3: 提交**

```bash
git add 电商详情图.html
git commit -m "feat(detail): 阶段2 单图调用与5路径响应解析"
```

---

### Task 8: Worker 池 + CONTENT_FILTER 自动重试

**Files:**
- Modify: `电商详情图.html`

**目标：** 实现 `runImageStage()` 与 `runOneTask(task, snap, maxRetries)`，并发跑所有 wait 任务，CONTENT_FILTER 自动重试 5 次。

- [ ] **Step 1: 在 `<script>` 里 Stage 2 单调用区块下方插入 Worker Pool 区块**

```js
  // ===== Worker Pool =====
  async function runOneTask(task, snap, maxRetries) {
    task.status = 'running';
    task.error = null;
    renderResultCard(task);                     // defined in Task 9
    updateStatusBar();

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      task.attempts = attempt;
      try {
        const url = await callImageAPI(task, snap);
        task.result = url;
        task.status = 'done';
        task.error = null;
        break;
      } catch (e) {
        const retryable = e.message.includes('CONTENT_FILTER');
        console.warn(`[Task ${task.id}] attempt ${attempt} failed:`, e.message);
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

  async function runImageStage() {
    const concurrency = parseInt($('concurrency').value, 10) || 4;
    const snap = snapshotImageConfig();
    const queue = state.tasks.filter(t => t.status === 'wait');

    const workers = Array(Math.min(concurrency, queue.length))
      .fill(0)
      .map(async () => {
        while (queue.length) {
          const task = queue.shift();
          if (!task) continue;
          await runOneTask(task, snap, MAX_RETRIES);
        }
      });
    await Promise.all(workers);
  }
```

- [ ] **Step 2: 浏览器冒烟（仅校验函数存在与不抛异常）**

F12：
```js
typeof runImageStage;     // 'function'
typeof runOneTask;        // 'function'
// 跑空队列不应该报错
state.tasks = [];
await runImageStage();    // 立即返回 undefined
```

注意 `runOneTask` 调用了 `renderResultCard`，下个任务才定义。本任务不实际跑非空队列，避免 ReferenceError。

- [ ] **Step 3: 提交**

```bash
git add 电商详情图.html
git commit -m "feat(detail): worker 池与 CONTENT_FILTER 自动重试"
```

---

### Task 9: 结果网格 + 单卡片渲染 + 单张重试

**Files:**
- Modify: `电商详情图.html`

**目标：** 实现 `renderResultCard(task)`、`renderResultGrid()`、`retryOne(taskId)`。卡片按 task.status 显示等待/运行/成功/失败四种态。

- [ ] **Step 1: 在 `<script>` 里 Worker Pool 区块下方插入 Result Grid 区块**

```js
  // ===== Result Grid =====
  function renderResultGrid() {
    const grid = $('resultGrid');
    grid.innerHTML = '';
    state.tasks.forEach(t => {
      const card = document.createElement('div');
      card.className = 'result-card';
      card.id = `task-${t.id}`;
      card.innerHTML = renderCardInner(t);
      grid.appendChild(card);
      bindCardActions(card, t);
    });
  }

  function renderResultCard(task) {
    const card = $(`task-${task.id}`);
    if (!card) {
      // 第一次出现：整体重渲染
      renderResultGrid();
      return;
    }
    card.innerHTML = renderCardInner(task);
    bindCardActions(card, task);
  }

  function renderCardInner(t) {
    let media = '';
    if (t.status === 'wait') {
      media = `<div class="media">⏳ 等待</div>`;
    } else if (t.status === 'running') {
      media = `<div class="media"><div class="spinner"></div></div>`;
    } else if (t.status === 'done' && t.result) {
      media = `<div class="media"><img src="${t.result}" data-zoom></div>`;
    } else if (t.status === 'error') {
      media = `<div class="media"><div class="err-text">❌ ${escapeHtml(t.error || '生成失败')}</div></div>`;
    } else {
      media = `<div class="media">—</div>`;
    }

    let actions = '';
    if (t.status === 'done') {
      actions = `<button class="btn btn-primary" data-act="download">💾 下载</button>`;
    } else if (t.status === 'error') {
      actions = `<button class="btn btn-ghost" data-act="retry">🔄 重试这张</button>`;
    } else {
      actions = `<button class="btn btn-ghost" disabled>···</button>`;
    }

    return media + `<div class="actions">${actions}</div>`;
  }

  function bindCardActions(card, t) {
    const img = card.querySelector('img[data-zoom]');
    if (img) img.addEventListener('click', () => zoomImg(img.src));
    const dl = card.querySelector('[data-act="download"]');
    if (dl) dl.addEventListener('click', () => downloadDataUrl(t.result, `detail_${t.id}.png`));
    const rt = card.querySelector('[data-act="retry"]');
    if (rt) rt.addEventListener('click', () => retryOne(t.id));
  }

  async function retryOne(taskId) {
    const task = state.tasks.find(t => t.id === taskId);
    if (!task) return;
    task.status = 'wait';
    task.error = null;
    task.attempts = 0;
    await runOneTask(task, snapshotImageConfig(), MAX_RETRIES);
  }
```

- [ ] **Step 2: 浏览器冒烟（用假 task 验证渲染）**

F12 控制台：
```js
state.tasks = [
  { id: 1, refIndex: 1, prompt: 'p1', size: null, aspect: null, status: 'wait',    result: null, error: null, attempts: 0 },
  { id: 2, refIndex: 1, prompt: 'p2', size: null, aspect: null, status: 'running', result: null, error: null, attempts: 1 },
  { id: 3, refIndex: 1, prompt: 'p3', size: null, aspect: null, status: 'done',    result: 'data:image/svg+xml;utf8,<svg xmlns=%22http://www.w3.org/2000/svg%22 width=%22100%22 height=%22100%22><rect width=%22100%22 height=%22100%22 fill=%22%2352c41a%22/></svg>', error: null, attempts: 1 },
  { id: 4, refIndex: 1, prompt: 'p4', size: null, aspect: null, status: 'error',   result: null, error: '5次尝试均失败: CONTENT_FILTER', attempts: 5 }
];
renderResultGrid();
```

期望：4 张卡片排成网格 — `⏳ 等待` / 转圈 / 绿色方块 + 💾 下载 / ❌ 错误 + 🔄 重试这张。点 💾 下载触发浏览器下载；点 🔄 重试这张会调 `retryOne(4)` → `runOneTask` → `callImageAPI` 真实请求（key 没填会立刻 401，卡片错文案变"HTTP 401: ..."）。点绿色方块图片，弹大图。

- [ ] **Step 3: 现在测一下 Task 4 的 clearAll**

F12：`clearAll()` → 弹确认 → 确认 → state.tasks 清空、网格清空、状态栏复位"就绪"。

- [ ] **Step 4: 提交**

```bash
git add 电商详情图.html
git commit -m "feat(detail): 结果网格、卡片渲染与单张重试"
```

---

### Task 10: "一键生成" 入口串联 + 输入校验 + LLM 重试

**Files:**
- Modify: `电商详情图.html`

**目标：** 实现 `generate()`（点按钮入口）和 `retryLLM()`，串联阶段 1 → 阶段 2，做完整输入校验，管理 phase 状态机。

- [ ] **Step 1: 在 `<script>` 里 Result Grid 区块下方插入 Generator 区块**

```js
  // ===== Generator (entry) =====
  function validateInputs() {
    if (!$('llmApiKey').value.trim())   { alert('请填写 LLM API Key');   return false; }
    if (!$('imageApiKey').value.trim()) { alert('请填写 图像 API Key'); return false; }
    if (!$('llmModel').value.trim())    { alert('请填写 LLM 模型');     return false; }
    if (!$('imageModel').value.trim())  { alert('请填写 图像模型');     return false; }
    if (state.refImages.length === 0)   { alert('请至少上传一张商品图'); return false; }
    if (!$('productDescription').value.trim()) { alert('请填写商品介绍'); return false; }
    return true;
  }

  let _taskIdSeq = 0;
  function nextTaskId() { return ++_taskIdSeq; }

  async function runStage1() {
    const llmKey = $('llmApiKey').value.trim();
    const llmModel = $('llmModel').value.trim();
    const count = $('count').value;

    state.phase = 'llm';
    state.llmRawResponse = null;
    clearError();
    updateStatusBar();

    let llmData;
    try {
      llmData = await callLLM(llmKey, llmModel, count);
    } catch (e) {
      state.phase = 'llm_failed';
      updateStatusBar();
      showError('red', `LLM 调用失败：${e.message}`, null);
      return null;
    }

    state.llmRawResponse = llmData.choices?.[0]?.message?.content || JSON.stringify(llmData);

    let parsed;
    try {
      parsed = parseLLMResponse(llmData);
    } catch (e) {
      state.phase = 'llm_failed';
      updateStatusBar();
      const raw = e.raw || state.llmRawResponse;
      showError('yellow', `LLM 返回无法解析：${e.message}`, raw);
      return null;
    }

    return parsed;
  }

  async function generate() {
    if (state.phase === 'llm' || state.phase === 'generating') return;
    if (!validateInputs()) return;

    // 重置任务列表
    state.tasks = [];
    renderResultGrid();

    const parsed = await runStage1();
    if (!parsed) return;

    // 创建任务并先全部渲染为 wait
    state.tasks = parsed.map(p => ({
      id: nextTaskId(),
      refIndex: p.refIndex,
      prompt: p.prompt,
      size: p.size,
      aspect: p.aspect,
      status: 'wait',
      result: null,
      error: null,
      attempts: 0
    }));
    renderResultGrid();

    state.phase = 'generating';
    updateStatusBar();

    await runImageStage();

    state.phase = 'done';
    updateStatusBar();
  }

  async function retryLLM() {
    if (state.phase === 'llm' || state.phase === 'generating') return;
    await generate();   // generate 自身会清空已有 tasks 并重跑全流程
  }
```

- [ ] **Step 2: DOMContentLoaded 里绑定 generate 按钮**

把 init 块改成：

```js
  window.addEventListener('DOMContentLoaded', () => {
    loadConfig();
    bindConfigPersistence();
    bindRefUploader();
    bindDescription();
    $('clearBtn').addEventListener('click', clearAll);
    $('downloadAllBtn').addEventListener('click', downloadAll);
    $('generateBtn').addEventListener('click', generate);
    updateStatusBar();
  });
```

- [ ] **Step 3: 浏览器冒烟（输入校验路径，无需联网）**

打开页面 → 直接点"🚀 一键生成详情图"，弹"请填写 LLM API Key"。
依次填一项点一次，每项依次报错：图像 Key → LLM 模型 → 图像模型 → "请至少上传一张商品图" → "请填写商品介绍"。
全填齐（Key 随便填 `sk-bad`），上传 1 张图，写"测试"两字 → 点生成 → 状态栏先变"LLM 生成提示词中..."→ 失败后顶部红条出"LLM 调用失败：HTTP 401: ..." + "🔁 重试 LLM"按钮 → 状态栏变"LLM 阶段失败"。

- [ ] **Step 4: 提交**

```bash
git add 电商详情图.html
git commit -m "feat(detail): 一键生成入口、输入校验与 LLM 重试"
```

---

### Task 11: 端到端真实接口冒烟（走通快乐路径）

**Files:**
- 仅运行测试，不改代码（除非冒烟暴露 bug）

**目标：** 用真实 LLM Key + 图像 Key 跑一次完整流程，确认两阶段串联工作。

- [ ] **Step 1: 准备**

打开 `电商详情图.html`：
- 填 LLM API Key（用户自己的 sparkcode key）
- 填图像 API Key
- LLM 模型填一个支持视觉的，例如 `gemini-2.5-pro` 或 `gpt-4o`（按项目实际可用的填）
- 图像模型保持默认 `gemini-3.1-flash-image-preview`
- 数量选 `3`（先小批量验证）
- 并发保持 `4`
- 强制尺寸/比例都留空

- [ ] **Step 2: 上传 1 张商品图**

随便找一张实物商品图（手机/水杯/包/玩偶都行）。

- [ ] **Step 3: 商品介绍写一段**

例如：「这是一款户外便携折叠桌，铝合金材质，承重 30kg，10 秒展开。请生成主图、使用场景图、细节特写各一张。」

- [ ] **Step 4: 点"🚀 一键生成详情图"**

期望流程：
1. 状态栏变"LLM 生成提示词中..."
2. 几秒到十几秒后，状态栏变"并发生图中 ｜ 成功 0 / 失败 0 / 总数 3"，结果区出 3 张等待卡片
3. 卡片陆续从转圈变图（或失败）
4. 全部完成后状态栏变"完成 ｜ 成功 X / 失败 Y / 总数 3"

任一异常（一直 LLM 阶段失败、所有图都失败）→ 看 F12 网络面板的请求/响应，定位是 Key/模型不可用还是代码 bug。**bug 才动代码**；配置问题在这里只记录，不修。

- [ ] **Step 5: 验证下载与重试**

- 任意成功卡片点 💾 下载 → 浏览器下载 `detail_<id>.png`
- 点右上 📦 批量下载 → 所有成功的依次下载（每张间隔 ~300ms）
- 任意失败卡片点 🔄 重试这张 → 单卡片回到运行态再出结果

- [ ] **Step 6: 验证 phase 互斥**

生成进行中再点"🚀 一键生成"应被禁用（按钮 disabled 灰色）。

- [ ] **Step 7: 验证刷新行为**

刷新页面 → API Key、模型名、数量、并发等配置仍在；上传的图、商品介绍、结果全部清空（按设计期望）。

- [ ] **Step 8: 如果 Step 4 出现可复现 bug，修后单独 commit；如无 bug，跳过。**

```bash
git add 电商详情图.html
git commit -m "fix(detail): <具体描述>"
```

---

### Task 12: 走完 spec §5 验收表 + 收尾

**Files:**
- 仅冒烟，不改代码

**目标：** 按 spec 的"测试"表逐项过一遍，无遗漏。

- [ ] **Step 1: 不填 Key 点生成**

清空 LLM API Key → 点生成 → 期望 alert "请填写 LLM API Key"。✅

- [ ] **Step 2: 填错 LLM Key（401）**

LLM Key 改成 `sk-bad-xxx`，其余正常 → 生成 → 红条 + "🔁 重试 LLM" 按钮可见 → 把 Key 改回正确的 → 点重试 LLM → 走通。✅

- [ ] **Step 3: LLM 返回纯文本兜底（解析失败路径）**

如果 LLM 模型有时会冒出非 JSON 解释，自然会触发；难以稳定复现时，F12 控制台手动注入：
```js
state.llmRawResponse = '我来帮你生成几条提示词...';
showError('yellow', 'LLM 返回无法解析：未能解析为 JSON 数组', state.llmRawResponse);
```
期望：黄条 + 折叠"展开 LLM 原文"内能看到原文。✅

- [ ] **Step 4: ```json``` 包裹解析（已在 Task 6 单测过，再确认整体跑通即可）**

跑实际 LLM 时若返回 ` ```json ... ``` ` 包裹也能解析成功。✅

- [ ] **Step 5: ref_index 越界**

Task 6 已单测过；跑真实流程时只有 1 张图却让 LLM 用 ref 2 时也会触发 → 黄条 ✅

- [ ] **Step 6: 上传 1 张图 + 数量 3**

期望：3 张结果卡，全部 refIndex=1（同一张参考图）。✅

- [ ] **Step 7: 上传 3 张图 + 数量"自动"**

期望：3-8 张结果卡，refIndex 分散在 1-3 之间。✅

- [ ] **Step 8: CONTENT_FILTER 自动重试**

很难主动触发；查 F12 网络面板，如某次请求返回 `finish_reason: 'content_filter'`，确认控制台有 `[Task X] attempt N failed: CONTENT_FILTER` 日志且 attempts 计数递增。如果整轮没出现，记录"未在本轮观察到，靠日志路径覆盖"即可。✅

- [ ] **Step 9: 关网络再点重试这张**

按 F12 → Network → 切到 Offline → 点失败卡的 🔄 重试这张 → 该卡片 error，其他卡不动 → 切回 Online。✅

- [ ] **Step 10: 验证"中途改配置不影响当前批次"**

跑一次大批量（数量 8、并发 2），生成开始后立刻把图像模型改成乱字符串 → 后续生成的图仍然用原模型成功（因为 `snapshotImageConfig` 已经定型）。✅

- [ ] **Step 11: 最终 commit（如果前面没动代码就跳过；动了就提）**

```bash
git status
# 若有改动：
git add 电商详情图.html
git commit -m "test(detail): §5 验收表全部过完"
```

- [ ] **Step 12: PR/合并由用户决定，不在本计划内**

---

## 自检结果

逐项核对 spec → 计划：

| Spec 要素 | 覆盖任务 |
|---|---|
| §1 两阶段流水线 | Task 6 (阶段1) + Task 7-8 (阶段2) + Task 10 (串联) |
| §2 7 个组件 + ClearButton + DownloadAllButton | Task 1 骨架；Task 2 ConfigPanel；Task 3 RefImageUploader；Task 4 PromptInput + Clear + DownloadAll；Task 5 StatusBar + ErrorBanner；Task 9 ResultGrid；Task 10 Generator |
| §2 state 形状 | Task 1 声明；Task 6/7/10 字段填充 |
| §3 SYSTEM_PROMPT | Task 6 |
| §3 三层兜底解析 | Task 6 + 6.Step 2 单测 |
| §3 ParseError + raw | Task 6（class）+ Task 5/10（展示） |
| §3 phase = 'llm_failed' | Task 10 |
| §4 snapshotImageConfig | Task 7 |
| §4 buildImageRequest 优先级 | Task 7 + 7.Step 2 单测 |
| §4 5 路径响应解析 | Task 7（extractImageFromResponse） |
| §4 worker 池 + MAX_RETRIES + CONTENT_FILTER 重试 | Task 8 |
| §4 retryOne | Task 9 |
| §5 输入校验表 | Task 10 validateInputs + Task 12.Step 1 |
| §5 运行时边界全部 | Task 12 逐项 |
| §5 localStorage schema | Task 2 |
| §5 测试表 | Task 11 + Task 12 |

**类型一致性**：`task.refIndex` / `task.size` / `task.aspect` 在 Task 6（解析输出）、Task 7（buildImageRequest）、Task 9（renderResultCard）、Task 10（Task 创建）一致使用。
`snapshotImageConfig()` 返回 `{apiKey, model, uiSize, uiAspect}` 在 Task 7 与 Task 8 一致。
`renderResultCard / renderResultGrid / clearError / updateStatusBar / retryLLM` 在被调用前都至少在前置任务里被声明（Task 5 之后）；Task 4 的 `clearAll` 调用了未来定义的函数，已在 Task 4 备注中说明且只在 Task 9.Step 3 才点击触发。

**Placeholder 扫描**：无 TBD/TODO；所有代码块完整可粘贴；所有冒烟步骤都给了具体期望表现。

---

## 执行选择

**Plan complete and saved to `docs/superpowers/plans/2026-04-16-ecommerce-detail-images.md`. Two execution options:**

**1. Subagent-Driven (recommended)** — 每个任务派一个新的 subagent，我在任务之间审核，迭代快。

**2. Inline Execution** — 在当前会话里直接执行，按检查点 batch。

**Which approach?**
