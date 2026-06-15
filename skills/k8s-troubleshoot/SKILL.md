---
name: k8s-troubleshoot
description: 快速使用 kubectl 對 Kubernetes 問題做唯讀排查與診斷。當使用者要查 namespace、pod restart、CrashLoopBackOff、ImagePullBackOff、Pending、deployment rollout、service 連線異常、configmap/secret、events、logs、probe 失敗、資源限制或節點排程問題時使用。適合直接接手 K8S 問題並執行指令整理結論。
user-invocable: true
---

## Connectivity-First Rule

When the user only asks to test connectivity, verify the smallest network
surface first and do not start a full Kubernetes investigation unless the
result points back to cluster runtime.

Use this order:

1. DNS resolution for the host.
2. HTTP request to the exact URL, if a URL is provided.
3. TCP port test only when HTTP cannot produce a response or when the user
   explicitly asks for a port-level check.

Interpretation:

- If HTTP returns any status code such as `200`, `301`, `401`, `403`, `404`, or
  `500`, the host and TCP port are reachable. Report the HTTP status and focus
  next on path, virtual host, routing, authentication, or upstream app behavior.
- If HTTP times out and TCP also fails, treat it as a network, firewall, DNS, or
  target availability problem.
- If a TCP test hangs but HTTP already returned a response, do not wait on or
  over-weight the TCP probe. Use the HTTP response as stronger evidence that
  port connectivity exists.
- Keep the first response concise: confirmed DNS/IP, HTTP status, TCP result if
  available, and the next concrete owner or check.

Only expand into Kubernetes checks such as pods, services, endpoints, events,
and logs when the user asks for K8s details, provides a namespace or workload,
or the simple connectivity test suggests the failure is inside the cluster.


使用此 skill 時，格式可為：
- `/k8s-troubleshoot`
- `/k8s-troubleshoot [namespace]`
- `/k8s-troubleshoot [namespace] [target]`
- `/k8s-troubleshoot [namespace] [target] [symptom]`

預設規則：
- 未指定 namespace 時，先列出目前 context 與 namespaces，再根據名稱猜測最可能目標
- 未指定 target 時，先從 namespace 全覽找異常 pod / deployment / service
- 預設只做 read-only 查詢；若需要 patch、restart、delete、rollout undo，先明確詢問使用者

## 執行原則

先做最小必要查詢，縮小問題範圍，再往下深挖。

輸出時要：
- 先寫明目前 context、namespace、target
- 先給結論，再列證據
- 明確區分「已確認」與「推論」
- 如果要使用者後續處理，直接列出建議動作

## 基本流程

### 1. 確認叢集與 namespace
先確認目前 context：
```bash
kubectl config current-context
```

如果 namespace 未指定或疑似不存在：
```bash
kubectl get ns
```

### 2. 取得 namespace 全覽
```bash
kubectl get pods,svc,deployment -n <namespace> -o wide
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

先找：
- 非 `Running` / `Completed` 的 pod
- restart 次數高的 pod
- `deployment` 沒有 ready replica
- `service` 無 endpoints 或對應不到 pod
- 近期 `Warning` 類事件

### 3. 如果有明確 target，直接深入
Pod：
```bash
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --tail=200
kubectl logs <pod> -n <namespace> --previous --tail=200
kubectl get pod <pod> -n <namespace> -o yaml
```

Deployment：
```bash
kubectl describe deploy <deployment> -n <namespace>
kubectl rollout status deploy/<deployment> -n <namespace> --timeout=120s
kubectl get rs -n <namespace> -l app=<app-label> -o wide
kubectl get deploy <deployment> -n <namespace> -o yaml
```

Service：
```bash
kubectl describe svc <service> -n <namespace>
kubectl get endpoints <service> -n <namespace> -o yaml
kubectl get pod -n <namespace> --show-labels
```

ConfigMap / Secret：
```bash
kubectl get configmap -n <namespace>
kubectl get secret -n <namespace>
kubectl get configmap <name> -n <namespace> -o yaml
kubectl get secret <name> -n <namespace> -o yaml
```

Node / 排程：
```bash
kubectl describe node <node-name>
kubectl top pod -n <namespace>
kubectl top node
```

## 常見症狀與查法

### A. Pod restart / CrashLoopBackOff
先查：
```bash
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous --tail=200
```

重點看：
- `Last State`
- `Exit Code`
- `OOMKilled`
- `Liveness probe failed`
- `Startup probe failed`
- 容器是否剛啟動就退出

判讀方向：
- `Exit Code 137`：先懷疑 OOM、被 kubelet kill、或 probe fail 後被終止
- `OOMKilled`：優先看 memory limit / 實際用量
- `probe failed`：看健康檢查路徑、port、初始延遲、啟動時間

### B. ImagePullBackOff / ErrImagePull
先查：
```bash
kubectl describe pod <pod> -n <namespace>
kubectl get deploy <deployment> -n <namespace> -o yaml
kubectl get secret -n <namespace>
```

重點看：
- image 名稱、tag 是否正確
- `imagePullSecrets` 是否存在未展開 placeholder
- 事件中是否有 `ErrImagePull`、`ImagePullBackOff`、`FailedToRetrieveImagePullSecret`
- 同 namespace 其他服務是否從同 registry 正常拉 image

### C. Pending / 排程失敗
先查：
```bash
kubectl describe pod <pod> -n <namespace>
kubectl describe node <node-name>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

