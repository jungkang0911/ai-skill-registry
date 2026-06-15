# todo-tracker skill

追蹤個人 TODO 清單，讀寫存放於 RAG 長期記憶的 `jungkang0911/rag-skill-todos`。

## 觸發方式

使用者說以下任何一種時啟動：
- `/todo`、`todo list`、`todo add`、`todo done`、`todo update`、`todo clean`
- 「看一下我的 todo」、「加個 todo」、「這個完成了」、「更新 todo」

## Source Key

```
jungkang0911/rag-skill-todos
```

## Token 讀取

```powershell
$token = (Get-Content "C:\Users\15168\.claude\rag.json" | ConvertFrom-Json).access_token
$headers = @{Authorization = "Bearer $token"}
```

## 支援動作

### list — 列出所有 TODO

讀取內容後，分組顯示：
- **未完成** `[ ]`
- **已完成** `[x]`

```powershell
$r = Invoke-RestMethod -Uri "https://rag.loanhusband.com/api/chunks?source_key=jungkang0911/rag-skill-todos" -Headers $headers
$r.rows[0].content
```

顯示格式：
```
## 未完成（N 項）
1. 項目說明（截止：...）
...

## 已完成（N 項）
1. 項目說明（完成：...）
```

---

### add <內容> — 新增 TODO

1. 讀取現有 content
2. 在對應分類（技術/工作/其他）的 `## 待處理` 區塊末尾加入：
   ```
   - [ ] **{內容}**
     - 加入：{今天日期}
   ```
3. Upsert 回 RAG

若使用者未指定分類，放入 `## 其他待辦`。

---

### done <關鍵字> — 標記完成

1. 讀取現有 content
2. 找到含關鍵字的 `[ ]` 行
3. 若只有一筆：直接改為 `[x]`，在項目下加 `  - 完成：{今天日期}`
4. 若有多筆：列出候選讓使用者選擇
5. Upsert 回 RAG

---

### update <關鍵字> <新內容> — 更新項目

1. 讀取現有 content
2. 找到含關鍵字的項目
3. 更新說明或子項目內容
4. 加上 `  - 更新：{今天日期}`
5. Upsert 回 RAG

---

### clean — 清除已完成項目

1. 讀取現有 content
2. 移除所有 `[x]` 項目（含子行）
3. 列出將被移除的項目請使用者確認
4. 確認後 Upsert 回 RAG

---

## Upsert 寫回範例

```powershell
$body = @{
    source_key = "jungkang0911/rag-skill-todos"
    title      = "TODO 追蹤清單"
    path       = "todos.md"
    summary    = "個人 TODO 追蹤，含技術待辦與工作追蹤"
    tags       = @("todo","追蹤")
    content    = $newContent
} | ConvertTo-Json -Depth 3

$bytes = [System.Text.Encoding]::UTF8.GetBytes($body)
Invoke-RestMethod -Uri "https://rag.loanhusband.com/api/memories" -Method POST `
    -Headers @{Authorization="Bearer $token"} -Body $bytes -ContentType "application/json"
```

## 注意事項

- 中文內容一律用 PowerShell `Invoke-RestMethod`，不用 Bash curl
- 每次動作前先 read 最新 content，避免覆蓋他人更新
- source_key 固定為 `jungkang0911/rag-skill-todos`，不建立新的
- 操作完成後回報：動作類型、影響項目數、目前未完成總數
