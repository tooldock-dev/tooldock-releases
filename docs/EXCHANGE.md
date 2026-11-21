# 資料交換（Exchange）規格書

> 狀態：**已實作**（平台 **1.4.0**）。`imports[].mimes`（〈7.1〉、〈8.1〉）為平台 **1.7.0** 新增（**已實作**），見 docs/RENDER.md。本文件定義擴充之間分享資料的格式與平台行為；實作以本文件為準，若需變更請先修改本文件。

## 1. 目的與範圍

讓一個工具產生的資料，可以交給另一個工具匯入，而**兩者不需要知道彼此**：

- 產生端只決定「這是什麼格式的資料」，不指定由誰接收。
- 任何工具只要產生同一種格式，接收端都能匯入；格式本身與產生者無關。
- 目的端工具沒有安裝時，資料可以匯出成檔案，日後再匯入。**沒有接收端不是錯誤。**
- 第一個要支援的情境：「平日作業輔助」類工具產生要在資料庫執行的 SQL，交給資料庫工具執行。SQL 會帶著參數一起傳遞，可以寫入資料，也可以查詢並把結果傳回（見〈6. `tooldock.sql`〉）。

不在範圍內：

- 擴充之間的即時雙向通訊。這部分仍然使用 `ctx.commands.execute`。
- 大型檔案傳遞。v1 只支援 10 MB 以內的純 JSON，附件留待後續版本（見〈11. 後續版本〉）。
- 平台內建資料庫驅動程式。SQL 由資料庫工具自己帶驅動程式執行，平台只負責轉送。

## 2. 名詞

| 名詞 | 說明 |
|---|---|
| **交換包**（package） | 一次分享的完整資料，由外層信封和 `payload` 組成，見〈4〉 |
| **格式**（type） | 交換包的種類，例如 `tooldock.sql`。路由只看格式 |
| **產生端**（producer） | 呼叫 `ctx.exchange.share()` 的擴充 |
| **接收端**（receiver） | 在 manifest 宣告 `contributes.exchange.imports` 的擴充 |
| **交換檔** | 交換包存成的檔案，副檔名 `.tdx`，內容就是交換包的 JSON |

## 3. 架構

```
 產生端擴充                        平台（ExtensionManager）                         接收端擴充
 ──────────                        ────────────────────────                         ──────────
 ctx.exchange.share(pkg) ──RPC──▶  1. 驗證信封
                                   2. 依 type + typeVersion 找接收端
                                      ├─ 1 個 ──────────────▶ 啟用 onImport:<type> ──▶ handler(ctx, pkg, info)
                                      ├─ 多個 → 工作台選擇器 ─┘                          預覽 → 使用者確認 → 處理
                                      └─ 0 個 → 依 fallback：存成 .tdx / 直接回報        │
                                   3. 回傳 ShareResult  ◀──────────────────────────── ImportResult
 ◀─────────────────────────────────┘

 開啟 .tdx 檔（指令／檔案關聯／第二個實例參數） ──▶ 從步驟 1 開始，info.origin = 'file'
```

平台負責：信封驗證、找出接收端、啟用接收端、選擇器、存成檔案、從檔案匯入。
擴充負責：定義格式內容、驗證 `payload`、預覽與確認、實際處理資料。

## 4. 交換包信封

```jsonc
{
  "format": "tooldock.exchange/1",
  "id": "0c9e6f4a-7d2b-4b8e-9a51-2f0c3d1e8b77",
  "type": "tooldock.sql",
  "typeVersion": 1,
  "title": "補登 11 月加班時數",
  "description": "依簽核單 #1123 更新 3 位同仁的加班時數",
  "source": { "extId": "help", "version": "0.1.0", "platform": "1.4.0" },
  "createdAt": "2026-11-19T09:30:00+08:00",
  "payload": { }
}
```

| 欄位 | 必填 | 規則 |
|---|---|---|
| `format` | ✔ | 固定為 `tooldock.exchange/1`。信封結構有不相容變更時才會改版 |
| `id` | ✔ | UUID v4。接收端可以用它判斷是否重複匯入 |
| `type` | ✔ | 格式名稱，規則見〈5.1〉 |
| `typeVersion` | ✔ | 正整數，格式本身的版本 |
| `title` | ✔ | 1–200 字，顯示在選擇器、存檔對話框和接收端預覽 |
| `description` | | 最多 2000 字 |
| `source` | ✔ | 產生者資訊。**只作為紀錄與顯示用途，不參與路由**；從檔案匯入時不保證可信 |
| `createdAt` | ✔ | ISO 8601，必須帶時區 |
| `payload` | ✔ | 物件，內容由 `type` 定義 |

平台驗證規則：

- 只驗證信封欄位；`payload` 的內容由接收端驗證。
- 整個交換包序列化成 UTF-8 JSON 後，不得超過 **10 MB**。
- 未知的信封欄位會保留並原樣轉交，不視為錯誤。
- 驗證失敗時，`share()` 會丟出例外，因為這是產生端的程式錯誤；從檔案匯入時則以訊息框告知使用者。

