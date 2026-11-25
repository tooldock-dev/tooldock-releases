# ToolDock 擴充開發指南

## 1. 建立擴充

外部開發者不需要平台原始碼，只要準備：
- 已安裝的 ToolDock
- Node.js 18 以上

```bash
npm create tooldock-extension@latest my-tool
```

```bash
cd my-tool
```

```bash
npm run dev
```

建立時會詢問 id、顯示名稱和範本：

| 範本 | 適用 |
|---|---|
| `view` | 有分頁畫面，前端呼叫後端處理資料（大多數工具） |
| `service` | 隨平台啟動的背景服務：定時執行、狀態列顯示 |

不想互動時可以直接帶參數：`--id my-tool --name "我的工具" --template view -y`；加上 `--no-install` 則不自動執行 `npm install`。

產生的專案已包含：

```
my-tool/
├── package.json        manifest（tooldock 欄位）與 dev / debug / validate / pack 腳本
├── backend/index.js    後端：activate(ctx) / deactivate()（JSDoc 型別，不需編譯）
├── ui/                 畫面（view 範本）
├── jsconfig.json       型別檢查設定（tooldock-types）
├── .vscode/            F5 啟動並偵錯
├── .tbxignore          打包時排除的檔案
└── README.md
```

| 開發工具套件 | 用途 |
|---|---|
| `create-tooldock-extension` | 專案產生器 |
| `tooldock-cli`（指令 `tooldock-ext`） | `dev`、`validate`、`pack`、`install`，沒有任何相依套件 |
| `tooldock-types` | 後端 `ctx`、畫面 `window.tooldock` 與 manifest 的型別定義 |

> 平台開發者在原始碼中也可以用 `npm run new-ext -- my-tool`，產生的專案會以 `file:` 連結本機的 `packages/`。

## 2. Manifest（package.json）

```jsonc
{
  "name": "my-tool",
  "version": "1.0.0",
  "main": "backend/index.js",            // 可省略：純前端擴充
  "tooldock": {
    "id": "my-tool",                      // 小寫英數、. 與 -
    "displayName": "我的工具",
    "description": "…",
    "icon": "🧩",                         // emoji，或相對路徑的 svg / png
    "engine": "^1.0.0",                   // 平台版本需求
    "activationEvents": ["onStartup"],    // 可省略：預設延遲載入
    "restartOnCrash": false,
    "keepAlive": false,                   // true = 不自動回收（1.3.0 起，見「自動回收」）
    "activationTimeout": 20000,
    "contributes": {
      "views": [{ "id": "myTool.main", "title": "我的工具", "icon": "🧩", "entry": "ui/index.html", "pinned": true }],
      "commands": [{ "id": "myTool.run", "title": "我的工具：執行" }],
      "configuration": {
        "myTool.endpoint": { "type": "string", "default": "", "title": "API 位址", "description": "…" },
        "myTool.mode": { "type": "string", "enum": ["fast", "safe"], "default": "safe" },
        "myTool.enabled": { "type": "boolean", "default": true }
      },
      "lifecycle": { "app:before-quit": "onBeforeQuit", "custom:month-end": "onMonthEnd" }
    }
  },
  "dependencies": { "dayjs": "^1.11.0" }
}
```

### 啟用事件

| 事件 | 時機 |
|---|---|
| `onStartup`／`*` | 進入 `app:ready` 時（常駐型工具） |
| `onView:<viewId>` | 開啟畫面時（宣告 views 後自動加入） |
| `onCommand:<cmd>` | 執行指令時（宣告 commands 後自動加入） |
| `onPhase:<phase>` | 進入任一生命週期階段時，例如 `onPhase:app:idle`、`onPhase:custom:month-end` |
| `onImport:<type>` | 收到資料交換包時（宣告 exchange.imports 後自動加入） |
| `onNotify:<channel>` | 通知服務需要這個通道時（宣告 notify.channels 後自動加入，1.6.0 起） |
| （隱含） | 畫面呼叫 `tooldock.invoke()` 時，如果擴充還沒啟動，會自動啟動 |

### 自動回收

