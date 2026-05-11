# Kubernetes 版本偏差實務案例研究（On-prem Case Studies）

> 更新日期：2026-05-09
> 狀態：公開來源整理，非官方建議。
> 適用範圍：Kubernetes 自管叢集（kubeadm、RKE2、OpenShift、Tanzu）的版本偏差實務案例；managed Kubernetes（EKS/AKS/GKE）應同時參照 provider 專屬指南。
> 邊界：本文不保存、推論或暗示任何特定公司內部 Kubernetes 版本、cluster 拓扑、hostname、IP、ticket 或 incident 資訊。

## 一句話摘要

版本偏差政策（Version Skew Policy）不只是理論規則，更是實務中避免服務中斷的關鍵指南。本文件彙整大型地端平台常見的版本偏差陷阱與解決方案。

## 目錄

```text
1. 案例 3：地端平台長期滯後 3 個 minor 的 kubelet
2. 案例 4：HA cluster 中 API server 版本混雜與 control plane 升級
3. 案例 5：CNI/CSI 元件版本限制與 skew 政策衝突
4. 案例 6：地端平台的版本跳躍與回卷策略
5. 案例 7：多個 node pool 的分批升級與 skew 管理
6. 案例 8：observability 工具鏈的版本相容性
7. 結論與最佳實践
```

## 案例 3：地端平台長期滯後 3 個 minor 的 kubelet

### 情境

某大型地端平台運行 Kubernetes 1.33，由於資源限制和測試負擔，plan 直接升級 control plane 到 1.35，跳過 1.34。kubelet 仍留在 1.33，而 CNI 和 CSI 元件也相對較舊。

### 問題

根據 version skew policy：
- kubelet 必須不比 API server 新，且可比 API server 舊最多 3 個 minor（1.25 以前為 2 個 minor）
- 1.35 API server 與 1.33 kubelet 的差距為 2 個 minor，理論上在允許範圍內
- 但實際上跳過 1.34 導致中間版本缺少，許多 security patches 和 bug fixes 未應用
- 長期滯後導致相容性風險累積

### 後果

1. **安全風險**：1.34 和 1.35 的 security patches 未部署
2. **功能缺失**：新功能無法使用，app teams 受限
3. **未來升級成本**：下次升級需補足中間版本，工作量倍增
4. **相容性問題**：某些新 API 欄位或 resource version 可能 kubelet 不理解

### 解決策

**原則：避免版本跳躍，制定定期升級計畫**

推奦做法：
1. **制定版本管理政策**：每季或每半年檢討版本，將差距控制在 1-2 個 minor 內
2. **分階段升級**：
   ```
   當前：1.33
   目標：1.35
   步驟：
    1. 升級到 1.34.x（最新 patch）
    2. 驗證穩定 2-4 週
    3. 升級到 1.35.x（最新 patch）
   ```
3. **利用 patch releases**：在進行 minor 升級前，確保目前版本已升到該 minor 的最新 patch
4. **建立測試環境**：每個版本都有專屬測試，確保相容性

### 地端平台特殊考量

- **測試資源有限**：建議建立自動化測試流水線，減少手動驗證負擔
- **change window 限制**：將升級分解為多個小批次，分散風險
- **app team coordination**：提前通知 app teams，提供相容性清單

## 案例 4：HA cluster 中 API server 版本混雜與 control plane 升級

### 情境

HA cluster 有兩個 control plane node，A 已升級到 1.36，B 仍在 1.35。由於 maintenance window 限制，無法同時升級所有 API server。

### 問題

根據 policy：
- HA cluster 中所有 API server 必須在 1 個 minor 內
- 但現實中常有部分節點滯後
- 這會收窄其他元件的可用版本範圍

### 後果

1. **control plane 元件限制**：scheduler/controller-manager 必須與最舊的 API server 相容
2. **kubelet 限制**：kubelet 版本範圍被壓縮
3. **kubectl 限制**：kubectl 版本選擇受限
4. **admission webhooks**：可能面臨新 API 欄位處理問題

### 解決策

**原則：最小化 API server 版本差距，或確保 load balancer 路由策略**

推�做法：
1. **優先同步所有 API server**：即使需要分階段，也應盡快縮小差距
2. **檢查 load balancer 配置**：
   - 如果 control plane 元件透過 LB 連線，必須等所有 API server 升到目標 minor 後才升級
   - 考慮配置 session affinity 或 routing 策略，確保元件連線到相容的 API server
3. **分批次但同步升級**：
   ```
   階段 1：升級 API server A 到 1.36
   階段 2：驗證穩定，確保 admission webhooks 相容
   階段 3：升級 API server B 到 1.36（盡快）
   階段 4：同步升級 control plane 元件
   ```
4. **監控與警報**：設置警報，當 API server 版本差距超過 1 minor 時通知

### 地端平台特殊考量

- **load balancer 限制**：某些地端 LB 可能不支援 advanced routing，需依賴同步升級
- **maintenance window 短**：考慮使用 rolling update 機制，縮短 downtime
- **app team impact**：提前與 app teams 協調，確保他們了解可能的影響

