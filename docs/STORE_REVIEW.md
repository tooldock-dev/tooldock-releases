# 管理者審查上架流程

這份文件說明 ToolDock 擴充商店的管理者如何**受理、審查、核准或退回**上架申請，以及上架後的異動、緊急處置與索引維運。

- **適用對象**：索引 repo（官方 [tooldock-dev/tooldock-registry](https://github.com/tooldock-dev/tooldock-registry)，或公司內部的索引 repo）的審核者與維護者。
- 開發人員的操作見 [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md)；商店的技術架構見 [STORE.md](STORE.md)；索引 repo 內的 `CONTRIBUTING.md` 是給發佈者看的精簡版審核清單，本文件是它的完整作業說明。

---

## 目錄

1. [審查的目的與責任範圍](#1-審查的目的與責任範圍)
2. [角色與權限](#2-角色與權限)
3. [索引 repo 的初始設定](#3-索引-repo-的初始設定)
4. [審核者的工作環境](#4-審核者的工作環境)
5. [流程總覽](#5-流程總覽)
6. [新上架審查步驟](#6-新上架審查步驟)
7. [決定與回覆](#7-決定與回覆)
8. [合併與上線確認](#8-合併與上線確認)
9. [異動既有項目](#9-異動既有項目)
10. [緊急處置](#10-緊急處置)
11. [上架後的持續監督](#11-上架後的持續監督)
12. [索引維運](#12-索引維運)
13. [附錄](#13-附錄)

---

## 1. 審查的目的與責任範圍

### 1.1 為什麼要人工審查

擴充後端是一般的 Node.js 程序，擁有與 ToolDock 相同的完整權限：讀寫使用者能存取的所有檔案、連到任何網址、執行任何程式。**平台沒有沙箱**。使用者在商店安裝擴充時，信任的依據就是索引：索引登記的 repo 與公鑰，決定了「這個 id 的擴充由誰發佈」。

所以每一筆收錄都代表管理者確認了：

1. **身分**：發佈者就是 repo 的擁有者，沒有冒名。
2. **信任來源**：登記的公鑰屬於這位發佈者，沒有與他人共用。
3. **內容一致**：Release 的套件與公開的原始碼相符。
4. **沒有明顯惡意**：在合理的審查範圍內，沒有發現可疑行為。
5. **說明誠實**：README 與用途宣告如實寫出擴充會做的事。

### 1.2 審查不保證什麼

請在回覆與公告中避免誤導使用者：

- **「已驗證發佈者」（`verified: true`）只代表身分經過確認，不代表程式碼經過安全稽核**。商店畫面也是這樣告知使用者。
- 簽章只證明套件出自持有私鑰的人。發佈者本身惡意、私鑰外流、或發佈者的 CI 被入侵時，簽章仍然有效。
- **上架後發佈的新版本不會經過 PR 審查**（見〈11〉）。審查的是「發佈者」，不是每一個版本。
- 從索引移除後，**已安裝的副本不會被移除或停用**，只是不再收到更新。

---

## 2. 角色與權限

| 角色 | 職責 | 需要的權限 |
|---|---|---|
| 發佈者 | 開發擴充、發佈 Release、對索引開 PR | 自己的擴充 repo |
| 審核者 | 依本文件審查 PR，核准或要求修改 | 索引 repo 的 Write（可核准 PR） |
| 維護者 | 合併 PR、處理緊急下架、維運索引建置與 Pages | 索引 repo 的 Maintain 或 Admin |

建議：

- 至少兩位審核者，**新上架與更換公鑰需要一位發佈者以外的審核者核准**。審核者本人的擴充由另一位審核者審查。
- 有 Admin 權限的人越少越好：能修改索引或 Pages 部署的人，可以換掉任何擴充的公鑰。
- 審核者對發佈者有利益關係（同公司、同專案）時，請交給其他審核者。

---

## 3. 索引 repo 的初始設定

只需要做一次。已經建立好的索引請逐項確認設定仍然正確。

### 3.1 建立 repo 與 Pages

1. 在 GitHub 建立 repo（例如公司內部的 `acme/tooldock-registry`），把官方索引 repo [tooldock-dev/tooldock-registry](https://github.com/tooldock-dev/tooldock-registry) 的內容複製到 repo 根目錄（`.github/` 必須在根目錄才會生效），再把 `extensions.json` 清空成 `[]`，只放自己要收錄的擴充。
2. **Settings → Pages → Build and deployment → Source** 選 **GitHub Actions**。
3. 到 **Actions** 手動執行一次「建立索引」，確認 `https://<owner>.github.io/<repo>/index.json` 可以開啟。
4. 把 ToolDock 的 `config/store.json` 預設來源改成這個網址（官方索引已設定好）。

### 3.2 保護設定（必要）

| 設定位置 | 設定 | 目的 |
|---|---|---|
| Settings → Branches（或 Rulesets）→ `main` | 必須經過 PR 才能合併 | 所有異動留下紀錄 |
| 同上 | 至少 1 位核准；有新 commit 時撤銷舊核准 | 避免核准後偷換內容 |
| 同上 | 必須通過狀態檢查「檢查上架 PR」 | 格式與公鑰錯誤不能合併 |
| 同上 | 禁止 force push、禁止刪除分支；規則也套用到管理員 | 保護歷史紀錄 |
| Settings → Environments → `github-pages` | 只允許 `main` 部署 | 避免其他分支發佈索引 |
| Settings → Actions → General | Fork PR 的 workflow 需要核准才執行 | 外部 PR 不能任意消耗 Actions |
| Settings → Collaborators and teams | 審核者 Write、維護者 Maintain，Admin 限 1–2 人 | 最小權限 |

### 3.3 建議設定

- **CODEOWNERS**：新增 `.github/CODEOWNERS`，把 `extensions.json`、`scripts/`、`schema/`、`.github/` 指派給審核者團隊，並在分支規則中要求 Code Owner 核准。修改建置腳本的 PR 比修改 `extensions.json` 更危險，請由維護者審查。
- **標籤**：建立 `新上架`、`更換公鑰`、`異動`、`下架`、`待發佈者修改`、`待審核`、`緊急` 標籤，方便追蹤。
- **公告管道**：在索引 repo 的 README 寫明緊急聯絡方式（例如安全通報用的 email 或 Security Advisory），以及下架公告會發在哪裡。

---

## 4. 審核者的工作環境

| 工具 | 用途 |
|---|---|
| Node.js 20 以上 | 執行索引的檢查與建置腳本、`tooldock-ext verify` |
| GitHub CLI（`gh`），已登入 | 查看 PR、下載 Release、查詢 repo 擁有者 |
| `unzip`、`diff` | 解開套件、比對原始碼 |
| ToolDock（最新版） | 實際安裝測試 |
| 索引 repo 的本機 clone | 試建索引 |

**安全原則**：審查的套件可能是惡意的。

- 每個審查案件用一個新的空資料夾（以下用 `~/review/<id>-<版本>`）。
- 安裝相依套件一律加 `--ignore-scripts`，不執行任何套件的安裝腳本。
- **不要**在審查用的資料夾執行 `npm run …`、`npx` 或擴充內的任何腳本。
- 實機測試用 `--user-data-dir` 指定獨立資料目錄，不要裝到自己平常使用的 ToolDock；有條件的話使用虛擬機或測試用電腦。
- 實機測試前，先完成程式碼審查（〈6.6〉）。

以下指令用這些變數，請依 PR 內容替換：

```bash
export ID=my-tool VER=1.0.0 REPO=your-name/tooldock-ext-my-tool KEY="MCowBQYDK2VwAyEA…" PR=123
```

```bash
export R=~/review/$ID-$VER && mkdir -p $R/release $R/pkg $R/src
```

---

## 5. 流程總覽

```
 發佈者開 PR
     │
     ▼
 ① 受理分類 ──── 範本未填、類型不明 ───────────▶ 要求補件
     │
     ▼
 ② CI 自動檢查 ── 失敗 ──────────────────────▶ 要求修正（發佈者 push 修正）
     │ 通過，且 job summary 顯示「收錄」
     ▼
 ③ 身分審查 ──── 冒名、PR 作者非擁有者 ───────▶ 退回
     │
     ▼
 ④ 套件完整性 ── 簽章、SHA-256、id、版本 ─────▶ 要求修正
     │
     ▼
 ⑤ 原始碼比對 ── 套件與 tag 的原始碼不符 ──────▶ 要求說明 / 退回
     │
     ▼
 ⑥ 程式碼與 manifest 審查 ── 可疑行為 ─────────▶ 要求說明 / 退回
     │
     ▼
 ⑦ 實機測試 ──── 無法安裝、與說明不符 ─────────▶ 要求修正
     │
     ▼
 ⑧ 決定：核准 / 要求修改 / 退回
     │ 核准
     ▼
 ⑨ 合併 → 索引建置 → 確認上線 → 留下審查紀錄
```

### 5.1 建議處理時間

以下是建議值，請依團隊人力調整，並寫在索引 repo 的 README 讓發佈者知道。

| 項目 | 建議 |
|---|---|
| 受理（加標籤、指派審核者、確認範本） | 2 個工作天內 |
| 新上架審查完成 | 10 個工作天內 |
| 發佈者回覆修改要求 | 30 天內未回覆即關閉 PR，可以重新開 |
| 一般異動（發佈者名稱、下架申請） | 5 個工作天內 |
| 更換公鑰 | 5 個工作天內；私鑰外流的申請依〈10〉立即處理 |
| 惡意行為通報 | 立即處理 |

---

## 6. 新上架審查步驟

### 6.1 受理與分類

```bash
gh pr view $PR --repo tooldock-dev/tooldock-registry
```

```bash
gh pr diff $PR --repo tooldock-dev/tooldock-registry
```

確認：

- [ ] PR 範本已填寫：擴充 id、repo、異動類型，「發佈者自我檢查」全部勾選。
- [ ] diff **只修改 `extensions.json`**，而且只新增一筆。同時修改 `scripts/`、`schema/`、`.github/` 的 PR 請退回，要求拆開（建置腳本的修改由維護者另外審查）。
- [ ] 新增的一筆 `verified` 是 `false`。
- [ ] 加上標籤 `新上架`、`待審核`，指派審核者。

### 6.2 CI 自動檢查

在 PR 的 Checks 頁面查看「檢查上架 PR」job，它會執行：

1. `node scripts/validate.js`：欄位、id 與 repo 格式、id 重複、公鑰能否載入為 Ed25519。
2. `node scripts/build-index.js`：實際讀取 Release、下載並驗證，**不發佈**。

打開 job summary 確認：

- [ ] 「檢查 extensions.json 格式與公鑰」通過。
- [ ] 收錄表格中**有這個擴充**，版本與 PR 說明的一致。
- [ ] 「問題」清單中**沒有**這個擴充的項目。

常見的 CI 訊息與處理：

| 訊息 | 原因 | 處理 |
|---|---|---|
| `不允許的欄位「…」` | 多了五個欄位以外的欄位 | 要求移除 |
| `id 與第 N 筆重複` | id 已被使用 | 要求改 id（發佈者需要改套件並重新發佈） |
| `publicKey 無法載入為 Ed25519 SPKI 公鑰` | 公鑰不完整或貼錯 | 要求貼上 `.pub` 檔的內容 |
| `repo 與第 N 筆相同`（警告） | 同一個 repo 發佈多個擴充 | 可以接受，附件以 id 區分；確認是同一位發佈者 |
| `沒有附件齊全且驗證通過的正式 Release` | 沒有 Release、是 draft / prerelease、缺 `.sig` | 要求發佈正式 Release |
| `簽章驗證失敗` | Release 不是用登記的公鑰簽的 | 要求確認 secret 與公鑰是同一組 |
| `套件 id 為「…」，與登記的「…」不符` | 套件 `tooldock.id` 與登記不同 | 要求一致 |
| `套件版本 … 與 tag … 不符` | 發佈流程錯誤 | 要求重新發佈 |
| `HTTP 403 …（API 次數已用完）` | GitHub API 限制 | 稍後按「Re-run jobs」 |

### 6.3 身分審查

**PR 作者與 repo 擁有者**

```bash
gh pr view $PR --repo tooldock-dev/tooldock-registry --json author --jq .author.login
```

```bash
gh api repos/$REPO --jq '{owner: .owner.login, type: .owner.type, private: .private, created: .created_at, fork: .fork}'
```

- [ ] repo 是**公開**的（`private: false`）。私有 repo 的 Release，ToolDock 無法直接下載。
- [ ] repo 擁有者是個人（`type: User`）時：PR 作者必須**就是**擁有者。
- [ ] repo 擁有者是組織（`type: Organization`）時：PR 作者必須是組織成員。可以用下列指令確認公開成員身分（回應 204 表示是成員）；成員身分不公開時，請發佈者到組織設定公開成員身分，或請組織的擁有者在 PR 留言確認。

```bash
gh api orgs/<組織>/members/<PR 作者> --silent && echo 是成員
```

- [ ] repo 不是 fork（`fork: false`）。是 fork 時，確認不是把他人的擴充改名重新發佈。

**發佈者名稱**

- [ ] `publisher` 與 repo 擁有者一致：個人用本名、暱稱或 GitHub 帳號；組織用組織名稱。
- [ ] 沒有冒用其他人、公司、知名產品或「ToolDock 官方」等容易誤解的名稱。
- [ ] 使用公司名稱時，repo 必須屬於該公司的 GitHub 組織。

**公鑰是否與他人共用**

```bash
grep -n "$KEY" extensions.json
```

- [ ] 這把公鑰只出現在這一筆，或只出現在**同一位發佈者**的其他擴充。
- [ ] 同一把公鑰出現在不同發佈者的項目中，代表私鑰外流或冒用：**退回**，並依〈10〉檢查既有項目。

**id 是否混淆**

- [ ] 不是內建擴充的 id：`code-editor`、`data-converter`、`markdown-editor`、`site-monitor`、`table-report`。
- [ ] 與既有擴充沒有只差一兩個字元、單複數、連字號位置等容易混淆的差別。
- [ ] 不是知名產品名稱，除非 repo 屬於該產品的擁有者。

### 6.4 套件完整性

下載 Release 附件並以 PR 登記的公鑰驗證，這和商店安裝時的檢查相同：

```bash
gh release download v$VER --repo $REPO --dir $R/release
```

```bash
npx -y -p tooldock-cli tooldock-ext verify $R/release/$ID-$VER.tbx --public-key "$KEY"
```

```bash
unzip -q $R/release/$ID-$VER.tbx -d $R/pkg && ls $R/pkg
```

- [ ] `verify` 通過（SHA-256 與簽章都正確）。
- [ ] 套件最上層只有一個資料夾，名稱就是擴充 id。
- [ ] `$R/pkg/$ID/package.json` 的 `tooldock.id`、`version` 與 PR、tag 一致。
- [ ] Release 不是 draft 或 prerelease。

### 6.5 原始碼比對

確認 Release 的套件確實是從 tag 的原始碼產生的，沒有多出原始碼中找不到的檔案。

```bash
git clone --depth 1 --branch v$VER https://github.com/$REPO $R/src
```

```bash
(cd $R/src && npm ci --ignore-scripts --omit=dev --no-audit --no-fund)
```

```bash
diff -rq $R/pkg/$ID $R/src | grep -v "^Only in $R/src"
```

只列出「套件裡有、原始碼裡沒有」與「內容不同」的檔案。原始碼裡有、套件裡沒有的檔案（測試、`.github`、`.tbxignore` 排除的檔案、`.map`）是正常的，已經濾掉。

| 結果 | 判斷 |
|---|---|
| 沒有輸出 | 套件與原始碼一致 |
| `Files … differ` 出現在 `node_modules` 內 | 先確認是否為平台相依的選用套件（CI 在 Linux 建置，你的電腦可能是 macOS）。其他差異請要求說明 |
| `Files … differ` 出現在擴充自己的檔案 | **要求說明**。正常情況下不應該發生 |
| `Only in $R/pkg/$ID` | 原始碼中沒有的檔案。若是 CI 建置的前端檔案，確認 `release.yml` 中有對應的建置步驟，必要時在本機重新建置後比對；否則**要求說明** |

另外檢查發佈流程本身：

```bash
cat $R/src/.github/workflows/release.yml
```

- [ ] 與 ToolDock 範本的 `release.yml` 相同，或差異都有合理用途（例如加入 `npm test`、前端建置步驟）。
- [ ] 沒有從外部網址下載並放進套件的步驟。

### 6.6 manifest 審查

```bash
node -e "console.log(JSON.stringify(require('$R/pkg/$ID/package.json'), null, 2))"
```

| 項目 | 確認內容 |
|---|---|
| `engine` | 寫法合法，與用到的功能相符（例如用了 `ctx.notify` 卻允許 1.6.0 以前的平台，程式必須有判斷） |
| `activationEvents` | 有 `onStartup` 或 `*` 時，README 要說明常駐理由；一般工具應該用延遲載入 |
| `keepAlive` | 為 `true` 時要有理由（例如保持資料庫連線） |
| `permissions.notify` | 程式有 `notify.send` 或 `window.notify` 時應該宣告；`reason` 與實際用途一致 |
| `permissions.commands` | 程式呼叫其他擴充的受保護指令時應該宣告 |
| `commands[].guarded` | 會造成外部影響（發訊息、寫資料庫、刪除資料）且其他擴充可呼叫的指令，必須標示。**替別人代發通知或代為執行動作的指令，沒有標示就退回** |
| `contributes.exchange` | 自訂格式以擴充 id 開頭；接收 `tooldock.sql` 的擴充見〈6.8〉 |
| `contributes.notify.channels` | 通道 id 以擴充 id 開頭；見〈6.8〉 |
| `scripts` | 沒有 `preinstall`、`install`、`postinstall` |
| `dependencies` | 只有執行時需要的套件；建置與測試工具應在 `devDependencies` |

也可以在**原始碼資料夾**用 `tooldock-cli` 跑一次檢查（只讀取檔案，不執行擴充程式碼）：

```bash
(cd $R/src && npx -y -p tooldock-cli tooldock-ext validate)
```

### 6.7 程式碼審查

審查範圍是**擴充自己的程式碼**（`node_modules` 以外的檔案）。後端程式碼優先，畫面次之。以下指令都在套件資料夾執行：

```bash
cd $R/pkg/$ID
```

> 這些搜尋只是找出**需要人工看的位置**，有結果不代表有問題，沒有結果也不代表沒問題。minified 的前端檔案（例如編輯器套件）每行很長，可以在指令後加 `| cut -c1-200` 縮短輸出，再另外判斷它是否能對應到原始碼（第 5 點）。

**1. 動態執行程式碼**

```bash
grep -rnE "\beval\s*\(|new Function\s*\(|vm\.(runIn|compile|Script)|require\s*\(\s*[^'\"\` )]" --include='*.js' --include='*.cjs' --include='*.mjs' --exclude-dir=node_modules .
```

- 下載遠端內容後 `eval`、`new Function`、`vm.run*`、寫成檔案再 `require`：**退回**。
- `require(變數)` 只用來載入自己資料夾內的模組（例如外掛式的資料來源）時可以接受，請確認路徑來源。

**2. 執行外部程式**

```bash
grep -rnE "child_process|\bexecSync\b|\bexecFile|\bspawn(Sync)?\s*\(|\bfork\s*\(" --include='*.js' --include='*.cjs' --include='*.mjs' --exclude-dir=node_modules .
```

- 每一處都要能對應到 README 說明的外部程式。
- 指令字串由使用者輸入拼接而成時（例如 `exec('cmd ' + input)`），要求改用 `execFile` 加參數陣列。

**3. 網路連線**

```bash
grep -rnoE "https?://[A-Za-z0-9.-]+[^\"'\` )]*" --include='*.js' --include='*.cjs' --include='*.mjs' --include='*.html' --exclude-dir=node_modules . | sort -u
```

```bash
grep -rnE "\bfetch\s*\(|https?\.request|net\.connect|tls\.connect|WebSocket|dgram" --include='*.js' --include='*.cjs' --include='*.mjs' --exclude-dir=node_modules .
```

- 列出的網址都要寫在 README，或是使用者自己設定的位址。
- 有遙測、使用統計、自動更新檢查時，README 必須寫明，最好可以關閉。
- 傳送的內容中包含使用者檔案、環境變數、系統資訊時，要特別確認用途。

**4. 存取與功能無關的檔案**

```bash
grep -rnE "\.ssh|\.aws|\.gnupg|\.kube|\.npmrc|\.netrc|Cookies|Login Data|Local State|keychain|Keychains|AppData|Application Support|ToolDock/extensions|process\.env\b" --include='*.js' --include='*.cjs' --include='*.mjs' --exclude-dir=node_modules .
```

- 讀取瀏覽器資料、SSH 或雲端金鑰、其他擴充的資料夾：**退回**。
- 讀取 `process.env` 整個物件並傳出去：**退回**。讀取特定、與功能相關的環境變數（例如 `HTTPS_PROXY`）可以接受。

**5. 混淆與二進位檔**

```bash
find . -type f \( -name "*.node" -o -name "*.exe" -o -name "*.dll" -o -name "*.so" -o -name "*.dylib" -o -name "*.wasm" -o -name "*.bin" \) -not -path "*/node_modules/*"
```

```bash
find . -type f \( -name "*.js" -o -name "*.mjs" -o -name "*.cjs" \) -not -path "*/node_modules/*" -size +100k
```

- 擴充自己的檔案中有二進位檔：要能從 repo 的原始碼建置，或來自知名套件且有說明。
- 大型或 minified 的 `.js`：確認 repo 中有原始碼與建置腳本（例如 `src-ui/` 與 `scripts/build-*.mjs`）。**無法對應到原始碼的混淆程式碼，退回**。

**6. 相依套件**

```bash
node -e "const l=require('$R/src/package-lock.json');const p=Object.entries(l.packages||{}).filter(([k,v])=>k&&!v.dev);console.log(p.length+' 個正式相依');for(const [k,v] of p) if(v.hasInstallScript) console.log('有安裝腳本：'+k)"
```

```bash
find node_modules -type f \( -name "*.node" -o -name "*.exe" -o -name "*.dll" -o -name "*.so" -o -name "*.dylib" \) 2>/dev/null
```

```bash
(cd $R/src && npm audit --omit=dev)
```

- 從 `.tbx` 安裝時**不會**執行任何 npm 腳本。但相依套件有安裝腳本，通常代表它需要原生編譯或下載額外檔案，可能在使用者電腦上無法運作，請要求說明。
- 有原生模組（`.node`）：確認能在目前的 Electron 版本載入（〈6.9〉實際測試），並請發佈者評估改用純 JS 或 WebAssembly 的套件。
- 套件名稱與知名套件只差一兩個字元（typosquatting）、下載次數極低、或剛發佈不久的不知名套件：請發佈者說明。
- `npm audit` 有高風險漏洞而且會影響擴充的用法：要求升級。

**7. 機密資料處理**

- [ ] 密碼、權杖用 `ctx.secrets` 保存，沒有寫進 `ctx.storage`、設定或日誌。
- [ ] 有 `ctx.secrets` 判斷（舊平台沒有）時，不支援的情況下不會改用明文保存，或已在畫面上明確告知使用者。

### 6.8 特定類型擴充的額外要求

| 擴充類型 | 必須確認 |
|---|---|
| **接收 `tooldock.sql`**（`exchange.imports` 含 `tooldock.sql`） | [EXCHANGE.md〈6.8〉](EXCHANGE.md#68-接收端的必要行為)七項全部做到，缺一項就要求修正：① 執行前預覽並確認，有查詢且來源為分享時顯示「查詢結果會傳回給…」；② 交換包不接受帳號密碼；③ 依語句分類拒絕不允許的語句，`allowDDL` 需要接收端設定也允許；④ 有 DML 時先試跑並回復；⑤ 同一個 `id` 重複匯入要警告；⑥ 有稽核紀錄；⑦ 一律使用參數綁定，不拼接 SQL。實機測試時逐項操作確認 |
| **通知通道**（`contributes.notify.channels`） | README 寫明訊息會送到哪個服務；`destinations` 的 id 穩定；`deliver` 失敗時回報中文原因；權杖用 `ctx.secrets` 保存 |
| **提供受保護指令給別人呼叫** | 代發通知、代為執行外部動作的指令都有 `guarded: true`；建議提供 `guardSummary`，讓確認對話框顯示人看得懂的說明 |
| **背景服務**（`onStartup`） | README 說明常駐原因、定時做什麼、多久一次；實機測試時觀察閒置時的 CPU 與記憶體 |
| **PDF 輸出使用 `allowRemote` 或 `javascript: true`** | README 說明需要的理由；`allowRemote` 時確認不會把文件內容送到外部 |
| **會刪除或覆寫使用者檔案** | 執行前有確認；README 有說明 |

### 6.9 實機測試

在獨立的資料目錄安裝，不影響審核者自己的 ToolDock（指定 `--user-data-dir` 時可以和平常使用的 ToolDock 同時執行）。以下為 macOS 的路徑，其他平台請替換執行檔位置：

```bash
export TD=/Applications/ToolDock.app/Contents/MacOS/ToolDock UD=$R/userdata
```

**A. 以商店流程安裝（建議，可以同時確認商店畫面的顯示）**

用 PR 分支的 `extensions.json` 在本機試建索引：

```bash
gh pr checkout $PR --repo tooldock-dev/tooldock-registry
```

```bash
GITHUB_TOKEN=$(gh auth token) node scripts/build-index.js --out $R/index/index.json
```

```bash
mkdir -p $UD && echo "{\"sources\":[{\"name\":\"審查\",\"url\":\"file://$R/index/index.json\"}]}" > $UD/store.json
```

```bash
"$TD" --user-data-dir=$UD
```

在商店找到這個擴充並安裝。

**B. 直接安裝套件**

```bash
"$TD" --user-data-dir=$UD --install-extension=$R/release/$ID-$VER.tbx
```

```bash
"$TD" --user-data-dir=$UD
```

**確認項目**

- [ ] 商店詳細頁的名稱、說明、圖示、發佈者、版本正確（方式 A）。
- [ ] 安裝確認畫面顯示的用途宣告與 README 一致（方式 A）。
- [ ] 安裝成功，擴充管理沒有錯誤；輸出面板（`⇧⌘U`）該擴充的頻道沒有錯誤。
- [ ] README 說明的主要功能都能操作。
- [ ] 第一次發通知、呼叫受保護指令時，有出現執行鎖確認；選「拒絕」時擴充不會崩潰。
- [ ] 擴充詳細頁的記憶體、CPU 在閒置時合理；延遲載入的擴充關閉畫面後程序會結束。
- [ ] 移除擴充後平台正常。
- [ ] 有條件的話，觀察網路連線（例如 macOS 的「活動監視器 → 網路」或 Little Snitch 等工具），確認只連到 README 列出的位址。

測試完畢後刪除整個審查資料夾：

```bash
rm -rf $R
```

### 6.10 是否給予「已驗證發佈者」

新上架預設維持 `verified: false`。只有在**能以 GitHub 以外的管道確認身分**時，審核者才把它改成 `true`（由審核者在 PR 中提交修改，或合併後另開 PR）：

| 可以給 `true` | 維持 `false` |
|---|---|
| repo 屬於 GitHub 上**已驗證網域**的組織（組織頁面有 Verified 標章），且 `publisher` 與組織名稱相符 | 個人帳號，且無法以其他管道確認本人 |
| 審核者能以其他管道確認本人（例如公司內部商店的同事、已知的合作夥伴，並留下確認紀錄） | 新建立的帳號或組織 |
| 長期維護、多次發佈無異常的既有發佈者，經兩位審核者同意 | 任何一點不確定 |

公司內部商店可以自訂規則，例如「內部組織的 repo 一律 `true`」，請寫在內部索引的 README。

---

## 7. 決定與回覆

### 7.1 判斷標準

| 決定 | 條件 | 動作 |
|---|---|---|
| **核准** | 〈6〉所有項目通過，或只剩不影響安全的建議 | Approve，在 PR 貼上審查紀錄（〈13.1〉），交給維護者合併 |
| **要求修改** | 可以修正的問題：README 不完整、缺用途宣告、缺 `guarded`、不必要的 `onStartup`、CI 失敗、套件與原始碼不一致但可能是流程錯誤 | Request changes，逐項列出問題與修正方式；標籤改為 `待發佈者修改` |
| **退回** | 冒名、PR 作者不是擁有者、公鑰與他人共用、明顯惡意行為、混淆過且無法對應原始碼的程式碼、刻意隱瞞的網路傳輸 | 關閉 PR 並說明原因；惡意行為依〈10.3〉處理 |

修改要求的處理：

- 擴充本身要修改時，發佈者要**發佈新版本**並在 PR 留言告知版本號。審核者對新版本重新執行〈6.4〉〈6.5〉，並審查與上一次審查版本之間的差異：

```bash
gh api repos/$REPO/compare/v<上次審查的版本>...v<新版本> --jq '.files[].filename'
```

- 只修改登記資料時，發佈者直接 push 到 PR 分支，審核者重新確認 diff 與 CI 即可。

### 7.2 回覆範本

**要求修改**

```markdown
感謝提交！審查後有以下需要修正的項目，修正後請在這裡留言告知：

1. **README 缺少網路連線說明**：程式會連到 `api.example.com`（`backend/sync.js` 第 42 行），請在 README 說明用途與傳送的內容。
2. **`myTool.send` 需要標示 `guarded`**：這個指令會代為發送訊息，其他擴充呼叫時應該先經過使用者確認。

擴充本身的修改請發佈新版本（例如 v1.0.1），並告知版本號。
```

**核准**

```markdown
審查通過 ✅ 審查版本：v1.0.0。審查紀錄如下，合併後最多約一小時會出現在商店。

之後發佈新版本只要推送新的 tag，不需要再開 PR。提醒：更換金鑰、repo 或發佈者名稱時，請依開發人員操作指南〈13〉開 PR。
```

**退回**

```markdown
很抱歉，這次無法收錄，原因如下：

- （具體原因，例如：PR 作者不是 repo 擁有者，也無法確認是該組織的成員。）

如果是誤會，歡迎提供說明後重新開 PR。
```

---

## 8. 合併與上線確認

### 8.1 合併

1. 確認核准後**沒有新的 commit**（有的話需要重新核准）。
2. 確認 PR 的 diff 仍然只有那一筆。
3. 使用 **Squash and merge**，commit 訊息寫「上架 `<id>`（審查版本 v`<版本>`）」。

### 8.2 確認索引已更新

合併到 `main` 會自動觸發「建立索引」workflow。

```bash
gh run list --repo tooldock-dev/tooldock-registry --workflow build-index.yml --limit 3
```

workflow 完成後（GitHub Pages 另有約 10 分鐘快取）：

```bash
curl -s https://tooldock-dev.github.io/tooldock-registry/index.json | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>{const e=JSON.parse(s).extensions.find(x=>x.id===process.argv[1]);console.log(e?{version:e.latest.version,publisher:e.publisher,verified:e.verified}:'尚未收錄')})" $ID
```

- [ ] job summary 的收錄表格中有這個擴充，版本與審查的一致。
- [ ] `index.json` 中有這個擴充。
- [ ] 在 ToolDock 商店按「重新整理」後可以找到。

### 8.3 留下紀錄

- 審查紀錄貼在 PR（〈13.1〉），PR 是之後追查的依據。
- 標籤改為 `新上架`（移除 `待審核`），關閉相關 issue。
- 如果審查時發現值得注意但不影響收錄的事項（例如依賴較少見的套件），寫在紀錄中，之後監督時可以參考。

---

## 9. 異動既有項目

所有異動都需要 PR，並由**該擴充 repo 的擁有者**提出。PR 只能修改該擴充自己的那一筆。

### 9.1 更換公鑰

風險最高的異動：換了公鑰，等於讓新金鑰的持有者可以發佈這個擴充。

- [ ] PR 作者是 repo 擁有者（〈6.3〉）。
- [ ] 原 repo 有**公告**：issue 或 Release 說明中寫明更換金鑰、新公鑰字串與原因，而且公告的帳號是 repo 擁有者。
- [ ] PR 中的新公鑰與公告中的完全相同。
- [ ] 新公鑰沒有被其他發佈者使用（〈6.3〉）。
- [ ] 發佈者已用新金鑰發佈新版本（開發人員操作指南〈13.1〉的建議順序）。PR 的 job summary 中，該擴充應該收錄這個新版本。
- [ ] 對新版本執行〈6.4〉〈6.5〉，並審查與上一個收錄版本之間的差異（〈7.1〉）。
- [ ] 原因合理（私鑰遺失、例行輪替、人員異動）。原因是「私鑰外流」時依〈10.2〉處理。

已安裝的使用者更新時，商店會因為金鑰不同而重新顯示確認畫面並警告，這是預期的行為。

### 9.2 更換 repo

**視同重新上架**，〈6〉全部重新審查。另外確認：

- [ ] 舊 repo 有公告新位置（README 或 issue），公告者是舊 repo 擁有者。
- [ ] 新 repo 的擁有者與舊 repo 相同，或有合理的關係（例如個人轉移到自己的組織）。

### 9.3 更改發佈者名稱

- [ ] PR 作者是 repo 擁有者。
- [ ] 新名稱依〈6.3〉的規則審查（不冒名、與 repo 擁有者一致）。
- [ ] 原為 `verified: true` 時，重新評估是否仍符合〈6.10〉。

### 9.4 調整「已驗證」

由審核者提出 PR，需要另一位審核者核准，並在 PR 中寫明依據（〈6.10〉）。

### 9.5 下架

| 情況 | 處理 |
|---|---|
| 發佈者自行申請 | PR 作者是 repo 擁有者即可合併 |
| repo 已刪除或改名 | 索引會自動不收錄（404 不沿用舊資料）。仍請開 PR 刪除該筆，保持清單正確 |
| 長期沒有附件齊全且驗證通過的 Release（job summary 持續出現問題） | 在擴充 repo 開 issue 通知發佈者，30 天內沒有回應就由維護者開 PR 下架 |
| 違反規範或惡意 | 依〈10.3〉 |

下架就是刪除 `extensions.json` 中的那一筆。合併後商店不再提供，**已安裝的使用者不會被移除**。下架後同一個 id 要重新上架，視同新上架審查；因惡意下架的 id 不再開放給他人使用，以免使用者誤以為是同一個擴充。

---

## 10. 緊急處置

### 10.1 原則

- 緊急處置**以速度優先**：維護者可以直接開 PR 並由另一位維護者立即核准合併；只有一位維護者在線時，可以先處理再補紀錄。
- 處置的目的是**停止散布**。索引沒有撤銷機制，已安裝的使用者只能透過公告自行移除。
- 所有緊急處置都要在索引 repo 開 issue（標籤 `緊急`）留下紀錄。

### 10.2 私鑰外流

1. **立即**開 PR 刪除該筆（或在發佈者已提供新公鑰時，直接更換為新公鑰），合併。
2. 手動執行一次「建立索引」workflow，讓索引立刻更新：

```bash
gh workflow run build-index.yml --repo tooldock-dev/tooldock-registry
```

3. 檢查外流期間是否有不是發佈者發佈的 Release：

```bash
gh release list --repo $REPO --limit 20
```

4. 在索引 repo 開 issue 公告：受影響的擴充、可能受影響的版本範圍、使用者該怎麼做。
5. 發佈者處理完畢後（新金鑰、新版本），依〈9.1〉重新收錄。

### 10.3 發現惡意行為

1. **立即**開 PR 刪除該筆並合併，手動執行「建立索引」。
2. 保存證據：下載可疑版本的 `.tbx`、記錄 SHA-256、截圖或摘錄可疑的程式碼位置，保存在維護者能存取的地方（不要公開散布惡意套件）。
3. 檢查同一位發佈者、同一把公鑰的其他擴充，必要時一併下架。
4. 在索引 repo 開 issue 公告：擴充 id、受影響版本、可疑行為摘要、**使用者該如何移除**（擴充管理中移除，或命令列 `--uninstall-extension=<id>`），以及是否需要更換可能外洩的密碼或權杖。
5. 公司內部商店：通知 IT 以部署工具在所有電腦執行移除：

```bash
/Applications/ToolDock.app/Contents/MacOS/ToolDock --uninstall-extension=<id>
```

6. 該 id 不再開放上架（〈9.5〉）。

### 10.4 索引或 Pages 被竄改

索引是單一信任根，能修改索引的人可以換掉任何擴充的公鑰。

1. 比對 `main` 的 git 歷史與目前線上的 `index.json`，找出被修改的項目。
2. 撤銷可疑人員的權限、輪替 Admin 帳號的憑證，檢查 Settings 中的協作者、Deploy keys、Actions secrets 與 Pages 設定。
3. 以正確的 `extensions.json` 重新執行「建立索引」。
4. 公告事件經過、受影響的時間範圍與擴充，請使用者在受影響期間安裝或更新過的擴充重新安裝。

---

## 11. 上架後的持續監督

**重要**：發佈者上架後，推送新的 tag 就會在約一小時內自動進入商店，**新版本不會經過 PR 審查**。審查的是發佈者與第一個版本，之後的版本依靠以下方式監督。

### 11.1 例行檢查

| 頻率 | 項目 | 方式 |
|---|---|---|
| 每週 | 「建立索引」的 job summary 是否有問題清單 | Actions 頁面查看最近一次執行 |
| 每週 | 新收錄的版本 | 比對本週與上週 job summary 的版本欄 |
| 每月 | 抽查新版本的程式碼差異，優先抽查：新增網路位址、新增 `child_process`、新增相依套件、`permissions` 變更、新增二進位檔 | 〈7.1〉的 compare 指令，再對有疑慮的版本執行〈6.7〉 |
| 每季 | 檢查索引 repo 的權限與分支規則（〈3.2〉）仍然正確 | Settings 頁面 |
| 每季 | 長期無更新、repo 已封存（archived）的擴充 | 通知發佈者，視情況下架 |

快速列出某個擴充在兩個版本之間新增的網路位址：

```bash
gh api repos/$REPO/compare/v1.0.0...v1.1.0 --jq '.files[] | .patch // ""' | grep -E "^\+" | grep -oE "https?://[A-Za-z0-9.-]+" | sort -u
```

### 11.2 使用者通報

- 在索引 repo 的 README 說明通報方式（issue 或安全通報管道）。
- 收到通報後，依通報內容執行〈6.7〉的相關檢查；確認是惡意行為時依〈10.3〉處理。
- 一般問題（錯誤、功能）請通報者到擴充自己的 repo 回報。

### 11.3 需要更嚴格控管時（公司內部商店）

公開索引的「新版本自動上架」不適合所有組織。需要**每個版本都經過審查**時，請改用 [STORE.md〈企業私有商店〉](STORE.md#企業私有商店)的方案 B（共用磁碟或內部伺服器鏡像）：

1. 管理者依本文件審查每一個版本。
2. 審查通過後，才把 `.tbx`、`.sha256`、`.sig` 複製到鏡像位置，並更新鏡像的 `index.json`。
3. 員工的 ToolDock 只設定內部鏡像來源（修改安裝版的 `resources/config/store.json`，或派送 `userData/store.json`）。

簽章仍然會用索引中的公鑰驗證，鏡像過程中被改動的檔案會被拒絕。

---

## 12. 索引維運

### 12.1 建置排程

「建立索引」workflow 在以下時機執行：

| 時機 | 說明 |
|---|---|
| push 到 `main` | 只在 `extensions.json`、`scripts/`、`schema/` 或 workflow 本身有異動時 |
| 每小時 | 第 17 分鐘 |
| 手動 | `gh workflow run build-index.yml` 或 Actions 頁面 |
| `repository_dispatch` | `event_type: extension-released`，需要有索引 repo 權限的 token |
| PR | 只檢查與試建，不發佈 |

### 12.2 判讀 job summary

| 訊息 | 意義 | 處理 |
|---|---|---|
| `收錄 vX（驗證通過）` | 正常 | — |
| `vX 附件不齊全（缺 …），略過` | 該版本缺附件，已改看較舊版本 | 通常是發佈者沒設定簽章 secret，可以通知發佈者 |
| `vX 未收錄 — 簽章驗證失敗` | 新版本不是用登記的公鑰簽的 | 可能是發佈者正在輪替金鑰（〈9.1〉）；若沒有輪替申請，**可能是 repo 被入侵**，請聯絡發佈者確認 |
| `vX 未收錄 — SHA-256 不符` | 附件被替換或上傳不完整 | 聯絡發佈者；持續發生時視為可疑 |
| `沿用上一版索引的 vX` | GitHub API 暫時性錯誤，沿用上一版資料 | 偶發可以忽略；持續發生請檢查 API 限制 |
| `沒有附件齊全且驗證通過的正式 Release，本次不收錄` | 擴充暫時從商店消失 | 確認原因，通知發佈者（〈9.5〉） |
| `HTTP 404` | repo 被刪除或改名 | 不會沿用舊資料，擴充從商店消失。聯絡發佈者，必要時依〈9.2〉〈9.5〉處理 |
| `API 次數已用完` | 速率限制 | Actions 內會自動提供 `GITHUB_TOKEN`；擴充數量很多時考慮降低排程頻率 |

### 12.3 延遲與快取

使用者看到變更的最長時間 = 索引建置（最多 1 小時）+ GitHub Pages 快取（約 10 分鐘）+ ToolDock 端快取（預設 30 分鐘，手動「重新整理」可略過）。緊急下架時請手動執行建置，並在公告中請使用者按「重新整理」。

### 12.4 GitHub Enterprise Server

在 GHES 上，Actions 會自動提供 `GITHUB_API_URL`，`build-index.js` 會改用內部 API，不需要修改。本機試建時請自行設定：

```bash
GITHUB_API_URL=https://github.example.com/api/v3 GITHUB_TOKEN=<token> node scripts/build-index.js --out dist/index.json
```

### 12.5 修改建置腳本或 schema

`scripts/`、`schema/`、`.github/workflows/` 的修改會影響所有擴充的驗證邏輯，請：

- 與 `extensions.json` 的異動分開成不同的 PR。
- 由維護者審查，特別注意驗證條件是否被放寬（例如不再驗證簽章、接受 prerelease）。
- 合併前在本機用目前的 `extensions.json` 試建，比對收錄結果是否與線上相同。

---

## 13. 附錄

### 13.1 審查紀錄範本

貼在 PR 的核准留言中：

```markdown
## 審查紀錄

- 擴充：`<id>`　審查版本：v<版本>　審核者：@<帳號>　日期：YYYY-MM-DD
- 異動類型：新上架 / 更換公鑰 / 更換 repo / 更改發佈者 / 下架

### CI
- [ ] 檢查上架 PR 通過，job summary 收錄 v<版本>

### 身分
- [ ] repo 公開、非 fork；PR 作者為擁有者（或組織成員：確認方式＿＿）
- [ ] publisher 與擁有者一致，未冒名
- [ ] 公鑰未與其他發佈者共用
- [ ] id 不與內建或既有擴充混淆

### 套件
- [ ] tooldock-ext verify 通過
- [ ] 套件與 tag 原始碼比對一致（差異說明：＿＿）
- [ ] release.yml 與範本一致（差異說明：＿＿）

### manifest
- [ ] engine 合理；啟用事件、keepAlive 有理由
- [ ] permissions 與程式一致；guarded 標示正確
- [ ] 無安裝腳本；dependencies 只含執行時套件

### 程式碼
- [ ] 無動態執行遠端程式碼
- [ ] 外部程式、網路位址皆與 README 相符（位址：＿＿）
- [ ] 無存取無關檔案；無混淆或無來源的二進位檔
- [ ] 相依套件與 npm audit 無重大問題
- [ ] 特定類型要求（tooldock.sql / 通知通道 / 背景服務）：不適用 / 已確認

### 實機測試
- [ ] 平台版本：＿＿　作業系統：＿＿
- [ ] 安裝、主要功能、執行鎖、移除皆正常

### 結論
- 決定：核准 / 要求修改 / 退回
- verified：false / true（依據：＿＿）
- 備註：
```

### 13.2 指令速查

| 用途 | 指令 |
|---|---|
| 查看 PR 與 diff | `gh pr view $PR --repo <索引 repo>`；`gh pr diff $PR --repo <索引 repo>` |
| PR 作者 | `gh pr view $PR --repo <索引 repo> --json author --jq .author.login` |
| repo 擁有者 | `gh api repos/$REPO --jq '.owner.login, .owner.type, .private, .fork'` |
| 組織成員 | `gh api orgs/<組織>/members/<帳號> --silent && echo 是成員` |
| 公鑰是否重複 | `grep -n "$KEY" extensions.json` |
| 下載 Release | `gh release download v$VER --repo $REPO --dir $R/release` |
| 驗證套件 | `npx -y -p tooldock-cli tooldock-ext verify <tbx> --public-key "$KEY"` |
| 取得原始碼 | `git clone --depth 1 --branch v$VER https://github.com/$REPO $R/src` |
| 安裝正式相依（不執行腳本） | `npm ci --ignore-scripts --omit=dev` |
| 比對套件與原始碼 | `diff -rq $R/pkg/$ID $R/src \| grep -v "^Only in $R/src"` |
| 版本差異 | `gh api repos/$REPO/compare/v<舊>...v<新> --jq '.files[].filename'` |
| 檢查格式 | `node scripts/validate.js` |
| 本機試建索引 | `GITHUB_TOKEN=$(gh auth token) node scripts/build-index.js --out dist/index.json` |
| 隔離安裝測試 | `ToolDock --user-data-dir=<資料夾> --install-extension=<tbx>` |
| 手動重建索引 | `gh workflow run build-index.yml --repo <索引 repo>` |

### 13.3 相關文件

| 文件 | 內容 |
|---|---|
| [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) | 開發人員操作指南：開發流程與上架說明 |
| [STORE.md](STORE.md) | 商店架構、信任模型、限制、企業私有商店 |
| [RELEASE.md](RELEASE.md) | 簽章與發佈檔格式 |
| [EXCHANGE.md](EXCHANGE.md) | 資料交換規格，含 `tooldock.sql` 接收端的必要行為 |
| [NOTIFY.md](NOTIFY.md) | 通知服務與執行鎖規格 |
| 索引 repo 的 `CONTRIBUTING.md`、`.github/PULL_REQUEST_TEMPLATE.md` | 給發佈者的上架步驟與審核清單 |