ToolDock 1.3.0 起，沒有宣告 `onStartup`／`*` 的擴充屬於延遲載入，用完後平台會自動結束它的程序，避免裝了很多工具後程序越積越多：

- **關閉分頁**：擴充的最後一個畫面關閉時，立即結束程序。
- **閒置回收**：擴充沒有開著的畫面，而且超過 5 分鐘沒有被使用（指令、前端呼叫、生命週期階段）時，結束程序。

正在處理請求時不會被回收。回收後只要再開啟畫面、執行指令或進入宣告的階段，就會依啟用事件自動重新啟動。回收等同停用：會呼叫 `deactivate()`，記憶體中的狀態、計時器與 `ctx.paths.temp` 都會清掉，需要保留的資料請放在 `ctx.storage` 或 `ctx.paths.data`。

以下擴充不會被自動回收：宣告 `onStartup`／`*` 的常駐擴充、manifest 設定 `"keepAlive": true` 的擴充、以 `--extension-dev-path` 載入的開發中擴充。

使用者可以在 `lifecycle.json` 的 `behavior` 調整：`deactivateOnViewClose`（預設 `true`）與 `extensionIdleTimeout`（毫秒，預設 `300000`；設為 `0` 關閉閒置回收）。

### 背景分頁卸載（ToolDock 1.10.0 起）

每個擴充畫面都跑在自己的 renderer 行程裡（每個 `tbx-ext://<id>` 是不同的 origin），一個畫面大約佔 80–100 MB，而且分頁切到背景後也不會釋放。畫面可以在 manifest 宣告允許平台卸載它：

```jsonc
"views": [{ "id": "myTool.main", "title": "我的工具", "discardWhenHidden": true }]
```

- 分頁在背景超過 `behavior.viewDiscardTimeout`（毫秒，預設 `600000` 即 10 分鐘；設為 `0` 關閉這個功能）後，平台會移除畫面的 iframe，分頁標題改成斜體。
- 切換回這個分頁時，畫面會**重新載入**，`td.view.restored` 為 `true`。
- 分頁仍然算開著：**後端不會因此被回收**，`ctx.ui.emit` 送出的事件在卸載期間會被丟棄，畫面重新載入時請向後端重新取得資料。
- 預設是 `false`，因為畫面的記憶體狀態（表單、捲動位置、未儲存的內容）會在卸載時消失。只在畫面能從後端或 `td.view.getState()` 還原時才開啟。

畫面端可以用 `window.tooldock.view` 配合（舊版平台沒有這個屬性，請先檢查）：

```js
const td = window.tooldock;
if (td.view) {
  if (td.view.restored) restore(await td.view.getState());     // 切回來時還原
  td.view.setState({ tab: 'history', scrollTop: 120 });         // 只存在這個分頁的生命期內，上限約 1 MB
  td.view.keepAlive(true);   // 有尚未儲存的內容、進行中的工作時暫時不要卸載
  td.view.keepAlive(false);  // 存好之後解除
}
```

## 3. 後端 API（`ctx`）

後端跑在獨立的 Node.js 程序，可以使用 Node 全部能力（`fs`、`child_process`、`fetch`、資料庫驅動…），也能 `require` 任何 npm 套件。

