# Kubernetes 版本偏差政策（Version Skew Policy）中文導讀

> 更新日期：2026-05-07
> 狀態：公開來源整理，非官方建議。
> 適用範域：Kubernetes upstream 版本偏差政策與元件升級順序基準；managed Kubernetes、OpenShift、RKE2、Tanzu 等仍需再查各自 provider/vendor matrix。
> 邊界：本文不保存、推論或暗示任何特定公司內部 Kubernetes 版本、cluster 拓扑、hostname、IP、ticket 或 incident 資訊。

## 一句話摘要

Kubernetes 版本偏差政策（Version Skew Policy）定義了升級期間元件間的安全版本差距，確保 cluster 穩定性和可用性；違反 skew 限制可能導致 control plane 無法升級、node 無法加入或 pod 無法調用。

## 版本偏差政策核心規則

Kubernetes 版本偏差政策適用於所有 core Kubernetes 元件，包括 `kube-apiserver`、`kubelet`、`kube-proxy`、`kube-controller-manager`、`kube-scheduler`、`cloud-controller-manager` 和 `kubectl`。

### 支援版本 skew 總覽

| 元件 | 最大 skew 限制 | 關鍵限制 |
|---|---|---|
| `kube-apiserver` | 1 個 minor 版本 | HA cluster 中所有 API server 必須在 1 個 minor 內 |
| `kubelet` | 3 個 minor 版本（舊於 1.25 為 2 個 minor） | 不可比 API server 新；可比 API server 舊最多 3 個 minor |
| `kube-proxy` | 3 個 minor 版本（舊於 1.25 為 2 個 minor） | 不可比 API server 新；可比 API server 舊最多 3 個 minor；需與同 node kubelet 在允許範圍內 |
| `kube-controller-manager` / `kube-scheduler` / `cloud-controller-manager` | 1 個 minor 版本 | 不可比所連線的 API server 新；預期與 API server minor 同一，但可舊 1 個 minor |
| `kubectl` | 1 個 minor 版本（舊或新） | 不可與所有 API server 的 skew 超過 ±1 minor |

### 實務範例（以 API server 1.36 為例）

| 元件 | 支援版本範圍 | 備註 |
|---|---|---|
| `kube-apiserver` | 1.36 | HA 中可包含 1.35 過渡 |
| `kubelet` | 1.36, 1.35, 1.34, 1.33 | 舊於 1.25 的規則更窄 |
| `kube-proxy` | 1.36, 1.35, 1.34, 1.33 | 需與同 node kubelet 相稱 |
| `control plane` | 1.36, 1.35 | 通常建議與 API server 相同 |
| `kubectl` | 1.37, 1.36, 1.35 | 可比 API server 新或舊 1 個 minor |

## 元件升級順序（關鍵！）

Kubernetes 官方明確規定了升級順序，違反順序可能導致 cluster 不穩定或無法運作。

### 從 1.35 升級到 1.36 的標準順序

```text
建議檢查順序：

  0. 先升到目前 minor 的最新 patch
       例：1.35.x → 最新 1.35 patch

  1. 檢查 admission webhooks / CRD / API deprecation
       新 API server 可能送出新欄位或新 resource version

  2. 升級 kube-apiserver
       HA: 確認 API server 間最多 1 minor skew

  3. 升級 kube-controller-manager / kube-scheduler / cloud-controller-manager
       前提：這些元件會連到的 API server 都已於目標 minor

  4. 分批 drain node，升 kubelet
       minor 版本 kubelet upgrade 前要 drain；不建議 in-place 直接升

  5. 升 kube-proxy 或確認延後仍在 skew 範圍內
       若 proxy 長期落後 3 minor，下次 control plane 升級前會卡oke

  6. 升級 kubectl / automation client / CI runner image
       避免操作端與 API server skew 超過 ±1 minor
```

### 版本跳躍與回卷策略

- **版本跳躍**：Kubernetes 官方不支援跳過 minor 版本，即使在單一實體 cluster 中。必須逐一升級（例如 1.33 → 1.34 → 1.35）。
- **回卷**：某些元件的升級是不可逆（例如 etcd 資料結構變化），必須在升級前驗證回卷路徑。
- **Patch 先升到最新**：在進行 minor 升級之前，確定目前版本已升到該 minor 的最新 patch。

## HA cluster 的陷阱與風險

在 HA cluster 中，如果 API server 間存在 skew，會收窄其他元件的可用範圍：

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
>- control plane rolling upgrade 期間要同時檢查所有 API server 的 min/max 版本。
>- controller/scheduler/cloud-controller-manager 若透過 load balancer 連線，必須等所有 API server 都完成目標 minor 後再升級。
>- node pool 升級批次要將 kubelet / kube-proxy 與 API server 最舊版本一起檢查。

## 實際案例研究

