# Kubernetes observability / incident response 導讀

> 建立日期：2026-05-12
> 範圍：公開 Kubernetes / SRE 文件整理；適用於大型 on-prem / hybrid Kubernetes 平台的通用 Day-2 維運設計。
> 邊界：這不是任何特定公司內部 incident playbook；不包含內部拓撲、版本、IP、hostnames、ticket、聊天紀錄或私有監控規則。

---

## TL;DR

Kubernetes incident response 不應只從「看一個 dashboard」開始，而要把 **signal → scope → owner → mitigation → learning** 串成固定路徑：

```text
User symptom / alert
        │
        ▼
┌────────────────────┐
│ 1. Declare scope    │  namespace / service / node pool / cluster / region
└─────────┬──────────┘
          ▼
┌────────────────────┐
│ 2. Collect signals  │  events + logs + metrics + object status + probes
└─────────┬──────────┘
          ▼
┌────────────────────┐
│ 3. Classify fault   │  workload / node / storage / network / control plane
└─────────┬──────────┘
          ▼
┌────────────────────┐
│ 4. Mitigate safely  │  rollback / drain / scale / traffic shift / quota guard
└─────────┬──────────┘
          ▼
┌────────────────────┐
│ 5. Learn blameless  │  timeline / contributing factors / action items
└────────────────────┘
```

最小可重用結論：

- **Events**：用來看近期 object 狀態轉換與 controller/kubelet 訊號；適合回答「剛才發生什麼」。
- **Logs**：Kubernetes 官方文件把 logging 視為 cluster-level concern；node/pod/container log 要被集中收集，避免只靠某台 node 的本機檔案。
- **Metrics**：resource metrics pipeline 提供 CPU / memory 等近即時使用量，適合 autoscaling 與 `kubectl top` 類觀察；長期 SLO/容量分析仍要靠完整監控系統。
- **Probes**：liveness / readiness / startup probes 是自動復原與流量切換邊界；錯誤 probe 會把 incident 放大。
- **Incident discipline**：SRE incident management / blameless postmortem 強調先恢復服務，再保留學習；不要把 postmortem 寫成責備文件。

---

## 為什麼這題值得放進 k8s-info？

大型地端平台常見問題不是「沒有工具」，而是 incident 當下訊號太多、責任邊界太模糊：

```text
alert fires
  ├─ app team: deployment / config / dependency?
  ├─ platform team: node / kubelet / CNI / CSI / apiserver?
  ├─ network team: BGP / firewall / VIP / DNS?
  └─ storage team: attach / mount / latency / capacity?
```

因此文件要累積的是 **低風險的調查順序**，不是環境專屬命令或內部 dashboard 名稱。

---

## 事件處理信號分層

### 1. Object status：先確定 Kubernetes 看到什麼狀態

適用問題：Pod Pending、CrashLoopBackOff、Service endpoint 不見、Node NotReady、PVC Pending。

```text
Kubernetes API objects
  ├─ Pod conditions / container statuses
  ├─ Node conditions
  ├─ Service + EndpointSlice
  ├─ PVC / PV / StorageClass
  └─ Deployment / ReplicaSet / StatefulSet rollout state
```

使用原則：

- 先看 object status，避免直接跳到 node shell 或猜 network/storage。
- 對 workload incident，先分清楚是「pod 沒被排程」、「container 啟動失敗」、「readiness 沒過」、「Service 沒 endpoint」。
- 對 node incident，先分清楚是 NotReady、pressure condition、taint、或 kubelet/container runtime 問題。

### 2. Events：回答「最近誰改變了狀態」

Kubernetes Event API 是 cluster resource，用來記錄與 object 相關的事件。事件很適合 incident 初判，但不應當成長期稽核或唯一事實來源。

```text
Event signal
  good for:   recent scheduling / image pull / attach / probe / eviction clues
  weak for:   long-term audit, exact root cause, external system evidence
```

實務邊界：

- Events 可能被壓縮、過期或只保留短期；postmortem 需要補上 logs、metrics、change record。
- 若同一事件大量重複，先把它當「症狀密度」而非 root cause。

### 3. Logs：從單點檔案轉成 cluster-level logging

Kubernetes 官方 logging architecture 強調 cluster-level logging：container logs、node logs 與系統 component logs 需要被收集到獨立後端，因為 pod/node 生命週期都可能短暫。

```text
container stdout/stderr
        │
        ▼
node log files / runtime log path
        │
        ▼
log agent / collector
        │
        ▼
central log backend
        │
        ▼
incident timeline / query / retention
```

低風險設計要點：

- 不把機密、token、個資寫進 application log；platform 文件也不應保存 raw log excerpt。
- incident note 只保存必要的 pattern / sanitized error class，不保存私有 URL、IP、帳號或訊息全文。
- 對 crash / restart 類事件，logs 要和 container restart count、last state、deployment revision 一起看。

### 4. Metrics：分清 resource metrics 與 SLO metrics

Kubernetes resource metrics pipeline 提供 CPU / memory 等近即時資源用量，常被 HPA / VPA / `kubectl top` 使用。但 incident response 還需要延伸到 request/error/latency、apiserver latency、etcd latency、CNI/CSI 指標等平台指標。

```text
Resource metrics pipeline
  kubelet / cAdvisor -> metrics-server -> API aggregation -> kubectl top / HPA

SRE observability
  service SLI -> alert -> dashboard -> trace/log correlation -> mitigation
```