產生端可以用 `ctx.exchange.create()` 產生信封，由平台自動填入 `format`、`id`、`source`、`createdAt`，見〈7.3〉。

## 5. 格式

### 5.1 命名

- 格式名稱符合 `^[a-z0-9][a-z0-9-]*(\.[a-z0-9][a-z0-9-]*)+$`，也就是至少兩段，以點分隔。
- `tooldock.*` 保留給平台標準格式。
- 擴充自訂的格式必須以**自己的擴充 id** 開頭，例如 id 為 `dbtool` 的擴充可以定義 `dbtool.migration`。平台載入 manifest 時會檢查這條規則，違反的宣告會被忽略並記錄警告。
- 其他擴充**可以接收、也可以產生**別人定義的格式，只是不能宣告它的 schema。

### 5.2 版本

- `typeVersion` 是整數。在同一個版本內只能做**向下相容**的變更，也就是新增選填欄位；接收端必須忽略不認識的欄位。
- 有不相容的變更時，`typeVersion` 加 1。
- 接收端用 `minVersion` / `maxVersion` 宣告它能處理的範圍，預設兩者都是 1。

### 5.3 平台標準格式（v1）

| 格式 | 用途 | 定義 |
|---|---|---|
| `tooldock.sql` | 要在資料庫執行的 SQL 與參數，包含查詢 | 〈6〉 |
| `tooldock.table` | 表格資料（欄位與列） | 〈5.4〉 |
| `tooldock.text` | 純文字、Markdown、CSV 等文字內容 | 〈5.5〉 |

標準格式的 JSON Schema 放在 `packages/types/exchange/<type>.v<version>.schema.json`，TypeScript 型別放在 `tooldock-types`，產生端可以用它們自我檢查。

### 5.4 `tooldock.table` v1

```jsonc
"payload": {
  "columns": [
    { "name": "emp_no", "title": "員工編號", "type": "string" },
    { "name": "hours",  "title": "時數",     "type": "decimal" }
  ],
  "rows": [
    ["A001", "12.5"],
    ["A002", null]
  ]
}
```

- `columns[].type` 使用〈6.4〉的型別名稱。
- `rows` 是二維陣列，每一列的長度必須等於 `columns` 的數量；值的表示法依照〈6.4〉的「值的 JSON 表示」。

### 5.5 `tooldock.text` v1

```jsonc
"payload": { "mime": "text/csv", "text": "a,b\n1,2", "filename": "export.csv" }
```

- `mime` 必填，限 `text/*` 或 `application/json`。
- `filename` 選填，接收端存檔時可作為預設檔名。
- 接收端可以用 `imports[].mimes` 只接收特定的 `mime`（1.7.0 起，〈7.1〉）。

### 5.6 自訂格式

在 manifest 宣告格式和它的 schema：

```jsonc
"contributes": {
  "exchange": {
    "formats": [
      { "type": "dbtool.migration", "version": 1, "title": "資料庫遷移腳本", "schema": "exchange/migration.v1.schema.json" }
    ]
  }
}
```

- `schema` 是相對於擴充根目錄的 JSON Schema 路徑，用來描述 `payload`。平台**不會**用它驗證資料，它的用途是文件與商店展示，讓其他開發者知道該怎麼產生這個格式。
- 商店索引會收錄 `formats`（見〈9〉），所以其他工具即使沒有安裝定義者，也查得到格式說明。

## 6. `tooldock.sql` v1

### 6.1 範例

```jsonc
"payload": {
  "dialect": "mssql",
  "target": { "name": "人事資料庫", "database": "HR" },
  "transaction": "all",
  "params": {
    "month":    { "type": "string", "value": "2026-11" },
    "approver": { "type": "string", "from": "prompt", "label": "核准人", "required": true },
    "now":      { "type": "datetime", "from": "context", "key": "now" }
  },
  "statements": [
    {
      "id": "update-hours",
      "sql": "UPDATE overtime SET hours = :hours, approved_by = :approver, updated_at = :now WHERE emp_no = :emp AND month = :month",
      "rows": [
        { "emp": "A001", "hours": { "type": "decimal", "value": "12.5" } },
        { "emp": "A002", "hours": { "type": "decimal", "value": "8" } }
      ],
      "expect": { "maxAffectedRows": 2 }
    },
    {
      "id": "log",
      "sql": "INSERT INTO overtime_log (month, approver, created_at) VALUES (:month, :approver, :now)"
    },
    {
      "id": "check",
      "sql": "SELECT emp_no, hours FROM overtime WHERE month = :month AND emp_no IN ('A001', 'A002') ORDER BY emp_no",
      "maxRows": 100
    }
  ],
  "expect": { "maxAffectedRows": 10 }
}
```

### 6.2 欄位

