# 擴充商店

ToolDock 的擴充商店以 **GitHub Releases** 為發佈來源，以一個**索引 repo** 決定收錄哪些擴充、信任哪把金鑰。平台本身不需要任何伺服器：索引是一個靜態的 `index.json`，套件直接從各擴充 repo 的 Release 下載。

> ⚠ **先講限制**：擴充後端在獨立的 Node.js 程序中執行，擁有與 ToolDock 相同的完整權限（讀寫檔案、連網、執行子程序），**平台沒有沙箱**。商店的簽章驗證只能證明「套件出自索引登記的發佈者，且下載過程沒有被竄改」，不能證明程式碼是安全的。詳見文末〈限制〉。

## 架構

```
 擴充 repo（每個擴充一個）                索引 repo（registry/ 範本）                    ToolDock
 ────────────────────────                ─────────────────────────                    ───────────
 push tag v1.2.0                          extensions.json  ← 人工審核的 PR
   └─ release.yml（範本內建）              （id、repo、publisher、publicKey、verified）
       pack --sign                                │
       └─ GitHub Release                          ▼
          ├ my-tool-1.2.0.tbx       ◀── build-index.js（每小時 / push / dispatch）
          ├ my-tool-1.2.0.tbx.sha256      讀 Releases → 下載 → 驗 SHA-256、簽章、id、version
          └ my-tool-1.2.0.tbx.sig         └─ index.json → GitHub Pages ──────────▶ 商店來源（store.json）
                    ▲                                                                   │
                    └──────────────────── 下載 .tbx / .sig（net.fetch）──────────────────┤
                                                                                        ▼
                                                     SHA-256 → Ed25519（索引中的公鑰）→ engine
                                                                                        │ 全部通過
                                                                                        ▼
                                                                         ExtensionManager.install()
```

| 位置 | 內容 |
|---|---|
| `registry/` | 索引 repo 範本，之後複製成獨立 GitHub repo（見〈建立索引 repo〉） |
| `config/store.json` | 平台的商店預設設定：來源、是否自動檢查更新、檢查週期 |
| `userData/store.json` | 使用者覆寫（商店「來源設定」頁面寫入） |
| `userData/store-cache/` | 各來源索引的快取，離線時沿用 |
| `userData/store-installs.json` | 從商店安裝的紀錄（repo、金鑰指紋、SHA-256），用來判斷更新能否一鍵完成 |
| `src/main/store.js` | 抓取與快取索引、驗證、安裝、更新檢查 |
| `src/renderer/workbench.js` | 活動列「商店」、列表 / 搜尋 / 詳細頁、安裝確認、來源設定頁、狀態列更新提示 |

## 信任模型

- **信任來源是索引**。安裝時只用索引中該擴充登記的 `publicKey` 驗證簽章；套件內附的任何金鑰、Release 說明裡的公鑰一律不採用。
- 索引必須以 `https://` 或 `file://` 讀取。明文 `http://` 的索引預設會被拒絕：攔截者可以換掉公鑰，讓簽章驗證失去意義。確定是受信任的內網時，可在該來源加上 `"allowInsecure": true`。
- `.tbx`、`.sha256`、`.sig` 本身可以走任何網址（包含 http），因為它們都會用索引的資料驗證。
- **「已驗證發佈者」**（`verified: true`）表示索引維護者確認過發佈者身分，不代表程式碼經過安全稽核。商店畫面會同時顯示這個說明。

## 發佈檔約定

擴充 repo 每個版本一個 Release，tag 為 `v<version>`，附件：

| 檔案 | 內容 |
|---|---|
| `<id>-<version>.tbx` | 擴充套件（zip） |
| `<id>-<version>.tbx.sha256` | `.tbx` 的 SHA-256，一行小寫 hex（讀取端會 trim，也接受 `sha256sum` 的「hex  檔名」格式） |
| `<id>-<version>.tbx.sig` | `crypto.sign(null, tbxBytes, privateKey)` 的結果，base64（解碼後 64 bytes） |

- 金鑰為 Ed25519。公鑰字串是 SPKI DER 的 base64，可用 `crypto.createPublicKey({ key: Buffer.from(s, 'base64'), format: 'der', type: 'spki' })` 載入。
- 擴充範本內建的發佈 workflow（`tooldock-ext pack --sign`）會產出上述附件，金鑰與 secret 設定見 `docs/RELEASE.md`。
- 標為 prerelease 或 draft 的 Release 不會收錄；缺任何一個附件的版本也不會收錄（例如沒有設定簽章 secret 時，Release 只有 `.tbx` 與 `.sha256`）。

## 索引格式

### `extensions.json`（人工維護）

