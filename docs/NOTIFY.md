# 通知服務（Notify）與執行鎖（Guard）規格書

> 狀態：**已實作**（平台 **1.6.0**）。本文件定義擴充之間「發送通知」的呼叫方式、通道擴充的介面，以及跨擴充呼叫敏感動作時的使用者授權機制。實作以本文件為準，若需變更請先修改本文件。

## 1. 目的與範圍

導入 Telegram 通知（`tooldock-ext-telegram` 1.0.0）之後，出現兩個問題：

1. **任何已安裝的擴充都能透過 `telegram.send` 直接發訊息，而且不經確認。** 使用者裝了一個工具，不會知道它會不會把資料送到外部聊天室。
2. **呼叫端綁死了目的地。** 網站監控如果要發 Telegram，就得寫 `ctx.commands.execute('telegram.send', …)`；之後加了 Mail，又要再改一次網站監控。

本規格提供兩個平台內建功能：

- **執行鎖（Guard）**：跨擴充呼叫敏感動作時，**第一次一定要使用者確認**。使用者可以選「允許這一次」、「永遠允許」或「拒絕」，決定會記錄下來。之後可以在平台內建的「執行鎖」畫面**撤銷授權**，並**查詢每一次呼叫的執行紀錄**（〈9〉）。
- **通知服務（Notify）**：呼叫端只說「發什麼內容、給誰」，不需要知道對方用 Telegram 還是 Mail。實際的發送由**通道擴充**（Telegram、未來的 Mail…）負責，處理完把每個目的地的結果回傳給呼叫端。

第一個要支援的情境：**網站監控發現網站異常 → 發通知給「維運人員」 → 第一次詢問使用者 → 送到 Telegram 維運群組 → 網站監控收到「已送達」的回應。**

不在範圍內：

- 防止惡意擴充。平台沒有沙箱（見〈10. 安全性〉），執行鎖防的是**未經使用者同意的自動化**與**誤用**。
- 收件人的「已讀／已處理」回報。v1 只回報「送達通道」的結果，人員確認回覆留到〈11. 後續版本〉。

## 2. 名詞

| 名詞 | 說明 |
|---|---|
| **呼叫端**（caller） | 呼叫 `ctx.notify.send()` 或執行受保護指令的擴充，例如網站監控 |
| **通道**（channel） | 實際發送的方式，由擴充在 manifest 宣告，例如 `telegram`、`mail`。平台內建 `desktop`（桌面通知） |
| **通道擴充**（channel provider） | 提供通道的擴充，例如 Telegram 通知 |
| **目的地**（destination） | 通道裡的一個收件位置，以 `通道/位址 id` 表示，例如 `telegram/chat-7f3a`（維運群組）、`mail/ops`。位址 id 由通道擴充決定，必須穩定，不能用會被改名的顯示名稱 |
| **對象**（audience） | 使用者在平台定義的收件群組，例如「維運人員」= `telegram/chat-7f3a` + `mail/ops`。呼叫端用對象名稱指派，不用知道通道 |
| **授權**（grant） | 使用者對「某個呼叫端 × 某個目標」做的決定：允許或拒絕 |
| **執行鎖**（guard） | 平台在交付前檢查授權、必要時詢問使用者的機制。通知與受保護指令共用 |

## 3. 架構

```
 呼叫端擴充                   平台（ExtensionManager + Notify + Guard）                     通道擴充
 ──────────                   ─────────────────────────────────────────                     ────────
 ctx.notify.send(n) ─RPC─▶  1. 驗證通知內容、節流、去重
                            2. 解析指派對象 to → 目的地清單
                               （沒有指定 → 呼叫端的預設路由 → 全域預設對象）
                            3. 執行鎖：逐一檢查 (呼叫端, 目的地) 的授權
                               ├─ 已允許 ─────────────────────────────┐
                               ├─ 已拒絕 → 該目的地 denied             │
                               └─ 沒有紀錄 → 授權對話框（合併成一個）   │
                                    允許一次 / 永遠允許 / 拒絕          │
                            4. 依通道分組、平行交付  ◀────────────────┘
                                    ├──────────────▶ 啟用 onNotify:telegram ──▶ deliver(ctx, n, dest)
                                    └──────────────▶ 啟用 onNotify:mail     ──▶ deliver(ctx, n, dest)
                            5. 彙整 NotifyResult、寫入執行紀錄  ◀──────────── DeliveryResult
 ◀──────────────────────────── NotifyResult
```

受保護指令（〈7.2〉）走同一個執行鎖，只是第 2、4 步換成「找出指令擁有者、交付 `command.execute`」。每一次經過執行鎖的呼叫（不論允許、拒絕、成功或失敗）都寫入**執行紀錄**（〈9.3〉）。

## 4. 執行鎖（Guard）

### 4.1 何時需要確認

| 呼叫來源 | 是否經過執行鎖 |
|---|---|
| 擴充 A 呼叫 `ctx.notify.send()`（任何目的地，**包含內建的 `desktop`**） | **是** |
| 擴充 A 呼叫 `ctx.window.notify()`（桌面通知） | **是**，視為發到 `desktop/local`，與上一列共用授權（〈7.1〉） |
| 擴充 A 執行擴充 B 宣告為 `guarded` 的指令 | **是** |
| 擴充 B 自己（後端、畫面、lifecycle hook）執行自己的指令 | 否 |
| 使用者從命令面板執行指令 | 否（使用者本人就是授權） |
| 平台核心指令（`workbench.*`、`store.*`…） | 否 |
| `ctx.window.toast()`（工作台內的提示） | 否（只在 ToolDock 視窗內顯示） |