```js
exports.activate = async (ctx) => {
  ctx.extension          // { id, path, version, manifest }
  ctx.paths              // { extension, data（持久資料夾）, temp（停用時刪除） }
  ctx.log.info('…')      // 顯示在輸出面板與擴充詳細頁

  // 持久化儲存（JSON）
  ctx.storage.get('key', defaultValue); ctx.storage.set('key', value);

  // 設定（manifest configuration + 使用者修改）
  ctx.settings.get('myTool.endpoint');
  ctx.settings.onDidChange((key, value) => {});

  // 畫面 ⇄ 後端
  ctx.ui.handle('query', async (a, b) => result);   // 畫面：await tooldock.invoke('query', a, b)
  ctx.ui.emit('progress', { pct: 50 });            // 畫面：tooldock.on('progress', fn)

  // 指令（命令面板、lifecycle hook、其他擴充都可呼叫）
  ctx.commands.register('myTool.run', async (...args) => {});
  await ctx.commands.execute('siteMonitor.checkAll');

  // 生命週期
  ctx.lifecycle.on('app:idle', async (payload) => {});
  await ctx.lifecycle.enter('custom:my-phase', { any: 'data' });

  // 機密資料（ToolDock 1.2.0 起）：密碼、權杖以作業系統鑰匙圈保護的金鑰加密，磁碟上只有密文
  // 每個擴充只能存取自己的項目，移除擴充時一併清除。舊版平台沒有 ctx.secrets，請先判斷
  if (ctx.secrets && await ctx.secrets.available()) {
    await ctx.secrets.set('token', 'xxx');
    const token = await ctx.secrets.get('token'); // 不存在時為 null
    await ctx.secrets.delete('token');
  }

  // 平台 UI 服務
  await ctx.window.showOpenDialog({ properties: ['openFile'] });
  await ctx.window.showSaveDialog({ defaultPath: 'out.xlsx' });
  await ctx.window.showMessageBox({ message: '確定？', buttons: ['是', '否'] });
  await ctx.window.notify('標題', '桌面通知內容');
  await ctx.window.toast('工作台提示', 'success');   // info | success | warn | error
  await ctx.window.openView('myTool.main');
  await ctx.window.setStatus({ id: 'x', text: '🧩 3', tooltip: '…', command: 'myTool.run' }); // text 空字串＝移除
  await ctx.shell.openPath(file); await ctx.shell.showItemInFolder(file); await ctx.shell.openExternal(url);

  // 停用時自動清理
  const t = setInterval(() => {}, 1000);
  ctx.subscriptions.push({ dispose: () => clearInterval(t) });
};

exports.deactivate = async (ctx) => {};          // 最多等 3 秒，之後強制結束程序
exports.onBeforeQuit = async (ctx, payload) => {}; // 對應 contributes.lifecycle
```

### 資料交換（ToolDock 1.4.0 起）

把資料交給其他工具，而不需要知道由誰接收。產生端只決定「格式」，平台負責找出接收端；沒有接收端時可以存成 `.tdx` 交換檔，日後再匯入。完整規格見 [EXCHANGE.md](EXCHANGE.md)。

**產生端**

```js
if (ctx.exchange) {   // 舊版平台沒有 ctx.exchange
  const pkg = ctx.exchange.create('tooldock.table', { columns, rows }, { title: '11 月加班時數' });
  const r = await ctx.exchange.share(pkg);          // { fallback: 'none' } 表示沒有接收端時不詢問存檔
  // r.status：delivered（r.result 是接收端的 ImportResult）| exported | no-receiver | cancelled | error
}
```

**接收端**：在 manifest 宣告，平台會自動加上 `onImport:<格式>` 啟用事件。匯入期間擴充不會被自動回收，也沒有逾時，等使用者確認多久都可以。

```jsonc
"contributes": {
  "exchange": {
    "imports": [{ "type": "tooldock.table", "handler": "importTable", "title": "用我的工具開啟", "view": "myTool.main" }],
    "formats": [{ "type": "my-tool.report", "version": 1, "title": "我的報表", "schema": "exchange/report.v1.schema.json" }]
  }
}
```

```js
exports.importTable = async (ctx, pkg, info) => {
  // info.origin：'share'（info.producer 是實際產生端）或 'file'（info.filePath；pkg.source 不可信）
  info.onCancel(() => { /* 使用者在狀態列取消匯入 */ });
  return { status: 'imported', message: '已載入 3 筆', data: { … } };  // rejected | cancelled
};
```

| 標準格式 | 內容 | 接收端 |
|---|---|---|
| `tooldock.sql` | 要在資料庫執行的 SQL 與參數，包含查詢 | 🛢 Oracle SQL 工具（商店，1.3.0 以上） |
| `tooldock.table` | 表格資料 | 📊 報表產生器、🔁 資料轉換工具（轉成 CSV）（商店擴充） |
| `tooldock.text` | 文字、CSV、JSON、Markdown 等 | 🔁 資料轉換工具；`text/markdown` 交給 📝 Markdown 編輯器（都是商店擴充） |

