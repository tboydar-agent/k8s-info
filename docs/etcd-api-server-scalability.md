# etcd / API server scalability 與 controller/informer internals 導讀

> 狀態：公開來源整理，非官方建議。
> 適用範圍：大型或成長中的 Kubernetes cluster、地端 / bare-metal / 私有雲平台的 control plane 容量規劃、on-call 初判與升級前風險盤點。
> 邊界：本文不保存、推論或套用任何特定公司內部 cluster size、etcd latency、API QPS、hostname、IP、dashboard、incident、ticket 或憑證。所有數字都來自公開文件，落地前必須以自己的測試與供應商支援矩陣驗證。

---

## 1. 為什麼這題重要？

很多 Kubernetes 故障表面看起來是「某個 workload 卡住」，實際上可能是 control plane 被放大效應壓垮：

```text
大量物件 / 大量 controller / 大量 watch
        │
        ▼
kube-apiserver request / watch / admission / LIST 壓力
        │
        ▼
etcd write latency / read latency / compaction / snapshot / disk I/O
        │
        ▼
controller reconcile 變慢、kubectl 變慢、Pod scheduling / Service update 延遲
```

因此，etcd / API server scalability 不是單純「把 control plane VM 加大」；它是：

- 物件數量與 watch fan-out 的問題；
- API Priority and Fairness（APF）隔離與排隊的問題；
- etcd disk / network / snapshot / compaction latency 的問題；
- controller / informer 是否做出大量 LIST、無界 queue、熱 loop 的問題；
- large-cluster limit 只是上限參考，不是承諾任何 workload 都能安全接近上限。

---

## 2. 公開來源中的硬邊界

Kubernetes 官方 large-cluster 文件在 v1.36 文件中描述 Kubernetes 支援到下列規模條件：

```text
No more than 110 pods per node
No more than 5,000 nodes
No more than 150,000 total pods
No more than 300,000 total containers
```

這些數字應該被視為「官方測試 / 支援形狀的上限語言」，而不是 on-prem 平台可以直接照抄的容量承諾。實務上還要同時看：

- 每個 namespace / CRD / controller 產生多少 Kubernetes 物件；
- admission webhook、aggregated API server、controller cache 的行為；
- etcd 盤延遲與 snapshot / compaction 尖峰；
- upgrade / resync / node reboot / network partition 時的瞬間 API 壓力；
- provider / distro / addon 的支援矩陣。

### control plane 元件關係

Kubernetes 官方架構文件把 kube-apiserver 描述為可水平擴充的控制面入口，etcd 則是所有 cluster data 的 backing store。這代表 API server replica 可以增加入口容量，但不能忽略背後的 etcd latency / quorum / disk I/O。

```text
client / controller / kubelet / scheduler
        │
        ▼
┌──────────────────────────────┐
│ kube-apiserver replicas       │  可水平擴充、需 LB / HA
│ - authn/authz/admission       │
│ - LIST/WATCH/WRITE path       │
│ - APF classification/queueing │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ etcd quorum                   │  一致性、低延遲 disk/network
│ - cluster state               │
│ - resource versions           │
│ - snapshots / compaction      │
└──────────────────────────────┘
```

---

## 3. etcd：先保守，再談擴大

Kubernetes 官方 `Operating etcd clusters for Kubernetes` 文件明確提醒：etcd 是 Kubernetes 所有 cluster data 的 backing store，必須有備份計畫；為了 Kubernetes 穩定性，生產 etcd 應該放在 dedicated machines 或 isolated environments，以確保資源需求。

etcd 官方硬體建議把重點放在：

- production 環境要有合適 CPU / memory / disk / network；
- etcd 對 disk write latency 很敏感；
- 一般至少需要約 50 sequential IOPS；重載 cluster 建議約 500 sequential IOPS；
- published cloud IOPS 常是 concurrent IOPS，可能比 sequential IOPS 高很多，不能直接等同；
- large / xLarge workload 會進入更多 client、request/sec、data size 的設計區間。