判斷依據是**平台記錄的呼叫者身分**，不是呼叫端自己傳入的參數，所以呼叫端無法冒用別的擴充。

### 4.2 呼叫者身分的來源

| 來源 | 程式位置 | 呼叫者 | 經過執行鎖 |
|---|---|---|---|
| 擴充後端 `ctx.*` | `ExtensionManager.services(ext)`，每個擴充程序各自一組 RPC 服務 | 該擴充 | 依〈4.1〉 |
| 擴充畫面 `window.tooldock.*` | `workbench.js` 依**送出訊息的 iframe 所屬分頁**取得 `tab.extId`，再經 `platform.call` 進入同一組 `services(ext)` | 該擴充 | 依〈4.1〉 |
| 命令面板、選單、工作台按鈕 | `main.js` router 的 `commands.execute`、`buildMenu()` | 使用者 | 否 |
| 生命週期設定檔 hook（`action: command` / `notify`） | `main.js` 的 `setHookRunner` | 使用者（設定檔由使用者撰寫） | 否 |
| 平台核心（商店、自我測試） | 主程序內部 | 平台 | 否 |

實作規則：

1. `executeCommand(command, args, { caller })` 新增第三個參數。`services(ext)` 一律傳 `{ caller: ext.id }`；其他主程序內的呼叫傳 `{ caller: null }` 表示使用者／平台。
2. **擴充不能透過任何參數改變 `caller`**。`platform.call` 的 `extId` 由工作台依 iframe 決定，畫面送來的訊息內容不採用。
3. **只看直接呼叫者**：A 執行 B 的一般指令，B 在指令裡呼叫 `ctx.notify.send()`，呼叫者是 B，不是 A。因此提供「代發」功能的擴充（把別人的請求轉成通知）必須自己把指令標示為 `guarded`，否則等於把自己的授權借給所有擴充。這點寫進 `EXTENSION_GUIDE.md`。

### 4.3 授權的單位

授權以 **(呼叫端擴充 id, 種類, 目標)** 為單位：

| 種類 | 目標 | 例 |
|---|---|---|
| `notify` | 目的地 `通道/位址 id` | `site-monitor` × `notify` × `telegram/chat-7f3a` |
| `command` | 指令 id | `report-bot` × `command` × `telegram.send` |

- 同一個呼叫端改送到**新的目的地**（例如另一個 Telegram 群組）時，要再確認一次。
- 授權**不綁擴充版本**：擴充更新後沿用。擴充**移除**時刪除它的所有授權；重新安裝要重新確認。
- 授權**不綁通知內容**：使用者核准的是「網站監控可以發到維運群組」，不是某一則訊息。

### 4.4 授權對話框

```
┌──────────────────────────────────────────────────────────────┐
│ 📡 網站監控 想發送通知                                        │
│                                                              │
│  標題：網站異常                                               │
│  內容：❌ 官網 異常：HTTP 502（連續 3 次）                     │
│                                                              │
│  發送到：                                                     │
│   ☑ ✈️ Telegram ─ 維運群組（群組）                           │
│   ☑ ✉️ Mail ─ ops@example.com                                │
│                                                              │
│  另有 2 則相同來源的通知等待中                                 │
│                                                              │
│  ☐ 拒絕後不再詢問                                             │
│                                                              │
│        [ 拒絕 ]     [ 允許這一次 ]     [ 永遠允許 ]           │
└──────────────────────────────────────────────────────────────┘
```

| 選擇 | 這一次 | 之後 |
|---|---|---|
| **允許這一次** | 發送到勾選的目的地 | 不記錄，下次同一組合仍會詢問 |
| **永遠允許** | 發送到勾選的目的地 | 記錄 `allow`，之後直接發送 |
| **拒絕** | 不發送 | 不記錄，下次仍會詢問 |
| **拒絕** + 勾選「不再詢問」 | 不發送 | 記錄 `deny`，之後直接回報 `denied`，不再跳出對話框 |
| 關閉對話框、按 Esc | 等同「拒絕」（不記錄） | — |
| 逾時（預設 10 分鐘） | 回報 `timeout` | 不記錄 |

規則：

1. **必須顯示實際內容**（標題、內容前 500 字、附件檔名），不能只顯示呼叫端名稱。使用者要知道「發的是什麼」才能判斷。
2. **取消勾選**的目的地這次不發送，也不記錄任何決定。
3. **合併等待中的請求**：對話框開著時，同一個呼叫端、同一組目的地的新請求不另開對話框，而是排在後面並顯示「另有 N 則等待中」。使用者的決定套用到所有等待中的請求（包含「允許這一次」）。不同呼叫端的請求依序排隊，一次只顯示一個對話框。
4. **視窗不在前景**：同時發出系統通知「網站監控要求發送通知，按此確認」，並在狀態列顯示「🔒 1 個待確認」。點擊後帶到對話框。
5. **沒有視窗**（自我測試、無介面模式）：直接回報 `denied`，`message` 註明「無法詢問使用者」。
6. 對話框按鈕預設焦點在「拒絕」，避免按 Enter 誤核准。
7. **呼叫端在等待期間被停用、移除或崩潰**：關閉它的對話框，等待中的請求回報 `denied`（`message`：「呼叫端已停止」），不記錄決定。
8. **平台結束**（`app:before-quit`）：所有等待中的請求回報 `timeout`，不記錄決定。

### 4.5 事先核准

