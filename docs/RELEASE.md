# 發佈擴充

用 `create-tooldock-extension` 建立的專案已附上 GitHub Actions 流程 `.github/workflows/release.yml`。推送 `v*` tag 後，CI 會自動驗證、打包、以 Ed25519 簽章，並建立 GitHub Release。ToolDock 商店會下載 Release 的附件，並用你登記在商店索引的公鑰驗證簽章。

需要 `tooldock-cli` 1.1.0 以上（`keygen`、`pack --sign`、`verify`）。舊專案請先升級：

```bash
npm install -D tooldock-cli@^1.1.0
```

舊專案沒有 workflow 時，可以從新版範本複製 `templates/_shared/_github/workflows/release.yml` 到專案的 `.github/workflows/release.yml`。

## 1. 產生金鑰

```bash
npx tooldock-ext keygen               # 預設存到 ~/.tooldock/keys
npx tooldock-ext keygen --out ./keys  # 指定資料夾
```

| 檔案 | 內容 |
|---|---|
| `tooldock-signing-key.pem` | 私鑰（PKCS8 PEM），權限 600。**不要提交到 git、不要分享** |
| `tooldock-signing-key.pub` | 公鑰字串，可以公開 |

終端機也會印出公鑰字串。資料夾裡已經有金鑰時，`keygen` 會拒絕覆寫，需要加上 `--force`。

一把金鑰可以用在你的所有擴充。請另外備份私鑰（例如密碼管理器）；私鑰遺失時，只能產生新金鑰，並更新商店索引裡的公鑰。

`pack` 一律不會打包 `tooldock-signing-key*.pem`，範本的 `.gitignore` 也已排除。

## 2. 設定 GitHub

在擴充的 GitHub repo 設定：

| 種類 | 名稱 | 內容 | 必要 |
|---|---|---|---|
| Secret | `TOOLDOCK_SIGNING_KEY` | 私鑰檔的完整內容（含 `-----BEGIN PRIVATE KEY-----` 與 `-----END PRIVATE KEY-----`） | 要上架商店就必要 |
| Variable | `TOOLDOCK_PUBLIC_KEY` | 公鑰字串。設定後 CI 會在發佈前用它再驗證一次，可以及早發現 secret 與商店登記的公鑰不是同一組 | 選用 |

用 GitHub CLI：

```bash
gh secret set TOOLDOCK_SIGNING_KEY < ~/.tooldock/keys/tooldock-signing-key.pem
gh variable set TOOLDOCK_PUBLIC_KEY --body "$(cat ~/.tooldock/keys/tooldock-signing-key.pub)"
```

或到 GitHub repo →「Settings → Secrets and variables → Actions」手動新增。

workflow 已宣告 `permissions: contents: write`，不需要另外建立 token。如果組織把 Actions 的預設權限限制為唯讀，請到「Settings → Actions → General → Workflow permissions」確認允許 workflow 自行宣告權限。

## 3. 登記公鑰到商店

把公鑰字串提交到 ToolDock 商店索引（提交方式見商店索引的說明）。商店只收錄能以登記公鑰驗證通過的版本。

## 4. 發佈

```bash
npm version patch        # 或 minor / major / 1.2.0；更新 package.json 並建立 v<版本> tag
git push --follow-tags
```

CI 流程：

1. `npm ci`（`package-lock.json` 必須提交）
2. `npm run validate`
3. 檢查 tag 是否等於 `v` + `package.json` 的 `version`，不一致就失敗
4. 有 `TOOLDOCK_SIGNING_KEY`：`tooldock-ext pack --sign`，產出 `.tbx`、`.sha256`、`.sig`
   沒有：`tooldock-ext pack --checksum`，只產出 `.tbx`、`.sha256`，並在 log 與摘要顯示「未簽章，商店不會收錄」警告
5. 有設定 `TOOLDOCK_PUBLIC_KEY` 時，執行 `tooldock-ext verify`
6. `gh release create` 建立 Release 並上傳附件。版本含 `-`（例如 `1.2.0-beta.1`）時標為預先發行版。Release 已存在時（重新執行 workflow）改為覆寫附件

## 在本機簽章與驗證

```bash
# 簽章：私鑰依序取自 --key、環境變數 TOOLDOCK_SIGNING_KEY
npx tooldock-ext pack --sign --key ~/.tooldock/keys/tooldock-signing-key.pem

# 驗證：檢查 .sha256（若存在）與 .sig；失敗時結束代碼為 1
npx tooldock-ext verify dist/my-tool-1.0.0.tbx --public-key "<公鑰字串>"
```

`verify` 預設讀取與 `.tbx` 同資料夾的 `<檔名>.sha256` 和 `<檔名>.sig`，也可以用 `--sha256 <檔案>`、`--sig <檔案>` 指定。公鑰可以改用環境變數 `TOOLDOCK_PUBLIC_KEY` 提供。

## 發佈檔格式

商店依照下列約定下載與驗證，請勿自行更改格式。

**演算法**：Ed25519，使用 Node.js 內建的 `crypto`。

**金鑰**

| | 格式 |
|---|---|
| 私鑰 | PKCS8 PEM |
| 公鑰字串 | SPKI DER 的 base64：`crypto.createPublicKey(key).export({ type: 'spki', format: 'der' }).toString('base64')`（Ed25519 為 60 個字元，以 `MCowBQYDK2VwAyEA` 開頭） |

**Release tag**：`v<package.json version>`，例如 `v1.2.0`。

**Release 附件**（`<id>` 為 `package.json` 的 `tooldock.id`）

| 檔名 | 內容 |
|---|---|
| `<id>-<version>.tbx` | `tooldock-ext pack` 的輸出：zip，最上層資料夾為擴充 id |
| `<id>-<version>.tbx.sha256` | 只有一行：`.tbx` 的小寫 hex SHA-256（64 個字元）。`tooldock-ext` 產出時結尾不加換行，讀取端應先去除前後空白 |
| `<id>-<version>.tbx.sig` | `crypto.sign(null, <.tbx 原始位元組>, privateKey)` 的 base64，結尾不加換行 |

**驗證方式**（商店與 `tooldock-ext verify` 相同）

```js
const crypto = require('crypto');
const publicKey = crypto.createPublicKey({ key: Buffer.from(publicKeyString, 'base64'), format: 'der', type: 'spki' });
const sha256Ok = crypto.createHash('sha256').update(tbx).digest('hex') === sha256Text.trim();
const sigOk = crypto.verify(null, tbx, publicKey, Buffer.from(sigText.trim(), 'base64'));
```

## 疑難排解

| 狀況 | 處理方式 |
|---|---|
| CI 顯示「版本不一致」 | tag 與 `package.json` 不符。刪除錯誤的 tag（`git tag -d vX && git push origin :refs/tags/vX`），用 `npm version` 重新建立 |
| CI 顯示「未簽章，商店不會收錄」 | 沒有設定 secret `TOOLDOCK_SIGNING_KEY`，或 secret 是空的 |
| `不是 PEM 格式的私鑰` | secret 內容不完整，請貼上私鑰檔的全部文字 |
| `以公鑰驗證簽章` 失敗 | `TOOLDOCK_SIGNING_KEY` 與 `TOOLDOCK_PUBLIC_KEY` 不是同一組金鑰 |
| `npm ci` 失敗 | 沒有提交 `package-lock.json`，或它與 `package.json` 不一致。在本機執行 `npm install` 後提交 |
| `gh release create` 權限不足 | 確認 workflow 權限設定（見「2. 設定 GitHub」） |
