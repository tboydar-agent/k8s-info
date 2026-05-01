# Kubernetes release intelligence（近一年，2025-04 到 2026-04）

> 狀態：初版研究整理。
> 更新時間：2026-05-01。
> 目的：幫助 platform / SRE / infrastructure 團隊快速掌握近一年 Kubernetes release 對「全球級資料中心、地端運算平台、7x24 穩定性、可擴展、好維護」的影響。
> 邊界：本文件只整理公開 upstream Kubernetes 資訊；不推論任何特定公司內部版本、架構或維運狀態。

---

## 0) TL;DR — 先問專家的 10 個問題

```text
如果要問真正能支撐全球級資料中心的 Kubernetes 專家，
不要只問「你會不會 kubectl」。

要問：
  1. 你怎麼設計 1 年以上的 minor upgrade pipeline？
  2. 你怎麼管 version skew：apiserver / kubelet / kube-proxy / kubectl？
  3. 你怎麼避免 patch release 造成 CNI / CSI / runtime / kernel 連鎖事故？
  4. 你怎麼讓 upgrade 前後的 SLO、error budget、rollout gate 可觀測？
  5. 你怎麼處理 API server / etcd / watch cache / informer 壓力？
  6. 你怎麼處理大型節點池的 Node pressure、PSI、CPU pinning、NUMA？
  7. 你怎麼驗證 storage path：CSI、multipath、mount、PVC、stateful workload？
  8. 你怎麼驗證 network path：CNI、kube-proxy、BGP、LB、nftables、DNS？
  9. 遇到外部網路事件，例如海纜斷線、跨區 packet loss、DNS/route 異常，你的監控多久知道？
 10. 你能不能從 upstream changelog / KEP / PR 追到底層實作，而不是只背 release note？
```

---

## 1) 近一年 minor release 地圖

資料來源：Kubernetes official releases / GitHub releases / changelog。

```text
2025-04-23  v1.33.0  ──► 2026-06-28 EOL
2025-08-27  v1.34.0  ──► 2026-10-27 EOL
2025-12-17  v1.35.0  ──► 2027-02-28 EOL
2026-04-22  v1.36.0  ──► 2027-06-28 EOL

Kubernetes upstream currently maintains the latest 3 minor branches.
Kubernetes 1.19+ receives approximately 1 year of patch support.
```

對大型平台的意思：

```text
不是「最新版最好」，而是：

  version window
    + patch cadence
    + skew policy
    + addon compatibility
    + node OS/kernel/CNI/CSI/runtime
    + app migration / CRD / admission policy
    + rollback / disaster recovery
```

---

## 2) 依重要度排序的 release intelligence

### ★★★★★ 1. Version support window：升級節奏本身就是可靠性能力

近一年從 v1.33 → v1.36，Kubernetes 仍維持「最新三個 minor release」與約一年 patch support 的基本節奏。

對大型平台的影響：

```text
如果你的平台同時跑多個 cluster / 多地區 / 多節點池：

  release N 出現
    ├─ 先做 lab / staging conformance
    ├─ 再做 addon matrix：CNI / CSI / ingress / LB / service mesh
    ├─ 再做 canary cluster
    ├─ 再做 low-risk workload pool
    ├─ 最後才碰核心 workload / stateful workload
    └─ 全程保留 downgrade / rollback / freeze window
```

專家追問：

```text
- 你的 cluster fleet 現在哪些 minor version？
- 每個版本距離 EOL 還有幾個月？
- 你的 upgrade lead time 是 30 天、90 天，還是 180 天？
- 如果某個 CNI/CSI 不支援目標版本，誰有權停止 rollout？
```

---

### ★★★★★ 2. Version skew policy：不是只升 apiserver，而是整個控制面與節點面協議

Kubernetes version skew policy 定義 kube-apiserver、kubelet、kube-proxy、controller-manager、scheduler、kubectl 等元件之間可接受的版本差距。