背景執行的呼叫端（例如半夜的網站監控）第一次發通知時，使用者可能不在電腦前，請求會逾時。所以「執行鎖」畫面（〈9.2〉）提供**新增授權**：選呼叫端、選目的地、允許。呼叫端也可以在自己的設定畫面放一個「測試通知」按鈕，引導使用者在安裝時就完成第一次確認。

### 4.6 授權紀錄檔

`userData/permissions.json`，只由主程序讀寫：

```json
{
  "version": 1,
  "grants": [
    {
      "caller": "site-monitor",
      "kind": "notify",
      "target": "telegram/chat-7f3a",
      "decision": "allow",
      "createdAt": 1795000000000,
      "lastUsedAt": 1795003600000,
      "useCount": 12
    }
  ],
  "mac": "base64…"
}
```

- `mac`：以 `safeStorage` 保護的平台金鑰對 `grants` 計算的 HMAC。驗證失敗時視為**沒有任何授權**（全部重新詢問），並在輸出面板警告。目的是讓「直接改檔案」不會悄悄取得授權；平台沒有沙箱，這不是完整的防護（〈10〉）。
- 目標的位址 id 消失（例如 Telegram 刪掉了那個聊天室）時保留紀錄，畫面標示「目的地已不存在」。

### 4.7 用途宣告與安裝後引導

授權一律在**執行時**取得（〈4.1〉），安裝時不授權：安裝當下收件位置可能還沒設定，使用者也看不到實際內容。為了讓使用者事先知道擴充會做什麼，並在電腦前完成第一次確認，增加兩件事。

**用途宣告**：擴充在 manifest 宣告會用到的受保護動作與原因。宣告**只是說明，不會授權**；沒有宣告的擴充執行時一樣會詢問。

```jsonc
"tooldock": {
  "permissions": {
    "notify": {
      "reason": "網站異常或恢復時通知你",        // 必填，最多 200 字
      "testCommand": "siteMonitor.testNotify"   // 選填：發送測試通知的指令（必須是自己宣告的指令）
    },
    "commands": [
      { "id": "telegram.send", "reason": "把每日報表發到 Telegram" }
    ]
  }
}
```

顯示位置：

| 位置 | 內容 |
|---|---|
| 商店的安裝／更新確認畫面 | 「這個擴充會：發送通知（原因）、執行『Telegram 通知：發送訊息』（原因）」，並註明「第一次執行時會再詢問你」 |
| 擴充詳細頁「權限用途」區塊 | 宣告內容、目前的授權、「發送測試通知」按鈕、「尚未完成第一次確認」提示 |
| 安裝後引導對話框 | 見下方 |

商店索引的 `latest.permissions` 收錄宣告摘要（與 `latest.exchange` 相同做法），安裝前就能顯示。

**安裝後引導**：符合下列條件的擴充，平台顯示一次引導對話框：

1. 宣告了 `permissions.notify`；
2. 已啟用且有效；
3. 還沒有任何通知授權；
4. 還沒有顯示過引導（記在 `state.json` 的 `guardOnboarded`）。

時機：從套件、資料夾或商店安裝完成後立即顯示；內建擴充與平台升級前就裝好的擴充，在工作台就緒後顯示（一次一個，排隊）。

```
┌──────────────────────────────────────────────────────┐
│ 📡 「網站監控」會發送通知                              │
│                                                      │
│  用途：網站異常或恢復時通知你                          │
│  也會執行：（permissions.commands，沒有則不顯示）       │
│                                                      │
│  通知第一次送出前需要你確認。現在發一則測試通知，        │
│  就能先完成確認，之後在背景發生異常時才不會等不到你。    │
│                                                      │
│               [ 稍後 ]   [ 發送測試通知 ]             │
└──────────────────────────────────────────────────────┘
```

- **發送測試通知**：有 `testCommand` 時，平台以使用者身分執行該指令（指令內呼叫 `ctx.notify.send`，呼叫者是該擴充，所以授權給的是擴充）；沒有時，平台以該擴充的身分發一則「〈擴充名稱〉：測試通知」到它的預設路由。接著就會出現〈4.4〉的授權對話框。
- 兩個按鈕都會記錄「已引導」，不再自動顯示；之後可以從擴充詳細頁的「發送測試通知」再做一次。
- 自我測試、命令列模式不顯示引導。

## 5. 通知內容

```ts
interface Notification {
  /** 標題，必填，最多 200 字。Mail 當主旨、Telegram 以粗體顯示 */
  title: string;
  /** 內文，純文字，最多 4000 字。通道自行轉換格式，呼叫端不必跳脫 HTML */
  body?: string;
  /** 等級，影響圖示與顏色，預設 info */
  level?: 'info' | 'success' | 'warning' | 'error';
  /** 指派對象，見〈6〉。省略時使用預設路由 */
  to?: string | string[];
  /** 分類，例如 site.down。用於紀錄篩選、對象的預設路由（〈6.2〉） */
  topic?: string;
  /** 補充欄位，通道以「名稱：值」逐行呈現 */
  fields?: Array<{ label: string; value: string }>;
  /** 相關連結 */
  url?: string;
  /** 附件，總大小上限 20 MB。不支援附件的通道略過並在結果註明 */
  attachments?: Array<{ filename: string; content: string | Uint8Array; mime?: string }>;
  /** 去重鍵：同一呼叫端、同一鍵在 dedupeWindow 內只發一次 */
  dedupeKey?: string;
  /** 去重時間窗（秒），預設 300 */
  dedupeWindow?: number;
  /** 靜音（通道支援時），預設依 level：info/success 靜音 */
  silent?: boolean;
}
```