| 欄位 | 必填 | 說明 |
|---|---|---|
| `dialect` | ✔ | `sqlite`、`mssql`、`postgres`、`mysql`、`oracle`、`generic`。`generic` 表示只使用 ANSI SQL，接收端可在任何資料庫執行 |
| `target` | ✔ | 目標資料庫的**邏輯描述**，見〈6.6〉。**不得**包含主機、帳號、密碼或連線字串 |
| `transaction` | | `all`（預設）：全部成功才提交；`statement`：每個語句各自一個交易；`none`：不使用交易 |
| `params` | | 共用參數，所有語句都能使用 |
| `statements` | ✔ | 1 到 1000 個語句，依序執行 |
| `expect` | | 整包的檢查條件，見〈6.7〉 |
| `options` | | 產生端的請求，例如 `{ "allowDDL": true }`。**只是請求**，接收端依自己的設定決定是否允許，見〈6.8〉 |

每個 `statements[]` 項目：

| 欄位 | 必填 | 說明 |
|---|---|---|
| `id` | | 識別碼，用在預覽和結果 |
| `sql` | ✔ | 一個 SQL 語句，參數寫成 `:名稱`。**一個項目只能有一個語句**，接收端應拒絕以 `;` 串接多個語句 |
| `params` | | 這個語句專用的參數 |
| `rows` | | 參數列。有 `rows` 時，每一列執行一次；`rows` 是空陣列時這個語句不執行；沒有 `rows` 時執行一次 |
| `expect` | | 這個語句的檢查條件 |
| `maxRows` | | 只對會回傳結果的語句有效：每次執行最多回傳幾列，見〈6.10〉 |

### 6.3 參數

**佔位符**一律寫成 `:名稱`，名稱符合 `[A-Za-z_][A-Za-z0-9_]*`。接收端負責轉成驅動程式的寫法，例如 mssql 的 `@名稱`、postgres 的 `$1`、mysql 的 `?`。解析時必須略過以下位置：

- 字串常值 `'…'` 和識別字引號 `"…"`、`` `…` ``、`[…]`
- 註解 `-- …` 與 `/* … */`
- postgres 的型別轉換 `::`

**參數值查找順序**：`rows[i]` → `statements[].params` → `payload.params`，先找到的優先。

- v1 不支援陣列參數，例如 `IN (:list)`。請展開成多個參數 `IN (:e1, :e2)`，或者用 `rows` 逐筆執行。
- SQL 中出現的參數找不到值：整包拒絕。
- 定義了參數但沒有被使用：只顯示警告。

**參數來源**（`from`，預設為 `value`）：

| `from` | 說明 | 其他欄位 |
|---|---|---|
| `value` | 固定值 | `value` |
| `prompt` | 匯入時由使用者在預覽畫面填寫 | `label`（必填）、`required`、`default`、`options`（下拉選項陣列）、`description` |
| `context` | 由接收端在執行時提供 | `key`，見下表 |

`context` 可用的 `key`：

| key | 型別 | 內容 |
|---|---|---|
| `now` | `datetime` | 開始執行的時間，同一包內所有語句共用同一個值 |
| `today` | `date` | 開始執行當天的日期，依本機時區 |
| `user` | `string` | 作業系統使用者名稱 |
| `package.id` | `string` | 交換包的 `id` |
| `source.extId` | `string` | 交換包的 `source.extId` |

`prompt` 和 `context` 只能出現在 `payload.params` 或 `statements[].params`，不能出現在 `rows` 裡，因為每一列都問使用者並不合理。

### 6.4 型別與值的表示

完整寫法是 `{ "type": "<型別>", "value": <值> }`。產生端也可以直接寫 JSON 純量，這時依下表推斷型別：

| JSON 值 | 推斷的型別 |
|---|---|
| 字串 | `string` |
| 整數 | `int` |
| 非整數的數字 | `float` |
| `true` / `false` | `bool` |
| `null` | `null` |

金額、日期這類值**必須**使用完整寫法，否則會被推斷成 `float` 或 `string`。

| 型別 | `value` 的 JSON 表示 | 說明 |
|---|---|---|
| `string` | 字串 | |
| `int` | 數字 | 必須在 ±2^53−1 以內，超過時請用 `bigint` |
| `bigint` | 十進位字串 | 例如 `"9007199254740993"` |
| `decimal` | 十進位字串 | 例如 `"12.50"`。**不得用 JSON 數字**，避免失去精度 |
| `float` | 數字 | |
| `bool` | `true` / `false` | |
| `date` | `"YYYY-MM-DD"` | |
| `time` | `"HH:mm:ss"` 或 `"HH:mm:ss.SSS"` | |
| `datetime` | ISO 8601，**必須帶時區** | 例如 `"2026-11-19T09:30:00+08:00"` |
| `binary` | base64 字串 | |
| `json` | 任意 JSON | 接收端序列化成字串後傳入 |
| `null` | `null` | 不需要型別時使用 |

有型別的 null 寫成 `{ "type": "decimal", "value": null }`，給需要知道參數型別的驅動程式使用，例如 mssql。

### 6.5 語句分類

接收端在預覽前，必須依照每個語句的第一個關鍵字（略過註解和 `WITH … AS (…)` 前綴）把它分類：

