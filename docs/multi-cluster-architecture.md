# Multi-cluster / multi-region Kubernetes architecture 導讀

> 狀態：公開來源整理；非特定公司架構建議。
> 最後更新：2026-05-13
> 適用範圍：需要把多個 Kubernetes cluster 視為一個平台組合來治理、連線與發布的 SRE / platform engineering 團隊。
> 不適用範圍：不能直接當作任何特定公司內部拓撲、IP plan、router config、DR RTO/RPO 或安全政策。

---

## TL;DR

Multi-cluster 不只是「多開幾個 cluster」。比較安全的思考順序是先決定隔離邊界，再決定服務發現與流量入口，最後才選控制面工具。

```text
┌──────────────────────────────────────────────────────────────────┐
│ Multi-cluster design question                                    │
└─────────────┬────────────────────────────────────────────────────┘
              ▼
  1. 為什麼要多 cluster？
     - blast radius / upgrade window / region / tenant / compliance
              ▼
  2. cluster 之間是否需要服務互相發現？
     - none / DNS only / exported service / global VIP / gateway
              ▼
  3. 誰能跨 cluster 發布與變更？
     - GitOps / fleet manager / policy controller / human approval
              ▼
  4. 事故時怎麼降級？
     - isolate / fail over / drain region / rollback / freeze deploy
```

核心原則：

- Kubernetes upstream 本身把「單一 cluster 內的 Service、Pod network、Node、workload」定義得很清楚；跨 cluster 的部分通常需要額外 API、控制器或 provider/vendor 功能。
- SIG Multicluster 的 Multi-Cluster Services API 用 `ServiceExport` / `ServiceImport` 描述「哪些 service 要跨 cluster 可見」，比手工共享所有 DNS/IP 更適合做低風險邊界。
- Cloud provider 的 multi-cluster 功能（例如 GKE Multi-cluster Services / Ingress、Azure Kubernetes Fleet Manager）可做參考，但它們是 provider-managed 語意；不能直接假設可以原樣搬到 on-prem。
- on-prem / bare-metal 場景通常要額外處理 L3/L4 load balancing、BGP/L2 announcement、DNS、PKI、identity、network policy、storage locality 與 observability 聚合。

---

## 來源與驗證狀態

