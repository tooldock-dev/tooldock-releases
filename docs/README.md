# ToolDock 開發者文件

這裡的文件對應 **ToolDock 1.7.0**。各項 API 標示了需要的最低平台版本，擴充的 `engine` 請依實際用到的功能設定。

## 從這裡開始

| 文件 | 給誰看 | 內容 |
|---|---|---|
| [開發人員操作指南](DEVELOPER_GUIDE.md) | 擴充開發者 | 從零開始的完整流程：環境安裝、建立專案、開發與除錯、測試、打包、簽章、發佈 Release、申請上架、改版、更換金鑰與下架 |
| [API 參考指南](EXTENSION_GUIDE.md) | 擴充開發者 | manifest 欄位、啟用事件、後端 `ctx` API、畫面 `window.tooldock` API、函式庫、除錯、平台命令列參數 |

## 功能規格

| 文件 | 內容 | 平台版本 |
|---|---|---|
| [資料交換（Exchange）](EXCHANGE.md) | 擴充之間分享資料：交換包、標準格式 `tooldock.table` / `tooldock.text` / `tooldock.sql`、`.tdx` 交換檔、接收端的必要行為 | 1.4.0 起；`mimes` 1.7.0 起 |
| [通知服務與執行鎖（Notify / Guard）](NOTIFY.md) | 發通知、通道擴充、受保護指令、用途宣告、授權與執行紀錄 | 1.6.0 起 |
| [文件輸出（Render）](RENDER.md) | `ctx.render.pdf()`、轉換環境、文件樣式 `document.css` | 1.7.0 起 |

## 發佈與商店

| 文件 | 給誰看 | 內容 |
|---|---|---|
| [發佈擴充](RELEASE.md) | 擴充開發者 | 簽章金鑰、GitHub Actions 自動發佈、發佈檔格式與驗證方式 |
| [擴充商店](STORE.md) | 開發者、IT 管理者 | 商店架構、信任模型、索引格式、企業私有商店、限制 |
| [管理者審查上架流程](STORE_REVIEW.md) | 索引審核者、維護者 | 審查步驟與指令、核准與退回標準、異動與緊急處置、索引維運 |

## 相關資源

| 資源 | 位置 |
|---|---|
| 下載 ToolDock | [Releases](https://github.com/tooldock-dev/tooldock-releases/releases/latest) |
| 開發工具套件 | npm：[create-tooldock-extension](https://www.npmjs.com/package/create-tooldock-extension)、[tooldock-cli](https://www.npmjs.com/package/tooldock-cli)、[tooldock-types](https://www.npmjs.com/package/tooldock-types) |
| 商店索引（申請上架） | [tooldock-dev/tooldock-registry](https://github.com/tooldock-dev/tooldock-registry) |
| 範例擴充 | [tooldock-ext-hello](https://github.com/tooldock-dev/tooldock-ext-hello)、[tooldock-ext-oracle-sql](https://github.com/tooldock-dev/tooldock-ext-oracle-sql)、[tooldock-ext-telegram](https://github.com/tooldock-dev/tooldock-ext-telegram) |
| 回報文件問題 | [Issues](https://github.com/tooldock-dev/tooldock-releases/issues) |

> 規格書中提到的 `src/…` 路徑是平台內部的實作位置，供參考用；開發擴充不需要平台原始碼。
