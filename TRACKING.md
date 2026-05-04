# Tracking

> k8s-info 的研究 backlog 與每日追蹤狀態。
> 原則：公開來源優先；無來源就標「待查」，不猜測任何特定公司內部資訊。

---

## Current status

```text
repo: k8s-info
created: 2026-05-01
focus: Kubernetes / Platform Engineering / large-scale on-prem compute
public specific-company internal K8s version: not tracked / not inferred
```

---

## Daily watch

| Topic | Query / Source | Status | Note |
|---|---|---|---|
| Kubernetes upstream release | https://kubernetes.io/releases/ | active | 追 minor / patch / EOL |
| Kubernetes patch releases | https://kubernetes.io/releases/patch-releases/ | active | 追 security/bugfix patch |
| Kubernetes GitHub releases | https://github.com/kubernetes/kubernetes/releases | active | 交叉驗證 |
| Large-scale Kubernetes public signal | `large-scale Kubernetes version` | active | 目前未確認內部版本 |
| Platform engineer public signal | `large-scale Computing Platform Engineer Kubernetes` | active | 公開職缺/徵才訊號 |
| 大型平台中文訊號 | `大型地端平台 Kubernetes 版本` | active | 中文公開資料 |

## Scheduled maintainers

```text
08:00 Asia/Taipei
  k8s-info-platform-version-daily
  focus: Kubernetes version / large-scale platform public signal tracking

08:30 Asia/Taipei
  k8s-info-deep-research-contributor-daily
  focus: broad/deep Kubernetes + platform engineering research contribution
```

---

## Research backlog

### P0 — version / upgrade baseline

- [x] 建立 Kubernetes release / EOL 摘要頁：見 `docs/kubernetes-release-intelligence-2025-2026.md`
- [x] 建立 version skew policy 中文導讀：見 `docs/kubernetes-version-skew.md`
- [ ] 建立 on-prem Kubernetes upgrade checklist
- [x] 建立 provider support matrix 對照：EKS / AKS / GKE / OpenShift / Rancher：見 `docs/kubernetes-provider-support-matrix.md`
- [ ] 每日追蹤 Kubernetes / large-scale platform public signals

### 2026-05-04 daily material update

```text
Upstream / patch train:
  - no newer stable Kubernetes release observed after v1.36.0
  - recorded branch-specific next patch targets from official patch page:
      v1.36.1 / v1.35.4 / v1.34.7
  - next monthly patch target remains 2026-05-12; cherry-pick deadline remains 2026-05-08
  - recorded release-summary-page lag caveat: use GitHub tags and raw changelogs as cross-checks

v1.36 upgrade-action themes:
  - metric rename checks for dashboards / alerts
  - scheduler plugin API change checks
  - DRA RBAC checks
  - legacy FlexVolume / git-repo / in-tree storage checks

Provider/vendor:
  - refreshed EKS / AKS / GKE / Rancher / OpenShift / Tanzu lifecycle notes from official docs
  - observed GKE versioning page last-updated marker: 2026-05-01 UTC
  - observed TKG v2.5.x release notes last-updated marker: 2026-04-30

Large-scale platform signal:
  - added CNCF AI sandboxing / workload-isolation signal
  - added CNCF KubeCon EU platform-engineering socio-technical signal

Docs:
  - added sources/2026-05-04.md
  - updated docs/kubernetes-version-watch.md

Privacy:
  - no specific-company internal Kubernetes version, architecture, contact, private link, or non-public data recorded
```

Source note: `sources/2026-05-04.md`

### 2026-05-03 daily material update

```text
Upstream / patch train:
  - recorded Kubernetes v1.36.0 as current latest release baseline
  - cross-checked v1.35.4 / v1.34.7 / v1.33.11 from GitHub releases and changelogs
  - recorded next monthly patch target: 2026-05-12

Provider/vendor:
  - refreshed EKS / AKS / GKE lifecycle notes
  - updated Rancher Manager v2.14.1 certified range to v1.33 through v1.35
  - replaced stale Tanzu uncertainty with live Broadcom TKG v2.5.x release-note evidence

Docs:
  - added sources/2026-05-03.md
  - updated docs/kubernetes-version-watch.md
  - updated docs/kubernetes-provider-support-matrix.md

Privacy:
  - no specific-company internal Kubernetes version, architecture, contact, private link, or non-public data recorded
```

Source note: `sources/2026-05-03.md`

### 2026-05-02 daily material update

```text
Additional deep-research contribution:
  - added docs/kubernetes-version-skew.md
  - translated upstream version skew policy into on-prem upgrade planning language
  - captured HA API server skew caveat, kubelet/kube-proxy -3 minor window, kubectl ±1 minor window, and control-plane-first upgrade order
  - added a 12-point on-prem pre-upgrade checklist with explicit provider/vendor/addon follow-ups

Upstream:
  - observed v1.36.0 as latest GitHub release
  - observed maintained upstream minors: 1.36 / 1.35 / 1.34
  - recorded 1.36 EOL 2027-06-28 and patch train status from official pages

Provider/vendor:
  - added provider support matrix watch for EKS / AKS / GKE / OpenShift / Rancher / Tanzu
  - separated upstream EOL from provider/vendor support windows

Privacy:
  - no specific-company internal Kubernetes version, architecture, contact, or non-public data recorded
```

Source note: `sources/2026-05-02.md`

### P1 — large-scale platform engineer map

- [x] 建立初版技術地圖：Bare Metal / OS / KVM / Container / K8S / Network / Storage / Security / IaC / ServiceMesh / BGP
- [x] 補 Linux node pressure debugging playbook：見 `docs/node-pressure-debugging.md`
- [ ] 補 BGP + Kubernetes load balancing 導讀
- [ ] 補 CSI / storage failure mode 導讀
- [ ] 補 multi-cluster / multi-region 架構筆記

### 2026-05-03 deep-research contribution

```text
Topic:
  Linux node pressure debugging

Docs:
  - added docs/node-pressure-debugging.md
  - mapped official node-pressure eviction signals to an on-call first-pass checklist
  - covered MemoryPressure, DiskPressure/inode, PIDPressure, taints, basic metrics limits, and privacy boundaries

Sources:
  - Kubernetes Node-pressure Eviction
  - Kubernetes Node Status
  - Kubernetes Troubleshooting Clusters
  - Kubernetes Resource Metrics Pipeline
  - Kubernetes Resource Management for Pods and Containers

Privacy:
  - no specific-company internal Kubernetes version, topology, hostname, IP, ticket, private link, personal contact, or non-public data recorded
```

Source note: `sources/2026-05-03.md`

### P2 — SRE / Day2 operations

- [ ] 建立 incident response template
- [ ] 建立 blameless postmortem template
- [ ] 建立 capacity planning 研究筆記
- [ ] 建立 performance tuning checklist
- [ ] 建立 7x24 on-call readiness checklist

### P3 — AI-assisted platform operations

- [ ] 建立 Claude / Copilot / Codex 用於 IaC review 的安全流程
- [ ] 建立 AI agent runbook review pattern
- [ ] 建立 SRE skill pack 草案
- [ ] 建立 AI 輔助 incident summary / postmortem flow

---

## Update policy

```text
有新公開可信來源：
  - 更新對應 docs 或 sources/YYYY-MM-DD.md
  - 更新本 TRACKING.md
  - commit / push

沒有新可信來源：
  - 不製造空 commit
  - 只在 cron 回報 no material update
```
