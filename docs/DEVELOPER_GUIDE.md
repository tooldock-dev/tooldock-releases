# 開發人員操作指南：開發流程與上架說明

這份指南從零開始，一步步帶你完成 ToolDock 擴充的**開發、測試、打包、簽章、發佈，到上架商店與後續維護**。每個步驟都附上實際要執行的指令、預期結果，以及常見錯誤的處理方式。

- **適用對象**：要替 ToolDock 開發擴充，並發佈到官方商店或公司內部商店的開發人員。
- **不需要**平台原始碼，只要安裝好 ToolDock。
- 所有開發者文件都公開在 [tooldock-releases/docs](https://github.com/tooldock-dev/tooldock-releases/tree/main/docs)。
- API 參考請查 [EXTENSION_GUIDE.md](EXTENSION_GUIDE.md)；發佈檔格式見 [RELEASE.md](RELEASE.md)；商店架構見 [STORE.md](STORE.md)。管理者審核的方式見 [STORE_REVIEW.md](STORE_REVIEW.md)，建議上架前也讀一遍，先知道審核者會看什麼。

---

## 目錄

0. [整體流程](#0-整體流程)
1. [準備開發環境](#1-準備開發環境)
2. [建立擴充專案](#2-建立擴充專案)
3. [設定 manifest](#3-設定-manifest)
4. [開發與除錯](#4-開發與除錯)
5. [測試](#5-測試)
6. [上架前自我檢查](#6-上架前自我檢查)
7. [打包](#7-打包)
8. [產生簽章金鑰](#8-產生簽章金鑰)
9. [設定 GitHub repo](#9-設定-github-repo)
10. [發佈第一個 Release](#10-發佈第一個-release)
11. [申請上架商店](#11-申請上架商店)
12. [發佈新版本](#12-發佈新版本)
13. [變更上架資訊與下架](#13-變更上架資訊與下架)
14. [公司內部發佈](#14-公司內部發佈)
15. [疑難排解](#15-疑難排解)
16. [附錄](#16-附錄)

---

## 0. 整體流程

```
 ① 開發                 ② 發佈                       ③ 上架（只做一次）            ④ 上線
 ─────────              ─────────                    ─────────────────            ─────────
 建立專案                keygen 產生金鑰               fork 索引 repo                索引每小時重建
 npm run dev            私鑰 → GitHub secret          extensions.json 加一筆         商店出現你的擴充
 撰寫、除錯、測試         npm version patch            開 PR                         使用者安裝
 npm run validate       git push --follow-tags        審核者審查 → 合併               │
 npm run install-local  └─ CI：驗證/打包/簽章/Release                                 ▼
                                                                                ⑤ 之後每次改版：
                                                                                   只要推新的 tag，
                                                                                   不必再開 PR
```

| 階段 | 你要做的事 | 需要多久 |
|---|---|---|
| 開發 | 第 1–6 節 | 依功能而定 |
| 第一次發佈 | 第 7–10 節（金鑰與 GitHub 設定只做一次） | 約 30 分鐘 |
| 上架申請 | 第 11 節 | 送出 PR 約 10 分鐘；審核時間依管理者排程 |
| 上線 | 合併後自動 | 最多約 1 小時，加上快取最多再 40 分鐘 |
| 後續改版 | 第 12 節 | 推 tag 後自動，約 1 小時內出現在商店 |

---

## 1. 準備開發環境

### 1.1 需要的軟體與帳號

| 軟體 | 版本 | 用途 | 下載 | 確認方式 |
|---|---|---|---|---|
| ToolDock | 最新版（至少 1.1.0；依擴充用到的功能而定，見〈3.3〉） | 執行與測試擴充 | [tooldock-releases](https://github.com/tooldock-dev/tooldock-releases/releases/latest)，見〈1.2〉 | 選單「關於 ToolDock」 |
| Node.js | 18 以上（CI 使用 22，建議本機也用 22 LTS） | 建立專案、打包、執行 CLI | [nodejs.org](https://nodejs.org/) | `node -v` |
| npm | 隨 Node.js 安裝 | 安裝套件 | — | `npm -v` |
| Git | 任一近期版本 | 版本控制、建立 tag | [git-scm.com](https://git-scm.com/downloads) | `git --version` |
| GitHub CLI（`gh`） | 任一近期版本 | 建立 repo、設定 secret、查看 Release、開 PR；本指南的指令都用它 | [cli.github.com](https://cli.github.com/) | `gh --version` |
| GitHub 帳號 | — | 存放原始碼與 Release、申請上架 | [github.com](https://github.com/signup) | — |
| VS Code（建議） | — | 範本已附偵錯設定，按 F5 就能在後端設中斷點 | [code.visualstudio.com](https://code.visualstudio.com/) | — |

### 1.2 安裝 ToolDock

ToolDock 的安裝檔發佈在公開的 GitHub repo [tooldock-dev/tooldock-releases](https://github.com/tooldock-dev/tooldock-releases)。到 **[最新版下載頁](https://github.com/tooldock-dev/tooldock-releases/releases/latest)**，依作業系統選擇檔案：

| 作業系統 | 檔案 |
|---|---|
| macOS（Apple Silicon，M1 以後） | `ToolDock-<版本>-mac-arm64.dmg` |
| macOS（Intel） | `ToolDock-<版本>-mac-x64.dmg` |
| Windows 10 / 11（64 位元） | `ToolDock-<版本>-win-x64.exe` |

也可以用 GitHub CLI 下載：

```bash
gh release download --repo tooldock-dev/tooldock-releases --pattern "*mac-arm64.dmg" --pattern SHA256SUMS.txt
```

**macOS**

1. 打開 `.dmg`，把 ToolDock 拖到「應用程式」資料夾。
2. 第一次開啟時，在 ToolDock 上**按右鍵 →「打開」**，再在對話框按「打開」。ToolDock 目前還沒有經過 Apple 公證，直接點兩下會被系統阻擋。
3. 右鍵也打不開時，在「終端機」執行：

```bash
xattr -dr com.apple.quarantine /Applications/ToolDock.app
```

**Windows**

1. 執行 `ToolDock-<版本>-win-x64.exe`，可以選擇安裝位置（預設是 `%LOCALAPPDATA%\Programs\ToolDock`）。
2. 出現「Windows 已保護您的電腦」時，按**「其他資訊」→「仍要執行」**。安裝檔目前沒有程式碼簽章，所以會出現這個警告。

**確認檔案完整**（選用）：每個版本都附有 `SHA256SUMS.txt`。macOS 在下載資料夾執行：

```bash
shasum -a 256 -c SHA256SUMS.txt --ignore-missing
```

Windows 在 PowerShell 執行 `Get-FileHash .\ToolDock-<版本>-win-x64.exe -Algorithm SHA256`，把結果和 `SHA256SUMS.txt` 對照。

**確認版本**：開啟 ToolDock，從選單「關於 ToolDock」查看版本。你的擴充用到的功能需要的平台版本見〈3.3〉；平台有新版時，下載新的安裝檔覆蓋安裝即可，已安裝的擴充與資料不受影響。

### 1.3 安裝 Node.js、Git 與 GitHub CLI

**macOS**（使用 [Homebrew](https://brew.sh/)）：

```bash
brew install node@22 git gh
```

**Windows**（使用內建的 winget，在 PowerShell 執行）：

```powershell
winget install OpenJS.NodeJS.LTS Git.Git GitHub.cli
```

不使用套件管理工具時，到〈1.1〉表格中的下載網址安裝。安裝完成後**重新開啟終端機**，確認三個指令都能執行：

```bash
node -v && git --version && gh --version
```

登入 GitHub CLI（依畫面指示用瀏覽器登入）：

```bash
gh auth login
```

第一次使用 Git 時，設定提交者名稱與 email：

```bash
git config --global user.name "你的名字"
```

```bash
git config --global user.email "you@example.com"
```

### 1.4 ToolDock 安裝位置

CLI 會依序在以下位置尋找 ToolDock：

1. `--app <路徑>` 參數
2. 環境變數 `TOOLDOCK_PATH`
3. 預設安裝位置：macOS 的 `/Applications`、`~/Applications`；Windows 的 `%LOCALAPPDATA%\Programs`、`%ProgramFiles%`；Linux 的 `/opt/ToolDock`

裝在其他位置時，建議把 `TOOLDOCK_PATH` 寫進 shell 設定檔：

```bash
export TOOLDOCK_PATH=/Applications/ToolDock.app
```

### 1.5 安裝開發工具套件（npm）

ToolDock 的擴充開發工具以三個 npm 套件發佈，**不需要平台原始碼**，也不需要全域安裝：

| 套件 | 指令 | 用途 | 安裝方式 |
|---|---|---|---|
| `create-tooldock-extension` | `npm create tooldock-extension` | 專案產生器，建立新專案時使用一次 | 不必安裝，`npm create` 會自動下載最新版執行 |
| `tooldock-cli` | `tooldock-ext` | `dev`、`validate`、`pack`、`install`、`keygen`、`verify` | 專案的 `devDependencies` |
| `tooldock-types` | — | 後端 `ctx`、畫面 `window.tooldock` 與 manifest 的型別定義，讓編輯器有自動完成與型別檢查 | 專案的 `devDependencies` |

**新專案**：用〈2.3〉的 `npm create tooldock-extension@latest` 建立時，`tooldock-cli` 與 `tooldock-types` 會自動寫進 `devDependencies` 並安裝，不需要另外處理。加了 `--no-install` 的話，進入專案資料夾後自己執行一次：

```bash
npm install
```

**既有專案**（例如手動建立、或從舊範本建立的專案）：在專案根目錄安裝：

```bash
npm install -D tooldock-cli tooldock-types
```

並在 `package.json` 加入腳本，之後就能用 `npm run dev` 等指令：

```jsonc
"scripts": {
  "dev": "tooldock-ext dev",
  "debug": "tooldock-ext dev --inspect",
  "validate": "tooldock-ext validate",
  "pack": "tooldock-ext pack",
  "install-local": "tooldock-ext install"
}
```

要讓編輯器取得型別，新增 `jsconfig.json`（新專案已內建）：

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "es2022",
    "lib": ["es2022", "dom", "dom.iterable"],
    "checkJs": true,
    "skipLibCheck": true,
    "noEmit": true,
    "types": ["node", "tooldock-types/ui"],
    "maxNodeModuleJsDepth": 0
  },
  "exclude": ["node_modules", "dist"]
}
```

後端以 JSDoc 標註型別：`/** @param {import('tooldock-types').ExtensionContext} ctx */`。

**確認版本與升級**：新版的 `tooldock-cli` 會檢查新的 manifest 欄位（例如資料交換、通知、`mimes`），`tooldock-types` 會加入新 API 的型別。**舊版 CLI 不會檢查它不認得的欄位**，錯誤要到使用者安裝後才會發現，所以開發時請使用最新版：

```bash
npm ls tooldock-cli tooldock-types
```

```bash
npm install -D tooldock-cli@latest tooldock-types@latest
```

升級後請提交更新過的 `package.json` 與 `package-lock.json`，CI 才會用相同的版本。

> - `tooldock-cli` 沒有任何相依套件，安裝很快，也不會把東西帶進擴充：`devDependencies` 一律不打包。
> - 不建議全域安裝（`npm install -g`）。每個專案各自鎖定版本，CI 與團隊成員才會用相同的工具。臨時在專案外使用時，可以用 `npx -p tooldock-cli tooldock-ext <指令>`。
> - 公司網路需要經過內部 npm 鏡像（例如 Artifactory、Nexus、Verdaccio）時，請確認鏡像有同步這三個套件，並在 `.npmrc` 設定 `registry`。

### 1.6 Windows 使用者

本指南的指令以 macOS 與 Linux 的 shell（bash / zsh）撰寫。在 Windows 上有兩種做法：

**做法 A（建議）：使用 Git Bash**。安裝 Git for Windows 時會一起安裝 Git Bash，本指南的指令幾乎都能直接執行。差別只在路徑：家目錄的 `~` 對應 `C:\Users\<你的帳號>`，ToolDock 位置的寫法是 `export TOOLDOCK_PATH="$LOCALAPPDATA/Programs/ToolDock/ToolDock.exe"`。

**做法 B：使用 PowerShell**。`npm`、`npx`、`git`、`gh` 指令都一樣，以下幾個寫法不同：

| 用途 | bash / zsh（本指南） | PowerShell |
|---|---|---|
| 設定 ToolDock 位置 | `export TOOLDOCK_PATH=…` | `$env:TOOLDOCK_PATH = "$env:LOCALAPPDATA\Programs\ToolDock\ToolDock.exe"` |
| 金鑰資料夾 | `~/.tooldock/keys` | `$HOME\.tooldock\keys` |
| 設定簽章 secret（〈9.1〉） | `gh secret set TOOLDOCK_SIGNING_KEY < ~/.tooldock/keys/tooldock-signing-key.pem` | `Get-Content $HOME\.tooldock\keys\tooldock-signing-key.pem -Raw \| gh secret set TOOLDOCK_SIGNING_KEY` |
| 設定公鑰 variable（〈9.1〉） | `gh variable set TOOLDOCK_PUBLIC_KEY --body "$(cat …pub)"` | `gh variable set TOOLDOCK_PUBLIC_KEY --body (Get-Content $HOME\.tooldock\keys\tooldock-signing-key.pub -Raw).Trim()` |
| 驗證套件（〈10.4〉） | `--public-key "$(cat …pub)"` | `--public-key (Get-Content $HOME\.tooldock\keys\tooldock-signing-key.pub -Raw).Trim()` |
| 查看套件內容（〈7.2〉） | `unzip -l dist/…tbx` | `tar -tf dist\…tbx` |
| 串接兩個指令 | `a && b` | PowerShell 7 可用 `a && b`；Windows PowerShell 5 請分兩行執行 |
| 暫存資料夾 | `/tmp/…` | `$env:TEMP\…` |

---

## 2. 建立擴充專案

### 2.1 決定擴充 id

id 是擴充在整個商店中的**唯一識別碼**，上架後不能更改（換 id 等於新的擴充），請先想清楚：

- 只能使用**小寫英數、`.` 與 `-`**，以英數開頭，最長 63 字。
- 不要與內建擴充相同：`data-converter`、`markdown-editor`、`site-monitor`、`table-report`。
- 不要與商店上既有的擴充、知名產品名稱混淆（審核者會檢查）。可以加上公司或個人前綴，例如 `acme-invoice`。
- 自訂資料交換格式、通知通道、指令 id 都會以它為前綴，越短越好。

### 2.2 選擇範本

| 範本 | 適用情境 | 內容 |
|---|---|---|
| `view` | 有分頁畫面，前端呼叫後端處理資料（大多數工具） | 畫面加後端，示範 `invoke`、`emit`、`storage`、對話框、指令、設定、生命週期 |
| `service` | 隨平台啟動的背景服務：定時執行、狀態列顯示 | 定時工作、狀態列、暫停與恢復、休眠與喚醒 |

### 2.3 建立專案

```bash
npm create tooldock-extension@latest my-tool
```

執行後會詢問 id、顯示名稱和範本。也可以一次帶齊參數：

```bash
npm create tooldock-extension@latest my-tool -- --id my-tool --name "我的工具" --template view -y
```

| 參數 | 說明 |
|---|---|
| `--id` | 擴充 id |
| `--name` | 顯示名稱 |
| `--template` | `view` 或 `service` |
| `-y` | 其餘全部使用預設值 |
| `--no-install` | 不自動執行 `npm install` |

完成後的專案結構：

```
my-tool/
├── package.json                    manifest（tooldock 欄位）與 dev / debug / validate / pack 腳本
├── backend/index.js                後端：activate(ctx) / deactivate()
├── ui/                             畫面（view 範本才有）
├── jsconfig.json                   型別檢查設定（tooldock-types）
├── .vscode/                        F5 啟動並偵錯
├── .github/workflows/release.yml   推送 v* tag 時自動發佈
├── .tbxignore                      打包時排除的檔案
├── .gitignore                      已排除 node_modules、dist、簽章私鑰
└── README.md
```

專案已內建三個開發工具套件（都是 `devDependencies`，不會被打包；手動安裝與升級見〈1.5〉）：

| 套件 | 用途 |
|---|---|
| `tooldock-cli`（指令 `tooldock-ext`） | `dev`、`validate`、`pack`、`install`、`keygen`、`verify` |
| `tooldock-types` | `ctx`、`window.tooldock` 與 manifest 的型別定義 |
| `@types/node` | Node.js 型別 |

### 2.4 建立 GitHub repo

上架商店的條件之一是**公開的 GitHub repo**，審核者會比對原始碼與發佈的套件。建議 repo 名稱用 `tooldock-ext-<id>`，例如 `tooldock-ext-my-tool`。

```bash
cd my-tool
```

```bash
git init && git add -A && git commit -m "初始版本"
```

```bash
gh repo create tooldock-ext-my-tool --public --source . --push
```

> **`package-lock.json` 一定要提交**。發佈的 CI 用 `npm ci` 安裝相依套件，沒有 lock 檔就會失敗。

---

## 3. 設定 manifest

manifest 就是 `package.json` 的 `tooldock` 欄位。完整欄位見 [EXTENSION_GUIDE.md〈2〉](EXTENSION_GUIDE.md#2-manifestpackagejson)，這裡只說明上架時特別重要的部分。

### 3.1 基本欄位

```jsonc
{
  "name": "my-tool",
  "version": "0.1.0",                  // x.y.z，發佈時 tag 必須是 v0.1.0
  "description": "一句話說明用途",
  "license": "MIT",                    // 上架需要授權條款
  "main": "backend/index.js",          // 純畫面擴充可以省略
  "tooldock": {
    "id": "my-tool",                   // 與商店登記的 id 必須相同
    "displayName": "我的工具",          // 商店與工作台顯示的名稱
    "description": "商店列表顯示的說明",
    "icon": "🧩",                      // emoji，或套件內的 svg / png（32 KB 以內才會顯示在商店）
    "engine": "^1.4.0",                // 需要的平台版本，見〈3.3〉
    "contributes": { … }
  }
}
```

### 3.2 版本號

使用語意化版本 `主版號.次版號.修訂號`：

| 變更內容 | 調整 | 指令 |
|---|---|---|
| 修正錯誤，不影響使用方式 | 修訂號 | `npm version patch` |
| 新增功能，相容舊版的設定與資料 | 次版號 | `npm version minor` |
| 不相容的變更：設定或資料格式改變、移除功能、提高 `engine` 主版號 | 主版號 | `npm version major` |
| 測試版 | 加後綴，例如 `1.2.0-beta.1` | `npm version 1.2.0-beta.1` |

含 `-` 的版本會被標成**預先發行版**，商店**不會收錄**，適合給測試人員手動下載安裝。

### 3.3 選擇 `engine`

`engine` 決定哪些平台版本可以安裝你的擴充。商店安裝前會檢查，不符合就拒絕安裝；有新版但 `engine` 不相容時，也不會提示更新。

支援的寫法：`*`、`1.2.3`（只允許這一版）、`^1.2.3`（1.2.3 以上、2.0.0 以下，**建議**）、`~1.2.3`（1.2.x）、`>=1.2.3`。

依照用到的功能，選擇最低需要的平台版本：

| 功能 | 最低平台版本 | 舊平台會怎樣 |
|---|---|---|
| 基本功能：畫面、指令、設定、生命週期、`storage` | 1.0.0 | — |
| `ctx.secrets`（以鑰匙圈保護的機密資料） | 1.2.0 | 沒有 `ctx.secrets` |
| `keepAlive`、自動回收 | 1.3.0 | 忽略 |
| 資料交換 `ctx.exchange`、`contributes.exchange` | 1.4.0 | 忽略，不會被呼叫 |
| 資源用量顯示 | 1.5.0 | — |
| 通知服務 `ctx.notify`、執行鎖、`guarded`、`permissions` | 1.6.0 | 忽略；**受保護指令不會詢問使用者** |
| PDF 輸出 `ctx.render`、交換的 `mimes` | 1.7.0 | 沒有 `ctx.render`；忽略 `mimes` |

兩種做法擇一：

- **提高 `engine`**：例如用到通知服務就設 `^1.6.0`。最簡單，但舊平台的使用者無法安裝。
- **維持較低的 `engine`，在程式裡判斷**：例如 `if (ctx.notify) { … } else { ctx.window.notify(…) }`。`npm run validate` 會在 `engine` 包含舊版平台時提出警告，提醒你需要判斷。

### 3.4 啟用事件

預設是**延遲載入**：開啟畫面、執行指令、前端呼叫時才啟動，用完自動回收。只有真正的背景服務才宣告 `onStartup`。`validate` 會對 `onStartup` 提出警告，審核者也會詢問理由。

### 3.5 用途宣告 `permissions`（1.6.0 起）

擴充會發通知、或會呼叫其他擴充的受保護指令時，**請宣告用途**。商店的安裝確認畫面會顯示給使用者看，審核者也會比對程式碼是否與宣告一致。

```jsonc
"tooldock": {
  "permissions": {
    "notify": { "reason": "網站異常或恢復時通知你", "testCommand": "myTool.testNotify" },
    "commands": [{ "id": "telegram.send", "reason": "把日報發到 Telegram" }]
  }
}
```

- `reason` 必填，最多 200 字，寫給一般使用者看。
- `testCommand` 選填，必須是自己宣告的指令。安裝後平台會引導使用者執行它，完成第一次確認。
- 宣告**不會**取得授權，只是說明。

### 3.6 受保護指令 `guarded`

你的指令如果會造成外部影響（發訊息、寫資料庫、刪除資料），而且**其他擴充可以呼叫**，就要標示 `"guarded": true`。特別是替別人「代發」通知或代為執行動作的指令，不標示就等於把你的授權借給所有擴充，審核者會要求修正。

```jsonc
"commands": [{ "id": "myTool.purge", "title": "我的工具：清除資料", "guarded": true }]
```

---

## 4. 開發與除錯

### 4.1 啟動開發模式

```bash
npm run dev
```

| 特性 | 說明 |
|---|---|
| 獨立資料目錄 | 使用 `ToolDock-ExtDev`，可以和平常使用的 ToolDock 同時開啟，互不影響 |
| 開發版優先 | 開發中的擴充優先於已安裝或內建的同 id 版本，並在啟動時就啟用，錯誤立即出現 |
| 存檔即重載 | 存檔後自動重新載入後端程序與已開啟的畫面 |
| 不自動回收 | 開發中的擴充不會被閒置回收 |
| 狀態列標示 | 狀態列變成橘色，顯示「🛠 擴充開發」 |

想從全新狀態開始（例如測試第一次安裝的引導流程）：

```bash
npm run dev -- --fresh
```

### 4.2 除錯

| 對象 | 方法 |
|---|---|
| 後端 | VS Code 按 F5，選「ToolDock：啟動並偵錯擴充」，可以設中斷點；重載後除錯器自動重連 |
| 後端（不用 VS Code） | `npm run debug`，再到 Chrome 開啟 `chrome://inspect` |
| `activate()` 本身 | `npm run dev -- --inspect-brk`，停在第一行等待除錯器 |
| 畫面 | 在分頁上按右鍵 →「檢查此畫面」 |
| 日誌 | `ctx.log.*` 與 `console.log` 顯示在輸出面板（`⇧⌘U`）和執行 `npm run dev` 的終端機 |
| 狀態 | 擴充詳細頁：PID、啟用原因、偵錯埠、記憶體與 CPU，以及停止、啟動、重新載入按鈕 |

### 4.3 撰寫程式

- 後端 `ctx` API、畫面 `window.tooldock` API：見 [EXTENSION_GUIDE.md〈3〉〈4〉](EXTENSION_GUIDE.md#3-後端-apictx)。
- 在檔案開頭加上 `// @ts-check`，編輯器就會依 `tooldock-types` 檢查型別。
- 需要保留的資料放在 `ctx.storage` 或 `ctx.paths.data`；`ctx.paths.temp` 與記憶體中的狀態在擴充被回收時會清除。
- 密碼、權杖一律用 `ctx.secrets`，不要寫進 `storage` 或設定。
- 計時器、連線、監看器請放進 `ctx.subscriptions`，停用時才會釋放。
- 錯誤訊息、通知內容請寫中文，讓使用者看得懂。

### 4.4 加入函式庫

```bash
npm install <套件>
```

| 規則 | 原因 |
|---|---|
| 執行時需要的套件放 `dependencies` | 打包時只帶走 `dependencies`（含完整相依樹） |
| 只在開發、測試、建置用的套件放 `devDependencies`（`npm install -D`） | 不會被打包，套件更小 |
| 避免原生模組（例如 `better-sqlite3`） | 需要依 Electron 版本重新編譯，平台升級後可能無法載入。改用 WebAssembly 或純 JS 的替代套件（例如 `sql.js`） |
| 避免帶有 `postinstall` 等安裝腳本的套件 | 審核者會檢查，有安裝腳本需要說明理由 |

### 4.5 使用前端框架（React、Vue…）

畫面可以用任何框架，但**發佈的 CI 只會執行 `npm ci`、`validate`、`pack`，不會幫你建置前端**。請擇一：

- **提交建置結果**：把建置後的檔案放在 `ui/`（例如 `ui/vendor/app.js`）並提交到 git，原始碼放在 `src-ui/` 並列入 `.tbxignore`。這是 Oracle SQL 工具的做法。
- **在 CI 建置**：在 `.github/workflows/release.yml` 的「驗證 manifest」之前加一個步驟，例如 `run: npm run build`。

不論哪一種，建置用的工具都放在 `devDependencies`。審核者會檢查 minified 的檔案是否能對應到 repo 中的原始碼，請保留建置腳本。

---

## 5. 測試

### 5.1 manifest 檢查

```bash
npm run validate
```

會檢查 manifest、進入點、畫面檔、指令註冊、設定型別、相依是否已安裝、資料交換與通知的宣告，以及用途宣告。

- **錯誤（✘）**：必須修正，否則 CI 會失敗，平台也可能忽略相關功能。
- **警告（⚠）**：不一定要改，但請確認理由。審核者可能會詢問。

### 5.2 單元測試（建議）

用 Node.js 內建的測試工具，不需要額外套件：

```jsonc
// package.json
"scripts": {
  "test": "node --test test/*.test.js"
}
```

把不依賴 `ctx` 的邏輯（解析、轉換、計算）拆成獨立模組，最容易測試。接著在 `.github/workflows/release.yml` 的「驗證 manifest」之前加入測試，測試失敗就不會發佈：

```yaml
      - name: 單元測試
        run: npm test
```

### 5.3 以安裝版測試

開發模式下有些行為不同（不回收、開發版優先），發佈前一定要用**打包後的套件**在一般模式測一次：

```bash
npm run install-local
```

這會打包並安裝到本機的 ToolDock。ToolDock 執行中時，安裝會轉交給執行中的視窗。

確認項目：

- [ ] 從擴充管理看到正確的名稱、圖示、版本與說明。
- [ ] 每個畫面都能開啟，每個指令都能從命令面板執行。
- [ ] 關閉所有畫面後程序會結束（延遲載入的擴充），再次開啟能正常恢復，資料沒有遺失。
- [ ] 設定頁的每個設定都能修改並生效。
- [ ] 會發通知的擴充：第一次發送時有執行鎖確認，選「拒絕」時擴充不會出錯。
- [ ] 資料交換：分享與接收都正常；沒有接收端時能存成 `.tdx`。
- [ ] 錯誤情況（檔案不存在、網路斷線、格式錯誤）有清楚的中文訊息，不會讓擴充崩潰。
- [ ] 移除擴充後，平台沒有殘留錯誤。

### 5.4 相容性測試

- **平台版本**：至少在 `engine` 允許的**最低版本**與**目前最新版本**各測一次。
- **作業系統**：使用者可能用 macOS、Windows 或 Linux。路徑請用 `path.join`，不要寫死 `/` 或 `\`；不要假設有 `bash` 或特定外部程式。
- **全新安裝**：`npm run dev -- --fresh` 或用 `--user-data-dir` 指定空資料夾，確認第一次使用的流程正常。

---

## 6. 上架前自我檢查

審核者會依 [STORE_REVIEW.md](STORE_REVIEW.md) 逐項檢查。先自己檢查一遍，可以省下來回修改的時間。

### 6.1 repo 與文件

- [ ] repo 是**公開**的，所有原始碼都在裡面（包含前端原始碼與建置腳本）。
- [ ] 有 `LICENSE` 檔案，`package.json` 的 `license` 欄位一致。
- [ ] `README.md` 清楚寫出（審核必看）：
  - 擴充的用途與使用方式
  - 會**讀寫哪些檔案或資料夾**
  - 會**連到哪些網路位址**（API、更新檢查、遙測…）；不連網也請明確寫出來
  - 會**執行哪些外部程式**（`child_process`）
  - 會發哪些通知、呼叫哪些其他擴充的指令
- [ ] `package-lock.json` 已提交。

### 6.2 程式碼

- [ ] 沒有下載並執行遠端程式碼：`eval`、`new Function`、`require` 下載來的檔案、`vm.runIn*` 執行遠端內容。
- [ ] 沒有讀取與功能無關的檔案：瀏覽器資料、SSH 金鑰、其他擴充的資料夾、使用者家目錄的其他設定。
- [ ] 沒有把資料傳到 README 沒有說明的位址。
- [ ] 沒有混淆過的程式碼；minified 的檔案都能從 repo 中的原始碼建置出來。
- [ ] 沒有 repo 中找不到原始碼的二進位檔（`.node`、`.exe`、`.dll`、`.so`…）。
- [ ] `package.json` 沒有 `postinstall`、`preinstall`、`install` 腳本。
- [ ] 機密資料用 `ctx.secrets` 保存；日誌中沒有印出密碼或權杖。
- [ ] `npm audit --omit=dev` 沒有高風險的已知漏洞（有的話請升級套件或說明影響）。

### 6.3 manifest

- [ ] `tooldock.id` 與要登記的 id 相同，沒有與內建或既有擴充混淆。
- [ ] `engine` 符合實際用到的功能（〈3.3〉）。
- [ ] 沒有不必要的 `onStartup`、`keepAlive`。
- [ ] 會發通知的擴充有 `permissions.notify`；會呼叫其他擴充受保護指令的有 `permissions.commands`。
- [ ] 會造成外部影響、且其他擴充可呼叫的指令標示了 `guarded`。
- [ ] `npm run validate` 沒有錯誤，警告都有理由。

### 6.4 特定類型擴充的額外要求

| 擴充類型 | 額外要求 |
|---|---|
| 接收 `tooldock.sql` | 必須做到 [EXCHANGE.md〈6.8〉](EXCHANGE.md#68-接收端的必要行為) 的七項：預覽並確認、不接受帳號密碼、語句分類、試跑、防止重複匯入、稽核紀錄、參數綁定。可以參考 [Oracle SQL 工具](https://github.com/tooldock-dev/tooldock-ext-oracle-sql) |
| 通知通道（`contributes.notify.channels`） | 通道 id 以自己的擴充 id 開頭；`destinations` 的 id 要穩定；60 秒內回應 `deliver`；README 說明訊息會送到哪個服務 |
| 產生 PDF 且使用 `allowRemote` 或 `javascript: true` | README 說明為什麼需要連網或執行 JavaScript |
| 背景服務（`onStartup`） | README 說明常駐的原因、會定時做什麼、多久一次 |

---

## 7. 打包

```bash
npm run pack
```

打包前會先執行 `validate`，然後產生 `dist/<id>-<版本>.tbx`。

### 7.1 套件包含什麼

| 包含 | 排除 |
|---|---|
| 專案中所有檔案 | `.git`、`.github`、`.vscode`、`.idea`、`dist/`、`*.tbx`、`*.log`、`*.map` |
| `dependencies` 與 `optionalDependencies` 的完整相依樹 | `devDependencies`（例如 `tooldock-cli`、`tooldock-types`） |
| | `package-lock.json`、`.gitignore`、`.tbxignore`、`.DS_Store` |
| | 簽章私鑰 `tooldock-signing-key*.pem`（一律排除） |
| | `.tbxignore` 列出的檔案 |

`.tbxignore` 使用簡化的 gitignore 語法，例如：

```
test/
src-ui/
scripts/
*.psd
```

### 7.2 檢查套件內容

`.tbx` 是 zip 檔，最上層資料夾是擴充 id。發佈前請看一下內容，確認沒有多帶測試資料、個人設定或機密：

```bash
unzip -l dist/my-tool-0.1.0.tbx
```

套件上限 200 MB，但請盡量精簡：越小下載越快，審核也越快。

---

## 8. 產生簽章金鑰

商店只收錄**有簽章**的套件，並用你登記的公鑰驗證。金鑰只要產生一次，可以用在你的所有擴充。

```bash
npx tooldock-ext keygen
```

| 產生的檔案（預設在 `~/.tooldock/keys/`） | 內容 | 處理方式 |
|---|---|---|
| `tooldock-signing-key.pem` | 私鑰（權限 600） | **絕對不要提交到 git、不要分享**。另外備份到密碼管理器 |
| `tooldock-signing-key.pub` | 公鑰字串（`MCowBQYDK2VwAyEA` 開頭，60 個字元） | 可以公開，上架時填進索引 |

- 資料夾裡已有金鑰時會拒絕覆寫；要重新產生請加 `--force`（舊金鑰會被覆蓋，請先確認）。
- 指定其他資料夾：`npx tooldock-ext keygen --out ./keys`（請確認該資料夾不會被提交）。
- **私鑰遺失**：只能產生新金鑰，並申請更換索引中的公鑰（〈13.1〉）。已安裝的使用者更新時會看到「金鑰已變更」的警告。
- **私鑰外流**：立即依〈13.4〉處理。

---

## 9. 設定 GitHub repo

### 9.1 設定 secret 與 variable

| 種類 | 名稱 | 內容 | 必要 |
|---|---|---|---|
| Secret | `TOOLDOCK_SIGNING_KEY` | 私鑰檔的**完整內容**，包含 `-----BEGIN PRIVATE KEY-----` 與 `-----END PRIVATE KEY-----` 兩行 | 上架商店必要 |
| Variable | `TOOLDOCK_PUBLIC_KEY` | 公鑰字串。設定後 CI 會在發佈前再驗證一次，及早發現 secret 設錯 | 建議 |

```bash
gh secret set TOOLDOCK_SIGNING_KEY < ~/.tooldock/keys/tooldock-signing-key.pem
```

```bash
gh variable set TOOLDOCK_PUBLIC_KEY --body "$(cat ~/.tooldock/keys/tooldock-signing-key.pub)"
```

也可以到 GitHub repo 的「Settings → Secrets and variables → Actions」手動新增。

### 9.2 確認 workflow 權限

workflow 已宣告 `permissions: contents: write`，不需要另外建立 token。如果組織把 Actions 的預設權限限制為唯讀，請到「Settings → Actions → General → Workflow permissions」確認允許 workflow 自行宣告權限。

---

## 10. 發佈第一個 Release

### 10.1 建立版本並推送

確認所有變更都已提交，然後：

```bash
npm version 1.0.0
```

```bash
git push --follow-tags
```

`npm version` 會修改 `package.json` 的版本、建立 commit，並建立 `v1.0.0` tag。之後也可以用 `patch`、`minor`、`major`（見〈3.2〉）。

> 不要手動建立 tag。**tag 必須等於 `v` 加上 `package.json` 的 `version`**，不一致時 CI 會失敗。

### 10.2 CI 做了什麼

推送 tag 後，GitHub Actions 的「Release」workflow 會依序執行：

| 步驟 | 內容 | 失敗時 |
|---|---|---|
| 1 | `npm ci` | lock 檔沒提交或與 `package.json` 不一致 |
| 2 | （若有加）`npm test` | 測試失敗 |
| 3 | `npm run validate` | manifest 有錯誤 |
| 4 | 檢查 tag 與版本一致 | 顯示「版本不一致」 |
| 5 | `tooldock-ext pack --sign`，產生 `.tbx`、`.sha256`、`.sig` | 私鑰格式錯誤。沒有 secret 時只產生 `.tbx`、`.sha256`，並警告「未簽章，商店不會收錄」 |
| 6 | 有 `TOOLDOCK_PUBLIC_KEY` 時，以公鑰驗證簽章 | secret 與公鑰不是同一組 |
| 7 | 建立 GitHub Release 並上傳附件；版本含 `-` 時標為預先發行版 | 權限不足 |

到 repo 的「Actions」頁面查看進度；完成後在 job summary 會列出發佈的檔案。

### 10.3 確認 Release

```bash
gh release view v1.0.0
```

附件必須**三個都在**：

| 檔名 | 內容 |
|---|---|
| `my-tool-1.0.0.tbx` | 擴充套件 |
| `my-tool-1.0.0.tbx.sha256` | SHA-256 |
| `my-tool-1.0.0.tbx.sig` | Ed25519 簽章 |

### 10.4 自己驗證一次

模擬商店的驗證流程，下載 Release 附件並以公鑰驗證：

```bash
gh release download v1.0.0 --dir /tmp/my-tool-release
```

```bash
npx tooldock-ext verify /tmp/my-tool-release/my-tool-1.0.0.tbx --public-key "$(cat ~/.tooldock/keys/tooldock-signing-key.pub)"
```

顯示驗證通過即可。也可以把下載的 `.tbx` 用「檔案 → 從套件安裝擴充」裝一次，確認是正常的。

### 10.5 重新發佈同一版

- **CI 中途失敗、修正 workflow 設定後**：在 Actions 頁面按「Re-run jobs」。Release 已存在時，CI 會覆寫附件。
- **程式碼有錯**：不要移動已推送的 tag，請改發新版本（例如 `1.0.1`）。使用者和索引可能已經取得舊的檔案，移動 tag 會讓 SHA-256 與已記錄的不一致。
- **只有 tag 打錯**（版本不一致，還沒產生 Release）：刪除 tag 後重新建立。

```bash
git tag -d v1.0.0 && git push origin :refs/tags/v1.0.0
```

---

## 11. 申請上架商店

上架只需要申請**一次**。合併之後，你每次發佈新版本都會自動出現在商店。

### 11.1 前提

- [ ] 已有**正式**（不是 draft、不是 prerelease）的 Release，三個附件齊全（〈10.3〉）。
- [ ] 已完成〈6〉的自我檢查。
- [ ] 你是 repo 的擁有者，或該組織的成員。**PR 必須由 repo 擁有者或組織成員開**，審核者會核對身分。

### 11.2 修改 `extensions.json`

官方索引 repo 是 [tooldock-dev/tooldock-registry](https://github.com/tooldock-dev/tooldock-registry)（公司內部商店請改用公司的索引 repo）。

```bash
gh repo fork tooldock-dev/tooldock-registry --clone
```

```bash
cd tooldock-registry && git switch -c add-my-tool
```

在 `extensions.json` 陣列中加入一筆：

```json
{
  "id": "my-tool",
  "repo": "your-name/tooldock-ext-my-tool",
  "publisher": "你的名字或組織",
  "publicKey": "MCowBQYDK2VwAyEA…",
  "verified": false
}
```

| 欄位 | 填寫方式 |
|---|---|
| `id` | 與套件 `package.json` 的 `tooldock.id` **完全相同** |
| `repo` | 發佈 Release 的 GitHub repo，格式 `owner/name` |
| `publisher` | 顯示給使用者的發佈者名稱（1–100 字）。必須與 repo 擁有者一致，不能冒用他人或公司名稱 |
| `publicKey` | `~/.tooldock/keys/tooldock-signing-key.pub` 的內容 |
| `verified` | **一律填 `false`**，由審核者決定是否改為 `true` |

只能有這五個欄位。送出前在本機檢查：

```bash
node scripts/validate.js
```

想在送出前確認你的 Release 能被收錄，可以試建索引（需要 Node.js 20 以上）：

```bash
GITHUB_TOKEN=$(gh auth token) node scripts/build-index.js --out dist/index.json
```

終端機出現「`my-tool：收錄 v1.0.0（v1.0.0，驗證通過）`」就表示沒問題。

### 11.3 開 PR

```bash
git commit -am "上架 my-tool" && git push -u origin add-my-tool
```

```bash
gh pr create --repo tooldock-dev/tooldock-registry --fill
```

PR 會自動帶出範本，請填寫擴充 id、repo、異動類型（新上架），並勾選「發佈者自我檢查」的每一項。

### 11.4 PR 送出之後

1. **自動檢查**：「檢查上架 PR」job 會檢查格式、公鑰，並試建索引。結果在 PR 的 Checks 頁面與 job summary。失敗時請依訊息修正後再 push 到同一個分支。
2. **人工審核**：審核者會檢查身分、下載套件比對原始碼、審查程式碼並實際安裝測試。可能會在 PR 留言要求修改，例如補充 README、加上 `guarded`、移除不必要的 `onStartup`。
   - 修改擴充本身：在擴充 repo 修正並**發佈新版本**，然後在 PR 留言告知新版本號。
   - 修改登記資料：直接 push 到 PR 分支。
3. **合併**：審核通過後由審核者合併。

### 11.5 確認上架

合併後，索引會在下一次建置時收錄你的擴充：

| 延遲來源 | 時間 |
|---|---|
| 索引建置 | 合併時會觸發一次；排程每小時一次 |
| GitHub Pages 快取 | 約 10 分鐘 |
| ToolDock 端索引快取 | 預設 30 分鐘；按商店的「重新整理」會略過快取 |

在 ToolDock 開啟商店（`⇧⌘S`），按「重新整理」，搜尋你的擴充，確認名稱、說明、版本、發佈者都正確，並實際從商店安裝一次。

---

## 12. 發佈新版本

上架之後，**發佈新版本不需要再開 PR**：

```bash
npm version patch
```

```bash
git push --follow-tags
```

### 12.1 新版本如何到達使用者

1. CI 建立新的 Release（〈10.2〉）。
2. 索引每小時重建時，取最新一個附件齊全、驗證通過的**正式** Release。驗證失敗時改看較舊的版本（最多往回 5 個），所以壞掉的新版本不會讓擴充從商店消失，但使用者也拿不到新版。
3. 使用者的 ToolDock 在閒置時與每 6 小時（預設）檢查更新，有新版時在狀態列顯示「⬆ N 個擴充可更新」。**不會自動安裝**，使用者按「更新」才會安裝。
4. repo 與金鑰都沒變時，使用者按一下就能完成更新；金鑰或 repo 變了，會重新顯示安裝確認並警告。

### 12.2 改版注意事項

- **資料相容**：使用者升級後，`ctx.storage`、`ctx.paths.data` 的舊資料仍在。資料格式改變時請在 `activate()` 中做轉換，並保留讀取舊格式的能力。
- **設定相容**：不要改既有設定的 key 或型別；要改請新增 key，並讀取舊值做轉換。
- **提高 `engine`**：舊平台的使用者不會看到這個更新（不列為可更新）。請在 Release 說明中寫清楚需要的平台版本，並考慮升主版號。
- **新增用途宣告**（例如新版開始發通知）：在 README 與 Release 說明中告知使用者。
- **測試版**：用 `1.2.0-beta.1` 這類版本號，商店不會收錄，可以把 Release 連結給測試人員手動安裝。
- **Release 說明**：CI 會自動產生 commit 列表。建議補上使用者看得懂的變更摘要，特別是不相容的變更與新的權限。

### 12.3 撤回有問題的版本

商店沒有撤回機制，已更新的使用者不會被降版。請**立即發佈修正版**（版本號要更大）。若問題嚴重（例如資料遺失或安全問題），同時把有問題的 Release 改為 prerelease 或刪除，索引下次建置時就會改提供較舊的版本，並在 repo 開 issue 告知使用者。

---

## 13. 變更上架資訊與下架

以下變更都需要對索引 repo 開 PR，並由 **repo 擁有者**提出。PR 範本的「異動類型」請勾選對應項目。

### 13.1 更換公鑰（金鑰輪替）

1. 產生新金鑰：`npx tooldock-ext keygen --out ~/.tooldock/keys-new`。
2. **先在原 repo 公告**：開一個 issue 或在 Release 說明中寫明「將更換簽章金鑰」，附上新公鑰字串與原因。
3. 把 GitHub secret `TOOLDOCK_SIGNING_KEY`（與 variable `TOOLDOCK_PUBLIC_KEY`）換成新金鑰，**發佈一個新版本**。
4. 開 PR 更新 `extensions.json` 的 `publicKey`，在 PR 中附上公告連結與步驟 3 的 Release 連結。

> 順序很重要：索引只用登記的公鑰驗證。步驟 3 的新版本在 PR 合併前會驗證失敗，索引會自動改提供較舊的舊金鑰版本，所以商店不會中斷；PR 合併後，新版本就會通過驗證並被收錄。反過來先合併 PR 的話，舊金鑰簽的版本立刻失效，在你發佈新版本之前，擴充會從商店消失。

使用者更新時，商店會因為金鑰不同而重新顯示確認畫面。

### 13.2 更換 repo

例如從個人帳號移到組織。**視同重新上架**，審核者會重新審查全部項目。在 PR 中說明原因，並在舊 repo 的 README 註明新位置。

### 13.3 更改發佈者名稱

開 PR 修改 `publisher`，說明原因（例如公司更名）。

### 13.4 私鑰外流（緊急）

1. 立即開 PR **移除或更換公鑰**，標題註明「緊急：私鑰外流」，並在 PR 中說明經過。若無法等待一般流程，請同時以索引 repo 公告的聯絡方式通知管理者。
2. 刪除 GitHub secret 中的舊私鑰；檢查 repo 的協作者與 Actions 紀錄，確認沒有被他人發佈的 Release。
3. 依〈13.1〉產生新金鑰並發佈新版本。
4. 在 repo 開 issue 告知使用者：受影響的版本範圍、該怎麼確認自己安裝的版本是否正常。

### 13.5 下架

開 PR 刪除該筆資料。合併後商店不再顯示與提供更新，**已安裝的使用者不受影響**（不會被移除）。建議在 repo 的 README 說明下架原因與替代方案。

---

## 14. 公司內部發佈

不上公開商店，只給公司內部使用時，有三種做法：

| 做法 | 適用 | 說明 |
|---|---|---|
| 公司內部索引 | 有內部 GitHub（GHES 或私有組織） | 管理者 fork 索引範本，上架流程與本指南相同，只是 PR 開到內部索引 repo。見 [STORE.md〈企業私有商店〉](STORE.md#企業私有商店) |
| 共用磁碟鏡像 | 隔離網路、或擴充 repo 是私有的 | 管理者把 `.tbx` 與索引放在共用磁碟（`file://`）。你只要照常發佈 Release，再通知管理者鏡像 |
| IT 直接派送 | 少量電腦、不需要商店 | 用命令列安裝，見下方 |

> ToolDock 下載 `.tbx` 時**不帶 GitHub 憑證**。擴充 repo 是私有的，商店就無法直接從 Release 下載，請改用共用磁碟鏡像。

IT 派送時，用打包好的 `.tbx` 以命令列安裝（成功時結束碼為 0）：

```bash
/Applications/ToolDock.app/Contents/MacOS/ToolDock --install-extension=./my-tool-1.0.0.tbx
```

Windows（PowerShell）：

```powershell
& "$env:LOCALAPPDATA\Programs\ToolDock\ToolDock.exe" --install-extension=.\my-tool-1.0.0.tbx
```

---

## 15. 疑難排解

### 15.1 開發與打包

| 狀況 | 處理方式 |
|---|---|
| `tooldock-ext: command not found`、`sh: tooldock-ext: not found` | 專案還沒安裝開發工具套件。執行 `npm install`；既有專案依〈1.5〉安裝 `tooldock-cli` |
| `npm create tooldock-extension` 失敗或下載不到 | 確認網路與 npm 鏡像設定（〈1.5〉）；`npm view create-tooldock-extension version` 應該能查到版本 |
| `validate` 沒有檢查出新欄位的錯誤、編輯器沒有新 API 的型別 | `tooldock-cli`、`tooldock-types` 版本太舊，依〈1.5〉升級 |
| `npm run dev` 找不到 ToolDock | 設定 `TOOLDOCK_PATH`，或加上 `--app <路徑>`（〈1.4〉） |
| 改了程式沒有生效 | 確認是用 `npm run dev` 啟動；看輸出面板是否有載入錯誤；必要時在擴充詳細頁按「重新載入」 |
| `相依套件未安裝：xxx` | 執行 `npm install` |
| 安裝後畫面空白 | 用「檢查此畫面」看主控台錯誤；確認 `ui/index.html` 有載入 `tooldock://sdk/ui-sdk.js`，前端建置結果有放進 `ui/` |
| 安裝後提示「需要平台版本…」 | `engine` 高於使用者的平台版本。請使用者升級 ToolDock，或降低 `engine` 並在程式裡判斷功能 |
| 原生模組載入失敗 | 改用純 JS 或 WebAssembly 的套件（〈4.4〉） |
| `.tbx` 太大 | 檢查 `unzip -l` 的內容；把測試、範例、原始碼加入 `.tbxignore`；確認建置工具在 `devDependencies` |

### 15.2 發佈（CI）

| 狀況 | 處理方式 |
|---|---|
| 「版本不一致」 | tag 與 `package.json` 不符。刪除錯誤的 tag 後用 `npm version` 重新建立（〈10.5〉） |
| 「未簽章，商店不會收錄」 | 沒有設定 secret `TOOLDOCK_SIGNING_KEY`，或內容是空的 |
| `不是 PEM 格式的私鑰` | secret 內容不完整，請貼上私鑰檔的全部文字（含 BEGIN / END 兩行） |
| 「以公鑰驗證簽章」失敗 | `TOOLDOCK_SIGNING_KEY` 與 `TOOLDOCK_PUBLIC_KEY` 不是同一組金鑰 |
| `npm ci` 失敗 | 沒有提交 `package-lock.json`，或它與 `package.json` 不一致。在本機執行 `npm install` 後提交 |
| `gh release create` 權限不足 | 確認 workflow 權限設定（〈9.2〉） |

### 15.3 上架與商店

| 狀況 | 可能原因與處理方式 |
|---|---|
| 「檢查上架 PR」失敗：不允許的欄位 | `extensions.json` 只能有 `id`、`repo`、`publisher`、`publicKey`、`verified` |
| 「檢查上架 PR」失敗：publicKey 無法載入 | 公鑰複製不完整，或貼成私鑰。請貼 `.pub` 檔的內容 |
| job summary 顯示「沒有附件齊全且驗證通過的正式 Release」 | Release 是 draft 或 prerelease；缺少 `.sig`；附件檔名不是 `<id>-<version>.tbx` |
| job summary 顯示「簽章驗證失敗」 | Release 用的私鑰與登記的公鑰不是同一組 |
| job summary 顯示「套件 id 為…與登記的…不符」 | `tooldock.id` 與 `extensions.json` 的 `id` 不同 |
| job summary 顯示「套件版本…與 tag…不符」 | `.tbx` 內的版本與 tag 不同，請用 `npm version` 重新發佈 |
| 合併後商店還是找不到 | 等待索引建置與快取（〈11.5〉），在商店按「重新整理」 |
| 新版本沒有出現在「可更新」 | 版本是 prerelease；附件不齊；新版 `engine` 與使用者的平台不相容；或還在快取時間內 |
| 使用者更新時看到「金鑰已變更」 | 公鑰更換後的正常現象（〈13.1〉）。若你沒有換過金鑰，請立即依〈13.4〉處理 |

---

## 16. 附錄

### 16.1 指令速查

| 時機 | 指令 |
|---|---|
| 建立專案 | `npm create tooldock-extension@latest my-tool` |
| 既有專案安裝開發工具 | `npm install -D tooldock-cli tooldock-types` |
| 確認／升級開發工具版本 | `npm ls tooldock-cli tooldock-types`；`npm install -D tooldock-cli@latest tooldock-types@latest` |
| 開發模式 | `npm run dev`（全新資料：`npm run dev -- --fresh`） |
| 偵錯 | `npm run debug`；`npm run dev -- --inspect-brk` |
| 檢查 manifest | `npm run validate` |
| 打包 | `npm run pack` |
| 打包並安裝到本機 | `npm run install-local` |
| 產生金鑰 | `npx tooldock-ext keygen` |
| 本機簽章 | `npx tooldock-ext pack --sign --key ~/.tooldock/keys/tooldock-signing-key.pem` |
| 驗證套件 | `npx tooldock-ext verify <檔案.tbx> --public-key <公鑰字串>` |
| 設定 secret | `gh secret set TOOLDOCK_SIGNING_KEY < ~/.tooldock/keys/tooldock-signing-key.pem` |
| 發佈新版 | `npm version patch && git push --follow-tags` |
| 查看 Release | `gh release view v<版本>` |

### 16.2 上架流程檢查表

複製到你的待辦清單使用：

```markdown
- [ ] 決定擴充 id（〈2.1〉）
- [ ] 建立專案與公開 GitHub repo（〈2〉）
- [ ] 設定 manifest：版本、engine、permissions、guarded（〈3〉）
- [ ] 開發與除錯（〈4〉）
- [ ] validate、單元測試、以安裝版測試、相容性測試（〈5〉）
- [ ] README、LICENSE、上架前自我檢查（〈6〉）
- [ ] 檢查套件內容（〈7〉）
- [ ] 產生金鑰並備份私鑰（〈8〉）
- [ ] 設定 GitHub secret 與 variable（〈9〉）
- [ ] 發佈 v1.0.0，確認三個附件並自己驗證（〈10〉）
- [ ] 對索引 repo 開 PR（〈11〉）
- [ ] 回應審核意見
- [ ] 合併後在商店確認並實際安裝一次（〈11.5〉）
```

### 16.3 相關文件

| 文件 | 內容 |
|---|---|
| [EXTENSION_GUIDE.md](EXTENSION_GUIDE.md) | manifest 完整欄位、後端 `ctx` 與畫面 `window.tooldock` API |
| [RELEASE.md](RELEASE.md) | 簽章與發佈檔格式的技術規格 |
| [STORE.md](STORE.md) | 商店架構、信任模型、企業私有商店 |
| [STORE_REVIEW.md](STORE_REVIEW.md) | 管理者審查上架流程 |
| [EXCHANGE.md](EXCHANGE.md) | 資料交換規格 |
| [NOTIFY.md](NOTIFY.md) | 通知服務與執行鎖規格 |
| [RENDER.md](RENDER.md) | PDF 輸出規格 |
| 索引 repo 的 `CONTRIBUTING.md` | 上架步驟與審核清單 |