- 內容是**純文字**加結構化欄位。各通道自己決定排版（Telegram 轉成 HTML 並跳脫、Mail 產生純文字與 HTML 兩份），呼叫端不需要針對通道寫格式。
- 不得放入帳號密碼。平台的執行紀錄只保留標題與內文前 200 字。

## 6. 指派對象與路由

### 6.1 `to` 的寫法

| 寫法 | 意義 | 例 |
|---|---|---|
| 省略 | 預設路由（〈6.2〉） | — |
| 對象名稱 | 使用者定義的收件群組 | `'維運人員'` |
| `通道/位址 id` | 指定單一目的地（進階用法，呼叫端得知道位址 id） | `'telegram/chat-7f3a'` |
| `通道` | 該通道的預設目的地 | `'telegram'` |
| 陣列 | 以上的聯集，重複的目的地只發一次 | `['維運人員', 'desktop']` |

**建議呼叫端只用對象名稱或省略**，並把對象名稱做成自己的設定（例如 `siteMonitor.notifyTo`，預設 `維運人員`），這樣呼叫端完全不用知道有哪些通道。

### 6.2 預設路由

`to` 省略時，依序使用：

1. 這個呼叫端的預設路由（使用者在「執行鎖」畫面為呼叫端或 `topic` 指定的對象）。
2. 全域預設對象（`notify.json` 的 `defaultAudience`，初始為「我」= `desktop/local`）。

### 6.3 對象不存在

`to` 指定的對象名稱還沒有定義時，授權對話框改成「網站監控 要發送給『維運人員』，但還沒有這個對象。要送到哪裡？」並列出所有通道的目的地讓使用者勾選。按「永遠允許」時同時**建立這個對象**並記錄授權；按「允許這一次」只發這一次，不建立對象。

### 6.4 對象設定檔

`userData/notify.json`：

```json
{
  "version": 1,
  "defaultAudience": "我",
  "audiences": [
    { "name": "我", "destinations": ["desktop/local"] },
    { "name": "維運人員", "destinations": ["telegram/chat-7f3a", "mail/ops"] }
  ],
  "routes": [
    { "caller": "site-monitor", "topic": "site.down", "audience": "維運人員" }
  ],
  "options": { "promptTimeoutSeconds": 600, "rateLimitPerMinute": 20, "logRetentionDays": 90, "logMaxRecords": 10000 }
}
```

`options` 是執行鎖與通知服務的設定，在「執行鎖」畫面的「對象與設定」分頁修改。

## 7. 其他受執行鎖保護的動作

### 7.1 桌面通知 `ctx.window.notify`

桌面通知會跳到其他程式之上，也會出現在系統通知中心，所以同樣要經過使用者確認：

- 1.6.0 起 `ctx.window.notify(title, body)` 視為 `ctx.notify.send({ title, body, to: 'desktop/local' })`，授權目標是 `desktop/local`，**與 `ctx.notify` 發到 `desktop` 共用同一筆授權**。
- 為了相容既有擴充，`ctx.window.notify` 被拒絕或逾時**不丟出例外**，回傳 `false`；送出時回傳 `true`（與現在相同）。
- 執行紀錄的呼叫方式顯示為 `window.notify`。
- 生命週期設定檔的 `notify` 動作是使用者自己寫的，不經過執行鎖（〈8.5〉）。

### 7.2 受保護指令

通知以外的敏感動作（例如 Oracle SQL 工具的「執行寫入」、Telegram 的 `telegram.send`），由提供指令的擴充在 manifest 標示：

```json
"commands": [
  { "id": "telegram.send", "title": "Telegram 通知：發送訊息", "guarded": true }
]
```

- 其他擴充執行時經過執行鎖，種類為 `command`、目標為指令 id。對話框顯示呼叫端、指令標題，以及參數的 JSON 預覽（前 500 字）。
- 擴充也可以提供 `guardSummary` 匯出函式，把參數轉成給人看的說明：`exports.guardSummary = (ctx, command, args) => '發送「備份完成」到 維運群組'`。
- 被拒絕時 `ctx.commands.execute` 丟出錯誤，`err.code === 'GUARD_DENIED'`（逾時為 `GUARD_TIMEOUT`）。

**Telegram 通知 1.1.0 起 `telegram.send` 標示為 `guarded`**，其他擴充改用 `ctx.notify.send`。

## 8. API

### 8.1 呼叫端：`ctx.notify`

```js
// 網站監控：網站由正常轉為異常時
if (ctx.notify) {
  const r = await ctx.notify.send({
    title: '網站異常',
    body: `❌ ${site.name} 異常：${st.error}`,
    level: 'error',
    topic: 'site.down',
    to: ctx.settings.get('siteMonitor.notifyTo', '維運人員'),
    fields: [{ label: '網址', value: site.url }, { label: '狀態碼', value: String(st.code || '-') }],
    url: site.url,
    dedupeKey: `down:${site.id}`,
  });
  if (r.status !== 'sent') ctx.log.warn(`通知未完整送達：${r.status} ${r.message || ''}`);
}
```

| 方法 | 說明 |
|---|---|
| `send(n): Promise<NotifyResult>` | 發送通知。**只有參數不合法時丟出例外**，其餘一律以 `status` 回報，與 `ctx.exchange.share` 相同 |
| `destinations(): Promise<DestinationInfo[]>` | 目前可用的目的地與對象（不含位址細節，只有 id、通道、顯示名稱），給呼叫端做設定畫面的下拉選單 |
| `audiences(): Promise<string[]>` | 已定義的對象名稱 |