重點看：
- CPU / memory request 是否太高
- node selector / affinity / taint toleration 是否不匹配
- PVC 是否綁定失敗
- 是否出現 `Insufficient cpu`、`Insufficient memory`

### D. Deployment rollout 卡住
先查：
```bash
kubectl rollout status deploy/<deployment> -n <namespace> --timeout=120s
kubectl describe deploy <deployment> -n <namespace>
kubectl get rs -n <namespace> -o wide
kubectl get pods -n <namespace> -l app=<app-label> -o wide
```

重點看：
- 新 ReplicaSet 是否建立
- 新 Pod 是否 ready
- 舊 Pod 是否卡在 terminating
- maxUnavailable / maxSurge 是否導致 rollout 緩慢

### E. Service 連不到 / API timeout
先查：
```bash
kubectl describe svc <service> -n <namespace>
kubectl get endpoints <service> -n <namespace> -o yaml
kubectl get pods -n <namespace> --show-labels
kubectl logs <pod> -n <namespace> --tail=200
```

重點看：
- service selector 是否選到 pod
- endpoint 是否為空
- pod 是否真的 listen 在正確 port
- app log 是否已有 downstream timeout / connection refused

### F. ConfigMap / Secret / 環境變數問題
先查：
```bash
kubectl describe pod <pod> -n <namespace>
kubectl get configmap <name> -n <namespace> -o yaml
kubectl get secret <name> -n <namespace> -o yaml
kubectl get deploy <deployment> -n <namespace> -o yaml
```

重點看：
- `envFrom` / `valueFrom` 是否指到不存在資源
- key 名稱是否正確
- 應用啟動 log 是否出現缺少設定或連線字串錯誤

## 快速排查命令組合

只知道 namespace，不知道哪裡壞：
```bash
kubectl get pods,svc,deployment -n <namespace> -o wide
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -30
kubectl top pod -n <namespace>
```

知道 deployment 名稱：
```bash
kubectl describe deploy <deployment> -n <namespace>
kubectl rollout status deploy/<deployment> -n <namespace> --timeout=120s
kubectl get rs -n <namespace> -o wide
kubectl get pods -n <namespace> -l app=<app-label> -o wide
```

知道 pod 名稱：
```bash
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --tail=200
kubectl logs <pod> -n <namespace> --previous --tail=200
```

## 輸出格式

以中文輸出，結構如下：

```markdown
# K8S 排查摘要
Context: <context>
Namespace: <namespace>
Target: <target or N/A>

## 結論
- 已確認：
- 推論：

## 關鍵證據
- <event / log / describe 重點>

## 現況
| Resource | Status | Restarts | Notes |
| --- | --- | --- | --- |

## 建議下一步
1. ...
2. ...
```

## 注意事項

- 預設只做查詢，不做修改
- 查 `secret` 時不要直接暴露敏感值；只描述名稱、key、是否存在
- 若使用者只說「幫我看一下 K8S」，先從全覽與 events 開始，不要一開始就 dump 大量 yaml
- 若事件已足夠說明問題，優先用事件與 describe 支持結論，不要過度擴張查詢
- 若需要修改資源，先說明風險與預計指令，再等使用者確認
