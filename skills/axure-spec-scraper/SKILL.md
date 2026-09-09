---
name: axure-spec-scraper
description: 使用 Playwright MCP 工具自動從 Axure 設計稿（axshare.com）擷取並整理 spec 內容成 Markdown。**務必在以下情況觸發此 skill**：使用者說「幫我抓 spec」、「把設計稿整理成文件」、「從 Axure 擷取規格」、「幫我解讀這個設計稿」、「輸出 pm_spec.md」、「抓 Axure」，或直接貼上 axshare.com / Axure 設計稿連結（即使使用者沒有明確說「使用 skill」）。這是整合設計規格的關鍵第一步。
---

# Axure Spec Scraper

使用 Playwright MCP 工具，從 Axure 分享連結中擷取設計稿 spec 內容，整理為可讀的 Markdown 文件。

> **工具說明：** 本 skill 統一使用 `mcp_playwright_browser_run_code_unsafe`（參數名 `code`）執行所有瀏覽器端操作。**請勿使用 `mcp_playwright_browser_evaluate`**（此工具不存在或 schema 不同）。`code` 的入口格式固定為 `async (page) => { ... }`。

### ⚠️ 關鍵技術提醒

- **頁面環境 vs Node.js 環境**：所有對 `window`、`document` 的訪問**必須**在 `page.evaluate()` 或 `page.waitForFunction()` 的**回調函數內**進行。
  - ❌ **錯誤**：`const doc = window.$axure.document;` （直接在 Node.js 執行 → window 不存在）
  - ✅ **正確**：`await page.evaluate(() => { const doc = window.$axure.document; return doc; })`
- 務必確保每一段訪問瀏覽器對象的代碼都被 `page.evaluate()` 或 `page.waitForFunction()` 的箭頭函數包裹。

### 執行前檢查清單

在執行每個步驟的代碼前，**必須確認**：

- [ ] 代碼入口是 `async (page) => { ... }` 格式
- [ ] 所有 `window.*` 訪問都在 `page.evaluate(() => { ... })` 內
- [ ] 所有 `document.*` 訪問都在 `page.evaluate(() => { ... })` 內
- [ ] `page.waitForFunction()` 的條件函數使用 `() => { ... }` 箭頭函數
- [ ] 頁面導航後，重新等待 Axure JS 水合（`page.waitForFunction(() => window.$axure)` ）

---

## 步驟一：確認 Playwright MCP 可用

使用 `tool_search` 查詢關鍵字 `playwright browser navigate snapshot`。

### 執行檢查清單
✓ 找到 `mcp_playwright_browser_navigate` 工具  
✓ 找到 `mcp_playwright_browser_run_code_unsafe` 工具  
✓ 找到 `mcp_playwright_browser_snapshot` 工具  

- **若以上工具都找到** → 繼續執行。
- **若找不到** → 通知使用者：

  > Playwright MCP 目前未連線。請在 VS Code 中執行：
  > 1. `Cmd+Shift+P` → 搜尋 **MCP: List Servers**
  > 2. 確認 `playwright` 狀態為 **Connected**
  > 3. 若未連線，請確認 `.vscode/mcp.json` 已設定 playwright server 並重新啟動 VS Code

  停止作業，等待使用者確認後再繼續。

---

## 步驟二：取得 Axure 網址

若使用者未提供網址，詢問：

> 請提供 Axure 設計稿的分享網址（格式通常為 `https://xxxx.axshare.com/?id=xxxxxx`）

---

## 步驟三：導航並取得頁面結構

### 執行檢查清單
✓ 已呼叫 `mcp_playwright_browser_navigate` 導航至 Axure 連結  
✓ 回應顯示 Page Title（頁面已載入）  
✓ 無密碼驗證框提示  

**步驟：**

1. 呼叫 `mcp_playwright_browser_navigate` 導航至目標網址（`https://{share-id}.axshare.com/?id={page-id}`）
2. 確認頁面載入（回應中有 Page Title）
3. **偵測 access code 驗證**：若 snapshot 或頁面標題出現密碼/驗證輸入框，停止並請使用者直接在瀏覽器輸入密碼，確認後再繼續。
4. 使用 `mcp_playwright_browser_run_code_unsafe` 等待 `$axure` 注入並取得頁面結構：