## 案例 5：CNI/CSI 元件版本限制與 skew 政策衝突

### 情境

地端平台使用 Calico 作為 CNI，CNI 元件有其自己的版本相容性要求。當 Kubernetes 版本升級時，CNI 可能尚未支援新版本。

### 問題

1. **CNI 版本限制**：Calico 可能只支援特定 Kubernetes 版本範圍
2. **CSI 驅動限制**：storage plugin 可能有自己的版本要求
3. **skew 衝突**：Kubernetes version skew policy 與 CNI/CSI 版本要求可能衝突

### 後果

1. **網路中斷**：CNI 不相容導致 pod 無法取得 IP 或網路策略失效
2. **storage 問題**：CSI 驅動不相容導致 PVC 掛載失敗
3. **cluster 不穩定**：元件間相容性問題導致服務不穩定

### 解決策

**原則：查閱 provider 版本矩陣，制定相容性計畫**

推�做法：
1. **查閱官方版本矩陣**：
   - Calico: https://www.projectcalico.org/calico-kubernetes-compatibility/
   - Cilium: https://docs.cilium.io/en/stable/installation/requirements/
   - 其他 CSI 驅動：參閱 vendor 文件
2. **制定升級順序**：
   ```
   優先順序：
     1. 確認 target Kubernetes 版本
     2. 查閱 CNI/CSI 支援矩陣
     3. 必要時先升級 CNI/CSI 元件
     4. 再升級 Kubernetes control plane
     5. 最後升級 node components
   ```
3. **建立測試環境**：在升級生產環境前，在測試環境驗證整個技術堆疊的相容性
4. **維護 upgrade window**：選擇合適的維護窗口，確保有足夠時間回滚

### 地端平台特殊考量

- **自訂 CNI/CSI**：若使用自定制版本，需額外驗證相容性
- **多個 CNI 部署**：某些地端平台可能有多個 CNI 部署，需個別驗證
- **第三方工具鏈**：observability 工具（如 Prometheus, Grafana）可能依賴特定 CNI metrics

## 案例 6：地端平台的版本跳躍與回卷策略

### 情境

某地端平台長期運行 Kubernetes 1.31，由於各種限制，現在想直接升級到 1.35，跳過 1.32-1.34。

### 問題

Kubernetes 官方不支援跳過 minor 版本，即使在單一實體 cluster 中。版本跳躍可能導致：

1. **API deprecation**：中間版本移除的 API 可能造成問題
2. **feature gate**：某些功能可能在中間版本啟用或關閉
3. **data format changes**：某些資料結構可能改變
4. **admission webhook changes**：webhook 可能依賴已移除的 API

### 後果

1. **cluster 無法啟動**：某些元件可能因相容性問題無法啟動
2. **pod 調用失敗**：kube-scheduler 或 controller-manager 可能無法正確處理物件
3. **data loss**：某些資料轉換可能導致資訊丢失
4. **難以回卷**：某些步驟可能不可逆（如 etcd 資料結構變更）

### 解決策

**原則：逐步升級，避免跳躍**

推奦做法：
1. **制定逐步升級計畫**：
   ```
   當前：1.31
   目標：1.35
   步驟：
    1. 升級到 1.32.x（最新 patch）
    2. 穩定化 2-4 週
    3. 升級到 1.33.x（最新 patch）
    4. 穩定化 2-4 週
    5. 升級到 1.34.x（最新 patch）
    6. 穩定化 2-4 週
    7. 升級到 1.35.x（最新 patch）
   ```
2. **每步都進行完整驗證**：包括功能測試、相容性測試、性能測試
3. **建立回卷計畫**：每一步都應有明確的回卷程序
4. **使用 kubeadm 等工具**：kubeadm 等工具會強制執行版本順序，避免跳躍

### 地端平台特殊考量

- **資源限制**：分階段升級需要更多時間和資源，需提前規劃
- **app team burden**：每一步升級都可能影響 app teams，需良好溝通
- **testing environment**：需要足夠的測試環境模擬每一步升級

## 案例 7：多個 node pool 的分批升級與 skew 管理

### 情境

地端平台有多個 node pool，包括不同用途（compute, memory, gpu）和不同 OS（Ubuntu, RHEL）的節點。升級時需要分批處理。

### 問題

1. **skew 管理複雜**：不同 node pool 可能有不同的升級進度
2. **污點與 toleration**：某些 node pool 可能有特殊配置
3. **pod 調和**：scheduler 可能將 pod 調和到不同版本 node
4. **CNI/CSI 影響**：不同 node pool可能使用不同的 CNI/CSI 配置

### 後果

1. **pod 調用失敗**：pod 可能被調和到不相容的 node
2. **service 中斷**：分批升級期間可能導致某些服務資源不足
3. **相容性問題**：不同版本 node 之間的通信問題

### 解決策

**原則：制定詳細的 node pool 升級計畫，確保 skew 在允許範圍內**

