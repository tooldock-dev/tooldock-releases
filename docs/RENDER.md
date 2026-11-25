# 文件輸出（Render）與 Markdown 編輯器規格書

> 1.9.0 起，Markdown 編輯器改由擴充商店提供，原始碼移到 [tooldock-ext-markdown-editor](https://github.com/tooldock-dev/tooldock-ext-markdown-editor)；以下〈7〉的內容仍適用。
>
> 狀態：**已實作**：平台 **1.7.0**（`ctx.render.pdf`、文件樣式、資料交換的 mime 篩選）、Markdown 編輯器 1.0.0、報表產生器 1.2.0；報表產生器的 PDF 匯出（1.3.0）為**草案，決定已定案**。本文件定義平台的「HTML 轉 PDF」服務 `ctx.render`、共用的文件樣式、資料交換的 mime 篩選，以及第一批使用它們的內建擴充：新增的 **Markdown 編輯器**、**報表產生器** 1.2.0（Markdown 來源，**已實作**）與 1.3.0（PDF 匯出）。實作以本文件為準，若需變更請先修改本文件。

## 1. 目的與範圍

使用者想要：

1. 在 ToolDock 裡**編輯 Markdown**，同時看到預覽，用工具列插入常用語法。
2. 把 Markdown **轉成 PDF**。
3. 把 Markdown 裡的**表格轉成 Excel**。

評估現有內建擴充後：

- **資料轉換**的轉換器都是「文字進、文字出」（`run(input) → string`），畫面是固定的輸入框和輸出框，不適合二進位檔案和編輯器。
- **報表產生器**以「資料來源 → 挑欄位 → 匯出」為主，已經有 Excel 和 Word 匯出器。Markdown 表格只要多一個資料來源就能沿用（〈8.1〉）。
- **PDF 沒有擴充能自己做好**。擴充後端在 `utilityProcess` 執行（`src/main/extension-host.js`），拿不到 `BrowserWindow` 和 `webContents.printToPDF`。擴充畫面是工作台裡的 iframe，`window.print()` 會跳出列印對話框，而且可能把整個工作台一起印出來。純 JS 的 PDF 函式庫（pdfkit、pdfmake）必須內嵌好幾 MB 到十幾 MB 的中文字型，標題、程式碼區塊、表格的排版也要自己處理。

所以本規格提供：

- **平台服務 `ctx.render.pdf()`**（〈4〉、〈5〉）：擴充給 HTML，平台在隱藏視窗用 Chromium 排版，再輸出 PDF。中文使用系統字型，擴充不需要帶任何函式庫。報表、Markdown，以及之後其他想輸出 PDF 的工具都能共用。
- **共用文件樣式 `tooldock://sdk/document.css`**（〈6〉）：中文字型、標題、表格、程式碼區塊和分頁規則。畫面預覽和 PDF 用同一份樣式，看到的就是印出來的。
- **新增內建擴充「Markdown 編輯器」**（〈7〉）：負責編輯、預覽、工具列，並匯出 HTML、PDF，表格交給報表產生器。
- **報表產生器**（〈8〉）：1.2.0 新增 Markdown 資料來源，1.3.0 新增 PDF 匯出。
- **資料交換的 mime 篩選**（〈7.7〉）：`imports[]` 新增 `mimes`，分享 Markdown 時直接交給 Markdown 編輯器，不再跳出選擇器。規則寫在 docs/EXCHANGE.md〈7.1〉、〈8.1〉。

第一個要支援的情境：**在 Markdown 編輯器寫好一份含表格的文件 → 匯出 PDF（有頁碼、書籤）→ 點「用報表產生器開啟表格」→ 匯出 Excel。**

不在範圍內：

- 平台內建 Markdown 解析。平台只處理 HTML，Markdown 由擴充自己解析（〈13〉D2）。
- HTML 轉 Word、轉圖片。列在〈10. 後續版本〉。
- 防止惡意擴充。平台沒有沙箱，本規格限制的是**轉換頁面能載入什麼**（〈5.2〉），保護的是文件內容不被外洩到網路（〈9〉）。

## 2. 名詞

| 名詞 | 說明 |
|---|---|
| **轉換**（render job） | 一次 `ctx.render.pdf()` 呼叫，從載入 HTML 到寫出 PDF |
| **轉換視窗** | 平台為一次轉換開的隱藏視窗，轉換結束就關閉 |
| **基準資料夾**（baseDir） | HTML 裡相對路徑資源（例如 `images/a.png`）的解析起點，也是轉換視窗唯一能讀取的本機資料夾 |
| **文件樣式** | 平台提供的 `tooldock://sdk/document.css`，套用在 `.td-doc` 底下 |
| **完成訊號** | 啟用 JavaScript 時，頁面告訴平台「畫完了，可以輸出」的方式（〈5.3〉） |

## 3. 架構

```
 擴充（後端或畫面）                 平台（ExtensionManager + Render）
 ─────────────────                 ──────────────────────────────────
 ctx.render.pdf(opts) ──RPC──▶  1. 驗證參數（不合法 → 丟出例外）
 tooldock.render.pdf(opts)      2. 排隊（全平台同時最多 2 個轉換）
                                3. 準備：HTML 寫成暫存檔，插入 <base>、文件樣式、額外 CSS、<title>
                                4. 開轉換視窗（隱藏、沙箱、獨立的記憶體 session、預設不執行 JavaScript）
                                   └─ 依〈5.2〉的規則放行或擋下每一個資源請求
                                5. 等待 load → document.fonts.ready →（啟用 JavaScript 時）完成訊號
                                6. printToPDF（紙張、邊界、頁首頁尾、書籤、標記 PDF）→ 寫入 output
                                7. 關閉視窗、刪除暫存檔，回傳 RenderResult
 ◀──────────────────────────────
```

轉換期間狀態列顯示「📄 產生 PDF 中：<標題>」，點擊可以取消（回報 `cancelled`）。轉換期間呼叫端不會被自動回收（與 `notify.send` 相同，用 `track()` 計入執行中）。

## 4. API

### 4.1 呼叫方式

```js
// Markdown 編輯器：匯出 PDF
if (!ctx.render) throw new Error('匯出 PDF 需要 ToolDock 1.7.0 以上');
const { canceled, filePath } = await ctx.window.showSaveDialog({
  defaultPath: doc.name.replace(/\.(md|markdown)$/i, '') + '.pdf',
  filters: [{ name: 'PDF', extensions: ['pdf'] }],
});
if (canceled) return;

const r = await ctx.render.pdf({
  html: `<h1>${title}</h1>${bodyHtml}`,
  output: filePath,
  baseDir: path.dirname(doc.file),   // 讓 ![](images/a.png) 讀得到
  title,
  stylesheet: 'document',
  page: { size: 'A4', margin: 15 },
  footer: '{page} / {pages}',
});
if (r.status !== 'done') ctx.window.toast(`PDF 沒有產生：${r.message}`, 'error');
else if (r.blocked.length) ctx.window.toast(`已產生，但有 ${r.blocked.length} 個資源被擋下（例如網路圖片）`, 'warning');
```

| 方法 | 說明 |
|---|---|
| `pdf(options): Promise<RenderResult>` | 把 HTML 轉成 PDF 並寫入 `options.output`。**只有參數不合法時丟出例外**（`err.code === 'RENDER_INVALID'`），其他情況都用 `status` 回報，與 `ctx.exchange.share`、`ctx.notify.send` 相同 |

擴充畫面也可以呼叫 `window.tooldock.render.pdf()`，呼叫者同樣是該擴充。

### 4.2 參數 `RenderPdfOptions`

| 欄位 | 型別 | 預設 | 說明 |
|---|---|---|---|
| `html` | string | — | HTML 內容。可以是完整文件，也可以只是 `<body>` 內的片段。UTF-8 編碼後最大 10 MB（與資料交換相同）。`html` 和 `file` 必須剛好給一個 |
| `file` | string | — | 本機 `.html`、`.htm` 檔案的絕對路徑 |
| `output` | string | **必填** | 輸出的 PDF 絕對路徑，副檔名必須是 `.pdf`，資料夾必須已存在。檔案已存在時直接覆蓋（通常由存檔對話框確認過） |
| `baseDir` | string | `file` 所在資料夾；`html` 時沒有預設 | 相對路徑資源的基準資料夾。`html` 沒給 `baseDir` 時，所有 `file:` 資源都會被擋下 |
| `title` | string | HTML 的 `<title>` | PDF 文件屬性的標題，也是頁首頁尾的 `{title}`。給了就取代 `<title>` |
| `stylesheet` | `'none'` \| `'document'` | `'none'` | `'document'` 時載入文件樣式並在 `<body>` 加上 `class="td-doc"`（〈6〉） |
| `css` | string | — | 額外的 CSS，放在文件樣式之後。最大 1 MB |
| `page.size` | `'A3'` \| `'A4'` \| `'A5'` \| `'Letter'` \| `'Legal'` \| `{ width, height }` | `'A4'` | 紙張。自訂尺寸以**公釐**為單位，各邊 50～2000 |
| `page.landscape` | boolean | `false` | 橫向 |
| `page.margin` | number \| `{ top, right, bottom, left }` | `15` | 邊界，單位公釐，0～100。有頁首頁尾時，上下邊界至少 12 公釐，太小會自動加大並寫入 `warnings` |
| `page.scale` | number | `1` | 縮放，0.1～2 |
| `page.ranges` | string | 全部 | 只輸出部分頁面，例如 `'1-3, 5'` |
| `page.preferCssPageSize` | boolean | `false` | 以 CSS `@page { size }` 為準，忽略 `page.size` |
| `header` / `footer` | string | 無 | 簡易頁首、頁尾文字，可用 `{page}`、`{pages}`、`{title}`、`{date}`（`YYYY-MM-DD`）。以 `｜` 分隔時依序放在左、中、右，沒有 `｜` 時置中，例如 `'{title}｜｜{page} / {pages}'`（〈4.3〉） |
| `headerHtml` / `footerHtml` | string | 無 | 進階：Chromium 的頁首頁尾樣板（〈4.3〉）。和 `header`、`footer` 擇一使用 |
| `outline` | boolean | `true` | 依 `<h1>`～`<h6>` 產生 PDF 書籤 |
| `tagged` | boolean | `true` | 產生標記 PDF（無障礙閱讀、複製文字順序較正確） |
| `background` | boolean | `true` | 印出背景色與背景圖（表頭底色需要） |
| `javascript` | boolean | `false` | 轉換頁面是否執行 JavaScript（〈5.3〉） |
| `allowRemote` | boolean | `false` | 是否允許載入 `http`、`https` 資源（〈5.2〉） |
| `timeout` | number | `60` | 逾時秒數，最大 300。從**開始轉換**算起，排隊時間不算 |

### 4.3 頁首與頁尾

簡易寫法 `header`、`footer` 由平台轉成樣板：字級 9pt、灰色、使用文件樣式的中文字型，左中右三欄。文字會先做 HTML 跳脫，再把 `{page}` 換成 `<span class="pageNumber">`、`{pages}` 換成 `<span class="totalPages">`。

進階寫法 `headerHtml`、`footerHtml` 直接交給 Chromium，可以使用 `pageNumber`、`totalPages`、`title`、`date`、`url` 這幾個 class。注意 Chromium 的限制：

- 樣板**不會**套用頁面的 CSS，也不會載入外部資源，字型和字級要寫在樣板自己的 `style` 裡。沒有指定字級時 Chromium 預設非常小，平台會補上 `font-size: 9pt`。
- 樣板裡的圖片只能用 `data:` URL。

### 4.4 回應 `RenderResult`

```ts
interface RenderResult {
  status: 'done' | 'timeout' | 'cancelled' | 'failed' | 'busy';
  /** 轉換代號，對應輸出面板的紀錄 */
  id: string;
  /** status 為 done 時：輸出的檔案、大小與頁數 */
  file?: string;
  bytes?: number;
  pages?: number;
  /** 被〈5.2〉擋下的資源網址，最多 20 筆（含 file: 的只留檔名） */
  blocked: string[];
  /** 不影響輸出的提醒，例如「邊界太小，已加大到 12 公釐」、「完成訊號逾時，已直接輸出」 */
  warnings: string[];
  /** 不是 done 時的原因 */
  message?: string;
  /** 毫秒，從開始轉換算起 */
  duration: number;
}
```

| status | 情況 |
|---|---|
| `done` | 已寫入 `output` |
| `timeout` | 超過 `timeout` 秒，沒有輸出 |
| `cancelled` | 使用者從狀態列取消、呼叫端被停用，或平台正在結束 |
| `failed` | 載入失敗、`printToPDF` 失敗、寫檔失敗（例如沒有權限、磁碟已滿）。`message` 說明原因 |
| `busy` | 這個擴充已經有 5 個轉換在排隊，沒有受理 |

`failed`、`timeout`、`cancelled` 時，平台保證 `output` 不會留下寫到一半的檔案：先寫到同一個資料夾的暫存檔，成功後才改名。

## 5. 轉換環境

### 5.1 轉換視窗

每次轉換開一個新的隱藏 `BrowserWindow`，結束就關閉，不重複使用：

| 設定 | 值 | 原因 |
|---|---|---|
| `show` | `false` | 使用者看不到 |
| `webPreferences.sandbox` | `true` | |
| `contextIsolation` / `nodeIntegration` | `true` / `false` | |
| `preload` | 無 | 轉換頁面拿不到 `window.tooldock`，也碰不到平台 RPC |
| `javascript` | `options.javascript` | 預設不執行 |
| `session` | 記憶體中的獨立 partition `render-slot-0`、`render-slot-1`（不寫入磁碟），每次轉換前清空 | 不共用工作台和擴充畫面的 cookie、localStorage。同時最多 2 個轉換，各用一個 slot，資源規則才能對應到正在使用它的那一個轉換 |
| 導覽 | 擋下 `will-navigate`、`window.open`、下載、所有權限請求（相機、通知…） | 頁面只能是交給它的那一份 |

每個 slot session 自己處理 `file:`、`http:`、`https:`、`tooldock:`、`tbx-ext:` 協定（`session.protocol.handle`），依目前轉換的規則放行或擋下（〈5.2〉）。

HTML 準備方式：平台把 `html`（或讀出 `file` 的內容）寫到 `<userData>/render-temp/<id>/index.html`，在 `<head>` 最前面插入：

1. `<meta charset="utf-8">`（原本沒有時）
2. `<base href="file:///<baseDir>/">`（有 `baseDir` 時）
3. `<link rel="stylesheet" href="tooldock://sdk/document.css">`（`stylesheet: 'document'` 時）
4. `<style>` 放入 `css`
5. `<title>`（有 `title` 時取代原本的）

片段（沒有 `<html>`）會先包成完整文件。暫存資料夾在轉換結束時刪除，平台啟動時也會清掉上次殘留的。

### 5.2 資源載入規則

slot session 的協定處理依下表放行或擋下；`session.webRequest.onBeforeRequest` 再擋掉表中沒有的協定，作為第二層：

| 來源 | 規則 |
|---|---|
| 這次轉換的暫存 HTML | 允許 |
| `baseDir` 底下的 `file:` | 允許。以實際路徑（`realpath`）比對，用符號連結跳出 `baseDir` 的會被擋下 |
| `baseDir` 以外的 `file:` | 擋下 |
| `tbx-ext://<呼叫端自己的 id>/` | 允許（擴充可以帶自己的 CSS、字型、圖片） |
| `tbx-ext://<其他擴充>/` | 擋下 |
| `tooldock://sdk/`、`tooldock://libs/` | 允許 |
| `tooldock://app/` | 擋下 |
| `data:`、`blob:` | 允許 |
| `http:`、`https:` | `allowRemote: true` 才允許 |
| 其他協定（`ftp:`、`ws:`…） | 擋下 |

被擋下的資源不會讓轉換失敗，只會記錄在 `RenderResult.blocked`（圖片顯示成空白）。

### 5.3 JavaScript 與完成訊號

預設不執行 JavaScript，頁面在 `load` 和 `document.fonts.ready` 之後就輸出。

`javascript: true` 時（例如頁面要用圖表函式庫畫圖）：

- 頁面在載入過程中設定 `window.TOOLDOCK_RENDER_WAIT = true`，表示要等它；畫完後設定 `window.TOOLDOCK_RENDER_READY = true`。平台每 100 毫秒檢查一次。
- 沒有設定 `TOOLDOCK_RENDER_WAIT` 時，和不執行 JavaScript 一樣，`load` 後就輸出。
- 等完成訊號最多等到 `timeout` 的一半。超過時直接輸出，並在 `warnings` 註記，不回報 `timeout`。

### 5.4 排隊、逾時與大小

- 全平台同時最多 **2** 個轉換，其他依呼叫順序排隊。每個擴充最多 5 個排隊中，再多就回報 `busy`。
- 逾時（`timeout`）從開始轉換算起。逾時、取消時立即關閉轉換視窗。
- `html` 最大 10 MB、`css` 最大 1 MB；`file` 指向的檔案最大 10 MB。圖片等資源平台不限制大小，由逾時限制總時間。
- 每次轉換寫一行到輸出面板：呼叫端、標題、頁數、大小、耗時、被擋下的資源數。

## 6. 文件樣式 `tooldock://sdk/document.css`

一份適合列印的中文文件樣式。所有規則都限定在 `.td-doc` 底下，所以擴充畫面可以同時載入 `ui-base.css` 和 `document.css`，不會互相影響。

| 項目 | 規則 |
|---|---|
| 字型 | 內文 `"PingFang TC", "Microsoft JhengHei", "Noto Sans CJK TC", "Noto Sans TC", sans-serif`；程式碼 `"SF Mono", Menlo, Consolas, "Noto Sans Mono CJK TC", monospace` |
| 字級與行高 | 內文 10.5pt、行高 1.7；h1～h6 為 20、16、14、12、11、10.5pt |
| 標題 | h1、h2 底部加細線；標題不會單獨留在頁尾（`break-after: avoid`） |
| 表格 | 全框線、表頭淺灰底；表頭在每一頁重複（`thead { display: table-header-group }`）；一列不會被拆到兩頁（`tr { break-inside: avoid }`） |
| 程式碼區塊 | 淺灰底、自動換行（`white-space: pre-wrap`），不會超出紙張；短的區塊不拆頁 |
| 圖片 | `max-width: 100%`，不拆頁 |
| 引用、清單、待辦清單、水平線 | 一般文件樣式；待辦清單顯示核取方塊 |
| 強制分頁 | `.page-break { break-after: page }` |
| 顏色 | 固定淺色（白底黑字），不跟隨深色主題，見〈13〉D6 |

報表產生器和 Markdown 編輯器的預覽都使用這份樣式，所以預覽看到的，就是 PDF 印出來的。

## 7. Markdown 編輯器（`markdown-editor` 1.0.0）

新增的內建擴充，放在 `extensions/markdown-editor`（1.7.0～1.8.x）。1.9.0 起改由擴充商店提供，原始碼在 [tooldock-ext-markdown-editor](https://github.com/tooldock-dev/tooldock-ext-markdown-editor)。

| manifest 項目 | 值 |
|---|---|
| `id` / `displayName` / `icon` | `markdown-editor` / Markdown 編輯器 / 📝 |
| `engine` | `^1.4.0`（需要資料交換）。PDF 匯出需要 1.7.0，舊平台上隱藏 PDF 按鈕，並提示「需要 ToolDock 1.7.0 以上」 |
| 相依套件 | `markdown-it`（解析）、`highlight.js`（程式碼上色，只帶常用語言） |
| 畫面 | `markdownEditor.main` |
| 指令 | `markdownEditor.open`（開啟）、`markdownEditor.new`（新文件）、`markdownEditor.openFile`（開啟檔案…）、`markdownEditor.exportPdf`（匯出 PDF） |

### 7.1 畫面

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 📂 開啟  💾 儲存 │ H▾ B I S <> │ • 1. ☑ ❝ │ 🔗 🖼 ▦ {} ― ⤓ │ 編輯│分割│預覽 │ 匯出▾ │
├────────────────────────────────────┬─────────────────────────────────────┤
│ # 月報                              │  月報                               │
│                                    │  ─────────                          │
│ | 項目 | 數量 |                     │  ┌──────┬──────┐                    │
│ |------|-----:|                     │  │ 項目 │ 數量 │                    │
│ | A    |   12 |                     │  ├──────┼──────┤                    │
│                                    │  │ A    │   12 │                    │
│            （編輯區）               │      （預覽：白底紙張、文件樣式）      │
├────────────────────────────────────┴─────────────────────────────────────┤
│ 月報.md　｜　第 3 行，第 8 欄　｜　412 字　｜　● 未儲存（草稿已保存）         │
└──────────────────────────────────────────────────────────────────────────┘
```

- **三種模式**：編輯、分割（預設）、預覽，記住上次的選擇。分割時可以拖曳中間的分隔線。
- **編輯區**使用 `<textarea>`（〈13〉D5），等寬字型，字級可調。
- **預覽的 HTML 由後端產生**（與 HTML、PDF 匯出共用同一份規則，`backend/markdown.js`），畫面只負責顯示。預覽與 HTML 匯出使用擴充自帶的 `ui/document.css` 複本（與平台的內容相同，`npm test` 會比對），所以在平台 1.4.0～1.6.x 也能預覽；PDF 使用平台的 `stylesheet: 'document'`。
- **預覽**在編輯停止 150 毫秒後更新，用白底紙張呈現，寬度約等於 A4 內文寬度。
- **捲動同步**：markdown-it 的區塊會帶原始行號（`token.map`），預覽元素加上 `data-line`。捲動任一側時，另一側捲到對應的區塊。可在設定關閉。
- **狀態列**：檔名、游標行列、字數（中文一字算一字，英文以空白分詞）、儲存狀態。
- **一次編輯一份文件**。開啟其他檔案前，如果有未儲存的變更，會詢問「儲存／不儲存／取消」。

### 7.2 工具列

| 群組 | 按鈕 | 插入的語法 | 沒有選取文字時 |
|---|---|---|---|
| 標題 | H▾（段落、H1～H6） | 行首 `# `～`###### ` | 套用到游標所在行 |
| 文字 | 粗體 B | `**…**` | 插入 `**粗體文字**` 並選取「粗體文字」 |
| | 斜體 I | `*…*` | 同上，「斜體文字」 |
| | 刪除線 S | `~~…~~` | 同上，「刪除線文字」 |
| | 行內程式碼 `<>` | `` `…` `` | 同上，「code」 |
| 段落 | 項目清單 • | 每行行首 `- ` | 套用到游標所在行 |
| | 編號清單 1. | 每行行首 `1. `、`2. `… | 同上 |
| | 待辦清單 ☑ | 每行行首 `- [ ] ` | 同上 |
| | 引用 ❝ | 每行行首 `> ` | 同上 |
| 插入 | 連結 🔗 | `[文字](網址)`；選取的是網址時放進括號 | 插入 `[連結文字](https://)` 並選取網址 |
| | 圖片 🖼 | 開檔案對話框選圖片，插入 `![檔名](相對路徑)`；文件還沒存檔時用絕對路徑並提示 | — |
| | 表格 ▦ | 點選格子選擇欄列數（最多 10 欄 × 7 列資料），插入含表頭的空表格，選取第一個表頭 | — |
| | 程式碼區塊 `{}` | ```` ```語言 ```` 包住選取的行；語言從下拉選單挑 | 插入空區塊 |
| | 水平線 ― | 獨立一行 `---` | — |
| | 分頁 ⤓ | 獨立一行 `<!-- pagebreak -->`（〈7.4〉） | — |

行為規則：

- **切換**：選取的文字已經有同樣的語法時（例如已經是 `**粗體**`），按鈕會移除語法。行首類的按鈕也一樣，所有選取的行都有前綴時就移除。
- **可以復原**：插入一律使用 `document.execCommand('insertText')`，讓 ⌘Z／Ctrl+Z 能逐步復原。
- 游標所在位置的語法會讓對應按鈕顯示為按下狀態（例如游標在粗體裡，B 會亮起）。

### 7.3 快捷鍵

macOS 使用 ⌘，Windows、Linux 使用 Ctrl。平台選單已經佔用的組合（⌘W、⌘⇧W、⌘⇧P、⌘⇧X、⌘⇧S、⌘⇧L、⌘⇧U，見 `src/main/main.js` 的選單）會先被選單攔下，不能使用。

| 快捷鍵 | 動作 |
|---|---|
| ⌘B / ⌘I / ⌘⇧D / ⌘E | 粗體／斜體／刪除線／行內程式碼 |
| ⌘K | 連結 |
| ⌘1～⌘6、⌘0 | 標題 1～6、取消標題 |
| ⌘S / ⌘⌥S / ⌘O / ⌘N | 儲存／另存新檔／開啟／新文件 |
| ⌘P | 匯出 PDF |
| Tab / ⇧Tab | 在清單中增加或減少縮排；不在清單時插入兩個空白 |
| Enter | 在清單項目中延續清單（自動加上 `- `、下一個編號、`- [ ] `）；在空的項目按 Enter 結束清單 |

### 7.4 Markdown 解析

- **markdown-it**：CommonMark 加上 GFM 表格、刪除線；`linkify` 開啟、`typographer` 關閉。
- **`html: false`**：Markdown 裡的 HTML 一律當文字顯示，預覽和 PDF 都不會執行文件裡的 HTML 或腳本（〈9〉）。
- **待辦清單**：`- [ ]`、`- [x]` 顯示成核取方塊（只能看，不能點）。
- **分頁**：獨立一行的 `<!-- pagebreak -->` 由自訂的區塊規則辨識，輸出 `<div class="page-break"></div>`。雖然 `html: false`，這一種寫法還是會被辨識；在 GitHub 等其他 Markdown 檢視器中，它是看不見的 HTML 註解。
- **程式碼上色**：highlight.js，認不出的語言不上色。
- **連結**：預覽中點擊連結時，`http`、`https` 用 `tooldock.shell.openExternal` 開啟，其他連結不處理。
- **本機圖片**：預覽在 `tbx-ext://` 頁面裡，讀不到 `file:`。相對路徑或絕對路徑的圖片由後端讀取後轉成 `data:` URL 顯示，限制在文件所在資料夾底下、單張 20 MB 以內，結果會快取到檔案修改時間改變為止。網路圖片在預覽中照常顯示；PDF 預設擋下，見〈7.5〉。

### 7.5 檔案與匯出

**開啟與儲存**

- 開啟 `.md`、`.markdown`、`.txt`。保留原本的換行字元（LF／CRLF）和 UTF-8 BOM；新文件使用 LF、沒有 BOM。
- **草稿**：未儲存的內容每 2 秒存到 `paths.data/drafts/`。不正常結束後重新開啟時，詢問要不要還原草稿。
- **外部修改**：畫面重新取得焦點時檢查檔案修改時間，被其他程式改過時詢問「重新載入／保留我的版本」。
- 最近開啟的 10 個檔案顯示在「開啟」的下拉選單。

**匯出選單**

| 項目 | 說明 |
|---|---|
| PDF… | 先選存檔位置，再呼叫 `ctx.render.pdf`，`baseDir` 是文件所在資料夾、`stylesheet: 'document'`；紙張、邊界、頁首頁尾使用〈7.6〉的設定。文件的第一個 `# 標題` 作為 PDF 標題，沒有時用檔名。網路圖片預設擋下，`markdownEditor.pdf.allowRemote` 開啟後才會載入 |
| HTML… | 單一 HTML 檔，內嵌文件樣式；勾選「內嵌圖片」時，本機圖片轉成 `data:` URL |
| 分享文字 | 用 `tooldock.text`（`mime: text/markdown`）分享給其他工具 |
| 用報表產生器開啟表格 | 文件中所有的 GFM 表格。只有一張時直接分享；多張時先讓使用者挑一張。用 `tooldock.table` 分享，報表產生器收到後就能匯出 Excel、Word、PDF（〈13〉D4） |

### 7.6 設定

| 設定 | 型別 | 預設 | 說明 |
|---|---|---|---|
| `markdownEditor.defaultMode` | string（`edit`、`split`、`preview`） | `split` | 開啟時的模式 |
| `markdownEditor.fontSize` | number | `14` | 編輯區字級 |
| `markdownEditor.syncScroll` | boolean | `true` | 編輯區和預覽同步捲動 |
| `markdownEditor.pdf.pageSize` | string（`A4`、`A3`、`A5`、`Letter`、`Legal`） | `A4` | |
| `markdownEditor.pdf.landscape` | boolean | `false` | |
| `markdownEditor.pdf.margin` | number | `15` | 公釐 |
| `markdownEditor.pdf.header` | string | `''` | 頁首，可用 `{page}`、`{pages}`、`{title}`、`{date}`，以 `｜` 分隔左中右 |
| `markdownEditor.pdf.footer` | string | `'｜{page} / {pages}｜'` | 頁尾（預設置中頁碼） |
| `markdownEditor.pdf.allowRemote` | boolean | `false` | PDF 是否載入網路圖片 |

### 7.7 資料交換

```jsonc
"exchange": {
  "imports": [
    { "type": "tooldock.text", "handler": "importText", "title": "在 Markdown 編輯器開啟", "view": "markdownEditor.main",
      "mimes": ["text/markdown", "text/x-markdown"] }
  ]
}
```

- 收到的內容開成新的未命名文件，不直接覆蓋目前的文件；有 `filename` 時當作預設檔名。
- **mime 篩選**（1.7.0 起，docs/EXCHANGE.md〈8.1〉）：Markdown 編輯器宣告 `mimes`，資料轉換不宣告（接收全部文字）。平台只保留最符合的一級接收端，結果如下表。

| 分享的 `mime` | 結果 |
|---|---|
| `text/markdown`、`text/x-markdown` | 直接交給 Markdown 編輯器 |
| `text/plain`、`text/csv`、`application/json`… | 直接交給資料轉換 |

- 想把 Markdown 交給資料轉換時，產生端可以用 `preferReceiver: 'data-converter'`。
- 平台 < 1.7.0 會忽略 `mimes`，兩個工具都出現在選擇器。這時 Markdown 編輯器收到其他 `mime` 仍然接受，並提示「這份內容是 `<mime>`，已當作純文字開啟」。
- 資料轉換不需要改版。

## 8. 報表產生器

### 8.1 Markdown 資料來源（1.2.0，已實作）

新增 `backend/sources/markdown.js`，加進 `sources/index.js` 的陣列，並加入 `markdown-it` 相依套件。

| 項目 | 規則 |
|---|---|
| `extensions` | `md`、`markdown` |
| `supportsQuery` | `false` |
| `listTables(file)` | 文件中的每一張 GFM 表格。名稱是表格前面最近的標題加上序號，例如「銷售統計（表 2）」；沒有標題時是「表 1」、「表 2」…。`rows` 是資料列數 |
| `read(file, { table })` | 表頭成為欄位名稱，空白的表頭命名為「欄位N」（與 CSV、Excel 來源相同），重複的依序加上「(2)」、「(3)」。儲存格去掉行內語法，只留文字：`**粗**` → `粗`、`` `code` `` → `code`、`[文字](網址)` → `文字`，跳脫的 `\|` 還原成直線符號 |
| 型別 | 交給現有的 `normalize()` 推斷，規則和 CSV 相同（有千分位逗號的數字視為文字，前導零的代碼保留為文字） |
| 沒有表格 | `listTables` 回傳空陣列，畫面提示「這份 Markdown 沒有表格」 |

只要這一項，報表產生器就能直接開啟 `.md` 檔並匯出 Excel。**這一項不依賴平台 1.7.0**，可以先做（〈12.2〉）。

儲存格裡的 `<br>` 讀成換行（GFM 表格的換行寫法）；跳脫的 `\<br\>` 保留原文。為了分辨這兩種情況，解析時關閉 markdown-it 的 `text_join`。

### 8.1.1 Markdown 匯出（1.2.0，已實作）

新增 `backend/exporters/markdown.js`，匯出格式變成 Excel／Word／CSV／Markdown 四種。

| 項目 | 規則 |
|---|---|
| 內容 | `# 標題`、副標題、資訊列（產出時間、筆數、製作人），接著是 GFM 表格 |
| 對齊 | 數字欄靠右（`---:`），其他靠左；依顯示寬度補空白（中文算 2），單欄最多補到 40，原始檔也容易閱讀 |
| 數字 | 照原值輸出，不加千分位，讀回時仍然是數字 |
| 跳脫 | `\`、`` ` ``、`*`、`~`、`[`、`]`、`<`、`>`、直線符號前面加 `\`；`_` 只在不夾在英數字中間時跳脫（`user_name` 保持原樣）；換行改成 `<br>` |
| 合計列 | 勾選「數字欄加總列」時，最後一列為 `**合計**` 與加粗的合計值 |
| 檔案 | UTF-8、沒有 BOM、LF |

用 Markdown 來源讀回匯出的檔案時，儲存格內容與原值相同（`npm test` 涵蓋）。

### 8.2 PDF 匯出（1.3.0）

新增 `backend/exporters/pdf.js`，匯出格式從「Excel／Word／CSV／Markdown」變成五種：

- 組出 HTML 表格，使用 `stylesheet: 'document'`，再用 `css` 套上設定的表頭底色（`tableReport.headerColor`）。
- 標題、副標題、產出時間、筆數、製作人的排列方式與 Excel 相同。
- 超過 6 欄時改用橫向，和 Excel 的規則一致。
- 頁尾：`'{title}｜｜第 {page} 頁 / 共 {pages} 頁'`。
- 新增設定 `tableReport.pdfMaxRows`（預設 5000），超過時截斷並在文件最後註記，做法和 Word 相同。
- 平台沒有 `ctx.render`（< 1.7.0）時，畫面不顯示 PDF 選項。
- 「用上次設定重新匯出」（`tableReport.exportLast`）支援 PDF。

## 9. 安全性

- **不經過執行鎖。** 產生 PDF 只寫入本機檔案，不會把資料送出去，也不會用到其他擴充的權限。擴充本來就能用 `fs` 寫檔，加上確認對話框並不會比較安全（〈13〉D3）。
- **轉換頁面預設不連網、不執行 JavaScript。** Markdown 檔案可能來自網路，裡面的網路圖片可以用來追蹤「誰在什麼時候打開了這份文件」。所以預設擋下 `http`、`https`，需要時由使用者在設定中開啟。
- **轉換頁面只能讀取 `baseDir`。** HTML 裡寫 `<img src="/etc/passwd">` 或 `../../` 都讀不到；符號連結以實際路徑比對（〈5.2〉）。
- **轉換頁面和平台隔離**：沒有 preload、沙箱、獨立的記憶體 session，不能導覽到其他頁面、不能開新視窗、不能下載（〈5.1〉）。
- **Markdown 編輯器關閉 HTML**（`html: false`），預覽和 PDF 都不會執行文件中的 HTML（〈7.4〉）。預覽的連結只開啟 `http`、`https`。
- 呼叫者身分由平台依 RPC 連線或 iframe 決定，用來判斷能讀取哪一個 `tbx-ext://`。
- 暫存 HTML 放在 `<userData>/render-temp/`，轉換結束就刪除，平台啟動時清除殘留。

## 10. 後續版本（不在 v1）

- **`ctx.render.image()`**：把 HTML 轉成 PNG（`capturePage`），給通知附件、縮圖使用。
- **回傳內容而不寫檔**：`output` 省略時回傳 PDF 內容，給通知服務的附件使用（需要先支援二進位附件）。
- **Markdown 編輯器**：Mermaid 圖表與 KaTeX 數學式（需要 `javascript: true` 和完成訊號）、多分頁同時編輯多份文件、CodeMirror 編輯器（語法上色、多游標）、匯出 Word（報表產生器已經有 `docx` 套件，可以考慮共用）、目錄 `[TOC]`。
- **PDF 文件屬性**：作者、主旨、關鍵字。Chromium 的 `printToPDF` 不提供，需要另外的 PDF 函式庫改寫中繼資料。
- **列印到印表機**：`ctx.render.print()`，使用系統列印對話框。

## 11. 相容性

| 情況 | 行為 |
|---|---|
| 平台 < 1.7.0 | 沒有 `ctx.render`，也沒有 `window.tooldock.render`、`tooldock://sdk/document.css`。擴充先判斷 `if (ctx.render)`；Markdown 編輯器、報表產生器隱藏 PDF 選項 |
| Markdown 編輯器在平台 1.4.0～1.6.x | 編輯、預覽、HTML 匯出、資料交換都能使用，只是 PDF 項目停用（提示需要 1.7.0）；預覽本來就使用擴充自帶的 `document.css` 複本 |
| 報表產生器 1.2.0 以上在平台 1.4.0～1.6.x | Markdown 資料來源可以使用；1.3.0 的 PDF 匯出隱藏 |
| 沒有安裝報表產生器 | 「用報表產生器開啟表格」依資料交換的規則處理：存成 `.tdx` 交換檔，或提示到商店尋找（docs/EXCHANGE.md） |
| 平台 < 1.7.0 的 `imports[].mimes` | 被忽略，所有接收 `tooldock.text` 的工具都出現在選擇器（〈7.7〉） |
| 系統沒有任何中文字型（極少數 Linux） | 中文會顯示成方塊。文件樣式的字型清單包含 Noto Sans CJK；輸出面板在偵測不到中文字型時提示安裝 `fonts-noto-cjk` |

## 12. 實作計畫

### 12.1 修改位置

| 位置 | 內容 |
|---|---|
| `src/main/render.js`（新增） | 參數驗證、排隊（全平台 2 個、每個擴充 5 個排隊中）、暫存 HTML 準備、轉換視窗與 `render` session、資源規則（〈5.2〉）、完成訊號、頁首頁尾樣板、`printToPDF`、先寫暫存檔再改名、計算頁數、取消與逾時、啟動時清除 `render-temp` |
| `src/main/extension-manager.js` | `services(ext)` 新增 `render.pdf`（以 `track()` 計入執行中）；擴充停用時取消它的轉換。`readManifest` 解析 `imports[].mimes`（非 `tooldock.text` 時忽略並警告）；找接收端時套用 mime 三級比對；`exchange.receivers` 支援 `{ mime }` |
| `src/main/protocol.js` | 把 `tooldock://`、`tbx-ext://` 也註冊到 `render` session |
| `src/main/main.js` | 平台結束時取消所有轉換 |
| `src/host/host-runtime.js` | `ctx.render.pdf`（RPC 不設逾時，由平台的 `timeout` 控制） |
| `src/sdk/ui-sdk.js` | `window.tooldock.render.pdf` |
| `src/sdk/document.css`（新增） | 〈6〉 |
| `src/renderer/workbench.js` | 狀態列「📄 產生 PDF 中」與取消 |
| `src/main/selftest-render.js`（新增） | 〈12.3〉 |
| `packages/types` | `RenderApi`、`RenderPdfOptions`、`RenderResult`；`ExtensionContext.render?`；`ExchangeImportContribution.mimes`、`ReceiverInfo.mimes`、`receivers()` 的 `options` |
| `packages/cli` | `validate` 檢查 `imports[].mimes` 的格式（`type/subtype` 或 `type/*`） |
| `docs/EXCHANGE.md` | 已加入 `mimes` 的規則（〈7.1〉、〈8.1〉、〈12〉） |
| `extensions/markdown-editor`（新增） | 〈7〉 |
| `extensions/table-report`（1.10.0 起移到 tooldock-ext-table-report） | 1.2.0：〈8.1〉（已完成，含 `npm test`）；1.3.0：〈8.2〉 |
| `docs/EXTENSION_GUIDE.md` | 新增「PDF 輸出」章節，連到本文件 |
| `README.md`、`package.json` | 平台 1.7.0、功能說明 |

### 12.2 里程碑

1. **報表產生器 Markdown 資料來源**（〈8.1〉，1.2.0，✅ 已完成）：不依賴平台更新，最快讓 md → Excel 可以使用。
2. **平台 1.7.0**（✅ 已完成）：`ctx.render.pdf`（`render.js`、資源規則、文件樣式）、資料交換的 `mimes` 篩選、自我測試。
3. **Markdown 編輯器：編輯與預覽**（✅ 已完成，1.0.0）：工具列、快捷鍵、捲動同步、草稿、HTML 匯出、資料交換。
4. **Markdown 編輯器：PDF 匯出**（✅ 已完成，1.0.0）：接上 `ctx.render.pdf`，完成〈1〉的情境。
5. **報表產生器 PDF 匯出**（〈8.2〉，1.3.0）。

### 12.3 自我測試新增項目

**平台**

- 片段和完整 HTML 都能輸出；`file` 模式的相對圖片以檔案所在資料夾解析。
- `output` 不是 `.pdf`、資料夾不存在、`html` 和 `file` 同時給或都沒給、超過 10 MB、紙張尺寸或邊界超出範圍時丟出 `RENDER_INVALID`。
- 資源規則：`baseDir` 內的圖片載入；`baseDir` 外、`../`、指向外面的符號連結、其他擴充的 `tbx-ext://`、`tooldock://app/` 被擋下並出現在 `blocked`；`allowRemote: false` 時本機測試伺服器收不到任何請求。
- 預設不執行 JavaScript：頁面中的 `<script>` 改不到 DOM。`javascript: true` 搭配 `TOOLDOCK_RENDER_WAIT`、`TOOLDOCK_RENDER_READY` 時等到完成訊號；完成訊號沒來時仍然輸出，並寫入 `warnings`。
- 頁首頁尾：`{page}`、`{pages}` 正確；文字中的 `<` 被跳脫；邊界太小時自動加大並寫入 `warnings`。
- `outline: true` 的 PDF 有書籤；`page.ranges` 只輸出指定頁；`landscape` 的頁面寬大於高。
- 逾時回報 `timeout`、取消回報 `cancelled`，兩者都不留下 `output`，暫存資料夾被刪除。
- 同時 3 個轉換時只有 2 個在執行；同一個擴充第 6 個排隊的請求回報 `busy`。
- 轉換期間呼叫端不被自動回收；呼叫端被停用時它的轉換回報 `cancelled`。
- 擴充畫面呼叫 `tooldock.render.pdf` 時以該畫面的擴充為呼叫者（只能讀自己的 `tbx-ext://`）。
- 轉換頁面沒有 `window.tooldock`，`window.open` 和導覽都被擋下。
- mime 比對：完全相同 > `type/*` > 沒有宣告；只保留最高一級；有宣告但不符合的被排除；`text/markdown; charset=utf-8` 與大小寫不同的 mime 視為相同；`preferReceiver` 指定較低一級的接收端時仍然交給它；從 `.tdx` 檔匯入套用同樣規則；非 `tooldock.text` 的 `mimes` 被忽略並記錄警告；`receivers(type, 1, { mime })` 的結果與實際交付一致。

**擴充**（內建擴充目前沒有測試，兩個擴充各自新增 `npm test`）

- Markdown 編輯器：工具列每個按鈕在「有選取」、「沒有選取」、「已有語法（切換移除）」三種情況的結果；清單的 Enter 延續與結束；`<!-- pagebreak -->` 輸出分頁、其他 HTML 被跳脫；CRLF 和 BOM 存檔後保留；表格抽取與報表產生器的來源結果一致。
- 報表產生器：Markdown 來源的表格命名、空白與重複表頭、行內語法去除、跳脫的 `\|`、數字推斷；PDF 匯出的截斷註記。

## 13. 決定紀錄

| # | 項目 | 決定 | 狀態 |
|---|---|---|---|
| D1 | PDF 的產生方式 | 平台提供 `ctx.render.pdf`，用隱藏視窗和 Chromium 的 `printToPDF`；不使用 pdfkit、pdfmake（中文字型太大、排版要自己處理） | 定案 |
| D2 | Markdown 解析放在哪裡 | 由擴充自己解析，平台只處理 HTML，保持平台通用 | 定案 |
| D3 | 是否經過執行鎖 | 不經過：只寫本機檔案，沒有對外的副作用（〈9〉） | 定案 |
| D4 | Markdown 編輯器如何輸出 Excel | 不自帶 `exceljs`，透過 `tooldock.table` 交給報表產生器，避免兩份匯出程式碼；報表產生器也能直接開 `.md`（〈8.1〉） | 定案 |
| D5 | 編輯區元件 | v1 使用 `<textarea>`。CodeMirror 6 需要打包成單一檔案，擴充目前沒有打包流程，列在後續版本 | 定案 |
| D6 | 預覽是否跟隨深色主題 | 不跟隨，固定白底紙張，預覽就是印出來的樣子 | 定案 |
| D7 | 分享文字時的選擇器 | 1.7.0 讓 `imports[]` 新增 `mimes`，依「完全相同 > `type/*` > 沒有宣告」三級比對，只保留最高一級；Markdown 編輯器宣告、資料轉換不宣告（〈7.7〉、docs/EXCHANGE.md〈8.1〉） | 定案 |
| D8 | 預設是否載入網路資源 | PDF 預設不載入，避免追蹤；Markdown 編輯器的預覽照常顯示網路圖片 | 定案 |
| D9 | 分頁語法 | 獨立一行 `<!-- pagebreak -->`：在其他 Markdown 檢視器中看不見 | 定案 |
| D10 | `pdf()` 的錯誤處理 | 只有參數不合法時丟出例外，其他情況以 `status` 回報（與 `exchange.share`、`notify.send` 一致） | 定案 |
| D11 | 程式碼上色 | 使用 highlight.js，只帶常用語言 | 定案 |
