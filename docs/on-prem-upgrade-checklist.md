# On-prem Kubernetes 升級檢查表（On-prem Upgrade Checklist）

> 更新日期：2026-05-07
> 狀態：公開來源整理，非官方建議。
> 適用範圍：Kubernetes 自管叢集（kubeadm、RKE2、OpenShift、Tanzu）的升級規劃基線；managed Kubernetes（EKS/AKS/GKE）應同時參照 provider 專屬指南。
> 邊界：本文不保存、推論或暗示任何特定公司內部 Kubernetes 版本、cluster 拓扑、hostname、IP、ticket 或 incident 資訊。

## 一句話摘要

Kubernetes 升級不只是版本號變更，而是一連串跨 control plane、node pool、CNI/CSI/addon 的相稱性驗證與滾動更新流程；缺少檢查表容易導致服務中斷、data loss 或 cluster 無法復原。

## 版本偏差政策與升級順序回顧

### 核心 skew 規則（以 API server 1.36 為例）

| 元件 | 最大 skew 限制 | 1.36 範例解讀 |
|---|---|---|  
| `kube-apiserver` | 1 個 minor 版本 | HA 中可包含 1.35 過渡，但不可長期混太多版本 |
| `kubelet` | 3 個 minor 版本（舊於 1.25 為 2 個 minor） | 不可比 API server 新；可比 API server 舊最多 3 個 minor；1.36 時可為 1.36/1.35/1.34/1.33 |
| `kube-proxy` | 3 個 minor 版本（舊於 1.25 為 2 個 minor） | 不可比 API server 新；可比 API server 舊最多 3 個 minor；需與同 node kubelet 相稱 |
| `control plane` | 1 個 minor 版本 | 不可比所連線的 API server 新；預期與 API server minor 同一，但可舊 1 個 minor |
| `kubectl` | 1 個 minor 版本（舊或新） | 不可與所有 API server 的 skew 超過 ±1 minor |

### 官方升級順序（關鍵！）

Kubernetes 官方明確規定了升級順序，違反順序可能導致 cluster 不穩定或無法運作。

#### 從 1.35 升級到 1.36 的標準順序

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

  5. 升級 kube-proxy 或確認延後仍在 skew 範圍內
       若 proxy 長期落後 3 minor，下次 control plane 升級前會卡住

  6. 升級 kubectl / automation client / CI runner image
       避免操作端與 API server skew 超過 ±1 minor
```

## On-prem 升級前 12 點檢查表

### 升級前 12 點檢查

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

### 日常監控與準備

```text
日常版本追蹤：

  1. Kubernetes upstream releases / patch releases
  2. 大型地端平台 + Kubernetes 公開訊號
  3. provider/vendor support window：EKS / AKS / GGE / OpenShift / Rancher / Tanzu