| 類別 | 關鍵字 | 預設 |
|---|---|---|
| 查詢 | `SELECT`、`VALUES`、`SHOW`、`EXPLAIN`（不含 `EXPLAIN ANALYZE`） | 允許，會回傳結果，見〈6.10〉 |
| DML | `INSERT`、`UPDATE`、`DELETE`、`MERGE`、`REPLACE`、`UPSERT` | 允許 |
| DDL | `CREATE`、`ALTER`、`DROP`、`TRUNCATE`、`RENAME` | **拒絕**，除非接收端設定允許 |
| 其他 | `EXEC`、`CALL`、`PRAGMA`、`GRANT`、`REVOKE`、`EXPLAIN ANALYZE` 等 | **拒絕**，除非接收端設定允許 |

分類的特殊規則：

- `WITH … AS (…)` 之後的主要語句決定類別。例如 `WITH x AS (…) SELECT …` 是查詢，`WITH x AS (…) UPDATE …` 是 DML。
- `SELECT … INTO <資料表>`（mssql、postgres 會建立新資料表）屬於 **DDL**；mysql 的 `SELECT … INTO @變數` / `OUTFILE` 屬於**其他**。
- `PRAGMA` 在 SQLite 可能修改資料庫設定，所以歸類為「其他」。
- `EXPLAIN ANALYZE` 會實際執行後面的語句，所以歸類為「其他」。
- `INSERT/UPDATE/DELETE … RETURNING`（或 mssql 的 `OUTPUT`）屬於 DML，同時也會回傳結果，見〈6.10〉。
- 無法判斷類別的語句一律歸類為「其他」。

### 6.6 目標資料庫

```jsonc
"target": { "name": "人事資料庫", "database": "HR" }
```

- `name` 必填，是給人看的邏輯名稱；`database` 選填，是提示用的資料庫名稱。
- 接收端在預覽畫面讓使用者選擇**實際連線**（連線的帳號密碼存在接收端的 `ctx.secrets`）。
- 接收端可以記住「`source.extId` + `target.name`」對應到哪個連線，下次預先選好，但仍然需要使用者確認。
- 實際連線的方言和 `dialect` 不同時（`generic` 除外），必須顯示警告。

### 6.7 檢查條件

```jsonc
"expect": { "maxAffectedRows": 10, "affectedRows": 3 }
```

- `maxAffectedRows`：影響筆數上限，超過就回復（rollback）並回報失敗。
- `affectedRows`：影響筆數必須完全相等。
- 放在語句上時檢查該語句所有列的合計；放在 `payload` 上時檢查整包的合計。
- 影響筆數只計算 DML。查詢回傳的列數不算在內，查詢的列數上限請用 `maxRows`。
- `transaction` 為 `none` 時無法回復。這時接收端必須在預覽階段就明確告知使用者「執行到一半失敗時，已執行的部分不會回復」。

### 6.8 接收端的必要行為

接收 `tooldock.sql` 的擴充**必須**做到以下幾點。平台沒有沙箱（見〈10〉），這些行為由接收端負責，平台無法強制：

1. **預覽並確認**。執行前必須顯示：標題、來源、目標連線、每個語句的 SQL 與分類、參數值（`prompt` 參數在這裡填寫）、`rows` 筆數（可展開查看）。有查詢語句而且 `info.origin` 為 `share` 時，還要明確顯示「查詢結果會傳回給『<產生端名稱>』」。使用者按下確認後才能執行。交換包沒有任何欄位可以跳過這一步，**只有查詢的交換包也一樣**，因為它會把資料庫內容交給另一個擴充。
2. **不接受帳號密碼**。交換包只提供邏輯目標，實際連線由使用者選擇。
3. **語句分類**。依〈6.5〉拒絕不允許的語句；`options.allowDDL` 只有在接收端設定也允許時才生效（兩者都允許，DDL 才能執行）。
4. **試跑**。交換包含有 DML，`transaction` 不是 `none`，而且資料庫支援交易時，應該先在交易中執行再回復，算出影響筆數並顯示在預覽中，再請使用者確認。試跑時查詢語句也會執行，接收端可以在預覽中顯示部分結果，但**只有確認後正式執行的結果**才會回傳。只有查詢的交換包不需要試跑。
5. **防止重複匯入**。記錄已匯入的 `id`。同一個 `id` 再次匯入時要明確警告，使用者必須再確認一次。
6. **稽核紀錄**。至少記錄：時間、`id`、`title`、`source`、實際連線名稱、每個語句的影響筆數、結果。
7. **參數綁定**。一律使用驅動程式的參數綁定，**不得**把值拼接進 SQL 字串。

### 6.9 匯入結果

`ImportResult.data`（見〈7.4〉）建議格式：

```jsonc
{
  "committed": true,
  "totalAffectedRows": 3,
  "durationMs": 128,
  "statements": [
    { "id": "update-hours", "affectedRows": [1, 1] },
    { "id": "log", "affectedRows": [1] },
    {
      "id": "check",
      "results": [
        {
          "columns": [
            { "name": "emp_no", "type": "string" },
            { "name": "hours",  "type": "decimal" }
          ],
          "rows": [["A001", "12.5"], ["A002", "8"]],
          "rowCount": 2,
          "truncated": false
        }
      ]
    }
  ]
}
```