```json
[
  {
    "id": "my-tool",
    "repo": "your-name/tooldock-my-tool",
    "publisher": "你的名字或組織",
    "publicKey": "MCowBQYDK2VwAyEA…",
    "verified": false
  }
]
```

Schema：`registry/schema/extensions.schema.json`。`node scripts/validate.js` 會另外實際載入每把公鑰，確認是 Ed25519。

### `index.json`（build-index.js 產生）

```json
{
  "schemaVersion": 1,
  "name": "ToolDock 擴充索引",
  "generatedAt": "2026-11-18T08:00:00.000Z",
  "extensions": [
    {
      "id": "my-tool",
      "repo": "your-name/tooldock-my-tool",
      "publisher": "你的名字或組織",
      "publicKey": "MCowBQYDK2VwAyEA…",
      "verified": false,
      "latest": {
        "version": "1.2.0",
        "tbxUrl": "https://github.com/your-name/tooldock-my-tool/releases/download/v1.2.0/my-tool-1.2.0.tbx",
        "sha256Url": "https://github.com/…/my-tool-1.2.0.tbx.sha256",
        "sigUrl": "https://github.com/…/my-tool-1.2.0.tbx.sig",
        "sha256": "9f2c…（64 個 hex）",
        "publishedAt": "2026-11-17T12:00:00Z",
        "engine": ">=1.0.0",
        "displayName": "我的工具",
        "description": "…",
        "icon": "🛠",
        "size": 48213,
        "exchange": { "imports": ["tooldock.table"], "formats": [{ "type": "my-tool.report", "version": 1, "title": "我的報表" }] }
      }
    }
  ]
}
```

- Schema：`registry/schema/index.schema.json`。
- `tbxUrl`、`sha256Url`、`sigUrl` 可以是絕對網址，也可以**相對於 index.json**（例如 `files/my-tool-1.2.0.tbx`），方便把整個索引連同套件複製到共用磁碟做離線鏡像。
- `sha256` 是建立索引時驗證過的值，安裝時以它為準；沒有這個欄位才去讀 `sha256Url`。
- `icon` 是 emoji，或由套件內圖示（png / svg / jpg / webp，32 KB 以內）轉成的 `data:image/...` URI；其他情況為 `null`，商店顯示預設圖示。
- `exchange`（1.4.0 起）是套件 `contributes.exchange` 的摘要：可接收的格式與定義的格式。資料交換找不到接收端時，「到商店尋找」用它篩選出可接收該格式的擴充；沒有交換宣告的擴充省略這個欄位。

### build-index.js 做了什麼

對 `extensions.json` 的每一筆：

1. 讀取 repo 的 Releases，略過 draft / prerelease，由新到舊找第一個附件齊全的版本。
2. 下載 `.tbx`、`.sha256`、`.sig`，驗證 SHA-256、以登記的公鑰驗證簽章，再確認套件內 `package.json` 的 id 與版本和登記資料、tag 一致。
3. 驗證失敗的版本不收錄，改看更舊的版本（最多往回 5 個）。都失敗就不收錄這個擴充，並寫進 job summary。
4. 讀取 GitHub API 時發生**連線錯誤**（5xx、逾時、次數用完）時，沿用上一版線上索引中的該筆資料，避免暫時性錯誤讓擴充從商店消失。404（repo 被刪除）或公鑰已更換時不沿用。

建置腳本不需要 `npm install`，只要 Node.js 20 以上。設定 `GITHUB_TOKEN` 可避開未驗證請求每小時 60 次的限制（Actions 內自動提供）；`GITHUB_API_URL` 可指向 GitHub Enterprise Server。

### 發佈方式：GitHub Pages

`registry/.github/workflows/build-index.yml` 把 `index.json` 部署到 GitHub Pages，**不 commit 回 repo**，理由：

- `index.json` 是建置產物，每小時重建。commit 回 repo 會產生大量機器人 commit，淹沒真正需要審核的 `extensions.json` 異動。
- commit 回 repo 需要給 workflow `contents: write`，push 又會觸發 push 事件，容易形成迴圈。Pages 部署只需要 `pages: write` 與 `id-token: write`，repo 內容維持唯讀。
- Pages 走 CDN、網址固定，適合當 ToolDock 的預設來源。

代價是歷史版本的 `index.json` 不在 git 中，需要時可查 Actions 的 artifact 與 job summary。

觸發時機：push 到 main（僅限 `extensions.json`、`scripts/`、`schema/` 或 workflow 異動）、每小時排程、手動、`repository_dispatch`（`event_type: extension-released`，擴充發佈後可主動通知）。PR 只做檢查與試建，不發佈。