| 類型 | 來源 | 本次使用方式 |
|---|---|---|
| Kubernetes official | [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/) | 單 cluster network baseline：Pod/Service/Node/Control-plane connectivity expectations。 |
| Kubernetes official | [Service](https://kubernetes.io/docs/concepts/services-networking/service/) | Service 抽象與 `type=LoadBalancer` 邊界。 |
| Kubernetes official | [Managing Workloads](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/) | workload rollout / management 是 cluster-local baseline。 |
| SIG Multicluster | [Multicluster Services API Overview](https://multicluster.sigs.k8s.io/concepts/multicluster-services-api/) | 跨 cluster service discovery 的官方 SIG API 方向。 |
| SIG Multicluster | [ServiceExport](https://multicluster.sigs.k8s.io/api-types/service-export/) / [ServiceImport](https://multicluster.sigs.k8s.io/api-types/service-import/) | 用 export/import 而不是全域暴露所有 service。 |
| Google Cloud | [GKE Multi-cluster Services](https://cloud.google.com/kubernetes-engine/docs/concepts/multi-cluster-services) | provider-managed implementation 參考。 |
| Google Cloud | [GKE Multi Cluster Ingress](https://cloud.google.com/kubernetes-engine/docs/concepts/multi-cluster-ingress) | global load balancing / ingress pattern 參考。 |
| Microsoft Azure | [Azure Kubernetes Fleet Manager fleets and member clusters](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/concepts-fleet) | fleet / member cluster governance pattern 參考。 |
| Open Cluster Management | [ManagedCluster concept](https://open-cluster-management.io/docs/concepts/cluster-inventory/managedcluster/) | cluster inventory / hub-managed-cluster vocabulary 參考。 |

待查：

- Gateway API 官方文件是否已有穩定的 multi-cluster guide；本次預期 URL 回傳 404，未納入核心來源。
- Karmada / OCM / Rancher / OpenShift ACM 等工具的最新版本差異，應另開 vendor/tool-specific 文件，不應混在本篇作為通用結論。
- 各環境的 DNS、PKI、network policy、east-west encryption、audit logging 與 failover SLO 需要以實際平台政策驗證。

---

## 為什麼需要 multi-cluster？

常見動機可以分成四類：

```text
┌─────────────────┬──────────────────────────────┬──────────────────────┐
│ 動機             │ 常見好處                      │ 常見風險              │
├─────────────────┼──────────────────────────────┼──────────────────────┤
│ Blast radius     │ cluster 故障不拖垮全部服務     │ 運維與觀測成本上升     │
│ Upgrade windows  │ 分批升級、分批驗證             │ 版本與 addon skew      │
│ Region / locality│ latency / data locality       │ 跨區網路與資料一致性   │
│ Tenant / boundary│ 權限、成本、法規邊界較清楚      │ 平台碎片化             │
└─────────────────┴──────────────────────────────┴──────────────────────┘
```

低風險判斷：

- 如果只是想讓團隊分權，先檢查 namespace / RBAC / quota / policy 是否足夠，不一定需要多 cluster。
- 如果是升級風險、region fault domain、tenant isolation 或合規邊界，multi-cluster 才比較有明確理由。
- 如果 workload 需要跨 cluster 強一致寫入，先把資料層與 RTO/RPO 講清楚；Kubernetes 多 cluster service discovery 不會自動解決資料一致性。

---

## 基準：Kubernetes upstream 定義的是 cluster-local primitives

Kubernetes official docs 對單一 cluster 的基礎語意很清楚：

- Cluster networking 要求 Pod、Node、Service 與 control plane components 之間有可預期的連線模型。
- Service 提供一組 Pod 的穩定抽象；`type=LoadBalancer` 需要環境提供外部 load balancer 實作。
- workload rollout / deployment 管理通常以單一 cluster 為邊界。

因此 multi-cluster 設計的第一個原則是：不要假設 Kubernetes 自動提供跨 cluster 的單一 service mesh、單一 scheduler、單一 storage plane 或單一 failure domain。

```text
單一 cluster：

  Pod ── Service ── Ingress/LoadBalancer
   │        │              │
   └── kubelet / CNI / kube-proxy / controller manager / API server

多 cluster：

  Cluster A                         Cluster B
  ┌──────────────┐                  ┌──────────────┐
  │ Service A    │                  │ Service A    │
  │ Pods         │                  │ Pods         │
  └──────┬───────┘                  └──────┬───────┘
         │                                  │
         └──── needs explicit export/import, DNS, gateway, LB, policy ────┘
```

---

## Pattern 1：完全隔離，只共享 GitOps / image / policy baseline

適合：

- 不需要 cluster 之間直接呼叫。
- 想降低 blast radius。
- 每個 cluster 都有獨立 ingress / DNS / monitoring slice。

```text
Git repo / artifact registry / policy baseline
             │
      ┌──────┴──────┐
      ▼             ▼
  Cluster A     Cluster B
  local DNS     local DNS
  local LB      local LB
```

優點：

- 最容易解釋 failure domain。
- 網路、DNS、identity 和 RBAC 邊界清楚。
- on-prem 環境可先用這個 pattern 累積升級與操作經驗。

風險：

- 全域流量切換與跨區 failover 需要外部 DNS / LB / traffic manager。
- 服務之間若開始互相呼叫，很容易長出手工 DNS 或 hard-coded endpoint。

最低檢查：

- 每個 cluster 的 ingress/LB/DNS owner 明確。
- shared GitOps pipeline 有 per-cluster approval / freeze / rollback。
- observability 聚合只保存必要 metadata，不保存 sensitive payload。

---

## Pattern 2：明確 export/import 的跨 cluster service discovery

SIG Multicluster 的 Multi-Cluster Services API 把跨 cluster service 可見性拆成：

- `ServiceExport`：在來源 cluster 宣告某個 Service 可被跨 cluster 使用。
- `ServiceImport`：在消費端或 importing cluster 表示一個跨 cluster service 的存在。

這個 model 的價值是「只出口被明確宣告的 service」，而不是把所有 cluster network 攤平成一張大網。

```text
Cluster A                                      Cluster B
┌──────────────────────────┐                  ┌──────────────────────────┐
│ Service: payments        │                  │ ServiceImport: payments  │
│ ServiceExport: payments  │ ── controller ─▶ │ DNS / VIP / endpoints     │
└──────────────────────────┘                  └──────────────────────────┘
```

適合：

- 少數 platform services 需要跨 cluster 被發現。
- 想保留 per-cluster ownership，但需要標準化 discovery API。
- 希望未來可替換 provider/vendor implementation。

風險：

- Service discovery 不等於安全授權；仍需 mTLS、NetworkPolicy、authn/authz 或 service mesh policy。
- 跨 cluster latency / retry / timeout 需要 app owner 明確處理。
- 若資料層沒有跨區設計，不能把 `ServiceImport` 當成 HA guarantee。

---

## Pattern 3：provider-managed global ingress / fleet

GKE 的 Multi-cluster Services / Multi Cluster Ingress 與 Azure Kubernetes Fleet Manager 代表另一種方向：由 provider 或平台控制面管理多 cluster membership、service discovery、ingress 或 fleet governance。

```text
Provider / fleet control plane
       │
       ├── member cluster A
       ├── member cluster B
       └── member cluster C

User-facing traffic
       │
       ▼
Global LB / Multi-cluster ingress / exported service routing
```

適合：

- 團隊已經在該 provider 上，且可以接受 provider-specific API。
- 需要跨 region / cross-cluster ingress 的 managed path。
- 平台希望集中管理 member clusters、更新策略與 placement。

風險：

- Provider-managed 語意不能直接搬到 bare-metal on-prem。
- 故障模式會跨越 Kubernetes、cloud LB、DNS、IAM、networking 與 provider control plane。
- Vendor lock-in 與 migration path 需要事先記錄。

on-prem 轉譯時應問：

- 誰扮演 provider control plane？
- 誰負責 global VIP / DNS failover / health check？
- 誰批准 cluster 加入或退出 fleet？
- 如果 hub/fleet manager 掛掉，member cluster 是否仍可服務既有流量？

---

## Pattern 4：hub-and-spoke 管理與 cluster inventory

Open Cluster Management 的 `ManagedCluster` vocabulary 可以用來理解 hub-and-spoke 管理模型：hub 保存 cluster inventory，managed clusters 回報狀態並接受策略/工作負載管理。

```text
                 ┌──────────────┐
                 │ Hub / manager │
                 │ inventory     │
                 │ policy        │
                 └──────┬───────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
  managed A        managed B        managed C
```

適合：

- cluster 數量多，需要 inventory、policy、placement 與狀態聚合。
- 要把「操作多個 cluster」和「把所有 cluster 網路打通」分開處理。

風險：

- Hub 是管理面，不應被誤解成所有 data-plane traffic 必經路徑。
- Hub 權限很高，要明確隔離 credentials、審計與變更 approval。
- 不同工具的 resource propagation / placement / policy 語意不同，需要逐一驗證。

---

## On-prem / bare-metal 設計檢查表

### 1. Failure domain

- [ ] cluster 邊界代表什麼：rack / room / region / tenant / lifecycle ring？
- [ ] 一個 cluster API server 不可用時，是否影響其他 cluster 的資料面流量？
- [ ] shared component（DNS、registry、GitOps controller、observability backend）是否反而變成單點？

### 2. Version and addon skew

- [ ] 每個 cluster 的 Kubernetes minor / patch / CNI / CSI / ingress / LB 版本有 inventory。
- [ ] 多 cluster controller 支援哪些 Kubernetes 版本範圍有官方文件。
- [ ] 升級順序：先 hub/fleet manager 還是先 member clusters？是否可 rollback？

### 3. Networking and traffic

- [ ] Pod CIDR / Service CIDR 是否跨 cluster 唯一？若不唯一，哪些元件會受影響？
- [ ] 跨 cluster 呼叫走 DNS、global VIP、gateway、service mesh 或 exported service？
- [ ] BGP/L2/MetalLB/Cilium/Calico 與外部 router 的責任邊界清楚。
- [ ] NetworkPolicy / firewall / mTLS / authn/authz 不依賴單一不明確的「內網可信」。

### 4. Data and storage

- [ ] Stateful workload 是否真的需要 active-active？還是 active-passive / backup-restore 即可？
- [ ] PVC / snapshot / backup / restore 是否跨 cluster 可驗證？
- [ ] RTO/RPO 由資料層承諾，不由 Kubernetes Service discovery 承諾。

### 5. Operations

- [ ] 每個 cluster 的 owner、escalation、maintenance window 有記錄。
- [ ] GitOps / CI/CD 有 per-cluster rollout、canary、pause、rollback。
- [ ] Logs / metrics / traces / Events 可跨 cluster 查詢，但不暴露 secret、token、private URL 或個資。
- [ ] Incident template 明確標記影響 cluster、region、service、source of truth 與未驗證假設。

---

## 事故初判流程

```text
User impact reported
        │
        ▼
Is impact single cluster or multi-cluster?
        │
        ├── single cluster
        │     ├─ check local API server / nodes / CNI / Service / Ingress / LB
        │     └─ isolate from global failover unless evidence says otherwise
        │
        └── multi-cluster
              ├─ check shared DNS / global LB / gateway / registry / GitOps
              ├─ check recent fleet-wide rollout or policy push
              ├─ check exported/imported services and health endpoints
              └─ decide: fail over, freeze deploy, or isolate cluster
```

低風險 on-call 問句：

1. 受影響範圍是單一 namespace、單一 cluster、還是多 cluster？
2. 近期是否有 shared component 變更：DNS、LB、certificate、GitOps、policy、image registry？
3. 是否只有跨 cluster traffic 出問題，cluster-local traffic 正常？
4. failover 後資料層是否支援該模式？有沒有可能造成 split-brain？
5. 現在要保護的是使用者流量、資料一致性，還是升級/發布速度？

---

## Prompt / runbook pattern：用 AI 協助但不外洩內部資料

可以用 AI 協助整理公開文件與抽象 runbook，但不要貼入內部 IP、hostname、ticket、credential、private dashboard URL 或未公開事件細節。

```text
請把以下公開來源整理成 multi-cluster Kubernetes 設計檢查表。
限制：
- 使用繁體中文。
- 只引用公開來源 URL。
- 不推論任何特定公司內部架構。
- 用「待查」標記需要環境驗證的部分。
- 輸出包含：failure domain、networking、service discovery、data layer、operations、incident response。
```

---

## 不可直接套用的邊界

本文件不能替代：

- 內部安全審查、網段規劃、PKI / IAM 設計。
- 真實 DR drill、backup restore test、region failover test。
- Vendor support matrix 與版本相容性確認。
- 對特定 cloud provider 或 on-prem router/LB 的正式設計文件。

在任何實作前，至少要把以下項目以公開或內部正式文件驗證：

```text
Kubernetes version
CNI / CSI / ingress / service mesh version
Service CIDR / Pod CIDR policy
DNS ownership
Certificate and identity model
NetworkPolicy / firewall boundary
Global LB / BGP / L2 owner
Backup / restore / RTO / RPO
Audit / logging retention
Change approval and rollback path
```
