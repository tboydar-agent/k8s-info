# Kubernetes version skew 中文導讀

> 更新日期：2026-05-02
> 狀態：公開來源整理，非官方建議。
> 適用範圍：Kubernetes upstream 與類 kubeadm / 自管叢集的升級規劃基線；managed Kubernetes、OpenShift、RKE2、Tanzu 等仍需再查各自 provider/vendor matrix。
> 邊界：本文不保存、推論或暗示任何特定公司內部 Kubernetes 版本、cluster 拓撲、hostname、IP、ticket 或 incident 資訊。

---

## 一句話摘要

Kubernetes version skew 不是「舊 node 還能不能跑」這麼簡單；它其實定義了控制平面、node agent、proxy 與 client 在升級期間可以共存的安全範圍。

```text
升級決策順序：

  support window
      ↓
  version skew policy
      ↓
  control plane upgrade order
      ↓
  node drain / kubelet rollout
      ↓
  CNI / CSI / admission webhook / CRD / client 相容性驗證
```

---

## 主要公開來源

| 來源 | 用途 |
|---|---|
| [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) | upstream 官方 skew 規則與元件升級順序 |
| [Kubernetes Releases](https://kubernetes.io/releases/) | 目前 upstream 維護分支、latest release、EOL |
| [Upgrade A Cluster](https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/) | 官方 cluster upgrade 高階流程 |
| [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) | kubeadm 升級流程與不支援跳 minor 的提醒 |
| [kubeadm upgrade reference](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/) | kubeadm upgrade 指令參考 |

---

## 2026-05-02 觀察到的 upstream baseline

Kubernetes 官方 releases 頁面顯示，Kubernetes project 維護最近三個 minor release branches；本日觀察到的 maintained minors 為 `1.36 / 1.35 / 1.34`。

```text
Upstream maintained minors observed:

  1.36  latest observed: 1.36.0  EOL: 2027-06-28
  1.35  latest observed: 1.35.3  EOL: 2027-02-28
  1.34  latest observed: 1.34.6  EOL: 2026-10-27
```

> 注意：provider/vendor 的支援窗可能比 upstream 不同。大型地端或混合雲平台應同時維護 upstream baseline、provider baseline、addon matrix，而不是只看一張表。

---

## 核心 skew 規則（以 1.36 API server 為例）

| 元件 | 官方 skew 重點 | 1.36 範例解讀 |
|---|---|---|
| `kube-apiserver` | HA cluster 中最新與最舊 `kube-apiserver` 必須在 1 個 minor 內 | HA API server 可在 1.36 / 1.35 過渡，但不能長期混太多版本 |
| `kubelet` | 不可比 `kube-apiserver` 新；可比 API server 舊最多 3 個 minor（舊於 1.25 的規則更窄） | API server 1.36 時，kubelet 可為 1.36 / 1.35 / 1.34 / 1.33 |
| `kube-proxy` | 不可比 `kube-apiserver` 新；可比 API server 舊最多 3 個 minor；也需與同 node kubelet 在允許範圍內 | API server 1.36 時，kube-proxy 可為 1.36 / 1.35 / 1.34 / 1.33 |
| `kube-controller-manager` / `kube-scheduler` / `cloud-controller-manager` | 不可比所連線的 `kube-apiserver` 新；預期與 API server minor 相同，但升級時可舊 1 個 minor | API server 1.36 時，這些 control-plane components 可為 1.36 / 1.35 |
| `kubectl` | 可比 `kube-apiserver` 舊或新 1 個 minor | API server 1.36 時，kubectl 可為 1.37 / 1.36 / 1.35 |

---

## HA cluster 的陷阱：最舊 API server 會收窄整體可用範圍

官方 skew policy 特別提醒：如果 HA cluster 裡的 `kube-apiserver` 還在過渡，例如同時有 1.36 與 1.35，其他元件要用「可能連到最舊 API server」來判斷。

```text
HA control plane upgrade window:

  kube-apiserver A: 1.36
  kube-apiserver B: 1.35
          │
          ├─ kubelet 1.36?  不安全：可能比 1.35 API server 新
          ├─ scheduler 1.36? 若透過 LB 可能連到 1.35，需等 API server 全部升完
          └─ kubectl 1.37?  可能對 1.35 超過 +1 minor
```

實務含意：

1. control plane rolling upgrade 期間不要只看「目標版本」；要看「所有 API server instance 的 min/max」。
2. controller/scheduler/cloud-controller-manager 若透過 load balancer 連到任一 API server，必須等所有 API server 都完成目標 minor 後再升。
3. node pool 升級批次要把 kubelet / kube-proxy 與 API server 最舊版本一起檢查。

---

## 官方升級順序轉成 on-prem 檢查語言

Kubernetes 官方 version skew policy 對 1.35 → 1.36 的示例順序是：先 API server，再 controller/scheduler/cloud-controller-manager，再 kubelet，最後 kube-proxy 可選擇同步或延後。

```text
建議檢查順序：

  0. 先升到目前 minor 的最新 patch
       例：1.35.x → 最新 1.35 patch

  1. 檢查 admission webhooks / CRD / API deprecation
       新 API server 可能送出新欄位或新 resource version

  2. 升級 kube-apiserver
       HA: 確認 API server 間最多 1 minor skew

  3. 升級 kube-controller-manager / kube-scheduler / cloud-controller-manager
       前提：這些元件會連到的 API server 都已在目標 minor

  4. 分批 drain node，升 kubelet
       minor 版本 kubelet upgrade 前要 drain；不建議 in-place 直接升

  5. 升 kube-proxy 或確認延後仍在 skew 範圍內
       若 proxy 長期落後 3 minor，下次 control plane 升級前會卡住

  6. 升級 kubectl / automation client / CI runner image
       避免操作端與 API server skew 超過 ±1 minor
```

---

## 不要把 skew policy 誤讀成「可以長期不升 node」

`kubelet` / `kube-proxy` 最多可落後 API server 3 個 minor，是為了支援分批 rollout 與大型叢集過渡，不代表長期停留在舊 node baseline 沒有風險。

```text
短期可接受：
  API server 1.36
  node pool A kubelet 1.36
  node pool B kubelet 1.35
  node pool C kubelet 1.34 / 1.33 during migration

長期高風險：
  node pool 永久停在 API server -3 minor
  下一次 API server minor upgrade 前，node 必須先追上
```

對大型地端平台，這會影響：

- maintenance window 與 drain capacity
- DaemonSet / PDB / disruption budget 設計
- CNI / CSI node plugin 相容性
- OS image / kernel / container runtime baseline
- observability 與 alert rule 是否仍支援舊 metric / label

---

## On-prem 升級前 12 點檢查

```text
[ ] 1. upstream support window：目標 minor 是否在目前維護分支內？
[ ] 2. provider/vendor matrix：OS、runtime、CNI、CSI、ingress、LB 是否支援目標 minor？
[ ] 3. API deprecation：kubectl convert / server-side dry-run / audit log 是否掃過？
[ ] 4. admission webhooks：是否支援新 API 欄位、版本與 matchPolicy: Equivalent？
[ ] 5. CRD + controller：operator/controller 是否聲明支援目標 minor？
[ ] 6. etcd：backup / restore drill / defrag / alarm status 是否完成？
[ ] 7. control plane：HA API server min/max skew 是否規劃好？
[ ] 8. node capacity：drain 時是否仍保有足夠 headroom 與 PDB 可用性？
[ ] 9. kubelet config：deprecated flags / config schema 是否檢查？
[ ] 10. kube-proxy / CNI：iptables / IPVS / eBPF datapath 是否有版本限制？
[ ] 11. client tooling：kubectl、CI image、GitOps controller 是否在 ±1 minor 或官方支援範圍？
[ ] 12. rollback：是否明確知道哪些步驟只能前進，哪些可回復？
```

---

## 待查事項

- 各 provider/vendor 對 version skew 的額外限制：EKS / AKS / GKE / OpenShift / RKE2 / Tanzu。
- CNI / CSI / service mesh 對 Kubernetes minor 的支援矩陣。
- 大型 bare-metal node pool 在 drain / cordon / OS image rollout 時的 capacity 模型。
- API deprecation 掃描工具與 GitOps pipeline 的整合方式。

---

## 隱私與安全邊界

```text
本文只保存：
  - Kubernetes 官方公開文件 URL
  - 通用 version skew / upgrade planning 摘要
  - 可驗證的 upstream release baseline

本文不保存：
  - 特定公司內部 Kubernetes version
  - cluster name / hostname / IP / ticket / incident
  - 個人 email / 電話 / 私訊
  - secrets / tokens / credentials
```
