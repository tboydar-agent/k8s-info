# k8s-info

`k8s-info` 是一個 Kubernetes / Platform Engineering 中文知識庫，聚焦「大型地端運算平台」需要的技術地圖、版本追蹤、SRE 維運模式與 AI-assisted operations。

> 狀態：非官方整理。  
> 語言：繁體中文（zh-TW）。  
> 風格：優先使用 ASCII art / 純文字圖，避免 Mermaid 在 GitHub mobile 上過小。  
> 隱私：不保存非必要個資、私訊、內部系統細節或未公開資訊。

---

## 最新更新重點

- ★★★★☆ `2026-05-01` 建立 [TSMC-scale platform engineer 技術地圖](docs/tsmc-platform-engineer-map.md)：把 Container / VM / Bare Metal / OS / K8S / Network / Storage / Security / IaC / ServiceMesh / BGP / KVM 等能力整理成可追蹤方向。
- ★★★★☆ `2026-05-01` 建立 [Kubernetes version watch](docs/kubernetes-version-watch.md)：每日追蹤 Kubernetes upstream、CNCF/供應商支援週期與公開 TSMC/K8s 訊號；目前不宣稱已知 TSMC 內部 K8s 版本。
- ★★★☆☆ `2026-05-01` 建立 [tracking backlog](TRACKING.md)：把 version、SRE、capacity、incident、AI agent for ops 等後續研究拆成可執行項目。

---

## 這個 repo 要回答什麼問題？

```text
如果要支撐台灣最大等級的地端運算平台，
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
│   ├── tsmc-platform-engineer-map.md
│   └── kubernetes-version-watch.md
├── sources/
│   └── README.md
└── .github/
    └── workflows/
```

---

## 建議閱讀順序

```text
5 分鐘：
  README.md

20 分鐘：
  docs/tsmc-platform-engineer-map.md

每日追蹤：
  docs/kubernetes-version-watch.md
  TRACKING.md
```

---

## 核心研究主題

```text
P0 — Kubernetes version / support window
  - upstream Kubernetes release / patch / EOL
  - managed K8s provider support matrix as comparison signal
  - on-prem upgrade pressure and skew policy
  - public TSMC job / talk / conference / article signals

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

## TSMC 相關處理原則

這個 repo 可以整理公開資訊與通用技術能力，但不假設或揭露任何未公開的 TSMC 內部資訊。

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

本 repo 初始方向來自一則 TSMC IT Computing Platform Engineer/Manager 徵才方向描述，核心關鍵字包含：

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