使用邊界：

- `kubectl top` 類 resource view 適合快速初判，不足以證明長期容量趨勢。
- 對 capacity planning，要保留時間序列與 peak / p95 / p99，不只看當下 snapshot。
- 對 control plane incident，要把 apiserver、etcd、controller、scheduler 分開看，避免把所有 API slow 都歸因於單一元件。

### 5. Probes：自動復原也可能自動放大事故

Kubernetes liveness / readiness / startup probes 是 workload 自動復原與流量切換的核心工具：

```text
startup probe   protects slow start
readiness probe controls traffic eligibility
liveness probe  triggers restart when app is unhealthy
```

常見 incident 放大模式：

- readiness 太嚴格：短暫 dependency 抖動導致所有 pod 從 Service endpoint 移除。
- liveness 太激進：冷啟動或 GC pause 被誤判為 dead，造成 restart loop。
- startup probe 缺失：慢啟動服務被 liveness 在初始化期間殺掉。

建議：probe 設計應該納入 release review / incident review，而不是只由 app repo 隨意設定。

---

## On-call 初判流程

```text
0. 安全邊界
   - 不在公開文件貼內部 dashboard、IP、hostname、聊天紀錄、token。
   - 不直接執行破壞性操作；先確認 blast radius 與 rollback path。

1. 宣告範圍
   - 單一 workload？namespace？node pool？cluster？region？
   - 是否只有新版本 / 新節點 / 特定 storage class / 特定 ingress path？

2. 建立時間線
   - alert first seen
   - deploy/config/change window
   - first failing pod/node/event
   - mitigation action
   - recovery timestamp

3. 收集五類 Kubernetes 訊號
   - object status
   - Events
   - logs
   - metrics
   - probes / rollout state

4. 分類 fault domain
   - workload: image, config, dependency, probe, resource request/limit
   - node: pressure, NotReady, runtime, disk, kernel, kubelet
   - network: Service/EndpointSlice, CNI, DNS, BGP/LB, policy
   - storage: PVC/PV, attach/mount, topology, CSI sidecar, backend latency
   - control plane: apiserver, etcd, scheduler, controller-manager

5. 選擇低風險緩解
   - rollback recent change
   - scale out only if capacity/quotas allow
   - cordon/drain suspect node with PDB awareness
   - shift traffic only if downstream capacity is verified
   - pause automation if it is amplifying the incident

6. Blameless learning
   - timeline + customer/platform impact
   - contributing factors
   - detection gap
   - mitigation gap
   - follow-up owners and due dates
```

---

## Incident note / postmortem 最小模板

```text
# Incident: <public-safe short title>

Status:
  detected / mitigating / resolved / learning

Scope:
  affected service class or platform component only; no private topology

Impact:
  user-visible symptom or platform symptom; avoid private customer data

Timeline:
  T0 alert
  T1 first hypothesis
  T2 mitigation
  T3 recovery

Signals used:
  - Kubernetes Events
  - object status
  - centralized logs, sanitized
  - metrics / SLI
  - recent change record

Contributing factors:
  - technical factor 1
  - process / detection factor 2

What went well:
  - ...

What was hard:
  - ...

Action items:
  - owner role, not personal name in public docs
  - due window
  - verification method

Privacy review:
  - no token / credentials
  - no private URL
  - no personal contact
  - no internal topology/IP/hostname
```

---

## 適用範圍

適合：

- 建立 Kubernetes on-call readiness checklist。
- 把 workload / node / storage / network / control-plane incident 分流。
- 設計公開、安全、可複用的 incident note / postmortem 模板。
- 教育平台工程師如何把 Events / logs / metrics / probes 串成時間線。

不適合直接套用：

- 特定公司內部 escalation policy。
- 特定監控產品 dashboard / alert rule 名稱。
- 特定網段、hostnames、cluster 名稱、tenant 名稱。
- 安全事件的完整 forensic 流程；security incident 需要額外法遵與資安流程。

---

## 待查事項

- 補一份 `7x24 on-call readiness checklist`，把 paging、handoff、runbook freshness、rollback drill 拆成檢查項。
- 補 `capacity planning / performance tuning` 筆記，連接 resource metrics、requests/limits、HPA/VPA、SLO burn rate。
- 補 `AI-assisted incident summary` 安全流程，明確規定哪些 sanitized signals 可以給 AI、哪些資料不可離開內部系統。

---

## 來源

- Kubernetes Docs — [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- Kubernetes Docs — [Resource metrics pipeline](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- Kubernetes Docs — [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- Kubernetes API Reference — [Event](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/)
- Kubernetes Docs — [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- Kubernetes Docs — [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- Google SRE Book — [Managing Incidents](https://sre.google/sre-book/managing-incidents/)
- Google SRE Book — [Postmortem Culture: Learning from Failure](https://sre.google/sre-book/postmortem-culture/)

---

## 隱私 / 安全檢查

```text
Recorded:
  - public Kubernetes documentation concepts
  - public SRE incident/postmortem principles
  - generic incident-response workflow

Not recorded:
  - specific-company internal Kubernetes versions
  - internal architecture, IP ranges, hostnames, dashboards, tickets
  - private links, chat messages, screenshots, credentials, tokens
  - personal contact details
```