推奦做法：
1. **分類 node pool**：根據用途和版本要件分類
2. **制定升級順序**：
   ```
   優先順序：
     1. 先升級 control plane（所有 API server）
     2. 升級 node pool 按優先順序：
        a. 先升級 general compute pool
        b. 再升級 specialized pools (memory, gpu)
        c. 最後升級特殊 OS pools
     3. 確保每個 node pool 升級後，kubelet 版本與 API server 相容
   ```
3. **使用 PDB 和 disruption budget**：確保升級期間服務可用性
4. **監控 skew 差距**：設置監控，當某個 node pool 的 skew 過大時警報
5. **污點與 toleration 管理**：確保 pod 不被調和到不相容的 node

### 地端平台特殊考量

- **多個 OS 版本**：不同 OS 可能需要不同的升級程序
- **hardware diversity**：GPU 節點可能有特殊的驅動程式要求
- **app labeling**：app teams 可能使用特定的 labels，需要協調

## 案例 8：observability 工具鏈的版本相容性

### 情境

地端平台使用 Prometheus + Grafana + Alertmanager 等 observability 工具鏈。Kubernetes 升級後，這些工具可能面臨版本相容性問題。

### 問題

1. **kube-state-metrics**：依賴 Kubernetes API，可能面臨新 API 欄位處理問題
2. **Prometheus**：可能依賴特定 Kubernetes 版本的 service discovery
3. **Grafana**：dashboard 可能依賴特定 metrics 格式
4. **custom exporters**：自定制 exporter 可能依賴舊 API

### 後果

1. **metrics 缺失**：某些 metrics 可能無法收集
2. **alert 失效**：alert rule 可能因 metrics 格式改變而失效
3. **dashboard 錯誤**：Grafana dashboard 可能顯示錯誤資訊
4. **monitoring gap**：監控覆蓋出現缺口

### 解決策

**原則：在升級計畫中包含 observability 工具鏈**

推薦做法：
1. **查閱工具鏈版本矩陣**：確認工具鏈支援目標 Kubernetes 版本
2. **制定升級順序**：
   ```
   優先順序：
     1. 先升級 observability 工具鏈（如果需要）
     2. 再升級 Kubernetes control plane
     3. 最後升級 node components
   ```
3. **測試 metrics pipeline**：在升級前，在測試環境驗證整個 metrics pipeline
4. **更新 alert rules 和 dashboards**：根據新 metrics 格式更新規則和圖表
5. **維護回卷計畫**：確保可以回卷到之前的版本

### 地端平台特殊考量

- **自訂 metrics**：某些自訂 metrics exporter 可能需要額外驗證
- **告警依賴**：某些業務關鍵告警可能依賴特定 metrics
- **歷史資料相容性**：長期 metrics 儲存可能受影響

## 結論與最佳實践

從這些實務案例中，我們總結以下最佳實践：

### 1. 避免版本跳躍

- 制定定期升級計畫，將版本差距控制在 1-2 個 minor 內
- 使用 patch releases 作為跳板，確保每個中間版本都有安全更新

### 2. 管理 HA cluster 的 skew

- 盡量同步升級所有 API server，保持版本一致
- 如果必須分批，確保 load balancer 路由策略或 session affinity
- 監控 API server 版本差距，避免超過 1 minor

### 3. 協調 CNI/CSI 與其他元件

- 查閱 provider 版本矩陣，確保所有元件相容
- 必要時先升級 CNI/CSI，再升級 Kubernetes
- 建立完整的測試環境，驗證整個技術堆疊

### 4. 制定詳細的升級計畫

- 使用官方升級順序：API server → control plane → kubelet → kube-proxy
- 每一步都進行完整驗證，包括功能、相容性、性能
- 建立回卷計畫，確保可以安全回退

### 5. 考慮地端平台特殊因素

- 資源限制：分階段升級，利用自動化減少負擔
- 維護窗口：規劃合適的維護窗口，提前通知所有 stakeholders
- app team coordination：良好溝通，提供相容性清單和升級指南
- 多樣性：處理多個 OS、hardware 和 node pool 的複雜性

### 6. 包含整個工具鏈

- 在升級計畫中包含 observability、CI/CD 和其他依賴工具
- 測試整個 pipeline，確保沒有中斷

## 相關文書

- [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [Kubernetes Releases](https://kubernetes.io/releases/)
- [On-prem Kubernetes Upgrade Checklist](on-prem-upgrade-checklist.md)
- [Kubernetes Version Lifecycle and Upgrade Strategy](kubernetes-version-lifecycle-and-upgrade-strategy.md)

## 追蹤與更新

本文件將持續根據實務經驗更新。如有新的案例或見解，歡迎貢獻。

---

> 本文件基於 Kubernetes 官方文件、公開最佳實務與實務案例整理。實際升級前請參閱最新官方文件，並根據自身環境調整。
> 不保存、推論或暗示任何特定公司內部 Kubernetes 版本、cluster 拓扑、hostname、IP、ticket 或 incident 資訊。