擴充畫面也可以用 `window.tooldock.notify.send()`，身分同樣是該擴充。

### 8.2 回應：`NotifyResult`

```ts
interface NotifyResult {
  /** 通知編號，可在執行紀錄查詢（〈9.3〉） */
  id: string;
  /** 整體結果 */
  status:
    | 'sent'            // 所有目的地都送達
    | 'partial'         // 部分送達
    | 'failed'          // 都沒送達（通道錯誤）
    | 'denied'          // 使用者拒絕，或所有目的地都已被拒絕
    | 'timeout'         // 授權對話框逾時
    | 'no-destination'  // 解析不出任何目的地（沒有對應的通道擴充）
    | 'throttled'       // 超過頻率限制（〈8.4〉）
    | 'duplicate';      // dedupeKey 在時間窗內已發送過
  message?: string;
  deliveries: Array<{
    destination: string;       // 'telegram/chat-7f3a'
    title: string;             // 'Telegram ─ 維運群組'
    status: 'sent' | 'failed' | 'denied' | 'skipped';
    /** 'once' | 'always' | 'denied' | 'remembered-deny' | 'timeout' | 'exempt'（使用者、生命週期設定檔）| 'none'（目的地不存在，沒有詢問） */
    decision: string;
    message?: string;          // 失敗原因（中文），或「此通道不支援附件」
    ref?: unknown;             // 通道回傳的參考資料，例如 Telegram messageIds
    at: number;
  }>;
}
```

### 8.3 通道擴充

Manifest：

```json
"contributes": {
  "notify": {
    "channels": [
      {
        "id": "telegram",
        "title": "Telegram",
        "icon": "✈️",
        "destinations": "listDestinations",
        "deliver": "deliver",
        "supports": { "attachments": true, "silent": true, "maxBody": 4096 }
      }
    ]
  }
}
```

- 通道 id 必須是**自己的擴充 id** 或以 `擴充id.` 開頭（`telegram`、`telegram.channel`），避免搶別人的名稱。`desktop` 保留給平台。
- 平台自動加上啟用事件 `onNotify:<通道 id>`，延遲載入的通道擴充在需要時才啟用。

後端：

```js
/** 列出目的地：在「執行鎖」畫面、授權對話框、ctx.notify.destinations() 時呼叫 */
exports.listDestinations = async (ctx) => chats().map((c) => ({
  id: c.id,                // 穩定的位址 id（不是聊天室名稱）
  title: c.name,
  detail: c.kind,          // '群組'、'頻道'、'個人'
  isDefault: c.isDefault,
}));

/**
 * 交付一則通知到一個目的地。
 * @param {object} n      Notification（已驗證、附件已轉成 Uint8Array）
 * @param {object} dest   { id, title }
 * @param {object} info   { caller: { id, name }, notificationId, signal }
 * @returns {{ status: 'sent' | 'failed', message?: string, ref?: any }}
 */
exports.deliver = async (ctx, n, dest, info) => { /* … */ };
```

- 交付逾時 60 秒，逾時視為 `failed`。重試（例如 Telegram 429）由通道自己處理。
- `deliver` 丟出例外時平台視為 `failed`，`message` 為例外訊息。
- 交付期間通道擴充標記為忙碌，不會被自動回收（沿用 `track`）。

### 8.4 頻率限制與去重

平台在執行鎖**之前**檢查，避免一次異常洗版、也避免連續跳出對話框：

| 限制 | 預設 | 設定 |
|---|---|---|
| 每個呼叫端每分鐘 | 20 則 | `notify.json` 的 `options.rateLimitPerMinute` |
| `dedupeKey` 時間窗 | 300 秒 | 呼叫端的 `dedupeWindow` |

超過時回報 `throttled` / `duplicate`，不會發送、不會詢問，但會寫入執行紀錄。

### 8.5 生命週期設定檔的 `notify` 動作

現有的 hook 動作 `notify` 只發桌面通知。1.6.0 起加上 `to` 與其他欄位，就改走通知服務：

```json
{ "action": "notify", "title": "每日備份完成", "body": "共 12 個檔案", "level": "success", "to": "維運人員" }
```

- 沒有 `to` 時維持原行為（只發桌面通知），舊設定檔不受影響。
- 設定檔由使用者撰寫，呼叫者視為使用者，**不經過執行鎖**（〈4.2〉），執行紀錄中的呼叫端顯示「生命週期設定」。
- 結果寫入執行紀錄與生命週期時間軸；失敗不會中斷該階段的其他 hook。

### 8.6 與資料交換的關係

兩者並存，用途不同：

| | 資料交換（`ctx.exchange.share`） | 通知服務（`ctx.notify.send`） |
|---|---|---|
| 用途 | 把資料交給**使用者**挑一個工具處理 | 把訊息**自動**送到人員 |
| 每次都要使用者操作 | 是（接收端預覽、確認） | 否，只有第一次確認授權 |
| 呼叫端知道收件位置 | 否 | 否（只用對象名稱） |

Telegram 通知接收 `tooldock.text`、`tooldock.table` 的預覽確認流程不變；使用者手動分享不需要執行鎖。

## 9. 平台內建畫面：「執行鎖」

### 9.1 為什麼內建在平台，而不是一般擴充

「執行鎖」畫面以平台核心畫面實作（與「擴充管理」、「生命週期」相同），出現在側邊欄、可以從命令面板開啟，**但不能停用或移除**：

