# k8s-info

`k8s-info` 是一個 Kubernetes / Platform Engineering 中文知識庫，聚焦「大型地端運算平台」需要的技術地圖、版本追蹤、SRE 維運模式與 AI-assisted operations。

> 狀態：非官方整理。
> 語言：繁體中文（zh-TW）。
> 風格：優先使用 ASCII art / 純文字圖，避免 Mermaid 在 GitHub mobile 上過小。
> 隱私：不保存非必要個資、私訊、內部系統細節或未公開資訊。

---

## 最新更新重點

- ★★★★☆ `2026-05-13` 新增 [Multi-cluster / multi-region Kubernetes architecture 導讀](docs/multi-cluster-architecture.md)：整理 cluster-local baseline、MCS `ServiceExport` / `ServiceImport`、provider-managed fleet/global ingress 與 on-prem 設計檢查表。
- ★★★★☆ `2026-05-12` 新增 [Kubernetes observability / incident response 導讀](docs/kubernetes-observability-incident-response.md)：把 Events、logs、metrics、probes 與 SRE incident/postmortem 原則整理成 on-call 初判與安全 postmortem 模板。
- ★★★★☆ `2026-05-11` 新增 [BGP / Kubernetes LoadBalancer 導讀](docs/bgp-kubernetes-load-balancing.md)：整理 bare-metal Service LoadBalancer、L2 announcement、BGP advertisement、MetalLB / Cilium / Calico 選型與故障初判。
- ★★★★★ `2026-05-09` 新增 [Kubernetes 版本偏差實務案例研究](docs/kubernetes-version-skew-on-prem-case-studies.md)：大型地端平台常見的版本偏差陷阱與解決方案，包含 CNI/CSI 相容性、HA cluster 管理、多 node pool 升級等實務案例。
- ★★★★★ `2026-05-07` 新增 [On-prem Kubernetes 升級檢查表](docs/on-prem-upgrade-checklist.md)：完整 on-prem 升級流程、檢查表與實務案例。
- ★★★★☆ `2026-05-06` 新增 [Kubernetes 版本生命周期與升級策略](docs/kubernetes-version-lifecycle-and-upgrade-strategy.md)：整合版本支援週期、version skew policy、升級順序與大型地端平台特殊考量。
- ★★★★☆ `2026-05-04` 新增 [CSI / Kubernetes storage failure modes 導讀](docs/csi-storage-failure-modes.md)：把 PVC/PV/StorageClass/CSI sidecars/snapshot 的常見故障拆成 on-call 初判流程。
- ★★★★☆ `2026-05-03` 新增 [Linux node pressure debugging playbook](docs/node-pressure-debugging.md)：整理 MemoryPressure / DiskPressure / PIDPressure、kubelet eviction signals 與 on-call 初判流程。
- ★★★★★ `2026-05-02` 新增 [Kubernetes version skew 中文導讀](docs/kubernetes-version-skew.md)：把 upstream skew policy 轉成 on-prem control plane / node pool 升級檢查語言。
- ★★★★☆ `2026-05-02` 新增 [Kubernetes provider support matrix watch](docs/kubernetes-provider-support-matrix.md) 與 [每日來源摘要](sources/2026-05-02.md)：分開追蹤 upstream / EKS / AKS / GGE / OpenShift / Rancher / Tanzu 支援窗。
- ★★★★★ `2026-05-01` 新增 [Kubernetes release intelligence（近一年）](docs/kubernetes-release-intelligence-2025-2026.md)：用 1–5 顆星整理 v1.33–v1.36 對全球級資料中心 SRE / platform engineering 的影響。
- ★★★★☆ `2026-05-01` 建立 [大型地端平台工程師技術地圖](docs/large-scale-platform-engineer-map.md)：把 Container / VM / Bare Metal / OS / K8S / Network / Storage / Security / IaC / ServiceMesh / BGP / KVM 等能力整理成可追蹤方向。
- ★★★★☆ `2026-05-01` 建立 [Kubernetes version watch](docs/kubernetes-version-watch.md)：每日追蹤 Kubernetes upstream、CNCF/供應商支援週期與公開大型平台/K8s 訊號；不宣稱任何特定公司內部 K8s 版本。
- ★★★☆☆ `2026-05-01` 建立 [tracking backlog](TRACKING.md)：把 version、SRE、capacity、incident、AI agent for ops 等後續研究拆成可執行項目。

---

## 這個 repo 要回答什麼問題？

```text
如果要支撐全球級 / 大型地端運算平台，
Kubernetes / VM / Bare Metal / Storage / Network / Security / SRE
到底需要懂到多深？

如果每天只能追一點，
哪些 upstream version / support window / OSS signal 最值得追？
```

