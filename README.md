# ToolDock 下載

ToolDock 是可擴充的桌面工具平台，介面類似 VS Code。所有工具都是擴充：每個擴充有自己的畫面和後端，可以從內建的「商店」安裝、更新或移除。

**[⬇ 下載最新版](https://github.com/tooldock-dev/tooldock-releases/releases/latest)**

| 作業系統 | 檔案 |
|---|---|
| macOS（Apple Silicon，M1 / M2 / M3 / M4） | `ToolDock-<版本>-mac-arm64.dmg` |
| macOS（Intel） | `ToolDock-<版本>-mac-x64.dmg` |
| Windows 10 / 11（64 位元） | `ToolDock-<版本>-win-x64.exe` |

每個版本都附有 `SHA256SUMS.txt`，可以用來確認下載的檔案完整、沒有被修改過。

已安裝 1.10.0 以上版本時，可以在「關於 ToolDock」按「檢查更新」，確認有沒有新版（macOS 在 ToolDock 選單，Windows 在「說明」選單）。

## 安裝

### macOS

1. 打開 `.dmg`，把 ToolDock 拖到「應用程式」資料夾。
2. 第一次開啟時，在 ToolDock 上**按右鍵 →「打開」**，再在對話框按「打開」。

ToolDock 目前還沒有經過 Apple 公證，直接點兩下會被系統阻擋。如果右鍵也打不開，請在「終端機」執行：

```bash
xattr -dr com.apple.quarantine /Applications/ToolDock.app
```

### Windows

1. 執行 `ToolDock-<版本>-win-x64.exe`。
2. 如果出現「Windows 已保護您的電腦」，按**「其他資訊」→「仍要執行」**。

安裝檔目前沒有程式碼簽章，所以會出現這個警告。安裝時可以選擇安裝位置，安裝完成後會建立桌面捷徑。

### 確認檔案完整

macOS / Linux：

```bash
shasum -a 256 -c SHA256SUMS.txt --ignore-missing
```

Windows PowerShell（把算出來的值和 `SHA256SUMS.txt` 對照）：

```powershell
Get-FileHash .\ToolDock-<版本>-win-x64.exe -Algorithm SHA256
```

## 工具從商店安裝

1.10.0 起 ToolDock 不內建任何工具，全部從左側活動列的「商店」安裝：

- **資料表報表產生器**：讀取 CSV、Excel、JSON、SQLite、Markdown 表格，輸出 Excel、Word、CSV 或 Markdown 報表。
- **資料轉換工具**：JSON、YAML、CSV、Base64、時間戳、民國年、雜湊等轉換。
- **網站監控**：定時檢查網站狀態與 SSL 憑證到期日。
- **Markdown 編輯器**：編輯與即時預覽，工具列插入常用語法，匯出 PDF 與 HTML。
- **程式碼編輯器**：建立專案、瀏覽資料夾、語法高亮編輯，查看 Git 異動清單與差異比對。
- **Oracle SQL 工具**：連線 Oracle 執行 SQL，查詢結果可以交給報表產生器。
- **Telegram 通知**：把通知送到 Telegram。

曾經內建的工具：Markdown 編輯器（1.7.0～1.8.x）、程式碼編輯器（1.8.x）在 1.9.0 起改由商店提供；資料表報表產生器、資料轉換工具、網站監控在 1.10.0 起改由商店提供。從商店安裝後，原本的資料與設定都還在。商店的擴充在安裝前，平台會先驗證 SHA-256 與發佈者的 Ed25519 簽章。

> 擴充的後端擁有完整的 Node.js 權限（檔案、網路、執行程式），平台沒有沙箱。請只安裝你信任的發佈者所提供的擴充。

## 開發擴充

```bash
npm create tooldock-extension@latest my-tool
```

擴充開發者也需要先安裝 ToolDock。從開發、測試、發佈到上架商店的完整步驟，請從 **[開發人員操作指南](docs/DEVELOPER_GUIDE.md)** 開始。

| 文件 | 內容 |
|---|---|
| [開發人員操作指南](docs/DEVELOPER_GUIDE.md) | 環境安裝、建立專案、開發與除錯、測試、打包、簽章、發佈、申請上架、改版與下架 |
| [API 參考指南](docs/EXTENSION_GUIDE.md) | manifest 欄位、後端 `ctx` 與畫面 `window.tooldock` API、函式庫、命令列參數 |
| [所有開發者文件](docs/) | 資料交換、通知與執行鎖、PDF 輸出、發佈檔格式、商店架構、上架審查流程 |

範例專案可參考 [tooldock-ext-hello](https://github.com/tooldock-dev/tooldock-ext-hello) 和 [tooldock-ext-oracle-sql](https://github.com/tooldock-dev/tooldock-ext-oracle-sql)。

## 回報問題

請到這個 repo 的 [Issues](https://github.com/tooldock-dev/tooldock-releases/issues) 回報問題。