- 授權與紀錄是主程序的資料（`permissions.json`、`guard-log.jsonl`），如果把管理畫面做成一般擴充，就得開放一組「可以改授權」的 API 給擴充，任何擴充都可能拿來替自己核准。
- 一般擴充可以被停用或移除；使用者撤銷授權的入口不能跟著消失。

畫面有四個分頁：**待確認**、**授權**、**執行紀錄**、**對象**。側邊欄圖示 🔒，有待確認的請求時顯示數字。

### 9.2 授權（撤銷與調整）

表格列出所有記住的決定：

| 欄位 | 說明 |
|---|---|
| 呼叫端 | 擴充名稱與 id；已移除的擴充不會出現（〈4.3〉） |
| 種類 | 通知 / 指令 |
| 目標 | 例如「✈️ Telegram ─ 維運群組」、「💻 桌面通知」、「Telegram 通知：發送訊息」；目的地已不存在時標示 |
| 決定 | 允許 / 拒絕 |
| 建立時間、最後使用、次數 | 最後使用與次數只計實際放行或擋下的呼叫 |

操作：

| 操作 | 效果 |
|---|---|
| **撤銷** | 刪除這筆授權。下一次呼叫**重新跳出確認**（等同從沒核准過） |
| **改為拒絕** / **改為允許** | 直接切換決定，不刪除紀錄 |
| **撤銷此擴充的全部授權** | 依呼叫端分組，一次撤銷 |
| **新增授權** | 事先核准（〈4.5〉）：選呼叫端、種類、目標、允許或拒絕 |
| **查看紀錄** | 跳到執行紀錄分頁，並篩選成這筆授權的呼叫端與目標 |

- 撤銷或修改**立即生效**：等待中的請求不受影響，之後的呼叫依新狀態處理。
- 每次撤銷、修改、新增也寫入執行紀錄（呼叫方式為「授權變更」），可以追查誰在什麼時候改過。
- 「擴充管理」的擴充詳細資料加上「授權」區塊，列出該擴充擁有的授權並提供相同的撤銷按鈕。

### 9.3 執行紀錄（查詢）

每一次經過執行鎖的呼叫都記一筆，包含被拒絕、逾時、節流、去重的呼叫。

紀錄欄位：

```ts
interface GuardRecord {
  id: string;            // 紀錄編號；通知時與 NotifyResult.id 相同
  at: number;            // 呼叫時間
  caller: string | null; // 擴充 id；null 表示使用者、'lifecycle' 表示生命週期設定檔
  via: 'notify.send' | 'window.notify' | 'command' | 'hook' | 'grant-change';
  kind: 'notify' | 'command' | 'grant';
  /** 通知：標題與內文前 200 字；指令：指令標題與 guardSummary（或參數前 200 字） */
  summary: string;
  topic?: string;
  /** 每個目標一筆 */
  targets: Array<{
    target: string;      // 'telegram/chat-7f3a'、'desktop/local'、'telegram.send'
    decision: 'once' | 'always' | 'denied' | 'remembered-deny' | 'timeout' | 'exempt';
    result: 'sent' | 'ok' | 'failed' | 'skipped' | 'not-run';
    message?: string;    // 失敗原因
  }>;
  status: string;        // 與 NotifyResult.status 相同；指令為 ok / error / denied / timeout
  waitedMs?: number;     // 等待使用者確認的時間
  durationMs: number;    // 從呼叫到回應的總時間
}
```

查詢條件（可組合）：

| 條件 | 說明 |
|---|---|
| 時間範圍 | 今天、最近 7 天、最近 30 天、自訂 |
| 呼叫端 | 下拉選擇，含「使用者」「生命週期設定」 |
| 種類 / 呼叫方式 | 通知、桌面通知、指令、授權變更 |
| 目標 | 目的地或指令 |
| 決定 | 允許這一次、永遠允許、拒絕、記住的拒絕、逾時 |
| 結果 | 成功、部分、失敗、被擋下（拒絕／逾時／節流／去重） |
| 關鍵字 | 比對摘要與 `topic` |

- 點一筆紀錄展開詳細資料：每個目標的決定與結果、失敗原因、等待確認的時間。
- 失敗的通知可以「重送」，重送是一筆新的呼叫，**仍經過執行鎖**，呼叫端記為原呼叫端。紀錄檔只存摘要，所以只能重送**平台這次執行期間**發出的通知（完整內容只保留在記憶體，最近 100 則，附件超過 1 MB 不保留）。
- 從紀錄可以直接「撤銷這筆授權」或「改為拒絕」。
- 「匯出 CSV」匯出目前篩選的結果（含 BOM）。

保存：

- 檔案 `userData/guard-log.jsonl`，一行一筆，只附加不改寫。
- 保留 **90 天且最多 10,000 筆**，超過時在平台啟動與每天第一次寫入時清除最舊的。設定在 `notify.json` 的 `options.logRetentionDays`、`options.logMaxRecords`，也可以在「對象與設定」分頁修改。
- 只保存摘要（前 200 字），不保存附件、`fields` 與指令的完整參數。
- 「清除紀錄」需要再次確認，清除後留下一筆「紀錄已清除」。

### 9.4 對象與待確認

- **對象**：新增、編輯對象；每個對象勾選目的地（依通道分組列出），可按「測試」發一則測試通知。設定全域預設對象與呼叫端的預設路由（〈6.2〉）。
- **待確認**：列出目前等待使用者決定的請求（呼叫端、摘要、目標、已等待多久、剩餘時間），點擊開啟授權對話框。

### 9.5 核心指令