---

## 技術地圖

```text
                       ┌──────────────────────────────┐
                       │  Platform Engineering Mission │
                       │  7x24 stable on-prem compute  │
                       └───────────────┬──────────────┘
                                       │
       ┌───────────────────────────────┼───────────────────────────────┐
       │                               │                               │
       ▼                               ▼                               ▼
┌──────────────┐                ┌──────────────┐                ┌──────────────┐
│ Day 1 Build  │                │ Day 2 Ops    │                │ Evolution    │
│ provisioning │                │ SRE / IR     │                │ automation   │
│ bootstrap    │                │ reliability  │                │ capacity     │
└──────┬───────┘                └──────┬───────┘                └──────┬───────┘
       │                               │                               │
       ▼                               ▼                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Bare Metal → OS → KVM/VM → Container Runtime → Kubernetes → Service Platform │
└─────────────────────────────────────────────────────────────────────────────┘
       │                 │                  │                 │
       ▼                 ▼                  ▼                 ▼
  Network/BGP       Storage/CSI        Security/Policy     Observability
```

---

## 內容結構

```text
k8s-info/
├── README.md
├── TRACKING.md
├── docs/
│   ├── README.md
│   ├── large-scale-platform-engineer-map.md
│   ├── kubernetes-version-watch.md
│   ├── kubernetes-version-skew.md
│   ├── node-pressure-debugging.md
│   ├── csi-storage-failure-modes.md
│   ├── bgp-kubernetes-load-balancing.md
│   ├── kubernetes-observability-incident-response.md
│   ├── multi-cluster-architecture.md
│   ├── kubernetes-provider-support-matrix.md
│   ├── kubernetes-release-intelligence-2025-2026.md
│   └── kubernetes-version-lifecycle-and-upgrade-strategy.md
├── sources/
│   ├── README.md
│   └── 2026-05-02.md
└── .github/
    └── workflows/
```

---

## 建議閱讀順序

```text
5 分鐘：
  README.md

20 分鐘：
  docs/kubernetes-release-intelligence-2025-2026.md
  docs/node-pressure-debugging.md
  docs/csi-storage-failure-modes.md

30 分鐘：
  docs/large-scale-platform-engineer-map.md

每日追蹤：
  docs/kubernetes-version-watch.md
  docs/kubernetes-version-skew.md
  docs/kubernetes-provider-support-matrix.md
  TRACKING.md
```

---

## 核心研究主題

```text
P0 — Kubernetes version / support window
  - upstream Kubernetes release / patch / EOL
  - managed K8s provider support matrix as comparison signal
  - on-prem upgrade pressure and skew policy
  - public large-scale platform job / talk / conference / article signals

P1 — On-prem platform engineering
  - bare metal lifecycle
  - OS baseline and kernel/network tuning
  - KVM / VM platform
  - container runtime and image supply chain
  - multi-cluster / multi-region architecture

P2 — SRE / Day2 operations
  - observability
  - incident response
  - blameless postmortem
  - capacity planning
  - performance tuning
  - automation-first operations

P3 — AI-assisted platform ops
  - Claude / Copilot / Codex for infra engineering
  - AI agent runbooks
  - SRE skill packs
  - safe automation and approval gates
```

---

## 公開大型平台訊號處理原則

這個 repo 可以整理公開資訊與通用技術能力，但不假設或揭露任何未公開的 特定公司內部資訊。

```text
可以寫：
  - 公開徵才訊號
  - 公開演講 / 研討會 / 投影片
  - Kubernetes upstream version / support policy
  - 大型地端平台的一般性技術挑戰
  - 從公開需求反推的學習地圖

不要寫：
  - 未公開內部 Kubernetes 版本
  - 內部 cluster 拓撲 / IP / hostname / ticket / incident
  - 個人聯絡資料
  - 私訊或非公開討論內容
  - 未驗證傳聞
```

---

## 參考起點

本 repo 初始方向來自一則 大型地端運算平台工程師方向描述，核心關鍵字包含：

```text
Container / VM / Bare Metal / OS / K8S / Network / Storage / Compute
Security / IaC / ServiceMesh / BGP / KVM
Capacity Planning / Performance Tuning
Incident Response / Blameless Postmortem
Automation-first / No manual operations
AI tools / Agentic Workflow / SRE Platform Engineering
```

> 註：本 repo 不保存原文中的個人聯絡方式；若需要應徵或推薦，請以官方職缺頁或當事人公開管道為準。

---

*Maintained by tboydar-agent*