- `affectedRows` 和 `results` 都是陣列，每個元素對應語句的一次執行（`rows` 的一列）。沒有 `rows` 時只有一個元素。
- DML 才有 `affectedRows`，會回傳結果的語句才有 `results`；`… RETURNING` 兩者都有。

### 6.10 查詢與回傳結果

範例：補登加班時數後，在同一個交易中讀回結果，交給產生端確認。

```jsonc
"statements": [
  { "id": "update-hours", "sql": "UPDATE overtime SET hours = :hours WHERE emp_no = :emp AND month = :month", "rows": [ … ] },
  {
    "id": "check",
    "sql": "SELECT emp_no, hours FROM overtime WHERE month = :month AND emp_no IN (:e1, :e2) ORDER BY emp_no",
    "params": { "e1": "A001", "e2": "A002" },
    "maxRows": 100
  }
]
```

**結果格式**：每次執行的結果就是 `tooldock.table` v1 的 `payload`（〈5.4〉），另外加上：

| 欄位 | 說明 |
|---|---|
| `rowCount` | 實際回傳的列數 |
| `truncated` | 因為 `maxRows` 或大小上限而截斷時為 `true` |

- 欄位型別由接收端依驅動程式回報的型別對應到〈6.4〉。無法對應時使用 `string`，值轉成字串。
- 值的表示法依照〈6.4〉。`decimal`、`bigint` 一律回傳字串，`datetime` 一律帶時區，`binary` 一律用 base64。
- 一個語句產生多個結果集時（例如 mssql 的預存程序），只回傳第一個，並在 `ImportResult.message` 中說明。

**數量限制**：

- `maxRows`：每次執行最多回傳的列數。沒有指定時使用接收端的預設值，建議為 10,000。接收端也可以設定上限，`maxRows` 超過上限時以上限為準。
- 整個 `ImportResult.data` 序列化後不得超過 **10 MB**，和交換包的上限相同。超過時，接收端從最後一個結果開始截斷，並把對應的 `truncated` 設為 `true`。
- 截斷**不是錯誤**，交易照常提交。產生端需要完整資料時，應該自己檢查 `truncated`。

**交易**：

- 查詢會依 `transaction` 的設定，在同一個交易中執行，所以排在 DML 之後的查詢可以讀到這個交易剛寫入的資料，例如讀回新產生的流水號。
- 只有查詢的交換包，接收端可以使用唯讀交易（資料庫支援時）。
- 交易最後被回復時（例如超過 `expect` 的上限），`committed` 為 `false`，而且**不回傳任何查詢結果**，避免產生端拿到回復前的資料。

**結果的去向**：

- `info.origin` 為 `share`：結果放在 `ImportResult.data`，透過 `ShareResult.result` 交給產生端。接收端畫面也應該顯示結果。
- `info.origin` 為 `file`：沒有產生端，結果只顯示在接收端畫面。接收端應該提供「分享結果」，把每個結果用 `tooldock.table` 分享出去（〈8.1〉），讓使用者可以交給報表等其他工具。

## 7. API

### 7.1 Manifest

```jsonc
"tooldock": {
  "engine": "^1.4.0",
  "contributes": {
    "exchange": {
      "formats": [ /* 〈5.6〉 */ ],
      "imports": [
        {
          "type": "tooldock.sql",
          "minVersion": 1,
          "maxVersion": 1,
          "handler": "importSql",
          "title": "執行到資料庫",
          "view": "dbtool.main"
        }
      ]
    }
  }
}
```

| `imports[]` 欄位 | 必填 | 說明 |
|---|---|---|
| `type` | ✔ | 接收的格式 |
| `minVersion` / `maxVersion` | | 能處理的 `typeVersion` 範圍，預設都是 1 |
| `handler` | ✔ | 後端匯出的函式名稱，簽章為 `(ctx, pkg, info) => ImportResult \| Promise<ImportResult>` |
| `title` | ✔ | 顯示在選擇器，例如「執行到資料庫」 |
| `view` | | 匯入時平台先開啟這個畫面再呼叫 handler，方便接收端顯示預覽 |
| `mimes` | | 1.7.0 起。只接收 `payload.mime` 符合的交換包，例如 `["text/markdown", "text/x-markdown"]`；可以寫 `text/*`。省略時接收全部。只適用於 payload 有 `mime` 欄位的格式（目前只有 `tooldock.text`），其他格式宣告時忽略並記錄警告。比對規則見〈8.1〉 |

- 平台會為每個 `imports[].type` 自動加上 `onImport:<type>` 啟用事件，所以接收端可以延遲載入。
- 同一個擴充對同一種格式只能宣告一個 `imports` 項目；重複時只採用第一個並記錄警告。
- 擴充必須有 `main` 才能宣告 `imports`，純畫面擴充無法接收。

### 7.2 啟用與回收

- 匯入期間，平台把接收端標記為**忙碌**（沿用 `ExtensionManager.track`），因此不會被自動回收，也不受 RPC 逾時限制，等待使用者確認多久都可以。
- handler 回傳後，接收端恢復一般的回收規則。

