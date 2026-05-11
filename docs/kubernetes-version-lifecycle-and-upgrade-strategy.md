1|# Kubernetes 版本生命周期與升級策略
     2|
     3|> 更新日期：2026-05-06
     4|> 狀態：公開來源整理，非官方建議。
     5|> 適用範圍：Kubernetes upstream 版本支援週期與升級規劃基線；managed Kubernetes、OpenShift、RKE2、Tanzu 等仍需再查各自 provider/vendor matrix。
     6|> 邊界：本文不保存、推論或暗射任何特定公司內部 Kubernetes 版本、cluster 拓扑、hostname、IP、ticket 或 incident 資訊。
     7|
     8|---
     9|
    10|## 一句話摘要
    11|
    12>Kubernetes 版本管理不是單純「升到最新版」，而是一個結合版本支援週期、version skew policy、provider matrix、以及地端平台限制的多層規劃過程。
    13|
    14|```text
    15>版本管理決策鏈：
    16>
    17>  support window
    18>      ↓
    19>  version skew policy
    20>      ↓
    21>  control plane upgrade order
    22>      ↓
    23>  node drain / kubelet rollout
    24>      ↓
    25>  CNI / CSI / admission webhook / CRD / client 相容性驗證
    26>```
    27|
    28|---
    29|
    30|## 主要公開來源
    31|
    32>| 來源 | 用途 |
    33>|---|---||
    34>| [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) | upstream 官方 skew 規則與元件升級順序 |
    35>| [Kubernetes Releases](https://kubernetes.io/releases/) | 目前 upstream 維護分支、latest release、EOL |
    36>| [Upgrade A Cluster](https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/) | 官方 cluster upgrade 高階流程 |
    37>| [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) | kubeadm 升級流程與不支援跳 minor 的提醒 |
    38>| [Kubernetes GitHub Releases](https://github.com/kubernetes/sig-release/tree/master/releases) | 版本 artifacts 與 release notes |
    39>|
    40|---  
    41|
    42|## 1. Kubernetes 版本支援週期
    43|
    44|### 1.1 支援期間與 patch 策略
    45|
    46|```text
    47>支援模型：
    48>
    49>  - 維護分支：最近三個 minor 版本（例如目前 1.36 / 1.35 / 1.34）
    50>  - patch support：
    51>      • 1.19 及 newer：約 1 年 patch support
    52>      • 1.18 及 older：約 9 個月 patch support
    53>  - patch releases：定期切出，必要時發行 urgent patch
    54>  - 決策權：Release Managers 小組
    55>```
    56|
    57|### 1.2 目前維護版本（2026-05-06）
    58|
    59|```text
    60>Upstream maintained minors observed:
    61>
    62>  1.36  latest: 1.36.0  EOL: 2027-06-28
    63>  1.35  latest: 1.35.3  EOL: 2027-02-28
    64>  1.34  latest: 1.34.6  EOL: 2026-10-27
    65>```
    66|
    67|### 1.3 End-of-Life (EOL) 排程
    68|
    69|根據 Kubernetes 官方 releases 頁面，版本 EOL 遵循以下模式：
    70|
    71|```text
    72>版本        EOL 日期
    73>------------ -------------------
    74>1.36        2027-06-28
    75>1.35        2027-02-28
    76>1.34        2026-10-27
    77>1.33        2026-06-28
    78>1.32        2026-02-28
    79>1.31        2025-11-11
    80>1.30        2025-07-15
    81>```
    82|
    83|> 注意：EOL 後不再提供安全更新或 bug fixes。使用者強烈建議升級到維護中的版本。
    84|
    85|### 1.4 版本釋出排程
    86|
    87|Kubernetes 遵循準定時釋出模型：
    88|
    89|```text
    90>cadence：
    91>  - 新 minor release：約每 3-4 個月一次
    92>  - patch releases：每月定期，必要時 urgent 釋出
    93>  - 範例排程：
    94>      1.36：2026-04-22 釋出
    95>      1.37：預估 2026-07-22 左右
    96>```
    97|
    98|詳細排程可參考 [Kubernetes Release Schedule](https://github.com/kubernetes/sig-release/tree/master/schedules) 目錄。
    99|
    100|---
    101|
    102|## 2. Version Skew Policy 詳解
    103|
    104>Version Skew Policy 定義了升級期間元件間的安全版本差距，確保 cluster 穩定性可用性。
    105|
    106|### 2.1 支援版本 skew 總覽
    107|
    108>| 元件 | 最大 skew 限制 | 關鍵限制 |
    109>|---|---|---|-----|
    110>| `kube-apiserver` | 1 個 minor 版本 | HA cluster 中所有 API server 必須在 1 個 minor 內 |
    111>| `kubelet` | 3 個 minor 版本（舊於 1.25 為 2 個 minor） | 不可比 API server 新；可比 API server 舊最多 3 個 minor |
    112>| `kube-proxy` | 3 個 minor 版本（舊於 1.25 為 2 個 minor） | 不可比 API server 新；可比 API server 舊最多 3 個 minor；需與同 node kubelet 在允許範圍內 |
    113>| `kube-controller-manager` / `kube-scheduler` / `cloud-controller-manager` | 1 個 minor 版本 | 不可比所連線的 API server 新；預期與 API server minor 相同，但可舊 1 個 minor |
    114>| `kubectl` | 1 個 minor 版本（舊或新） | 不可與所有 API server 的 skew 超過 ±1 minor |
    115|
    116|### 2.2 實務範例（以 API server 1.36 為例）
    117|
    118>| 元件 | 支援版本範圍 | 備註 |
    119>|---|---|---|-----|
    120>| `kube-apiserver` | 1.36 | HA 中可包含 1.35 過渡 |
    121>| `kubelet` | 1.36, 1.35, 1.34, 1.33 | 舊於 1.25 的規則更窄 |
    122>| `kube-proxy` | 1.36, 1.35, 1.34, 1.33 | 需與同 node kubelet 相容 |
    123>| `control plane` | 1.36, 1.35 | 通常建議與 API server 相同 |
    124>| `kubectl` | 1.37, 1.36, 1.35 | 可比 API server 新或舊 1 個 minor |
    125|
    126|### 2.3 HA cluster 的陷阱與風險
    127|
    128>在 HA cluster 中，如果 API server 間存在 skew，會收窄其他元件的可用範圍：
    129|
    130>```text
    131>HA control plane upgrade window:
    132>
    133>  kube-apiserver A: 1.36
    134>  kube-apiserver B: 1.35
    135>          │
    136>          ├─ kubelet 1.36?  不安全：可能比 1.35 API server 新
    137>          ├─ scheduler 1.36? 若透過 LB 可能連到 1.35，需等 API server 全部升完
    138>          └─ kubectl 1.37?  可能對 1.35 超過 +1 minor
    139>```
    140|
    141>實務含意：
    142>
    143>- control plane rolling upgrade 期間要同時檢查所有 API server 的 min/max 版本。
    144>- controller/scheduler/cloud-controller-manager 若透過 load balancer 連線，必須等所有 API server 都完成目標 minor 後再升級。
    145>- node pool 升級批次要將 kubelet / kube-proxy 與 API server 最舊版本一起檢查。
    146|
    147|### 2.4 元件升級順序
    148|
    149>從 1.35 升級到 1.36 的標準順序：
    150>
    151>```text
    152>建議檢查順序：
    153>
    154>  0. 先升到目前 minor 的最新 patch
    155>       例：1.35.x → 最新 1.35 patch
    156>
    157>  1. 檢查 admission webhooks / CRD / API deprecation
    158>       新 API server 可能送出新欄位或新 resource version
    159>
    160>  2. 升級 kube-apiserver
    161>       HA: 確認 API server 間最多 1 minor skew
    162>
    163>  3. 升級 kube-controller-manager / kube-scheduler / cloud-controller-manager
    164>       前提：這些元件會連到的 API server 都已在目標 minor
    165>
    166>  4. 分批 drain node，升 kubelet
    167>       minor 版本 kubelet upgrade 前要 drain；不建議 in-place 直接升
    168>
    169>  5. 升 kube-proxy 或確認延後仍在 skew 範圍內
    170>       若 proxy 長期落後 3 minor，下次 control plane 升級前會卡住
    171>
    172>  6. 升級 kubectl / automation client / CI runner image
    173>       避免操作端與 API server skew 超過 ±1 minor
    174>```
    175|
    176|---
    177|
    178|## 3. 升級策略與最佳實践
    179|
    180>### 3.1 大型地端平台的特殊考量
    181>
    182>大型地端平台的升級決策需額外考量：
    183>
    184>```text
    185>地端升級因素：
    186>
    187>  - 維護窗口與 drain capacity
    188>  - DaemonSet / PDB / disruption budget 設計
    189>  - CNI / CSI node plugin 相容性
    190>  - OS image / kernel / container runtime baseline
    191>  - observability 與 alert rule 是否仍支援舊 metric / label
    192>  - app team migration cost
    193>  - 7x24 製造可靠性壓力
    194>```
    195|
    196>### 3.2 升級前 12 點檢查
    197|
    198>```text
    199>Before version bump:
    200>
    201>  [ ] 1. upstream support window：目標 minor 是否在目前維護分支內？
    202>  [ ] 2. provider/vendor matrix：OS、runtime、CNI、CSI、ingress、LB 是否支援目標 minor？
    203>  [ ] 3. API deprecation：kubectl convert / server-side dry-run / audit log 是否掃過？
    204>  [ ] 4. admission webhooks：是否支援新 API 欄位、版本與 matchPolicy: Equivalent？
    205>  [ ] 5. CRD + controller：operator/controller 是否聲明支援目標 minor？
    206>  [ ] 6. etcd：backup / restore drill / defrag / alarm status 是否完成？
    207>  [ ] 7. control plane：HA API server min/max skew 是否規劃好？
    208>  [ ] 8. node capacity：drain 時是否仍保有足夠 headroom 與 PDB 可用性？
    209>  [ ] 9. kubelet config：deprecated flags / config schema 是否檢查？
    210>  [ ] 10. kube-proxy / CNI：iptables / IPVS / eBPF datapath 是否有版本限制？
    211>  [ ] 11. client tooling：kubectl、CI image、GitOps controller 是否在 ±1 minor 或官方支援範圍？
    212>  [ ] 12. rollback：是否明確知道哪些步驟只能前進，哪些可回復？
    213>```
    214|
    215|### 3.3 版本跳躍與回滚策略
    216|
    217>- **版本跳躍**：Kubernetes 官方不支援跳過 minor 版本，即使在單一實體 cluster 中。必須逐一升級（例如 1.33 → 1.34 → 1.35）。
    218|- **回滚**：某些元件的升級是不可逆（例如 etcd 資料結構變化），必須在升級前驗證回滚路徑。
    219|- **Patch 先升到最新**：在進行 minor 升級之前，確保目前版本已升到該 minor 的最新 patch。
    220|
    221|### 3.4 長期不升風險
    222|
    223>雖然 `kubelet` / `kube-proxy` 最多可落後 API server 3 個 minor，但這是為了支援分批 rollout，不代表長期停留在舊版本沒有風險：
    224>
    225>```text
    226>短期可接受：
    227>  API server 1.36
    228>  node pool A kubelet 1.36
    229>  node pool B kubelet 1.35
    230>  node pool C kubelet 1.34 / 1.33 during migration
    231>
    232>長期高風險：
    233>  node pool 永久停在 API server -3 minor
    234>  下一次 API server minor upgrade 前，node 必須先追上
    235>```
    236|
    237|---
    238|
    239|## 4. EOL 後處理策略
    240|
    241>### 4.1 版本到達 EOL 後
    242>
    243>- **立即後果**：不再提供安全更新或 bug fixes。
    244>- **風險層級**：高 - 暴露於未修補的安全漏洞。
    245>- **建議行動**：制定並執行升級計畫，優先升級到維護中的版本。
    246|
    247>### 4.2 升級路徑規劃
    248>
    249>當版本到達 EOL 時：
    250>
    251>1. **評估當前版本**：確認當前版本與 EOL 版本差距。
    252>2. **選擇目標版本**：選擇最新的維護版本（通常是最新 minor）。
    253>3. **規劃跳板**：如果差距大於 1 個 minor，規劃中間版本跳板。
    254>4. **驗證相容性**：檢查所有依賴元件的相容性。
    255>5. **執行升級**：按照版本 skew policy 和升級順序執行。
    256>6. **監控與驗證**：升級後監控 cluster 穩定性和應用程式行為。
    257>
    258>### 4.3 特殊情況：EOL 後 emergency patch
    259>
    260>有時候，即使版本到達 EOL，社群仍可能發行 emergency patch 來解決重大安全問題：
    261>
    262>```text
    263>範例：
    264>  - 1.26.15 在 EOL 後釋出以解決 Go CVEs
    265>  - 1.25.16 在 EOL 後釋出以解決 CVE-2023-5528
    266>  - 1.24.17 在 EOL 後釋出以解決 CVE-2023-3676 和 CVE-2023-3955
    267>```
    268>
    269>但這不應該作為依賴，EOL 後的最佳實践還是升級到維護版本。
    270|
    271|---
    272|
    273|## 5. 大型地端平台升級策略要點
    274|
    275>### 5.1 容量規劃與維護窗口
    276>
    277>- **drain capacity**：計算同時 drain 的 node 數量，確保應用程式高可用性。
    278>- **維護窗口**：規劃足夠長的維護窗口，考慮到大型 cluster 的升級時間。
    279>- **批次升級**：將 node 分批升級，降低風險。
    280|
    281>### 5.2 相容性矩陣管理
    282>
    283>- **CNI / CSI**：檢查 CNI 與 CSI 驅動是否支援目標版本。
    284>- **CRI**：確認 container runtime (containerd, CRI-O, Docker) 是否相容。
    285>- **OS / kernel**：檢查 node OS 與 kernel 版本要求。
    286>- **監控與日誌**：確保 Prometheus exporter、Fluentd 等工具相容。
    287>
    288>### 5.3 應用程式相容性
    289>
    290>- **admission controller / webhook**：驗證新的 API 版本支援。
    291>- **CRD**：確認自定義資源定義與控制器相容。
    292>- **應用程式碼**：檢查應用程式碼中是否有硬編碼的 API 版本。
    293>
    294>### 5.4 測試策略
    295|
    296>- **金絲雀測試**：在小規模環境中先行測試。
    297>- **負載測試**：執行負載測試，確保性能符合期望。
    298>- **故障注入**：使用 chaos engineering 工具測試韌性。
    299>- **回滚測試**：驗證回滚程序的可行性。
    300|
    301|---
    302|
    303|## 6. 持續追蹤與更新
    304|
    305>### 6.1 每日檢查項目
    306|
    307>```text
    308>每日版本追蹤：
    309>
    310>  1. Kubernetes upstream releases / patch releases
    311>  2. 大型地端平台 + Kubernetes 公開訊號
    312>  3. provider/vendor support window：EKS / AKS / GKE / OpenShift / Rancher / Tanzu
    313>```
    314|
    315>### 6.2 可信來源優先序
    316|
    317>```text
    318>A. 官方/一手來源
    319>  - Kubernetes official releases
    320>  - Kubernetes GitHub releases
    321>  - 大型企業官方 careers pages
    322>  - 大型地端平台 engineering / blog / conference pages
    323>
    324>B. 高可信二手來源
    325>  - CNCF / KubeCon / iThome Kubernetes Summit speaker pages
    326>  - cloud provider official support matrix: EKS / AKS / GKE
    327>  - vendor docs: Red Hat OpenShift / Rancher / VMware Tanzu
    328>
    329>C. 只作線索
    330>  - LinkedIn jobs
    331>  - Glassdoor / Cake / job mirrors
    332>  - forum posts / social posts
    333>  - conference slides not hosted by organizer or speaker
    334>```
    335|
    336|---
    337|
    338|## 7. 隱私與安全邊界
    339|
    340>```text
    341>本文只保存：
    342>  - Kubernetes 官方公開文件 URL
    343>  - 通用 version skew / upgrade planning 摘要
    344>  - 可驗證的 upstream release baseline
    345>
    346>本文不保存：
    347>  - 特定公司內部 Kubernetes version
    348>  - cluster name / hostname / IP / ticket / incident
    349>  - 個人 email / 電話 / 私訊
    350>  - secrets / tokens / credentials
    351>```
    352|
    353|--- 
    354|
    355|## 相關文件
    356|
    357>- [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
    358>- [Kubernetes Releases](https://kubernetes.io/releases/)
    359>- [Version Skew Management](kubernetes-version-skew.md)
    360>- [Version Watch](kubernetes-version-watch.md)
    361>|]