```

## On-prem 升級實務步驟

### 階段一：準備與驗證（事前 1-2 週）

1. **版本規劃**
   - 確認當前版本與目標版本
   - 檢查 upstream 維護狀態（避免 EOL 版本）
   - 規劃中間跳板版本（如果差距 > 1 minor）

2. **Backup 與災難復原**
   - 執行 etcd 全備份（使用 velero 或 etcdctl）
   - 驗證 backup 可讀取
   - 記錄 cluster 狀態（manifest、critical workloads）

3. **相稱性掃描**
   - 掃描 API deprecation（使用 `kubectl convert` 與 dry-run）
   - 檢查 admission webhooks 是否支援新 API 版本
   - 驗證 CRD 與 controller 的版本宣告

4. **Provider/Vendor 確認**
   - 檢查 OS 版本（例如 Ubuntu/CentOS 對 kubelet 的支援）
   - 確認 container runtime（containerd/docker）版本相容性
   - 檢查 CNI plugin 版本（Calico/Flannel/Cilium）
   - 驗證 CSI driver 版本（尤其是 storage backend）

### 階段二：Control Plane 升級

1. **單一 API server 升級（非 HA 或先第一個節點）**
   ```bash
   # kubeadm 範例
   kubeadm upgrade plan <版本>
   kubeadm upgrade apply <版本>
   ```

2. **HA 環境：所有 API server 同步升級**
   - 確保所有 API server 在同一 minor 版本後再繼續
   - 使用 load balancer 時，注意 control plane 元件的連線目標

3. **其他 Control Plane 元件**
   - 升級 kube-controller-manager、kube-scheduler、cloud-controller-manager
   - 確認這些元件可以連到所有 API server

### 階段三：Node Pool 滾動升級

1. **批次規劃**
   - 規劃 drain 批次（通常 5-10% 節點一次）
   - 確認 PDB 與 disruption budget 允許的同時停機數量
   - 保留足夠 headroom 以應應突發需求

2. **單一批次流程**
   ```bash
   # 1. cordon 節點
   kubectl cordon <節點名稱>

   # 2. drain 節點（考慮 PDB 與 grace period）
   kubectl drain <節點名稱> --ignore-daemonsets --delete-emptydir-data

   # 3. 升級 kubelet（kubeadm 範例）
   #    SSH 到節點執行：
   #    - 升級 OS 套件（如果必要）
   #    - 升級 kubelet：
   apt-get install -y kubelet=<版本>
   #    - 重新啟動 kubelet
   systemctl restart kubelet

   # 4. 解除 cordon，節點回復服務
   kubectl uncordon <節點名稱>
   ```

3. **重複直到所有節點完成**

### 階段四：Addons 與後續檢查

1. **Addons 升級**
   - 升級 CNI plugin
   - 升級 CSI drivers
   - 升級 ingress controllers
   - 升級 cluster autoscaler（如果有）

2. **驗證步驟**
   - 執行 `kubectl get componentstatuses` 確認 control plane 健康
   - 執行 `kubectl get nodes` 確認所有節點 `Ready`
   - 執行 smoke test 應用程式（例如 deploy 簡單 pod）
   - 執行 E2E 測試（如果有）

3. **Rollback 準備**
   - 確認 etcd 備份可還原
   - 準備降版指令（如果有）
   - 監控 24-48 小時，確保穩定性

## 常見陷阱與風險緩解

### 陷阱 1：HA Control Plane 中部分節點未升級

**情境**：API server A 已升級到 1.36，API server B 仍在 1.35。

**風險**：
- control plane 元件（scheduler, controller-manager）若透過 LB 連線，可能連到 1.35 或 1.36，導致版本不相稱。
- kubelet 若升級到 1.36，但連到 1.35 API server，可能超出 ±3 minor 限制（1.36 - 1.35 = 1，但 1.36 比 1.35 新，而 kubelet 不可比 API server 新）。

**緩解**：HA 升級時，應同時升級所有 API server 到目標版本，或至少確保所有 API server 在同一 minor 版本範圍內，再升級其他元件。

### 陷阱 2：跳過 Minor 版本

**情境**：直接從 1.33 升級到 1.35，跳過 1.34。

**風險**：
- kubelet 1.33 與新 API server 1.35 的 skew 超過 3 個 minor（實際地是跳過了 1.34）。
- kube-proxy 同樣超出允許範圍。
- 新 API server 可能使用 kubelet 不理解的新 API 欄位。

**後果**：新 API server 啟動後，既有 node 上的 kubelet 無法與之通識，node 狀態變為 `NotReady`，pod 無法調用。

**緩解**：必須先將 kubelet 和 kube-proxy 升級到 1.34，再升級 API server 到 1.35，遵循正確順序。

### 陷阱 3：長期不升級 node

**情境**：kubelet 長期比 API server 舊 3 個 minor。

**風險**：
- 安全更新缺失（EOL 版本不再提供安全補丁）
- 新功能無法用戶
- 相稱性問題隨著版本差距拉大
- 未來升級時需要更多步驟和驗證

**緩解**：制定定期升級計畫，將版本差距控制在 1-2 個 minor 以內。

## 版本跳躍與回卷策略

### 版本跳躍

Kubernetes 官方不支援跳過 minor 版本，即使在單一實體 cluster 中。必須逐一升級（例如 1.33 → 1.34 → 1.35）。

### 回卷策略

某些元件的升級是不可逆（例如 etcd 資料結構變化），必須在升級前驗證回卷路徑。

### Patch 先升到最新

在進行 minor 升級之前，確定目前版本已升到該 minor 的最新 patch。

## 實務案例研究

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

## 版本偏差檢查清單（推奨）

### 升級前 12 點檢查（重點項目）

```text
升級前 12 點檢查（重點）：

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

## 總結

Kubernetes 升級是一個系統性的過程，需要仔細規劃與執行。遵循 version skew policy、使用檢查表、並進行逐步驗證，可以大幅降低升級風險。對於 on-prem 環境，額外的注意事項包括：
- 自行管理 control plane 的 HA 考量
- 手動處理 CNI/CSI/addon 相容性
- 規劃 node drain 的 capacity 與 PDB
- 準備 etcd 備份與回復流程

建議在任何升級計畫中，將版本偏差檢查納入標準流程，並定期審查 cluster 的版本狀態。

## 相關文件

- [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [Kubernetes Releases](https://kubernetes.io/releases/)
- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Version Skew Management](docs/kubernetes-version-skew.md)
- [Kubernetes 版本生命周期與升級策略](docs/kubernetes-version-lifecycle-and-upgrade-strategy.md)

---  
*本文件基於 Kubernetes 官方文件與公開最佳實務整理，僅供參考。實際升級前請參閱最新官方文件。*