### 7.3 產生端：`ctx.exchange`

```ts
interface ExchangeApi {
  /** 建立信封，平台填入 format、id、source、createdAt */
  create<T>(type: string, payload: T, meta: { title: string; description?: string; typeVersion?: number }): ExchangePackage<T>;

  /** 分享交換包，見〈8〉的流程 */
  share(pkg: ExchangePackage, options?: ShareOptions): Promise<ShareResult>;

  /**
   * 查詢目前能接收這個格式的擴充（不含自己、不含已停用的）；省略 typeVersion 時不限版本。
   * 1.7.0 起可以帶 { mime }，結果套用〈8.1〉的 mime 比對；省略時不比對 mime
   */
  receivers(type: string, typeVersion?: number, options?: { mime?: string }): Promise<ReceiverInfo[]>;
}

interface ShareOptions {
  /** 沒有接收端時的處理：'export'（預設）詢問是否存成 .tdx；'none' 直接回報 */
  fallback?: 'export' | 'none';
  /** 優先交給這個擴充；它無法接收時，照一般流程處理。只是建議，不會強制 */
  preferReceiver?: string;
}

interface ShareResult {
  status: 'delivered' | 'exported' | 'no-receiver' | 'cancelled' | 'error';
  receiver?: { extId: string; name: string };
  /** status 為 delivered 時，接收端的處理結果 */
  result?: ImportResult;
  /** status 為 exported 時，存檔路徑 */
  filePath?: string;
  /** status 為 error 時的訊息 */
  message?: string;
}

interface ReceiverInfo { extId: string; name: string; title: string; icon: string; /** 1.7.0 起，imports[].mimes */ mimes?: string[]; }
```

- `share()` **只有在信封驗證失敗時才丟出例外**。其他情況，包括沒有接收端、使用者取消、接收端出錯，一律以 `status` 回報。
- 畫面端以 `window.tooldock.exchange` 提供相同的 `create`、`share`、`receivers`。畫面端無法直接取得擴充版本等資訊，所以它的 `create` 回傳 `Promise<ExchangePackage>`。

### 7.4 接收端：handler

```ts
type ImportHandler = (ctx: ExtensionContext, pkg: ExchangePackage, info: ImportInfo) => ImportResult | Promise<ImportResult>;

interface ImportInfo {
  /** 'share'：由其他擴充分享；'file'：從 .tdx 檔匯入 */
  origin: 'share' | 'file';
  /** origin 為 file 時的檔案路徑 */
  filePath?: string;
  /** origin 為 share 時，由平台填入的實際產生端；預覽畫面顯示「查詢結果會傳回給『<產生端名稱>』」時使用 */
  producer?: { extId: string; name: string };
  /** 使用者是否已在工作台取消 */
  cancelled: boolean;
  /** 使用者在工作台取消匯入時觸發；已經取消時註冊會立即觸發 */
  onCancel(listener: () => void): Disposable;
}

interface ImportResult {
  status: 'imported' | 'rejected' | 'cancelled';
  /** 給使用者看的一句話，例如「已更新 3 筆」 */
  message?: string;
  /** 給產生端的結構化結果，例如〈6.9〉 */
  data?: unknown;
}
```

- handler 丟出例外時，平台把它轉成 `ShareResult { status: 'error', message }`，並記錄到接收端的輸出面板。
- handler 回傳 `undefined` 視為 `{ status: 'imported' }`；回傳的 `status` 不是 `imported`、`rejected`、`cancelled` 之一時視為錯誤。
- `info.origin === 'file'` 時，**`source` 欄位不可信**（檔案可能被修改），接收端不應依 `source.extId` 決定信任程度。
- 產生端**不能假設一定拿得到** `result`：沒有接收端時交換包會存成檔案，使用者之後可能在別的時間、別台電腦匯入，結果不會回到產生端。需要查詢結果的產生端，應該在 `status` 不是 `delivered` 時告訴使用者「結果請到接收端查看」，或者用 `fallback: 'none'` 避免存檔。

## 8. 平台流程

### 8.1 分享

1. 驗證信封（〈4〉），失敗就丟出例外。
2. 找出接收端：已啟用、`imports` 有這個 `type`、`typeVersion` 在範圍內，而且不是產生端自己。
   - **mime 比對**（1.7.0 起）：payload 有 `mime` 時，取它的本體（去掉 `;` 之後的參數、轉成小寫），把接收端分成三級：① `mimes` 有完全相同的項目；② `mimes` 有符合的 `type/*`；③ 沒有宣告 `mimes`。`mimes` 有宣告但都不符合的接收端排除。**只保留最高的一級**。例如 Markdown 編輯器宣告 `text/markdown`、資料轉換沒有宣告：分享 `text/markdown` 時只剩 Markdown 編輯器，直接交付；分享 `text/csv` 時只剩資料轉換。
   - `preferReceiver` 在三級之內都有效：產生端指定的接收端只要沒有被排除，就直接交給它。
   - 「下次自動使用這個工具」的紀錄只在保留下來的接收端中套用。