對大型平台的影響：

```text
Control plane first:
  kube-apiserver
    ↓
  controller-manager / scheduler / cloud-controller-manager
    ↓
  kubelet / kube-proxy / node agents
    ↓
  kubectl / automation clients
```

真正的風險不只在 Kubernetes binary，而在：

```text
client-go / controller runtime
  + CRD conversion webhook
  + admission webhook
  + GitOps controller
  + autoscaler / scheduler plugin
  + CSI / CNI / device plugin
```

專家追問：

```text
- 你的 automation client / kubectl / controller-runtime 版本矩陣在哪？
- 你是否允許 kubelet 落後幾個 minor？為什麼？
- 你怎麼防止新版 kubectl 或 controller 使用舊 apiserver 不支援的欄位？
```

---

### ★★★★★ 3. v1.36 對 observability / alerting 的破壞性影響：metric rename 不能漏

v1.36 changelog 有多個 action required：例如 controller-manager volume operation metric rename、etcd bookmark metric rename。

對大型平台的影響：

```text
metric rename 看起來小，
但大型平台可能有：

  Prometheus rules
  Grafana dashboard
  SLO burn-rate alert
  pager routing
  runbook link
  incident automation

任一個沒更新，
就可能在真正事故時「系統已壞，但告警沒響」。
```

專家追問：

```text
- upgrade 前是否有 alert rule diff？
- deprecated / renamed metrics 有沒有自動掃描？
- dashboard 和 alert 是否有 canary validation？
- 有沒有用 recording rules 做兼容層？
```

---

### ★★★★★ 4. Network path：kube-proxy / nftables / EndpointSlice / topology-aware routing 仍是高風險區

近一年 release 裡，network 相關值得注意：

```text
v1.33:
  - EndpointSlice hints deprecated path，鼓勵改用 Service trafficDistribution
  - kube-proxy / nftables 行為與 kernel / nft 版本相容性需要注意

v1.34+ patch:
  - kube-proxy nftables mode 修正，涉及 nft 1.1.3 相容性

v1.36:
  - Service .spec.externalIPs 增加 warning / deprecation
```

對大型平台的影響：

```text
Network 是最容易被誤判的層：

  app timeout
    可能是 CNI
    可能是 kube-proxy
    可能是 DNS
    可能是 BGP route
    可能是 external LB
    可能是跨區 packet loss
    可能是海纜 / ISP / transit 事件
```

若目標是比一般電信客服更早掌握異常，應設計：

```text
synthetic probes:
  region A → region B
  region A → internet edge
  cluster → DNS resolver
  pod → service VIP
  pod → external dependency
  node → BGP peer

signal fusion:
  packet loss + RTT + DNS latency + route change + app error rate
```

專家追問：

```text
- 你能不能區分 cluster 內部 service routing 與外部 transit 問題？
- 有沒有跨區 synthetic probe？頻率多少？
- BGP session flap、route withdrawal、packet loss 有沒有統一事件時間線？
- 你的 incident commander 多久能知道「這不是 app bug」？
```

---

### ★★★★★ 5. Storage path：CSI / mount / PVC / stateful workload 是 upgrade 前必測核心

近一年 release 裡，多個 storage / CSI / mount / PVC 訊號值得注意：

```text
v1.33:
  - CSI driver IsLikelyNotMountPoint 行為理解需正確
  - MutableCSINodeAllocatableCount alpha，用於避免 scheduler 依賴過期 volume capacity
  - multipath / iSCSI / Fibre Channel 修正
  - pod resize / storage capacity scoring 等牽涉 stateful scheduling

v1.36:
  - flex-volume kubeadm integrated support removed path
  - git-repo volume plugin disabled by default
  - PVC unused status alpha
  - CSI volume scheduling opt-in behavior
```

對大型平台的影響：

```text
storage incident 通常恢復慢、影響大、rollback 難：

  scheduler decision
    → attach
    → mount
    → filesystem permission
    → pod start
    → app recovery
    → data correctness
```