etcd 官方 performance 文件也提醒，latency 與 throughput 會互相影響；snapshot、compaction、gRPC 與 disk 行為都可能讓 latency 變化。因此，在 Kubernetes on-prem 場景中，應該把 etcd SLO 拆成可觀測問題：

```text
etcd health 不只看 up/down

1. request latency 是否上升？
2. leader 是否穩定？
3. disk fsync / backend commit 是否變慢？
4. DB size 是否接近 quota，需要 defrag / compaction 規劃？
5. snapshot / backup 是否可恢復，而不是只有產生檔案？
6. API server request latency 是否跟 etcd latency 同步變差？
```

### Event 分離：只在大規模場景當作選項

Kubernetes large-cluster 文件提到，為了改善大 cluster 效能，可以把 Event objects 存在額外的 dedicated etcd instance，並讓 API server 使用它來儲存 events。這是高階設計選項，不應在小型 cluster 中過早複雜化；但在 event churn 很大、on-call 看到 events 寫入壓力明顯時，值得列為架構評估項目。

---

## 4. API Priority and Fairness：避免壞 controller 餓死其他人

Kubernetes API Priority and Fairness（APF）文件說明，APF 會更細緻地分類與隔離 request，並透過有限 queue 與 fair queuing，避免行為不佳的 controller 在同一 priority level 內餓死其他 request。文件也指出 APF 會套用到 watch requests；在沒有 APF 時，watch requests 不受舊式 `--max-requests-inflight` 限制。

把這段轉成平台維運語言：

```text
症狀：
  kubectl / controller / node status update latency 上升

不要立刻做：
  只調大 max-inflight 或只加 API server CPU

先拆：
  1. 哪一類 request？read / write / watch / LIST？
  2. 哪些 user-agent / controller / service account？
  3. 是否 admission webhook 或 aggregated API server 放慢？
  4. APF queue / seat / priority level 是否顯示壅塞？
  5. etcd latency 是否同步上升？
```

APF 是流量隔離與保護機制，不是讓無界 controller 或高 churn workload 免費通過的加速器。大型平台應保留「誰可以在事件中優先通過」的策略設計，例如 node / workload health 相關控制迴路優先於低優先度批次查詢。

---

## 5. Controller / informer：watch 不是零成本

Kubernetes controller 文件用 thermostat 類比說明 controller 是 control loop：watch cluster state，再把目前狀態推向 desired state。這個模型對平台設計的關鍵含意是：

- controller 不是只在「有變更」時免費醒來；它通常需要 informer cache、workqueue、reconcile retry；
- 大量 CRD / custom controller 會放大 API server LIST/WATCH 與 memory 壓力；
- 熱 loop controller 可能造成同一物件反覆 update，進一步增加 etcd writes；
- upgrade、resync、controller restart 會造成 cache rebuild / LIST 峰值。

```text
CRD / object count 增加
        │
        ├─ controller informer cache memory 增加
        ├─ initial LIST / relist 成本增加
        ├─ watch fan-out 增加
        └─ reconcile queue 變長
                │
                ▼
        API server / etcd / controller 三方一起慢
```

### 對 platform team 的設計提醒

在導入 operator、GitOps controller、policy engine、service mesh controller 或自製 CRD 前，應該要求最少四個公開 / 可測資料點：

1. controller watches 哪些 resources？是否 cluster-scoped？
2. 每個 workload / namespace 會增加多少 CRD object？
3. controller 是否有 rate limit、backoff、max concurrent reconcile？
4. 大量 object、API server slow、etcd slow 時，它是降級、排隊、還是熱 loop？

---

## 6. On-call 初判流程