| 指令 | 標題 |
|---|---|
| `guard.show` | 執行鎖：開啟 |
| `guard.showLog` | 執行鎖：查詢執行紀錄（可帶 `{ caller, target }` 預先篩選） |
| `guard.revokeAll` | 執行鎖：撤銷所有授權（需再次確認） |
| `notify.test` | 通知：發送測試通知 |

擴充**不能**查詢或修改授權與執行紀錄。呼叫端要知道自己某一次通知的結果，用 `ctx.notify.send()` 回傳的 `NotifyResult` 即可。

## 10. 安全性

- **平台沒有沙箱。** 擴充可以自己連線 Telegram API；執行鎖保護的是**通道擴充的憑證**（例如 Telegram 的 Bot Token，以 `ctx.secrets` 依擴充分開保存，其他擴充讀不到）與**使用者的知情同意**，不是防惡意擴充。商店的審核與簽章仍是主要防線。
- 呼叫者身分由平台依 RPC 連線決定，不採用呼叫端傳入的任何欄位。
- 授權紀錄檔有 HMAC 檢查（〈4.6〉），被改動時全部失效。
- 授權對話框必須顯示實際內容與目的地；預設焦點在「拒絕」。
- 執行紀錄只保存摘要（前 200 字），不保存附件、`fields` 與指令完整參數；保留 90 天、最多 10,000 筆（〈9.3〉）。
- 授權的檢視、撤銷與執行紀錄只在平台內建畫面提供，沒有給擴充的 API（〈9.1〉）。

## 11. 後續版本（不在 v1）

- **人員回覆（ack）**：`send({ ack: true })` 時 Telegram 附上「已處理」按鈕、Mail 附上確認連結；收件人按下後平台發出 `ctx.notify.onDidAck(id, fn)`，網站監控可以把該異常標示為「已有人處理」。需要通道擴充提供 `onAck` 事件，Telegram 要開始輪詢 `getUpdates`。
- **升級規則**：N 分鐘內沒人回覆，改發給下一個對象。
- **安靜時段**：指定時段只發 `error` 等級。
- **其他種類的執行鎖**：網路對外連線、讀取其他擴充的資料等，沿用 `(呼叫端, 種類, 目標)` 的授權模型。

## 12. 相容性

| 情況 | 行為 |
|---|---|
| 平台 < 1.6.0 | 沒有 `ctx.notify`；manifest 的 `contributes.notify`、`guarded` 被忽略。呼叫端先判斷 `if (ctx.notify)`，否則退回 `ctx.window.notify` |
| 呼叫端仍用 `ctx.commands.execute('telegram.send', …)` | Telegram 1.1.0 起經過執行鎖（第一次詢問）；Telegram 1.0.0 維持原行為 |
| 沒有安裝任何通道擴充 | 只有 `desktop`；指定 `telegram` 的請求回報 `no-destination` |
| 既有擴充呼叫 `ctx.window.notify`（例如網站監控 1.0.0） | 1.6.0 起第一次會跳出確認；被拒絕時回傳 `false`、不丟出例外（〈7.1〉） |
| 通道擴充被停用 | 它的目的地視為不存在，回報 `no-destination`（或 `partial`），授權保留 |

## 13. 實作計畫

### 13.1 修改位置

| 位置 | 內容 |
|---|---|
| `src/main/guard.js`（新增） | 授權紀錄（`permissions.json` 讀寫、HMAC）、`check(caller, kind, target, prompt)`、等待中請求的合併與排隊、逾時；授權的撤銷／修改／新增；執行紀錄（`guard-log.jsonl` 附加、查詢、保留期限清理、匯出 CSV） |
| `src/main/notify.js`（新增） | 通知驗證、節流與去重、對象解析（`notify.json`）、依通道分組交付、彙整 `NotifyResult`；`desktop` 通道（含 `ctx.window.notify` 改走執行鎖，〈7.1〉） |
| `src/main/extension-manager.js` | `readManifest` 解析 `contributes.notify`、`commands[].guarded`，加上 `onNotify:*`；`executeCommand(command, args, { caller })` 對 `guarded` 指令呼叫 guard；服務 `notify.send`、`notify.destinations`、`notify.audiences`；移除擴充時清除授權 |
| `src/host/host-runtime.js` | `ctx.notify`；處理 `notify.destinations`、`notify.deliver`、`guard.summary` RPC（依匯出名稱呼叫） |
| `src/main/main.js` | 核心指令 `guard.show`、`guard.showLog`、`guard.revokeAll`、`notify.test`；router 新增 `guard.*` 方法（只給工作台用，不在 `services(ext)` 裡）；router 的 `commands.execute`、`buildMenu()`、hook 的 `command` 動作傳 `{ caller: null }`；hook 的 `notify` 動作有 `to` 時改走通知服務（〈8.5〉） |
| `src/renderer/workbench.js` | 授權對話框、狀態列「🔒 待確認」、側邊欄「執行鎖」畫面（待確認、授權、執行紀錄、對象，〈9〉）；「擴充管理」詳細資料的授權區塊 |
| `src/sdk/ui-sdk.js` | `window.tooldock.notify` |
| `src/main/selftest-notify.js`（新增） | 〈13.3〉 |
| `packages/types` | `NotifyApi`、`Notification`、`NotifyResult`、通道 handler 型別 |
| `packages/cli` | `validate` 檢查 `contributes.notify`（通道 id 前綴、handler 名稱） |
| `extensions/site-monitor` | 1.1.0：異常與恢復改用 `ctx.notify.send`，新增設定 `siteMonitor.notifyTo`；設定畫面加「測試通知」 |
| `tooldock-ext-telegram`（另一個 repo） | 1.1.0：宣告 `telegram` 通道、`listDestinations`、`deliver`（沿用 `sendTo`）；`telegram.send` 標示 `guarded` 並提供 `guardSummary`；engine `^1.6.0` |
| `docs/EXTENSION_GUIDE.md` | 新增「通知」章節，連到本文件 |

