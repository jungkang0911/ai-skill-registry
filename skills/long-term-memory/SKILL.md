---
name: long-term-memory
description: >
  操作 https://rag.loanhusband.com/api 的長期記憶系統。
  支援 save / read / search / list / catalog / update 六個動作，縮寫 ltm 同效。
  使用者不需要記指令，直接用自然語言說要做什麼即可。
triggers:
  - "long-term-memory"
  - "ltm"
---

# long-term-memory

RAG 長期記憶系統 Skill。

Skill 來源：`https://rag.loanhusband.com/share/AI_MEMORY_SKILL.md`
Fallback Prompt：`https://rag.loanhusband.com/share/AI_MEMORY_PROMPT.md`
Base URL：`https://rag.loanhusband.com/api`
Auth：Bearer token（email 登入取得，存於 rag.json）

---

## 安裝指令（使用者對 AI 說）

```
幫我從 https://rag.loanhusband.com/share/AI_MEMORY_SKILL.md 安裝 long-term-memory skill，
Email <你的email>，RAG 密碼 <密碼>
```

AI 執行：
1. 讀取 SKILL.md（本檔）
2. 呼叫 `POST /api/auth/login` 取得 token
3. 將 token 寫入 `~/.claude/rag.json`
4. 確認登入成功，回報 namespace

---

## 使用方式

使用者**不需要記指令或 source_key**，直接用自然語言：

- 「幫我把這次對話保存起來」
- 「查我的 namespace 裡跟 skill 有關的」
- 「搜尋跟 SAP 有關的記憶」
- 「我之前存了哪些東西？」

你負責整理摘要、決定 source_key、執行 API 呼叫。

---

## 對話開場流程（縮小範圍）

每次對話開始，若使用者說明正在做什麼，**不要全量載入記憶**，執行以下流程：

1. 從使用者的描述推測 namespace 和關鍵字
2. 執行 `GET /api/chunks?namespace=<ns>&q=<關鍵字>`
3. 只顯示標題清單（不顯示內容）：
   ```
   找到以下相關記憶：
   1. jungkang0911/cloudplay-yunyu — CloudPlay 雲遊開發記錄
   2. jungkang0911/cloudplay-db    — 資料庫架構記錄
   ```
4. 等使用者說「載入第1個」或「新的」
5. **只載入那一筆**的完整 content，作為本次對話的上下文
6. 其他記憶不載入，保持對話範圍精準

若沒找到相關記憶，直接告知並開始新的。

---

## Browse 流程（標題找，編號操作）

使用者說「查我的 xxx 記憶」或「查 rpa 裡跟 xxx 有關的」時：

1. 執行 `GET /api/chunks?namespace=<ns>&q=<關鍵字>`
2. 顯示編號清單
3. 等使用者說「讀第1個」、「更新第2個」、「刪第1個」後再執行

**保存對話時若未指定 source_key：**
1. 先執行 `GET /api/chunks?namespace=<ns>&q=<對話主題關鍵字>`
2. 顯示相關的現有記憶清單
3. 詢問：「要更新哪一筆，還是建立新的？」
4. 使用者選擇後才執行儲存

---

## 刪除

```
DELETE /api/memories/<source_key>
Authorization: Bearer <token>
```

cascade 刪除該 source_key 的所有 documents 和 chunks。只能刪除自己 namespace 的記憶。

---

## source_key 規則

格式：`<namespace>/<id>`，僅小寫英文、數字、連字符

| 用途 | 格式 | 範例 |
|------|------|------|
| 個人筆記 | `<email前綴>/<主題>` | `jungkang0911/pr006` |
| 團隊共用 | `<共用namespace>/<id>` | `rpa/pr006`、`team/convention` |

Namespace 由系統從 email 自動派生（`jungkang0911@gmail.com` → `jungkang0911`）。

---

## API 對應

### 認證

```
POST /api/auth/login
Content-Type: application/json

{ "email": "<email>", "password": "<密碼>" }
```

回傳：`{ access_token, refresh_token, expires_in, email, namespace }`