專家追問：

```text
- upgrade 前是否有 stateful workload replay test？
- CSI driver 是否逐版本驗證？
- multipath / FC / iSCSI / NVMe / local PV 各自怎麼測？
- PVC stuck / mount residual / attach detach storm 的 runbook 在哪？
```

---

### ★★★★☆ 6. v1.33 的 node observability：PSI metrics 是懂 Linux 的 SRE 要看的訊號

v1.33 新增 Pressure Stall Information metrics 到 node metrics。

對大型平台的影響：

```text
CPU / memory / IO 使用率不是全部。
PSI 關心的是「工作被資源壓力卡住的時間」。

高階 SRE 會問：
  CPU usage 高，但 thread 有沒有 stalled？
  memory reclaim 是否造成 latency spike？
  IO pressure 是否正在拖慢 kubelet / container runtime / app？
```

專家追問：

```text
- 你有沒有把 PSI 納入 node SLO？
- PSI 與 app p99 latency 有沒有關聯圖？
- 壓力發生時，你先看 kubelet、container runtime、kernel，還是只看 pod restart？
```

---

### ★★★★☆ 7. API server / etcd / watch cache：大型 cluster 的瓶頸常在控制面資料流

v1.33 出現多個與 API Machinery / Etcd / watch / informer / cache 相關訊號，例如 ListFromCacheSnapshot、apiserver storage digest、informer ordering 等。

對大型平台的影響：

```text
大型 cluster 的控制面壓力來源：

  LIST storm
  WATCH reconnect storm
  informer fanout
  controller retry loop
  CRD object size / count
  admission webhook latency
  etcd compaction / defrag / disk latency
```

專家追問：

```text
- 你知道最大 LIST request 來自哪個 controller 嗎？
- etcd p99 fsync latency 與 apiserver p99 request latency 有沒有關聯？
- admission webhook 失敗時 fail-open 還是 fail-close？
- 如果 watch cache 不一致，你怎麼證明？
```

---

### ★★★★☆ 8. DRA / GPU / device plugin：AI workload 讓 scheduler 與 device allocation 變成平台核心能力

v1.33 到 v1.36 持續有 Dynamic Resource Allocation（DRA）相關 API / RBAC / scheduler / kubelet plugin 變化。

對大型平台的影響：

```text
AI / GPU / FPGA / NIC offload / special devices 不是單純 node label。
真正問題是：

  device discovery
  allocation lifecycle
  topology awareness
  isolation / RBAC
  scheduler integration
  failure / taint / eviction
```

專家追問：

```text
- 你把 GPU 當 node pool，還是當可分配 device resource？
- DRA driver 的 RBAC 如何最小化？
- device health 變差時，pod 是等、遷移、降級，還是直接 fail？
```

---

### ★★★★☆ 9. Scheduler evolution：PodGroup / topology-aware workload scheduling 指向批次與大規模工作負載

v1.36 changelog 有 PodGroup、TopologyAwareWorkloadScheduling、WorkloadAwarePreemption 等訊號。

對大型平台的影響：

```text
傳統 service workload：重點是可用性與水平擴展。
大型 batch / AI / HPC workload：重點是 gang scheduling、topology、device locality、preemption。

如果平台同時支援 app、batch、AI，
就不能只靠預設 scheduler 心智模型。
```

專家追問：

```text
- 你的 scheduler profile 怎麼切？
- 有沒有外部 scheduler / queueing layer？
- topology-aware scheduling 對 network / storage / GPU locality 的影響怎麼量化？
```

---

### ★★★☆☆ 10. kubelet / node lifecycle：restart、sidecar、CrashLoopBackOff、in-place resize 仍是 Day2 重點

近一年 release 中持續修正 kubelet restart、sidecar container、CrashLoopBackOff、in-place pod vertical scaling、container restart rules 等問題。