```js
// 【頁面環境】此代碼在頁面上下文中執行，可訪問 window、document
async (page) => {
  // ✓ 在頁面環境中等待 $axure 注入
  await page.waitForFunction(() => window.$axure && window.$axure.document, { timeout: 10000 });
  
  // ✓ 在頁面環境中訪問並返回 $axure 數據
  return await page.evaluate(() => {
    const doc = window.$axure.document;  // ✓ window 存在
    return JSON.stringify(doc.pages || doc.sitemap, null, 2);
  });
}
```

此步驟可取得所有頁面的 `pageName`、`id`、`url`、資料夾結構（rootNodes）。

---

## 步驟四：擷取指定頁面 spec 內容

### 執行檢查清單
✓ 頁面已完全載入（能訪問 $axure）  
✓ 使用者未指定頁面時，使用當前頁  

**只抓使用者指定的頁面（預設為當前頁）**，不需遍歷所有頁面。

使用 `mcp_playwright_browser_run_code_unsafe` 從 spec frame 抓取文字。**方法一最常用；若回傳空字串，改用方法二。**

### 方法一：主頁面或 iframe（常規場景）

```js
// 【頁面環境】此代碼在頁面上下文中執行
async (page) => {
  // ✓ page.evaluate() 保證在頁面環境中執行
  return await page.evaluate(() => {
    const result = [];

    // 嘗試直接從主頁面的 iframe 內取得（同 origin 場景）
    const iframe = document.querySelector('iframe');
    let body = null;
    try {
      if (iframe && iframe.contentDocument) {
        body = iframe.contentDocument.body;
      }
    } catch (e) {
      // cross-origin iframe，先跳過
    }

    // 若 iframe 不可用，嘗試取主頁面 body（部分 Axure 版本 spec 直接在主頁）
    if (!body) body = document.body;

    body.querySelectorAll('p, h1, h2, h3, h4, h5, h6, li, td, th').forEach(el => {
      const directText = Array.from(el.childNodes)
        .filter(n => n.nodeType === Node.TEXT_NODE)
        .map(n => n.textContent.trim())
        .join('');
      const text = directText || el.innerText.trim();
      if (text) result.push(text);
    });

    // 去除重複行並過濾空白
    return [...new Set(result)].filter(Boolean).join('\n---\n');
  });
}
```

### 方法二：透過 Playwright frame API（當方法一回傳空字串時）

若方法一回傳空字串或錯誤，改用 `mcp_playwright_browser_run_code_unsafe`：

```js
// 【混合環境】頁面層級訪問 frames()，frame 層級訪問 DOM
async (page) => {
  // ✓ 這裡在 Node.js 環境，page 是 Playwright Page 對象
  const frames = page.frames();  // ✓ page.frames() 是 Node.js API
  
  const specFrame = frames.find(f => 
    f.url().includes('axshare.com') && f !== page.mainFrame()
  );
  
  if (!specFrame) return 'no spec frame found';
  
  // ✓ 切換到 frame 的頁面環境
  return await specFrame.evaluate(() => {  // ✓ frame.evaluate() 在 frame 的頁面環境執行
    const result = [];
    document.body.querySelectorAll('p, h1, h2, h3, h4, h5, h6, li, td, th').forEach(el => {
      const text = el.innerText.trim();
      if (text) result.push(text);
    });
    return [...new Set(result)].filter(Boolean).join('\n---\n');
  });
}
```

若需要切換頁面，使用 `mcp_playwright_browser_navigate`：
- URL 格式：`https://{share-id}.axshare.com/#id={page-id}&p={encoded-page-name}`
- 切換後重新執行步驟三（等待 `$axure` 水合）再執行本步驟。

---

## 步驟五：轉換為 Markdown

根據擷取內容，依以下結構輸出：

