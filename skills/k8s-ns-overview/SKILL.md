---
name: k8s-ns-overview
description: 快速了解公司內部 k8s namespace 環境概況（pods、services、configmap 設定、resource 使用）
user-invocable: true
---

使用者呼叫此 skill 時，格式為 `/k8s-ns-overview <namespace> [context]`。

- `namespace`：必填，指定要查詢的服務 namespace（如 `app-middleware`、`member-center` 等）
- `context`：選填，預設使用 `kubectl config current-context`；可指定切換（如 `aks-clg-sit01`）

若未提供 namespace，執行 `kubectl get namespaces` 列出清單後，請使用者指定再繼續。

## 已知 Context 對照

| Context | 系統 | 環境 |
|---|---|---|
| `aks-ch-dev01` | CH | Dev |
| `aks-clg-sit01` | CLG | SIT |
| `aks-clg-uat01` | CLG | UAT |
| `aks-sigv-ph-uat01` | SIGV PH | UAT |

## 執行步驟

### 0. 處理認證 / SSL Proxy 問題

公司網路環境可能因 Proxy 自簽憑證導致 `kubelogin` 無法連到 Azure AD，執行 kubectl 前先設定：

```powershell
$env:AZURE_CLI_DISABLE_CONNECTION_VERIFICATION = "1"
```

若使用者有指定 context，先切換：
```
kubectl config use-context <context>
```

若仍然失敗，提示使用者：
> kubelogin 認證失敗，可能需要切換網路／VPN，或手動執行 `az login` 後重試。

### 1. 確認 context
```
kubectl config current-context
```

### 2. 取得 namespace 全覽
```
kubectl get pods,svc,deployment -n <namespace> -o wide
```
- 標示出非 Running 的 pod（含 restarts 次數高的）
- 標示出 EXTERNAL-IP 不為 none 的 service

### 3. 取得 ConfigMap 關鍵設定
列出該 namespace 所有 configmap：
```
kubectl get configmap -n <namespace>
```
針對名稱包含 `.configs` 的 configmap 讀取內容：
```
kubectl get configmap <name> -n <namespace> -o jsonpath='{.data}'
```
從內容中整理並分組顯示：
- **API 端點** (`ApiUrl__*`)：列出服務名稱 → URL
- **資料庫連線** (`ConnectionStrings__*`)：列出 DB 名稱 → Host/ServiceName（密碼遮蔽）
- **功能開關 / 環境旗標**：`ProdEnvironment`, `TestMode`, `LogSetting__*` 等
- **Timeout 設定**：`TimeOutSetting__*`, `SlowHostedTime`

若找不到含 `.configs` 的 configmap，跳過此區塊並繼續。

### 4. 取得 Deployment resource 設定
```
kubectl get deployment -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[0].resources}{"\n"}{end}'
```
列出每個 deployment 的 CPU/Memory requests 與 limits。

### 5. 取得最近 Events（若有異常）
```
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20
```
只顯示 Warning 等級的事件。

### 6. 取得 Pod restart 統計
```
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.containerStatuses[*]}{.restartCount}{end}{"\n"}{end}'
```

## 輸出格式

以 Markdown 表格和區塊呈現，結構如下：

```
# K8s Namespace 概況: <namespace>
Context: <current-context>

## Pods 狀態
| Pod | Status | Restarts | Age |
...

## Services
| Service | Type | ClusterIP | Port |
...

## API 端點設定
| 服務 | URL |
...

## 資料庫連線
| 名稱 | Host | ServiceName |
...

## 環境旗標
- ProdEnvironment: N/Y
- TestMode: Y/N
- DebugLogFlag: Y/N
...

## Resource 配置
| Deployment | CPU req/limit | Memory req/limit |
...

## 近期警告事件（若有）
...
```

## 注意事項
- 此 skill 只做 read-only 查詢，不會修改任何 k8s 資源
- 若 namespace 不存在，提示使用者並列出可用的 namespaces
- 若某個步驟查詢失敗，記錄錯誤後繼續執行其餘步驟