對大型平台的影響：

```text
Day2 SRE 最常處理的是「不是整個 cluster 壞，而是一批 node / 一類 pod 行為怪」。

要看：
  kubelet restart 後 pod 狀態
  sidecar initContainer Always restartPolicy
  startupProbe / liveness / readiness 互動
  resize / eviction / QoS / cgroup 行為
```

專家追問：

```text
- kubelet restart 的 chaos test 有沒有？
- sidecar workload 在 upgrade 後如何驗證？
- CrashLoopBackOff 參數是否適合你的 incident automation？
```

---

## 3) Release 對照摘要

```text
v1.33.0  2025-04-23
  high-signal:
    - PSI node metrics
    - API server / watch cache / informer 改善
    - DRA / device allocation 多項演進
    - EndpointSlice / topology-aware routing 方向
    - CSI / storage / multipath 修正

v1.34.0  2025-08-27
  high-signal:
    - DRA kubelet gRPC API v1 / v1beta1 deprecated
    - resource / scheduler / storage API 持續演進
    - kubeconfig kuberc 方向
    - 大型平台需重點看 CNI/CSI/runtime/addon matrix

v1.35.0  2025-12-17
  high-signal:
    - 需要搭配 patch release 看 regression 修正
    - StatefulSet / kubelet / kube-proxy / audit latency 等 patch 訊號
    - 大型平台不應只看 .0，應看 .1/.2/.3 patch train

v1.36.0  2026-04-22
  high-signal:
    - metric rename：volume operation / etcd bookmark
    - DRA granular RBAC
    - flex-volume kubeadm integrated support removed
    - Service externalIPs warning/deprecation
    - scheduler PodGroup / topology-aware workload scheduling
    - MemoryQoS / cgroup v2 memory.min
```

---

## 4) 對「全球級資料中心 SRE」的能力要求

```text
Level 1: 會用 Kubernetes
  - kubectl / deployment / service / ingress

Level 2: 會維運 Kubernetes
  - upgrade / backup / restore / monitoring / alert

Level 3: 會 debug Kubernetes
  - apiserver / etcd / kubelet / CNI / CSI / scheduler / runtime / Linux

Level 4: 會設計大型平台
  - multi-cluster / multi-region / capacity / fault domain / SLO / automation

Level 5: 會從 upstream source / KEP / PR 推導風險
  - 看到 changelog 就知道要測哪條 data path
  - 看到 patch release 就知道哪些 cluster 不該 rollout
  - 看到外部網路事件能快速切分 app / cluster / ISP / transit / submarine cable
```

---

## 5) 建議建立的內部 release review 模板

```text
Kubernetes release review

1. Version window
   current:
   target:
   EOL:
   skew allowed:

2. Addon matrix
   CNI:
   CSI:
   ingress/LB:
   service mesh:
   autoscaler:
   GitOps/controller-runtime:

3. Data path risk
   network:
   storage:
   node/kernel/runtime:
   control plane/etcd:

4. Observability diff
   metrics renamed:
   metrics deprecated:
   dashboards affected:
   alerts affected:
   SLO gates:

5. Rollout plan
   lab:
   staging:
   canary:
   production waves:
   rollback:
   freeze window:

6. External dependency awareness
   ISP / transit / DNS / route:
   synthetic probes:
   regional packet loss:
   submarine cable / cross-border latency:
```

---

## 6) Sources

- Kubernetes releases: https://kubernetes.io/releases/
- Kubernetes patch releases: https://kubernetes.io/releases/patch-releases/
- Kubernetes version skew policy: https://kubernetes.io/releases/version-skew-policy/
- Kubernetes GitHub releases: https://github.com/kubernetes/kubernetes/releases
- Kubernetes v1.33 changelog: https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.33.md
- Kubernetes v1.34 changelog: https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.34.md
- Kubernetes v1.35 changelog: https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.35.md
- Kubernetes v1.36 changelog: https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.36.md