## 平台端

### 來源設定

預設值在 `config/store.json`；使用者覆寫寫在 `userData/store.json`，**逐欄覆寫**預設值（`sources` 整組取代）。刪除使用者檔即可還原預設。這與 `lifecycle.json`「預設 + 使用者覆寫」的做法相同；差別在於 lifecycle 是整份取代，store 是逐欄合併，所以使用者只改檢查週期時，IT 之後調整的預設來源仍會生效。

```json
{
  "sources": [
    { "name": "公司內部", "url": "https://tooldock-registry.corp.example/index.json" },
    { "name": "ToolDock 官方", "url": "https://tooldock-dev.github.io/tooldock-registry/index.json" },
    { "name": "離線鏡像", "url": "file:///Volumes/Share/tooldock/index.json" }
  ],
  "autoCheckUpdates": true,
  "checkIntervalHours": 6,
  "cacheTtlMinutes": 30
}
```

- 同一個擴充 id 出現在多個來源時，以**排在前面的來源**為準。企業可以把內部來源放在最前面，以內部版本取代公開版本。
- 可在商店的「來源設定」頁面加入、移除來源與調整更新檢查，或用「編輯設定檔」直接修改。
- 支援 `https://`、`file://`；`http://` 需要 `allowInsecure`。

### 安裝流程

`Store.install(id)` 依序執行，任何一步失敗都中止並回報原因，已安裝的版本不受影響：

1. **下載** `.tbx`：`http(s)` 用 Electron 的 `net.fetch`（走系統 proxy 與憑證設定），`file://` 直接讀檔。上限 200 MB。
2. **SHA-256**：與索引的 `sha256` 比對。
3. **Ed25519 簽章**：下載 `.sig`，用索引登記的公鑰驗證。
4. **相容性與一致性**：讀取套件內的 `package.json`，確認 `tooldock.engine` 符合目前平台版本，且 id、版本與索引一致。
5. 寫到暫存檔，呼叫既有的 `ExtensionManager.install()`。

畫面上會依序顯示「下載 → SHA-256 → 簽章 → 相容性 → 安裝」的進度。

### 安裝確認與一鍵更新

- 第一次安裝時，先顯示**發佈者**（含是否已驗證）、**來源 repo**、商店來源、簽章金鑰指紋與權限警告，使用者確認後才安裝。
- 從商店安裝過的擴充會記錄 repo 與金鑰指紋。之後的更新若 repo 與金鑰都沒變，按「更新」即可完成（一鍵更新）；如果金鑰或 repo 變了，會再次顯示確認視窗並警告金鑰已變更。

### 更新檢查

- 在 `app:idle` 生命週期階段（工作台就緒 3 秒後）檢查一次，之後每 `checkIntervalHours` 小時檢查一次。
- 有新版時，在狀態列顯示「⬆ N 個擴充可更新」、活動列的商店圖示顯示數字、跳出一次提示；**不會自動安裝**。點狀態列會打開商店的「可更新」清單。
- 新版的 `engine` 不相容時不列為可更新，詳細頁會說明需要的平台版本。
- 也可以從命令面板執行「商店：檢查擴充更新」。

### IPC 與指令

IPC 路由（`src/main/main.js` 的 `router`）：`store.state`、`store.refresh`、`store.install`、`store.checkUpdates`、`store.addSource`、`store.removeSource`、`store.updateConfig`、`store.resetConfig`、`store.editConfig`、`store.reloadConfig`。主程序推給工作台的事件：`store.changed`、`store.updates`、`store.progress`。

命令面板：「商店：開啟擴充商店」（⇧⌘S）、「商店：檢查擴充更新」、「商店：重新整理索引」、「商店：編輯來源設定檔」。

### 自我測試

`npm run selftest` 會產生測試金鑰、簽出 `.tbx`，建立本機 `file://` 假索引，並驗證：`app:idle` 自動檢查、成功安裝、雜湊不符被拒、簽章不符被拒、engine 不相容被拒、透過商店畫面確認後安裝、偵測到可更新（不自動安裝）後一鍵更新。截圖為 `07-store.png` 到 `10-store-sources.png`。測試不連網。

## 上架流程

發佈者：

1. `npx tooldock-ext keygen` 產生金鑰，私鑰存為擴充 repo 的 Actions secret（見 `docs/RELEASE.md`）。
2. 推 tag `v<version>`，讓範本內建的 workflow 建立附件齊全的 Release。
3. 到索引 repo 開 PR，在 `extensions.json` 加入一筆（`verified` 填 `false`）。
4. 合併後最多約一小時出現在商店。之後發佈新版只要推 tag，不必再開 PR。