3. 依接收端數量處理：
   - **1 個**：直接交付。
   - **多個**：工作台顯示選擇器，列出 `imports[].title` 和擴充名稱，加上「存成檔案…」選項，並有「下次自動使用這個工具」核取方塊（依格式記在 `userData/exchange.json`）。有 `preferReceiver` 而且它在名單中時，直接交付給它。使用者關閉選擇器時回報 `cancelled`。
   - **0 個**：
     - `fallback: 'export'`：顯示訊息框「目前沒有可以接收『<格式標題>』的工具」，按鈕為「存成檔案…」、「到商店尋找」（〈9〉）、「取消」。存檔後回報 `exported`，取消則回報 `no-receiver`。
     - `fallback: 'none'`：直接回報 `no-receiver`。
4. 交付：如有 `view` 先開啟畫面，以 `onImport:<type>` 啟用接收端，呼叫 handler，回傳 `delivered` 和 handler 的結果。
5. 匯入進行中，狀態列顯示「匯入中：<title>」，點擊後可以取消。取消會觸發 `info.onCancel`，`share()` 立即回報 `cancelled`；接收端之後回傳的結果會被忽略。

### 8.2 從檔案匯入

進入點：

- 核心指令 `exchange.importFile`（「交換：匯入交換檔 (.tdx)」）。
- 工作台拖放 `.tdx` 檔案。
- 作業系統檔案關聯（macOS `open-file` 事件、Windows／Linux 的啟動參數；平台已在執行時由 `second-instance` 轉交）。

流程與〈8.1〉相同，差別在於：`info.origin` 為 `file`、沒有產生端（所以不排除任何擴充）、沒有接收端時改顯示「目前沒有可以接收的工具」加上「到商店尋找」，而且不提供存檔選項。

### 8.3 交換檔

- 副檔名 `.tdx`，MIME 類型 `application/vnd.tooldock.exchange+json`。
- 內容是交換包的 UTF-8 JSON（不含 BOM），建議縮排 2 格，方便人工檢查。
- 預設檔名：`<title 去除檔名不允許字元>-<YYYYMMDD-HHmm>.tdx`。
- 交換檔是純 JSON，**不需要平台也能產生**：舊版平台或外部程式只要寫出符合〈4〉的 JSON，就能在 1.4.0 以上匯入。

## 9. 商店

- `build-index.js` 把每個擴充的 `contributes.exchange.imports[].type` 和 `formats` 收錄進 `index.json`，欄位為 `exchange: { imports: string[], formats: [{ type, version, title }] }`。這些資訊來自套件內容，所以和 `engine`、`displayName` 一樣放在 `latest` 底下（`latest.exchange`）；沒有交換宣告的擴充省略這個欄位。
- 「到商店尋找」會開啟商店，篩選出 `exchange.imports` 包含該格式的擴充。
- 商店的擴充詳細頁顯示「可接收的格式」和「定義的格式」。

## 10. 安全性

- **平台沒有沙箱**（見 `docs/STORE.md`〈限制〉）。已安裝的擴充本來就能直接連線資料庫，交換機制不會讓它多出權限。〈6.8〉要求的確認流程，防的是**誤操作**和**來路不明的交換檔**，不是防惡意擴充。
- `.tdx` 檔可能來自 email 或網路分享，內容一律視為不可信：`source` 可以被偽造，`title` 和 `description` 可以寫得很誤導人。所以接收端預覽時，**必須顯示實際的 SQL 與參數**，不能只顯示標題。
- **查詢會把資料庫內容交給產生端**。產生端拿到資料後能做什麼，平台無法限制，所以接收端的確認畫面必須清楚標示資料的去向（〈6.8〉第 1 點）。來路不明的 `.tdx` 檔則沒有這個問題，因為從檔案匯入時結果只會顯示在接收端。
- 交換包不得包含帳號密碼。標準格式的 schema 沒有可以放這類資料的欄位；自訂格式的設計者也應遵守。
- 平台不會記錄 `payload` 的內容，只在生命週期時間軸和輸出面板記錄：時間、`id`、`type`、`title`、產生端、接收端、結果。

## 11. 後續版本（不在 v1）

- **附件**：`.tdx` 改成 zip（`package.json` 加上附件檔），支援 `tooldock.file` 與超過 10 MB 的資料。屆時信封升到 `tooldock.exchange/2`，v1 的純 JSON 仍然可以匯入。
- **語句串接**：把前一個查詢的結果當成後續語句的參數，例如先查出主管名單再逐筆寫入。v1 只能由產生端拿到結果後，再分享一個新的交換包。
- **陣列參數**：支援 `IN (:list)`。
- **剪貼簿**：交換包複製到剪貼簿，從另一個工具貼上。

## 12. 相容性

