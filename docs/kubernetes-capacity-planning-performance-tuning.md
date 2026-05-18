# Kubernetes capacity planning / performance tuning 導讀

> 範圍：大型或地端 Kubernetes 平台的公開、通用 capacity planning 與 performance tuning 研究筆記。
> 邊界：不可直接套用到任何特定公司內部環境；實際 CPU / memory / storage / network 門檻、node shape、CNI/CSI、autoscaler、SLO 與成本模型都必須依現場量測與變更流程驗證。
> 風格：source-grounded、低風險、先觀察再調整；避免把單一 provider 或單一 workload 的數字當成通用真理。

---

## 1. 一句話摘要

```text
capacity planning 不是「把 node 加大」；
performance tuning 也不是「把 limit 調高」。

比較安全的順序是：
  SLO / workload profile
    -> requests / limits / quota / node allocatable
    -> autoscaling / scheduling / eviction boundary
    -> control-plane and API scalability guardrails
    -> 小步變更 + rollback + 觀測
```

對大型地端平台來說，最常見的風險不是單點 CPU 不夠，而是多層假設互相衝突：Pod request 偏低、LimitRange 預設不透明、system-reserved 不足、node eviction 門檻不符合實際磁碟/記憶體壓力、HPA 只看平均 utilization、API server / etcd 物件量與 watch 流量成長但沒有被納入容量模型。

---

## 2. 主要公開來源