審核者依 `registry/CONTRIBUTING.md` 的審核清單逐項確認，重點包括：PR 作者是否為 repo 擁有者、`publisher` 是否冒名、公鑰是否與他人共用、`.tbx` 內容是否與原始碼相符、後端有無可疑行為。

## 建立索引 repo

`registry/` 是範本，本 repo 不會發佈它。要建立真正的索引：

1. 在 GitHub 建立新 repo（例如 `tooldock-dev/tooldock-registry`），把 `registry/` 的**內容**複製到 repo 根目錄（`.github/` 要在根目錄才會生效）。
2. repo 的 **Settings → Pages → Build and deployment → Source** 選 **GitHub Actions**。
3. 到 **Actions** 手動執行一次「建立索引」，確認 `https://<owner>.github.io/<repo>/index.json` 可以開啟。
4. 把 `config/store.json` 的預設來源改成上一步的網址。官方索引已建立於 [tooldock-dev/tooldock-registry](https://github.com/tooldock-dev/tooldock-registry)，預設來源為 `https://tooldock-dev.github.io/tooldock-registry/index.json`。
5. （選用）設定 branch protection：`main` 需經 PR 與至少一位審核者核准，並要求「檢查上架 PR」通過。

## 企業私有商店

依情況擇一或並用：

**A. 內部索引 repo（GitHub Enterprise Server 或 github.com 的私有組織）**
Fork `registry/` 範本，`extensions.json` 只放內部擴充。在 GHES 上 Actions 會自動提供 `GITHUB_API_URL`，build-index 會改用內部 API。`index.json` 發佈到內部 Pages 或任何內部 HTTPS 伺服器，再把網址放到來源清單的最前面。

> 注意：ToolDock 下載 `.tbx` 時**不帶 GitHub 憑證**。如果擴充 repo 是私有的，`browser_download_url` 無法直接下載，請改用方案 B，把套件鏡像到員工可讀取的位置。

**B. 共用磁碟 / 離線鏡像（`file://`）**
把 `index.json` 與所有附件放到同一個資料夾，`tbxUrl` 等網址改成相對路徑（例如 `files/my-tool-1.2.0.tbx`），來源設成 `file:///Volumes/Share/tooldock/index.json`（Windows：`file:///S:/tooldock/index.json`）。完全不需要網路，適合隔離網路環境。簽章仍會用索引中的公鑰驗證，所以鏡像過程中被改動的檔案會被拒絕。

**C. 由 IT 統一設定預設來源**
修改安裝版的 `resources/config/store.json`（與 `lifecycle.json` 相同位置），或用部署工具派送 `userData/store.json`。也可以把官方來源移除，只留內部來源。

## 限制

- **沒有沙箱**。擴充後端是一般的 Node.js 程序，可以讀寫使用者能存取的所有檔案、連到任何網址、執行任何程式。商店的驗證機制保護的是「傳遞過程」，不是「程式行為」。
- **簽章不等於安全**。簽章只證明套件出自持有私鑰的人。發佈者本身惡意、私鑰外流、或發佈者的 CI 被入侵時，簽章都會是有效的。
- **索引是單一信任根**。能修改索引 repo（或其 Pages 部署）的人，可以換掉任何擴充的公鑰。請對索引 repo 啟用 branch protection、要求 PR 審核，並限制有 Pages 部署權限的人。
- **沒有撤銷機制**。從索引移除擴充或更換公鑰後，已安裝的副本不會被移除或停用，只是不再收到更新。
- **沒有防降版與凍結保護**。商店只在索引版本較新時提示更新；但若索引被竄改成提供舊版，首次安裝會裝到舊版，攻擊者也能讓索引停在舊版本（凍結攻擊）。
- **安裝後不再驗證**。檔案寫入使用者擴充目錄後，平台不會再檢查完整性；本機上能寫入該目錄的程式都可以修改擴充。
- **不自動安裝**，也不處理擴充之間的相依關係。
- **私有 GitHub repo 的 Release 無法直接下載**（見〈企業私有商店〉）。
- **更新偵測有延遲**：索引每小時重建一次，GitHub Pages 另有約 10 分鐘的快取；平台端的索引快取預設 30 分鐘（`cacheTtlMinutes`），手動「重新整理」或「檢查更新」會略過快取。
- 自動更新檢查掛在 `app:idle` 階段。如果自訂的 `lifecycle.json` 移除了 `app:idle`，就不會自動檢查（手動檢查仍可用）。
- 下載使用 `net.fetch`，會套用系統 proxy；`file://` 來源則由主程序直接讀檔，不經過 `net.fetch`。