```text
[使用者回報 kubectl / rollout / scheduler / controller 變慢]
        │
        ▼
1. 先分層：client latency / API server latency / etcd latency / controller backlog
        │
        ├─ kubectl 只有某 resource 慢？看 CRD / aggregated API / webhook
        ├─ 所有 writes 慢？看 etcd fsync/backend commit + API write latency
        ├─ watches 掉線或重建？看 API server watch / APF / network churn
        └─ controller reconcile 慢？看 queue depth、error retry、leader election
        │
        ▼
2. 看公開且低風險的 cluster-level 指標
        │
        ├─ kube-apiserver request duration / request total / inflight / APF queue
        ├─ etcd request / fsync / backend commit / leader changes / DB size
        ├─ controller-manager workqueue depth / retries / duration
        └─ scheduler pending / scheduling duration
        │
        ▼
3. 先做 non-mutating 降風險動作
        │
        ├─ 暫停非必要批次 controller / 高 churn 任務
        ├─ 限制大範圍 LIST / watch 客戶端
        ├─ 延後非必要 rollout / GitOps sync wave
        └─ 保存 metrics / Events / logs 的時間窗
        │
        ▼
4. 需要 mutation 前先走變更審批
        │
        ├─ APF config / admission / controller replica / etcd defrag 都要審核
        └─ 不在壓力中臨時套用未測參數
```

---

## 7. 升級 / 擴容前檢查表

### A. 物件與 watch 面

- [ ] cluster 是否接近官方 large-cluster 上限任一維度：nodes / pods / containers / pods-per-node？
- [ ] namespaces、Services、Secrets、ConfigMaps、CRDs、Custom Resources 的成長率是否可見？
- [ ] 哪些 controllers / agents 會 watch 全 cluster？
- [ ] 是否有大量短生命週期 Jobs / Pods / Events？
- [ ] 是否有 dashboard、scanner、backup、inventory 工具做頻繁 full LIST？

### B. API server 面

- [ ] kube-apiserver replica 與 load balancing 是否 HA？
- [ ] request latency 是否分 verb / resource / subresource / response code？
- [ ] APF priority level / flow schema 是否保護 node、system controller、on-call 讀取？
- [ ] admission webhook timeout / failurePolicy / latency 是否可觀測？
- [ ] aggregated API server 或 extension API 是否有 SLO？

### C. etcd 面

- [ ] etcd 是否在 dedicated / isolated 環境，避免與 noisy workload 共用 disk？
- [ ] backup / restore 是否演練，而不是只有產出 snapshot？
- [ ] disk sequential IOPS / fsync latency 是否符合實測需求？
- [ ] DB size、quota、compaction、defrag 是否有例行節奏？
- [ ] 是否需要評估 Event 分離 etcd，而不是讓 event churn 壓主 store？

### D. controller / informer 面

- [ ] 每個重要 controller 的 workqueue depth / retry / reconcile duration 是否可觀測？
- [ ] 自製 controller 是否有 rate limit、backoff、bounded concurrency？
- [ ] CRD schema / status update 是否避免頻繁寫入大物件？
- [ ] resync / restart / failover 時是否做過壓力測試？

---

## 8. 不可直接套用的地方

不要把本文轉成以下未驗證結論：

- 「某公司內部 Kubernetes 一定是某版本 / 某規模」；
- 「5,000 nodes 是所有 cluster 都能安全達到的承諾」；
- 「API server 慢就只要增加 replica」；
- 「etcd 慢就立刻 defrag / 改 APF / 調大 inflight」；
- 「所有 on-prem 都應該分離 Event etcd」。

本文只能作為公開來源導讀與 checklist。任何生產變更都需要：版本相容性、供應商支援、壓力測試、回復方案、變更審批與觀測指標。

---

## 9. 待查事項

- Kubernetes 官方是否有針對 v1.36 以後 large-cluster / scalability 測試更新更細的 release note。
- SIG Scalability / perf-tests 是否有近期公開測試報告可補成「測試方法」篇。
- 常見 operator / GitOps / policy controller 的 public scalability guidance 是否可整理成單獨矩陣。
- APF metrics 與 etcd metrics 的實際 PromQL 範例，需避免保存任何內部 label / hostname / IP。

---

## Sources

- Kubernetes: [Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/)
- Kubernetes: [Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- Kubernetes: [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- Kubernetes: [API Priority and Fairness](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/)
- Kubernetes: [Kubernetes Metrics Reference](https://kubernetes.io/docs/reference/instrumentation/metrics/)
- Kubernetes: [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- etcd: [Hardware recommendations](https://etcd.io/docs/v3.6/op-guide/hardware/)
- etcd: [Performance](https://etcd.io/docs/v3.6/op-guide/performance/)