Token 有效期約 1 小時，到期前自動用 refresh_token 換新。

### save（新增）
```
POST /api/memories
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "source_key": "<namespace>/<id>",
  "title": "記憶標題",
  "path": "notes.md",
  "summary": "<content 前 80 字>",
  "tags": "<namespace>,<id>",
  "content": "<內容>"
}
```

### read（讀取）
```
GET /api/chunks?source_key=<namespace>/<id>
Authorization: Bearer <access_token>
```

### search（搜尋）
```
GET /api/chunks?q=<關鍵字>
GET /api/chunks?namespace=<ns>&q=<關鍵字>
Authorization: Bearer <access_token>
```

### list（列出）
```
GET /api/chunks?namespace=<ns>
Authorization: Bearer <access_token>
```

### catalog（全覽）
```
GET /api/catalog
Authorization: Bearer <access_token>
```

### update（更新內容）
直接重新 POST，伺服器會 upsert（相同 source_key + path 覆蓋）：
```
POST /api/memories   ← 同 save
Authorization: Bearer <access_token>
```

> 注意：`PATCH /api/chunks/:chunk_id` 只能更新 metadata（status、tags、summary），
> **無法更新 content**，content 變更一律用 POST。

---

## 認證處理

按以下優先順序取得 token：

1. **設定檔** `~/.claude/rag.json`（推薦）
   ```json
   {
     "email": "your@email.com",
     "access_token": "<token>",
     "refresh_token": "<refresh>",
     "expires_at": 1234567890000
   }
   ```
   讀取方式（PowerShell）：
   ```powershell
   $cfg = Get-Content "$env:USERPROFILE\.claude\rag.json" -Raw | ConvertFrom-Json
   $token = $cfg.access_token
   ```

2. **Token 過期處理**：
   若 `expires_at < (Get-Date).ToUnixTimeMilliseconds()`，
   執行 refresh 並更新 rag.json：
   ```powershell
   $body = @{ refresh_token = $cfg.refresh_token } | ConvertTo-Json
   $r = Invoke-RestMethod -Uri "https://rag.loanhusband.com/api/auth/refresh" `
        -Method POST -ContentType "application/json; charset=utf-8" `
        -Body ([System.Text.Encoding]::UTF8.GetBytes($body))
   # 更新 rag.json 中的 access_token / refresh_token / expires_at
   ```

3. 若 rag.json 不存在或 refresh 失敗，提示：
   「請重新登入：`POST /api/auth/login` with email + password，
   或至 https://rag.loanhusband.com 登入後取得 token。」
   **不要**要求使用者在對話中輸入明文密碼

---

## 安全規則

若 save / update 的內容包含以下任一項，拒絕並警告：
- API key：`sk-`、`Bearer `、`ghp_` 開頭
- 私鑰：`-----BEGIN.*PRIVATE KEY-----`
- 密碼欄位：`password\s*[=:]`、`secret\s*[=:]`
- 含密碼的資料庫連線字串

---

## 中文編碼注意事項

在 Windows 上執行 API 呼叫時，**必須使用 PowerShell**（`Invoke-RestMethod`），
不可使用 Bash / Git Bash 的 `curl` 傳送含中文的 JSON。

```powershell
$cfg = Get-Content "$env:USERPROFILE\.claude\rag.json" -Raw | ConvertFrom-Json
$token = $cfg.access_token

$body = @{ source_key="..."; content="中文內容" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://rag.loanhusband.com/api/memories" -Method POST `
  -Headers @{ Authorization="Bearer $token" } `
  -Body ([System.Text.Encoding]::UTF8.GetBytes($body))
```

---

## 錯誤處理

| 狀況 | 處理 |
|------|------|
| 401 | Token 無效，執行 refresh 或重新登入 |
| 403 | 嘗試存取他人 namespace，檢查 source_key |
| 404 on read | source_key 不存在，建議 `ltm catalog` 確認 |
| 網路失敗 | 提示確認網路或 VPN |

---
