---
name: ntfy-push
description: >
  透過 ntfy 發送推播通知到使用者裝置。
  任務完成、失敗、需要注意時主動推播。
triggers:
  - "ntfy"
  - "ntfy-push"
  - "推播"
---

# ntfy-push

透過 ntfy HTTP API 發送推播通知。

## 設定

- Server：`https://ntfy.sh`
- Topic：`JAlert`
- Auth：不需要（public topic）

## 發送方式（PowerShell）

```powershell
$message = "你的訊息"
Invoke-RestMethod `
  -Method Post `
  -Uri "https://ntfy.sh/JAlert" `
  -Headers @{ Title = "Claude" } `
  -ContentType "text/plain; charset=utf-8" `
  -Body ([System.Text.Encoding]::UTF8.GetBytes($message))
```

## 使用時機

- 長時間任務完成後
- 發生錯誤或需要使用者確認時
- 使用者明確說「做完通知我」

## 規則

- 訊息保持簡短（一兩句）
- 不要為每個小步驟都發通知，只在重要節點推播
- 可加 `Priority` header：`min / low / default / high / urgent`
- 可加 `Tags` header：ntfy 支援的 emoji tag，例如 `white_check_mark`