| 類別 | 來源 | 本文件使用方式 |
|---|---|---|
| Pod resource model | [Kubernetes Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) | requests / limits 是 scheduler 與 cgroup enforcement 的核心輸入 |
| Node allocatable | [Reserve Compute Resources for System Daemons](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/) | 把 kube / system daemons 與 eviction reservation 從 workload capacity 中扣除 |
| Namespace guardrails | [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) / [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/) | 用 quota / default / min / max 限制 namespace 資源風險 |
| Autoscaling | [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | HPA 根據觀測 metrics 與 target 計算 replica 數；不是完整 capacity planner |
| Node pressure | [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/) | 把 memory / disk / inode / PID 壓力視為 capacity model 的安全邊界 |
| Large cluster | [Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/) | 官方 scalability threshold 與大型叢集設計提醒 |
| Component metrics | [Metrics For Kubernetes System Components](https://kubernetes.io/docs/concepts/cluster-administration/system-metrics/) | 控制面與元件 metrics 是 tuning 後驗證依據 |
| SRE overload | [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/) / [Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) | 把 overload / cascading failure 視為 capacity planning 的可靠度問題 |

---

## 3. Capacity model：先分四個層次

```text
┌───────────────────────────────────────────────────────────────┐
│  Business / user SLO                                           │
│  latency, availability, recovery, batch deadline, cost boundary │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Workload profile                                              │
│  requests, limits, peak/idle ratio, burst, startup, I/O shape   │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Cluster resource envelope                                     │
│  node allocatable, quota, scheduler fit, autoscaling, eviction  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Control-plane / platform envelope                             │
│  API QPS, object count, watch/list churn, etcd, controllers     │
└───────────────────────────────────────────────────────────────┘
```

### 3.1 Workload requests / limits

Kubernetes 官方文件把 `requests` 描述為 scheduler 用來尋找可容納 node 的輸入；`limits` 則與 runtime / cgroup enforcement 有關。這表示：

- request 太低：scheduler 以為 cluster 還有容量，但實際可能進入 CPU contention、memory pressure 或 eviction。
- request 太高：排程與成本保守，可能造成低利用率與 pending pods。
- limit 太低：CPU throttling 或 OOMKilled 可能看起來像應用程式效能問題。
- 沒有 limit / request：namespace 可能需要 LimitRange 或 admission policy 做最低 guardrail。

### 3.2 Node allocatable 不是 node 總容量

在地端平台做容量規劃時，不能直接把 `node capacity` 全部賣給 workload。Kubernetes 官方 node allocatable 模型要求把 kube daemons、system daemons 與 eviction reservation 納入考量。

```text
Node capacity
  - kube-reserved
  - system-reserved
  - eviction-reserved / hard eviction threshold
  = allocatable for pods
```

實務含義：若只看總 CPU / memory，而沒有明確保留 kubelet、container runtime、CNI、CSI、log agent、monitoring agent、node exporter 等系統資源，就可能在尖峰時讓「平台本身」和 workload 搶資源，最後表現成 NodePressure、Pod eviction 或 kubelet 不穩。

### 3.3 Namespace quota / LimitRange 是風險隔離，不是效能保證

ResourceQuota 與 LimitRange 可以限制 namespace 的總 request/limit、物件數量、預設值與 min/max。它們適合做多租戶 guardrail：

- 避免單一 namespace 無上限建立 workload 或 consume requests。
- 讓沒有填 request/limit 的 Pod 取得預設值。
- 限制 request/limit ratio，避免極端 overcommit。

但 quota 不會自動找出正確 sizing；它只是在錯誤 sizing 擴散前設界線。

---

## 4. Performance tuning：安全的調整順序

```text
Observe
  ├─ workload latency / error / saturation
  ├─ Pod request/limit vs actual usage
  ├─ node allocatable / pressure / eviction
  ├─ HPA events and replica decisions
  └─ API server / scheduler / controller / etcd metrics

Hypothesize
  ├─ CPU throttling? memory OOM? disk pressure?
  ├─ insufficient replicas? slow startup? bad readiness?
  ├─ scheduler fragmentation? quota/LimitRange constraint?
  └─ control-plane/API bottleneck?

Change one layer
  ├─ requests/limits
  ├─ replica/HPA target
  ├─ node pool shape / allocatable reservation
  ├─ quota / LimitRange default
  └─ controller/API client behavior

Verify and rollback
```

### 4.1 不要只用平均 CPU utilization 判斷容量

HPA 官方文件說明 HPA controller 會週期性觀測 metrics，並依 target 計算 desired replicas。這很有用，但它不是完整 capacity planner：

- 平均 CPU 可能掩蓋 tail latency、hot shard、單 Pod memory leak 或 I/O bottleneck。
- scale out 需要啟動時間、image pull、readiness、cache warm-up。
- 如果 request 設錯，CPU utilization percentage 也會被扭曲。
- HPA 不會自動修正 namespace quota、node allocatable 或 API server bottleneck。

### 4.2 Node pressure 是容量模型的「紅線」

Node-pressure eviction 文件把 kubelet 的 eviction signal 分成 memory、nodefs/imagefs/containerfs、inode、PID 等方向。對 capacity planning 的含義是：

```text
如果常態 capacity 模型會逼近 eviction threshold，
那不是「偶發壓力」，而是容量模型本身沒有安全餘裕。
```

因此 capacity review 應該把以下訊號列入週期檢查：

- `MemoryPressure` / `DiskPressure` / `PIDPressure` node condition。
- eviction events 與 `Evicted` pods。
- image GC / container log 成長 / ephemeral-storage requests。
- kubelet、runtime、CNI/CSI、logging agent 的 system resource 使用量。

---

## 5. 大型叢集與控制面 guardrails

Kubernetes 官方大型叢集文件提供 scalability considerations。即使某個叢集低於官方最大 threshold，也不代表控制面沒有瓶頸；平台團隊仍應觀察：

- API server request rate、latency、inflight request、watch/list pattern。
- etcd object count、database size、request latency、leader changes。
- controller reconciliation backlog。
- scheduler queue / scheduling latency。
- 大量 controller / operator / CI / GitOps tool 是否造成 bursty API traffic。

```text
Workload capacity OK
  不等於
Control-plane capacity OK
```

對地端平台而言，capacity planning 應該把「新增 node / pod / namespace / CRD / operator / GitOps sync frequency」視為會增加控制面負載的變更，而不只是 compute 資源變更。

---

## 6. On-prem capacity planning checklist

```text
[ ] 1. SLO / SLI
      - latency、error rate、availability、batch deadline 是否明確？

[ ] 2. Workload inventory
      - stateless / stateful / daemonset / batch / control-plane addon 分類？
      - peak / idle / startup / failover profile 是否不同？

[ ] 3. Requests / limits quality
      - 每個 workload 是否有 CPU / memory requests？
      - 是否有 throttling / OOMKilled / Pending / Evicted pattern？
      - LimitRange 預設值是否會誤導 HPA utilization？

[ ] 4. Node allocatable
      - kube-reserved / system-reserved / eviction threshold 是否明確？
      - CNI / CSI / log / monitoring agents 是否納入 system overhead？

[ ] 5. Namespace / tenant guardrails
      - ResourceQuota 是否限制 request/limit/object count？
      - 是否有例外流程與容量審核？

[ ] 6. Autoscaling behavior
      - HPA target metric 是否代表 SLO 風險？
      - scale-up / scale-down stabilization 是否符合流量型態？
      - node autoscaling / bare-metal provisioning 是否有實際供給延遲？

[ ] 7. Storage / network capacity
      - PVC / IOPS / throughput / inode / ephemeral-storage 是否被納入？
      - Service / ingress / CNI datapath 是否有瓶頸觀測？

[ ] 8. Control-plane capacity
      - API server / etcd / scheduler / controller metrics 是否有 dashboard？
      - GitOps / operators / CI 是否造成 API burst？

[ ] 9. Failure-mode reserve
      - N+1 / zone failure / rack failure / maintenance drain 後是否仍有容量？
      - 升級期間是否有 surge / unavailable budget？

[ ] 10. Change safety
      - 每次只調一層，保留 rollback，記錄 before/after metrics。
```

---

## 7. 低風險調整範例

| 現象 | 先查 | 避免直接做 | 較安全下一步 |
|---|---|---|---|
| Pod latency 高但 CPU 平均不高 | tail latency、throttling、GC、I/O、hot shard | 只看平均 CPU 就大幅加 node | 對照 request/limit、Pod-level metrics、服務 SLO |
| 經常 Pending | scheduler events、quota、node allocatable、taints、topology constraints | 直接調高 quota 或移除 constraints | 找出是資源不足、約束太嚴，還是 fragmentation |
| HPA 反覆震盪 | HPA events、metrics delay、request 是否正確、startup/readiness | 只提高 maxReplicas | 調整 target/stabilization，修正 request，確認啟動時間 |
| Node DiskPressure | nodefs/imagefs/containerfs、logs、image GC、ephemeral-storage request | 只刪 Pod 或清 log | 建立 log rotation、ephemeral-storage request/limit、image lifecycle |
| API server latency 升高 | system component metrics、audit/API request pattern、controller burst | 只加 worker node | 找出 API client/watch/list pattern，檢查 controller/GitOps 行為 |

---

## 8. 待查事項

- Kubernetes 官方大型叢集 threshold 與實際地端平台 node shape / workload density 的差距。
- Vertical Pod Autoscaler、Cluster Autoscaler、Karpenter 或 bare-metal provisioning 工具在本 repo 是否需要獨立整理。
- etcd performance / compaction / defrag / backup restore 是否需要獨立 runbook。
- 如何把 capacity review 轉成 monthly platform review template 與 pre-upgrade capacity gate。

---

## 9. 不可直接套用的邊界

- 本文件不推論任何特定公司內部 cluster size、Kubernetes 版本、node 規格、CIDR、SLO、成本或 incident。
- 本文件不提供 production 調參值；所有 requests、limits、reserved resources、eviction thresholds、HPA target 都必須依公開文件、測試環境、現場 metrics 與 change approval 決定。
- 本文件不保存 dashboard URL、私人 issue、ticket、kubeconfig、Secret、token 或個人聯絡資料。

---

## 10. 相關文件

- [Linux node pressure debugging playbook](node-pressure-debugging.md)
- [Kubernetes observability / incident response 導讀](kubernetes-observability-incident-response.md)
- [On-prem Kubernetes 升級檢查表](on-prem-upgrade-checklist.md)
- [CSI / Kubernetes storage failure modes 導讀](csi-storage-failure-modes.md)