### 13.2 里程碑

1. **執行鎖**：`guard.js`、授權對話框、`guarded` 指令、命令面板豁免、`ctx.window.notify` 改走執行鎖、「執行鎖」畫面的授權與執行紀錄分頁。
2. **通知服務**：`notify.js`、`ctx.notify`、內建 `desktop` 通道、對象與預設路由、對象分頁。
3. **Telegram 通道**：Telegram 1.1.0 發佈到商店。
4. **網站監控**：1.1.0 改用通知服務，驗證完整情境（〈1〉）。
5. **Mail 通道**：另一個擴充 `tooldock-ext-mail`（SMTP，憑證放 `ctx.secrets`），只需實作〈8.3〉，呼叫端不用改。

### 13.3 自我測試新增項目

- 呼叫者身分：擴充 A 無法以 B 的名義送出（身分由連線決定）。
- 第一次發送跳出詢問；「允許這一次」不記錄、「永遠允許」記錄 `allow`、「拒絕＋不再詢問」記錄 `deny` 且之後不再詢問。（自我測試以可替換的 `guard.prompt` 直接回答）
- 對話框開著時同一組請求合併，決定套用到全部。
- 逾時回報 `timeout`；沒有視窗回報 `denied`。
- `permissions.json` 被改動（HMAC 不符）時視為沒有授權。
- 移除擴充後它的授權被清除。
- 自己的 `guarded` 指令、命令面板執行不經過執行鎖；其他擴充執行時經過，拒絕時 `err.code === 'GUARD_DENIED'`。
- 對象解析：省略 `to`、對象名稱、`通道/id`、`通道`、陣列去重；對象不存在時的詢問與建立。
- 部分目的地失敗回報 `partial`；通道 `deliver` 丟出例外、逾時回報 `failed`；通道擴充停用時 `no-destination`。
- 節流 `throttled`、去重 `duplicate`，兩者都不跳出對話框。
- 延遲載入的通道擴充在交付時被啟用，交付期間不被回收。
- 擴充畫面發出的 `notify.send` 以該畫面的擴充為呼叫者；A 執行 B 的一般指令、B 再發通知時，呼叫者是 B。
- 等待確認期間呼叫端被停用回報 `denied`；平台結束時回報 `timeout`；兩者都不記錄決定。
- hook `notify` 沒有 `to` 時只發桌面通知；有 `to` 時走通知服務且不跳出對話框。
- `ctx.notify` 發到 `desktop` 與 `ctx.window.notify` 都要確認，且共用同一筆授權；`ctx.window.notify` 被拒絕時回傳 `false`、不丟出例外。
- 撤銷授權後下一次呼叫重新詢問；「改為拒絕」後直接回報 `denied`；撤銷、修改、新增都寫入執行紀錄。
- 執行紀錄：允許、拒絕、逾時、節流、去重都有紀錄；依呼叫端、目標、決定、結果、時間範圍查詢的結果正確；超過保留期限或筆數時清除最舊的。
- 擴充的 `services(ext)` 沒有任何 `guard.*` 方法（擴充無法查詢或修改授權）。

## 14. 決定紀錄

| # | 項目 | 決定 | 狀態 |
|---|---|---|---|
| D1 | 授權單位 | 呼叫端 × 目的地：換新的聊天室要再問；不綁內容、不綁版本 | 定案 |
| D2 | 「拒絕」是否記住 | 預設只拒絕這一次；勾選「不再詢問」才記住 | 定案 |
| D3 | 確認逾時 | 10 分鐘，回報 `timeout`，可在 `notify.json` 的 `options.promptTimeoutSeconds`（「對象與設定」分頁）調整 | 定案 |
| D4 | 桌面通知 | **需要確認**：`ctx.notify` 發到 `desktop` 與 `ctx.window.notify` 都經過執行鎖，共用授權（〈7.1〉） | 定案（改變） |
| D5 | 「允許這一次」與合併 | 套用到對話框開著時累積的同組請求 | 定案 |
| D6 | 「處理後有回應」 | 只回傳送達結果（`NotifyResult`）；收件人「已處理」回報列在後續版本（〈11〉） | 定案 |
| D7 | 命令面板、生命週期設定檔 | 視為使用者本人，免確認 | 定案 |
| D8 | 呼叫者身分 | 由平台依連線或 iframe 決定，只看直接呼叫者 | 定案 |
| D9 | `send` 的錯誤處理 | 只有參數不合法丟出例外，其餘以 `status` 回報（與 `exchange.share` 一致） | 定案 |
| D10 | `telegram.send` | Telegram 1.1.0 起標示 `guarded`，建議改用 `ctx.notify` | 定案 |
| D11 | 執行鎖的管理畫面 | 平台內建核心畫面「執行鎖」，不能停用或移除；提供撤銷授權與執行紀錄查詢；擴充沒有對應 API（〈9〉） | 定案 |
| D12 | 執行紀錄保存 | 90 天、最多 10,000 筆，只存摘要（〈9.3〉） | 定案 |
| D13 | 授權時機 | 執行時詢問；安裝時只顯示用途宣告，不授權；安裝後引導發送測試通知完成第一次確認（〈4.7〉） | 定案 |