```markdown
# {頁面標題}　畫面說明

> **來源：** {原始網址}
> **路徑：** {breadcrumb 路徑}
> **頁面 ID：** `{id}`
> **擷取日期：** {YYYY-MM-DD}

---

## 重要異動

{異動摘要}

---

## 改版項目總覽

| 項目標號 | 區塊 | 說明摘要 |
|:---:|---|---|
| 1 | {區塊名} | {說明} |

---

## {標號}. {區塊名稱}

{詳細說明，使用條列與子清單}

### {子標題（如有排序規則、補充說明等）}

{內容}
```

**格式原則：**
- 標號對應的區塊各自為一個 `##` 段落
- 條列項目保持 `- ` 格式
- 有序清單（險種列表等）使用 `1.` 格式
- 表格用於排序規則、欄位對照等結構性資料
- 重要詞彙用 `**粗體**` 標示

---

## 步驟六：輸出檔案

若使用者要求存檔（如「輸出 pm_spec.md」），使用 `create_file` 建立檔案於 workspace 根目錄。

檔案命名慣例：`pm_spec.md`（或使用者指定名稱）

輸出完成後，告知使用者：

> ✅ `pm_spec.md` 已產出至根目錄。  
> 下一步：執行 **figma-spec-extractor** 擷取視覺規格，產出 `design_spec.md`。  
> 待 design_spec 完成後，可執行 **frontend-spec-synthesizer** 讀取 `pm_spec.md` + `design_spec.md`，產出 `frontend_sdd.md`（三份文件皆留在專案根目錄）。

---

## 常見實作錯誤

### ❌ 錯誤 #1：在 Node.js 環境訪問 window

```js
// ❌ 錯誤 — window is not defined（在 Node.js 執行）
async (page) => {
  await page.waitForFunction(() => window.$axure && window.$axure.document, { timeout: 10000 });
  const doc = window.$axure.document;  // ❌ 在這裡 window 不存在！
  return JSON.stringify(doc.pages);
}
```

**修正：** 將訪問 window 的代碼放在 `page.evaluate()` 的回調中

```js
// ✅ 正確 — window 訪問在頁面環境
async (page) => {
  await page.waitForFunction(() => window.$axure && window.$axure.document, { timeout: 10000 });
  return await page.evaluate(() => {  // ✅ 進入頁面環境
    const doc = window.$axure.document;  // ✓ window 現在存在
    return JSON.stringify(doc.pages);
  });
}
```

### ❌ 錯誤 #2：混淆 page 和 frame 的環境

```js
// ❌ 錯誤 — page.frames() 後試圖直接訪問 frame 內容
const frames = page.frames();  // Node.js 環境
const body = frames[1].document.body;  // ❌ frames[1].document 不可用
```

**修正：** 用 `frame.evaluate()` 進入 frame 的頁面環境

```js
// ✅ 正確
const frames = page.frames();
const content = await frames[1].evaluate(() => {
  return document.body.innerText;  // ✓ frame 的頁面環境
});
```

---

## 注意事項與除錯指引

| 現象 | 原因 | 處理方式 |
|---|---|---|
| `window is not defined` | 訪問 window 的代碼不在 `page.evaluate()` 或 `page.waitForFunction()` 的回調中 | **務必**將所有 window/document 訪問包在 `await page.evaluate(() => { ... })` 中 |
| `$axure is not defined` | JS 尚未水合完畢 | 確保 `waitForFunction(() => window.$axure && window.$axure.document)` 在 `evaluate` 之前執行 |
| `no iframe` / 空字串 | cross-origin iframe 或 spec 在主頁面 | 改用方法二（Playwright `page.frames()` API，見步驟四） |
| 頁面出現密碼輸入框 | Axure access code 保護 | 請使用者直接在瀏覽器輸入密碼後再繼續 |
| 內容重複 | Axure DOM 有巢狀文字節點 | 已在擷取邏輯加入 `Set` 去重 |
| 回傳 `SecurityError` | iframe 跨 origin | 切換為步驟四方法二（`specFrame.evaluate()`） |

- `window.$axure.document.pages.rootNodes` 包含完整頁面樹狀結構（含資料夾）
- 不需要截圖（`mcp_playwright_browser_take_screenshot`），文字擷取已足夠
- **絕對不要使用 `mcp_playwright_browser_evaluate`**，此工具不存在或 schema 不同，會導致驗證錯誤