### 案例 1：不當升級導致 node 無法加入

**情境**：cluster 運行在 1.33，直接升級 API server 到 1.35，跳過 1.34。

**問題**：
- kubelet 仍在 1.33，與新 API server 的 skew 超過 3 個 minor（1.35 - 1.33 = 2，但實際地是跳過了 1.34，導致 skew 計算錯誤）。
- kube-proxy 仍在 1.33，同樣超出允許範圍。
- 新 API server 可能用戶 kubelet 不理解的新 API 欄位。

**後果**：新 API server 啟動後，既有 node 上的 kubelet 無法與之通識，node 狀態變為 `NotReady`，pod 無法調用。

**解決策**：必須先將 kubelet 和 kube-proxy 升級到 1.34，再升級 API server 到 1.35，遵循正確順序。

### 案例 2：HA cluster 中部分升級

**情境**：HA cluster 有兩個 API server，A 升級到 1.36，B 仍在 1.35。

**問題**：
- control plane 元件（scheduler, controller-manager）如果透過 LB 連線，可能連到 1.35 或 1.36。
- 如果 scheduler 升級到 1.36，但連到 1.35 API server，可能因為版本不相稱而失敗。
- kubelet 如果升級到 1.36，但連到 1.35 API server，可能超出 ±3 minor 限制（1.36 - 1.35 = 1，但實際地是 1.36 比 1.35 新，而 kubelet 不可比 API server 新）。

**解決策**：HA cluster 升級時，應同時升級所有 API server 到目標版本，或至少確保所有 API server 在同一 minor 版本範圍內，再升級其他元件。

## 版本偏差檢查清單

### 升級前檢查

```text
升級前 12 點檢查：

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

### 日常監控

```text
日常版本追蹤：

  1. Kubernetes upstream releases / patch releases
  2. 大型地端平台 + Kubernetes 公開訊號
  3. provider/vendor support window：EKS / AKS / GGE / OpenShift / Rancher / Tanzu
```

## 常見問題（FAQ）

### Q: 可以長期運行 kubelet 比 API server 舊 3 個 minor 嗎？

**A)**: 雖然 Kubernetes 允許 kubelet 最多比 API server 舊 3 個 minor，但這是為了支援分批 rollout，不代表長期停留在舊版本沒有風險。長期不升級可能導致：
>- 安全更新缺失（EOL 版本不再提供安全補丁）
>- 新功能無法用戶
>- 相稱性問題隨著版本差距拉大
>- 未來升級時需要更多步驟和驗證

建議制定定期升級計畫，將版本差距控制在 1-2 個 minor 以內。

### Q: 如果我的 cluster 是 single-instance，還需要遵守 skew 政策嗎？

**A)**: 是的。即使是 single-instance cluster，Kubernetes 官方仍要求遵守 skew 政策。跳過 minor 版本或違反 skew 限制可能導致：
>- API server 與 kubelet 通訊失敗
>- kubectl 操作失敗
>- 無法安裝或升級 ci某些 add-on

### Q: 如何處理 EOL 版本？

**A)**: 當版本到達 EOL（End-of-Life），Kubernetes 不再提供安全更新或 bug fixes。應該：
>1. 評估當前版本與 EOL 版本差距
>2. 選擇目標版本（通常是最新維護版本）
>3. 規劃中間版本跳板（如果差距大於 1 個 minor）
>4. 驗證相稱性
>5. 執行升級（按照版本 skew policy 和升級順序）
>6. 監控與驗證

### Q: 版本偏差政策對 managed Kubernetes 的影響？

**A)**: 對於 EKS、AKS、GGE 等 managed Kubernetes，控制平面（control plane）由 provider 管理，用戶通常不需要擔心 API server 的版本。但用戶仍需注意：
>- node 上的 kubelet 和 kube-proxy 版本需符合 provider 的要求
>- kubectl 版本需與 API server 相稱
>- 自定義 add-on 的相稱性

Provider 通常會提供支援矩陣，例如 EKS 支援特定版本範圍。

## 總結

Kubernetes 版本偏差政策是確保 cluster 穩定性和可用性的關鍵準則。遵守這些規則可以：
>- 避免升級期間的服務中斷
>- 確保元件間的相稱性
>- 簡化升級流程
>- 減少生產環境風險

建議在任何升級計畫在，將版本偏差檢查納入標準流程，並定期審查 cluster 的版本狀態。

## 相關文件

- [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [Kubernetes Releases](https://kubernetes.io/releases/)
- [Version Skew Management](kubernetes-version-skew.md)（本文件）
- [Kubernetes 版本生命周期與升級策略](kubernetes-version-lifecycle-and-upgrade-strategy.md)

---
*本文件基於 Kubernetes 官方文件與公開最佳實務整理，僅供參考。實際升級前請參閱最新官方文件。*