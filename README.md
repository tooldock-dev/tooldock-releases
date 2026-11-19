# ToolDock 下載

ToolDock 是可擴充的桌面工具平台，介面類似 VS Code。所有工具都是擴充：每個擴充有自己的畫面和後端，可以從內建的「商店」安裝、更新或移除。

**[⬇ 下載最新版](https://github.com/tooldock-dev/tooldock-releases/releases/latest)**

| 作業系統 | 檔案 |
|---|---|
| macOS（Apple Silicon，M1 / M2 / M3 / M4） | `ToolDock-<版本>-mac-arm64.dmg` |
| macOS（Intel） | `ToolDock-<版本>-mac-x64.dmg` |
| Windows 10 / 11（64 位元） | `ToolDock-<版本>-win-x64.exe` |

每個版本都附有 `SHA256SUMS.txt`，可以用來確認下載的檔案完整、沒有被修改過。

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

## 內建工具與商店

安裝後就有三個內建工具：

- **資料表報表產生器**：讀取 CSV、Excel、JSON、SQLite，輸出 Excel 或 Word 報表。
- **資料轉換工具**：JSON、YAML、CSV、Base64、時間戳、民國年、雜湊等轉換。
- **網站監控**：定時檢查網站狀態與 SSL 憑證到期日。

左側活動列的「商店」可以安裝更多擴充，例如「Oracle SQL 工具」。商店的擴充在安裝前，平台會先驗證 SHA-256 與發佈者的 Ed25519 簽章。

> 擴充的後端擁有完整的 Node.js 權限（檔案、網路、執行程式），平台沒有沙箱。請只安裝你信任的發佈者所提供的擴充。

## 開發擴充

```bash
npm create tooldock-extension@latest my-tool
```

擴充開發者也需要先安裝 ToolDock。詳細說明請見 [create-tooldock-extension](https://www.npmjs.com/package/create-tooldock-extension)，範例專案可參考 [tooldock-ext-hello](https://github.com/tooldock-dev/tooldock-ext-hello) 和 [tooldock-ext-oracle-sql](https://github.com/tooldock-dev/tooldock-ext-oracle-sql)。

## 回報問題

請到這個 repo 的 [Issues](https://github.com/tooldock-dev/tooldock-releases/issues) 回報問題。