- 自訂格式必須以自己的擴充 id 開頭（例如 `my-tool.report`），`tooldock.*` 保留給平台；違反時平台忽略並記錄警告，`tooldock-ext validate` 會列為錯誤。
- 標準格式的 TypeScript 型別在 `tooldock-types`（`SqlPayload`、`TablePayload`、`TextPayload`、`ImportHandler`…），JSON Schema 在 `tooldock-types/exchange/*.schema.json`。
- 接收 `tooldock.sql` 的擴充必須做到〈6.8〉的要求：預覽並確認、試跑、語句分類、防止重複匯入、稽核紀錄、參數綁定。可以參考或沿用 [Oracle SQL 工具](https://github.com/tooldock-dev/tooldock-ext-oracle-sql)：`backend/sql-exchange.js`（解析、分類、型別檢查）不依賴資料庫驅動程式，其他資料庫只要替換執行的部分（`backend/exchange-exec.js`）。
- 使用者可以用「檔案 → 匯入交換檔」、命令面板「交換：匯入交換檔 (.tdx)」、把 `.tdx` 拖進視窗，或在檔案總管雙擊 `.tdx`（安裝版有檔案關聯）匯入。
- **只接收特定的文字**（1.7.0 起）：接收 `tooldock.text` 時可以加上 `"mimes": ["text/markdown"]`（也可以寫 `text/*`）。分享時平台只保留最符合的一級接收端：完全相同 > `type/*` > 沒有宣告，所以分享 Markdown 會直接交給宣告 `text/markdown` 的工具，不跳出選擇器。舊平台會忽略 `mimes`，handler 仍要自己處理不符合的內容。`ctx.exchange.receivers(type, version, { mime })` 的結果與實際交付一致。

### 產生 PDF（ToolDock 1.7.0 起）

擴充給 HTML，平台用隱藏視窗排版後輸出 PDF，中文使用系統字型，擴充不需要帶任何 PDF 函式庫。完整規格見 [RENDER.md](RENDER.md)。

```js
if (ctx.render) {   // 舊版平台沒有 ctx.render
  const { canceled, filePath } = await ctx.window.showSaveDialog({ filters: [{ name: 'PDF', extensions: ['pdf'] }] });
  if (!canceled) {
    const r = await ctx.render.pdf({
      html: '<h1>月報</h1><table>…</table>',   // 或 file: '/絕對路徑/報表.html'
      output: filePath,
      baseDir: '/文件所在資料夾',               // 相對路徑的圖片從這裡讀
      stylesheet: 'document',                   // 平台的中文文件樣式（tooldock://sdk/document.css）
      page: { size: 'A4', landscape: false, margin: 15 },          // 公釐
      header: '{title}｜｜{date}', footer: '｜第 {page} 頁 / 共 {pages} 頁｜', // 以 ｜ 分左中右
    });
    // r.status：done（r.pages、r.bytes）| timeout | cancelled | failed | busy；r.blocked 是被擋下的資源
  }
}
```

- 只有參數不合法時丟出例外（`err.code === 'RENDER_INVALID'`），其他情況都用 `status` 回報。
- 轉換頁面**預設不連網、不執行 JavaScript**，只能讀 `baseDir` 底下的檔案和自己的 `tbx-ext://`。需要網路圖片時加 `allowRemote: true`；需要圖表函式庫時加 `javascript: true`，並在頁面設定 `window.TOOLDOCK_RENDER_WAIT = true`，畫完後設 `window.TOOLDOCK_RENDER_READY = true`。
- 轉換期間擴充不會被自動回收；使用者可以從狀態列取消。全平台同時最多 2 個轉換，其餘排隊。
- 畫面的預覽可以載入同一份 `tooldock://sdk/document.css`，把內容放在 `<article class="td-doc">` 裡，預覽看到的就是印出來的樣子。

### 通知與執行鎖（ToolDock 1.6.0 起）

發通知時只給**內容**與**對象**，不需要知道使用者用 Telegram 還是 Mail。對象（例如「維運人員」）由使用者在平台的「執行鎖」畫面設定要送到哪些目的地；實際發送由**通道擴充**負責。完整規格見 [NOTIFY.md](NOTIFY.md)。

```js
if (ctx.notify) {   // 舊版平台沒有 ctx.notify，請退回 ctx.window.notify
  const r = await ctx.notify.send({
    title: '網站異常：官網', body: 'HTTP 502（連續 3 次）', level: 'error',
    to: '維運人員',                 // 對象名稱；省略時用使用者設定的預設路由
    topic: 'site.down', fields: [{ label: '網址', value: url }], url,
    dedupeKey: `down:${site.id}`,   // 5 分鐘內同一個鍵只發一次
  });
  // r.status：sent | partial | failed | denied | timeout | no-destination | throttled | duplicate
  // r.deliveries：每個目的地的結果（destination、status、decision、message）
}
```

**執行鎖**：擴充第一次發到某個目的地時，平台會跳出確認，使用者可以選「允許這一次」、「永遠允許」或「拒絕」（可勾選不再詢問）。**桌面通知 `ctx.window.notify` 也一樣**，被拒絕時回傳 `false`、不丟出例外。`send` 會等到使用者回應才回傳，最久 10 分鐘（逾時回報 `timeout`），不要假設通知一定會送出。建議在設定畫面放一個「測試通知」按鈕，讓使用者安裝後就完成第一次確認。

**用途宣告**：在 manifest 說明會發通知、會呼叫哪些受保護指令。商店安裝確認畫面、擴充詳細頁會顯示；宣告了 `notify` 的擴充裝好後，平台會引導使用者發一則測試通知，先完成第一次確認。宣告只是說明，**不會因此取得授權**。

```jsonc
"tooldock": {
  "permissions": {
    "notify": { "reason": "網站異常或恢復時通知你", "testCommand": "myTool.testNotify" },  // testCommand 選填，必須是自己的指令
    "commands": [{ "id": "telegram.send", "reason": "把日報發到 Telegram" }]
  }
}
```

沒有 `testCommand` 時，平台以你的擴充身分發一則「〈擴充名稱〉：測試通知」到預設路由。如果你的通知會送到特定對象（例如讀取設定的 `to`），請提供 `testCommand`，確認的才會是實際會用到的目的地。

**受保護指令**：其他擴充呼叫會造成外部影響的指令（例如發訊息、寫資料庫）時，在 manifest 標示 `guarded`，第一次會詢問使用者；被拒絕時 `ctx.commands.execute` 丟出 `err.code === 'GUARD_DENIED'`（逾時為 `GUARD_TIMEOUT`）。擴充自己、命令面板執行不受影響。

```jsonc
"commands": [{ "id": "myTool.purge", "title": "我的工具：清除資料", "guarded": true }]
```

```js
// 選用：確認對話框顯示的說明（預設顯示參數 JSON）
exports.guardSummary = (ctx, command, args) => command === 'myTool.purge' ? `清除 ${args[0]} 天前的資料` : null;
```

> 平台只看**直接呼叫者**：A 執行你的一般指令，你在指令裡呼叫 `ctx.notify.send`，呼叫者是你，用的是你的授權。所以替別人「代發」通知或代為執行動作的指令，一定要標示 `guarded`，否則等於把你的授權借給所有擴充。

**通道擴充**：提供新的發送方式（例如 Mail），只要宣告通道並匯出兩個函式，呼叫端不用改：

```jsonc
"contributes": {
  "notify": {
    "channels": [{ "id": "my-mail", "title": "Mail", "icon": "✉️", "destinations": "listDestinations", "deliver": "deliver", "supports": { "attachments": true } }]
  }
}
```

```js
exports.listDestinations = async (ctx) => [{ id: 'ops', title: 'ops@example.com', isDefault: true }]; // id 要穩定，不能用會改的名稱
exports.deliver = async (ctx, n, dest, info) => {
  // n：title、body、level、fields、url、attachments（Uint8Array 或文字）、silent；內容是純文字，排版與跳脫由通道負責
  // info.caller：呼叫端（已經過使用者確認）；60 秒內要回應，重試由通道自己處理
  return { status: 'sent', ref: { messageId } };   // 或 { status: 'failed', message: '中文原因' }
};
```

通道 id 必須是自己的擴充 id 或以「擴充id.」開頭，`desktop` 保留給平台；平台會自動加上 `onNotify:<通道 id>` 啟用事件。

## 4. 畫面 API（`window.tooldock`）

```html
<link rel="stylesheet" href="tooldock://sdk/ui-base.css">   <!-- 與平台一致的深/淺色主題 -->
<script src="tooldock://sdk/ui-sdk.js"></script>
<script>
  const td = window.tooldock;
  await td.invoke('query', 1, 2);
  const off = td.on('progress', (d) => {});
  await td.dialog.open({...}); await td.dialog.save({...}); await td.dialog.message({...});
  td.toast('完成', 'success'); td.notify('標題', '內容');   // 1.6.0 起桌面通知經過執行鎖
  if (td.notify.send) await td.notify.send({ title: '匯出完成', to: '我' });   // 通知服務（1.6.0 起）
  td.openView('otherView'); td.executeCommand('myTool.run');
  td.shell.openPath(p); td.shell.openExternal(url);
  await td.settings.get(); await td.settings.update('myTool.mode', 'fast');
  if (td.render) await td.render.pdf({ html, output: filePath, stylesheet: 'document' });   // 1.7.0 起
  if (td.exchange) {   // ToolDock 1.4.0 起；畫面端的 create 回傳 Promise
    const pkg = await td.exchange.create('tooldock.text', { mime: 'text/csv', text }, { title: '匯出結果' });
    const r = await td.exchange.share(pkg);
  }
  if (td.view) {   // 背景分頁卸載（1.10.0 起，見〈2. Manifest〉的「背景分頁卸載」）
    if (td.view.restored) restore(await td.view.getState());
    td.view.setState(state); td.view.keepAlive(true);
  }
  td.viewId; td.extensionId; td.escape(html);
</script>
```

畫面可以用任何前端框架（React、Vue…），把打包後的檔案放進擴充即可，以 `tbx-ext://<id>/` 為根目錄載入。`localStorage` 依擴充隔離。

`ui-base.css` 提供 CSS 變數（`--bg`、`--fg`、`--accent`、`--ok`、`--err`…）和這些基本元件樣式：`button.secondary/ghost/danger`、`.card`、`.badge.ok/warn/err`、`table.grid`、`.row`、`.stack`。

## 5. 函式庫

| 方式 | 做法 | 適用 |
|---|---|---|
| 擴充自帶 | 在擴充資料夾執行 `npm install xxx`，打包時一起帶走 | 大部分情況，版本獨立互不干擾 |
| 平台共用 | 在 `<userData>/libs` 執行 `npm install xxx`（擴充側欄有「開啟共用 lib」按鈕） | 多個擴充共用的大型套件、公司內部共用模組 |

解析順序：先找擴充自己的 `node_modules`，找不到再找共用 lib 目錄。

> 原生模組（例如 `better-sqlite3`）需要依 Electron 版本重新編譯。建議改用 WebAssembly 或純 JS 的替代套件（例如本專案用 `sql.js`），換 Electron 版本時就不必重新編譯。

### 範例：讓報表產生器支援 MySQL

報表產生器的原始碼在 [tooldock-ext-table-report](https://github.com/tooldock-dev/tooldock-ext-table-report)（1.10.0 以前內建在平台）：

```bash
git clone https://github.com/tooldock-dev/tooldock-ext-table-report && cd tooldock-ext-table-report
npm install && npm install mysql2
```

接著新增 `backend/sources/mysql.js`，實作 `{ id, label, extensions, supportsQuery, listTables, read }`，再把它加進 `backend/sources/index.js` 的陣列，用 `npm run dev` 試用。平台本身不需要任何改動。

## 6. 打包與發佈

```bash
npm run pack
```

打包前會先執行 `validate`，再產生 `dist/<id>-<版本>.tbx`。套件只包含 `dependencies` 的完整相依樹；`devDependencies`、`.git`、`.vscode`、source map，以及 `.tbxignore` 列出的檔案都會排除。

```bash
npm run install-local
```

這個指令會打包並安裝到本機的 ToolDock。如果 ToolDock 正在執行，安裝會轉交給執行中的視窗，完成後顯示提示。

使用者可以用以下任一方式安裝：

- 選單「檔案 → 從套件安裝擴充」，或擴充側欄的 📦。
- 把解壓縮後的資料夾放進 `<userData>/extensions/`，平台會自動偵測。
- 用命令列安裝，適合 IT 大量部署（見第 8 節）。

如果安裝的套件與內建擴充同 id，會覆蓋內建版本；之後移除這個套件，就會自動還原內建版本。

### 上架到商店

範本內建 `.github/workflows/release.yml`。設定一次就能自動發佈：

1. 執行 `tooldock-ext keygen` 產生簽章金鑰。
2. 把私鑰設為 GitHub secret `TOOLDOCK_SIGNING_KEY`。
3. 之後每次推送 `v<版本>` tag，GitHub Actions 就會自動驗證、打包、簽章，並建立 Release。

上架商店時，對索引 repo 發 PR，登記擴充的 repo 和公鑰即可。完整步驟見 [RELEASE.md](RELEASE.md)；商店架構與企業私有商店的做法見 [STORE.md](STORE.md)。

## 7. 除錯

`npm run dev` 會以 `--extension-dev-path` 啟動 ToolDock，有以下特性：

- **獨立資料目錄**（`ToolDock-ExtDev`）：可以和正式使用的 ToolDock 同時開啟。加上 `-- --fresh` 則每次使用全新的暫存目錄。
- **開發版優先**：開發中的擴充優先於已安裝或內建的同 id 版本，並在平台啟動時就啟用，錯誤會馬上出現。
- **存檔即重載**：存檔後自動重新載入後端程序和已開啟的畫面。
- **狀態列標示**：狀態列變成橘色，顯示「🛠 擴充開發」。

| 對象 | 方法 |
|---|---|
| 後端 | VS Code 按 F5 選「ToolDock：啟動並偵錯擴充」，可設中斷點；重新載入後偵錯埠不變，除錯器會自動重連。也可以執行 `npm run debug`，再到 Chrome 開啟 `chrome://inspect` |
| `activate()` 本身 | `npm run dev -- --inspect-brk`，停在第一行等待除錯器 |
| 畫面 | 在分頁上按右鍵 →「檢查此畫面」，開發者工具會直接定位到該擴充的 iframe |
| 日誌 | 輸出面板（`⇧⌘U`）依擴充分頻道顯示；執行 `npm run dev` 的終端機也會同步輸出 |
| 狀態 | 擴充詳細頁顯示 PID、啟用原因、偵錯埠，並有停止、啟動、重新載入按鈕 |

找不到 ToolDock 時，用 `--app <路徑>` 或環境變數 `TOOLDOCK_PATH` 指定。可以指向 `.app`、執行檔，或平台原始碼目錄。

## 8. 平台命令列參數

| 參數 | 說明 |
|---|---|
| `--extension-dev-path=<資料夾>` | 以開發模式載入擴充（可重複指定） |
| `--inspect-extensions[=port]` | 擴充後端開啟 Node inspector（預設 9229，每個擴充依序加 1） |
| `--inspect-brk-extensions[=port]` | 同上，並暫停在第一行 |
| `--install-extension=<.tbx 或資料夾>` | 安裝後結束，不開視窗（可重複指定）；平台執行中時轉交給執行中的視窗 |
| `--uninstall-extension=<id>` | 移除後結束 |
| `--list-extensions` | 列出擴充後結束 |
| `--user-data-dir=<資料夾>` | 指定資料目錄 |
| `--dev` | 所有擴充都存檔即重載（平台開發用） |

命令列模式成功時結束碼為 0，任何一項失敗則為 1，可以直接寫進部署腳本。例如在 macOS 上：

```bash
/Applications/ToolDock.app/Contents/MacOS/ToolDock --install-extension=./my-tool-1.0.0.tbx
```