| 情況 | 行為 |
|---|---|
| 平台 < 1.4.0 | 沒有 `ctx.exchange`、`window.tooldock.exchange`；manifest 的 `contributes.exchange` 會被忽略 |
| 產生端想同時支援舊平台 | 先判斷 `if (ctx.exchange)`；舊平台可以改用 `ctx.window.showSaveDialog` 自己寫出 `.tdx` 檔（〈8.3〉） |
| 接收端在舊平台 | 不會被呼叫，其餘功能正常 |
| 接收端只支援較舊的 `typeVersion` | 不會出現在接收端名單中，等同沒有接收端 |
| 平台 < 1.7.0 | `imports[].mimes` 被忽略，所有接收端都會出現在選擇器；有宣告 `mimes` 的接收端要自己處理不符合的 `mime`（例如提示「已當作純文字開啟」） |

## 13. 實作計畫

### 13.1 修改位置

| 位置 | 內容 |
|---|---|
| `src/main/extension-manager.js` | `readManifest` 解析並驗證 `contributes.exchange`、自動加上 `onImport:*`；找出接收端；交付（`track` + 啟用 + `exchange.import` RPC）；服務 `exchange.share`、`exchange.receivers` |
| `src/main/exchange.js`（新增） | 信封驗證、`create`、`.tdx` 讀寫、預設接收端紀錄（`userData/exchange.json`）、可替換的互動（存檔路徑、沒有接收端的訊息框、檔案錯誤） |
| `src/main/lifecycle.js` | 時間軸記錄平台事件（`record`），資料交換的紀錄出現在生命週期時間軸 |
| `src/host/host-runtime.js` | `ctx.exchange`；處理 `exchange.import`（依 handler 名稱呼叫匯出函式）、`exchange.cancel` |
| `src/main/main.js` | 核心指令 `exchange.importFile`、`exchange.resetDefaults`（「交換：重設預設接收工具」）、`open-file`、`second-instance` 傳入 `.tdx` 路徑 |
| `src/renderer/workbench.js` | 接收端選擇器、沒有接收端的提示、狀態列「匯入中」與取消、拖放 `.tdx` |
| `src/sdk/ui-sdk.js` | `window.tooldock.exchange` |
| `src/preload/preload.js` | 拖放檔案的實際路徑（`webUtils.getPathForFile`） |
| `src/main/selftest-exchange.js`（新增） | 〈13.3〉的自我測試 |
| Oracle SQL 工具（商店擴充，另一個 repo） | `tooldock.sql` 的接收端：`backend/sql-exchange.js` 不依賴驅動程式，其他資料庫工具可直接沿用；`backend/exchange-exec.js` 是 Oracle 的執行部分 |
| `packages/types` | `ExchangeApi` 等型別、標準格式的型別與 JSON Schema |
| `packages/cli` | `tooldock-ext validate` 檢查 `contributes.exchange` |
| `registry/scripts` | 收錄 `exchange` 欄位 |
| 平台 `package.json` | 版本 1.4.0；electron-builder 加上 `.tdx` 的 `fileAssociations` |
| `docs/EXTENSION_GUIDE.md` | 新增「資料交換」章節，連到本文件 |

### 13.2 里程碑

1. **信封與路由**：`exchange.js`、manifest 解析、`ctx.exchange`、單一接收端交付、沒有接收端時存檔。
2. **工作台**：選擇器、預設接收端、狀態列取消、從檔案匯入（指令與拖放）。
3. **標準格式**：`tooldock-types` 型別與 schema、CLI 驗證。
4. **範例**：
   - `data-converter` 接收 `tooldock.text`。
   - `table-report` 接收 `tooldock.table`，並把報表分享成 `tooldock.table`。
   - `tooldock.sql` 由商店的 Oracle SQL 工具（1.3.0）接收，完整實作〈6.8〉。最初以內建的「資料庫執行器」（SQLite）作為參考實作，後來改由 Oracle SQL 工具取代並移除。
5. **商店與檔案關聯**：索引收錄、到商店尋找、`.tdx` 檔案關聯。

### 13.3 自我測試新增項目

- 信封驗證：缺欄位、`type` 不合法、超過 10 MB 時丟出例外。
- 自訂格式未以自己的擴充 id 開頭時被忽略。
- 單一接收端交付，延遲載入的接收端會被啟用，匯入期間不會被回收。
- `typeVersion` 超出範圍時視為沒有接收端。
- 沒有接收端：`fallback: 'none'` 回報 `no-receiver`；`fallback: 'export'` 存成 `.tdx`。目前的自我測試沒有處理對話框的機制，`exchange.js` 需要提供可替換的存檔路徑選擇函式，讓自我測試直接指定暫存目錄的路徑。
- 從 `.tdx` 匯入時 `info.origin` 為 `file`。
- handler 丟出例外時回報 `error`，產生端不受影響。
- 取消匯入時回報 `cancelled`。
- `tooldock.sql` 的接收端測試放在 Oracle SQL 工具的 repo（`npm test`）：參數解析（略過字串、註解、`::`、Oracle 的 `q'[…]'`）、型別轉換、語句分類；連到 Oracle 時另外測試同一交易讀得到寫入、`maxAffectedRows` 超過時回復且不回傳查詢結果、試跑、`maxRows` 截斷、`transaction: statement`。
