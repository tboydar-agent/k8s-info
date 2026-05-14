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
- [x] 建立 Kubernetes 版本生命周期與升級策略：見 `docs/kubernetes-version-lifecycle-and-upgrade-strategy.md`
- [x] 建立 on-prem Kubernetes upgrade checklist：見 `docs/on-prem-upgrade-checklist.md`
- [x] 建立 provider support matrix 對照：EKS / AKS / GGE / OpenShift / Rancher / Tanzu：見 `docs/kubernetes-provider-support-matrix.md`
- [x] 建立 Kubernetes version skew 实务案例研究：見 `docs/kubernetes-version-skew-on-prem-case-studies.md`
- [ ] 每日追蹤 Kubernetes / large-scale platform public signals

### 2026-05-06 daily material update

```text
Upstream / patch train:
  - v1.36.0 officially released: 2026-04-22
  - v1.35.3 released: 2026-03-19
  - v1.34.6 released: 2026-03-19
  - v1.33.10 released: 2026-03-19
  - Next patch targets:
      v1.36.1 (not yet released)
      v1.35.4 (not yet released)
      v1.34.7 (not yet released)
  - v1.33 series entered maintenance mode: 2026-04-28

Provider / vendor:
  - EKS: standard 1.35 / 1.34 / 1.33; extended 1.32 / 1.31 / 1.30 (unchanged)
  - AKS: standard support 1.35 / 1.34 / 1.33; LTS 1.35 / 1.34 / 1.33 / 1.32 / 1.31 / 1.30 / 1.29 (unchanged)
  - GKE: standard/extended support model remains (unchanged)

Large-scale platform signal:
  - Platform engineering trends: Kubernetes as table stakes, focus on container runtimes (containerd, CRI-O), lightweight orchestration (K3s, Nomad), service mesh 2.0 (Ambient Mesh), GitOps as default, developer experience tools (Dapr, Crossplane, Backstage).
  - These are directional observations and do not constitute specific version updates.

Docs:
  - added sources/2026-05-06.md
  - updated docs/kubernetes-version-watch.md

Privacy:
  - No specific-company internal Kubernetes version, architecture, contact, private link, or non-public data recorded.

Note: Patch baselines have been corrected. The previous record of v1.35.4 / v1.34.7 / v1.33.11 has been updated to reflect actual release status.
```

Source note: `sources/2026-05-06.md`

### 2026-05-11 daily material update

```text
Upstream / patch train:
  - v1.36.0 remains latest upstream minor baseline.
  - Kubernetes official patch releases and GitHub releases now both verify:
      v1.35.4 / v1.34.7 / v1.33.11
  - v1.36.1 remains a next patch target / not observed as released in official rows.
  - This supersedes the 2026-05-06 caution that official page verification only showed v1.35.3 / v1.34.6 / v1.33.10.

Provider / vendor:
  - EKS: standard 1.35 / 1.34 / 1.33; extended 1.32 / 1.31 / 1.30.
  - AKS: 1.36 lifecycle row observed: upstream Apr 2026, preview May 2026, GA Jun 2026, standard EOL Jun 2027, platform support until 1.40 GA.
  - GKE: 1.36 release schedule milestones observed; page last-updated marker 2026-05-08 UTC.
  - OpenShift/Tanzu exact docs URLs need follow-up due HTTP 403/404 in automated fetch; no unsupported claim added.

Docs:
  - added sources/2026-05-11.md
  - updated docs/kubernetes-version-watch.md
  - updated TRACKING.md

Privacy:
  - No specific-company internal Kubernetes version, architecture, contact, private link, token, or non-public data recorded.
```

Source note: `sources/2026-05-11.md`

### 2026-05-12 daily material update

```text
Upstream / patch train:
  - No newer stable upstream release observed today.
  - v1.36.0 remains latest upstream minor baseline.
  - Released patch baseline remains v1.35.4 / v1.34.7 / v1.33.11.
  - Official patch page next-target rows now show:
      v1.36.1 / v1.35.5 / v1.34.8
  - GitHub release API did not show those next-target versions as published stable tags in the fetched page.

Provider / vendor:
  - EKS remains standard 1.35 / 1.34 / 1.33; extended 1.32 / 1.31 / 1.30.
  - AKS 1.36 roadmap remains preview May 2026, GA Jun 2026, standard EOL Jun 2027, platform support until 1.40 GA.
  - GKE 1.36 schedule remains visible; page last-updated marker remains 2026-05-08 UTC.
  - SUSE/Rancher support matrix was reachable and exposes RKE2/K3s entries including 1.35 / 1.34 / 1.33; no new exact certification claim added.

Docs:
  - added sources/2026-05-12.md
  - updated docs/kubernetes-version-watch.md
  - updated TRACKING.md

Privacy:
  - No specific-company internal Kubernetes version, architecture, contact, private link, token, or non-public data recorded.
```

Source note: `sources/2026-05-12.md`

### 2026-05-13 daily material update

```text
Upstream / patch train:
  - Kubernetes GitHub releases now show the 2026-05 patch train as published stable tags:
      v1.36.1 / v1.35.5 / v1.34.8 / v1.33.12
  - Raw upstream changelogs cross-check the same patch entries.
  - v1.36 remains newest upstream minor baseline.
  - Official release landing page still describes maintained branches as 1.36 / 1.35 / 1.34.
  - Official patch-release page did not surface explicit new patch rows in the fetched HTML snapshot; keep watching for row-level cut/date details.

Provider / vendor:
  - EKS remains standard 1.35 / 1.34 / 1.33; extended 1.32 / 1.31 / 1.30.
  - AKS 1.36 lifecycle remains preview May 2026, GA Jun 2026, standard EOL Jun 2027, platform support until 1.40 GA.
  - GKE release schedule page last-updated marker advanced to 2026-05-12 UTC.
  - SUSE/Rancher support matrix remained reachable; no new exact certification claim added.

Docs:
  - added sources/2026-05-13.md
  - updated docs/kubernetes-version-watch.md
  - updated TRACKING.md

Privacy:
  - No specific-company internal Kubernetes version, architecture, contact, private link, token, or non-public data recorded.
```

Source note: `sources/2026-05-13.md`

### 2026-05-14 daily material update

```text
Upstream / patch train:
  - Kubernetes official patch-release page now exposes May 2026 patch train rows:
      v1.36.1 / v1.35.5 / v1.34.8 / v1.33.12
  - Official row details observed:
      1.36.1  cherry-pick deadline 2026-05-08, target date 2026-05-13
      1.35.5  cherry-pick deadline 2026-05-08, target date 2026-05-12
      1.34.8  cherry-pick deadline 2026-05-08, target date 2026-05-12
      1.33.12 cherry-pick deadline 2026-05-08, target date 2026-05-12
  - Next official active-branch watch targets are now 1.36.2 / 1.35.6 / 1.34.9.
  - v1.36 remains newest upstream minor baseline; v1.33 remains near-EOL maintenance line with EOL 2026-06-28.

Provider / vendor:
  - EKS page remains reachable; no EKS 1.36 availability claim recorded.
  - AKS 1.36 lifecycle remains preview May 2026, GA Jun 2026, standard EOL Jun 2027, platform support until 1.40 GA.
  - GKE release schedule page last-updated marker advanced to 2026-05-13 UTC.
  - SUSE/Rancher support matrix remained reachable; no new exact certification claim added.

Docs:
  - added sources/2026-05-14.md
  - updated docs/kubernetes-version-watch.md
  - updated TRACKING.md

Privacy:
  - No specific-company internal Kubernetes version, architecture, contact, private link, token, or non-public data recorded.
```

Source note: `sources/2026-05-14.md`

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
- [x] 補 BGP + Kubernetes load balancing 導讀：見 `docs/bgp-kubernetes-load-balancing.md`
- [x] 補 CSI / storage failure mode 導讀：見 `docs/csi-storage-failure-modes.md`
- [x] 補 multi-cluster / multi-region 架構筆記：見 `docs/multi-cluster-architecture.md`

### 2026-05-13 deep-research contribution

```text
Topic:
  Multi-cluster / multi-region Kubernetes architecture

Docs:
  - added docs/multi-cluster-architecture.md
  - mapped Kubernetes cluster-local primitives to multi-cluster design boundaries
  - covered SIG Multicluster MCS ServiceExport / ServiceImport, provider-managed fleet/global ingress patterns, hub-and-spoke inventory, and on-prem safety checks
  - added incident triage flow and explicit non-applicability boundaries for internal topologies

Sources:
  - Kubernetes Cluster Networking / Service / Managing Workloads docs
  - SIG Multicluster Multicluster Services API, ServiceExport, ServiceImport docs
  - GKE Multi-cluster Services and Multi Cluster Ingress docs
  - Azure Kubernetes Fleet Manager concepts
  - Open Cluster Management ManagedCluster concept

Privacy:
  - no specific-company internal cluster topology, CIDR, hostname, IP, router config, credentials, personal contact, or non-public data recorded
```

Source note: `sources/2026-05-13.md`

### 2026-05-04 deep-research contribution

```text
Topic:
  CSI / Kubernetes storage failure modes

Docs:
  - added docs/csi-storage-failure-modes.md
  - mapped official PV/PVC/StorageClass/CSI sidecar/snapshot mechanics into a low-risk on-call triage flow
  - covered PVC Pending, attach/mount failures, topology conflicts, resize/snapshot capability gaps, and reclaim/finalizer boundaries

Sources:
  - Kubernetes Persistent Volumes
  - Kubernetes Storage Classes
  - Kubernetes Volumes
  - Kubernetes Volume Snapshots
  - Kubernetes CSI developer docs
  - Kubernetes CSI sidecar containers: external-provisioner, external-attacher, external-resizer, node-driver-registrar

Privacy:
  - no specific-company internal Kubernetes version, topology, hostname, IP, ticket, private link, personal contact, or non-public data recorded
```

Source note: `sources/2026-05-04.md`

### 2026-05-11 deep-research contribution

```text
Topic:
  BGP / Kubernetes LoadBalancer on bare metal

Docs:
  - added docs/bgp-kubernetes-load-balancing.md
  - mapped Kubernetes Service LoadBalancer boundaries to L2/BGP announcement choices
  - compared MetalLB, Cilium BGP/L2, and Calico service IP advertisement from official docs
  - added low-risk design checklist and first-pass troubleshooting flow

Sources:
  - Kubernetes Service docs
  - MetalLB concepts/configuration/advanced BGP configuration
  - Cilium BGP Control Plane and L2 Announcements docs
  - Calico BGP peering and service IP advertisement docs

Privacy:
  - no specific-company internal network topology, IP ranges, router configs, credentials, personal contact, or non-public data recorded
```

Source note: `sources/2026-05-11.md`

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

- [x] 建立 observability / incident response 導讀與 incident note / postmortem template：見 `docs/kubernetes-observability-incident-response.md`
- [x] 建立 blameless postmortem template：見 `docs/kubernetes-observability-incident-response.md`
- [ ] 建立 capacity planning 研究筆記
- [ ] 建立 performance tuning checklist
- [ ] 建立 7x24 on-call readiness checklist

### 2026-05-12 deep-research contribution

```text
Topic:
  Kubernetes observability / incident response

Docs:
  - added docs/kubernetes-observability-incident-response.md
  - mapped Kubernetes Events, logs, resource metrics, probes, and object status into a low-risk on-call triage flow
  - added a public-safe incident note / blameless postmortem template
  - updated README.md, docs/README.md, sources/2026-05-12.md, and TRACKING.md

Sources:
  - Kubernetes Debug Services
  - Kubernetes Resource metrics pipeline
  - Kubernetes Logging Architecture
  - Kubernetes Event API reference
  - Kubernetes liveness/readiness/startup probes
  - Kubernetes Nodes
  - Google SRE Book: Managing Incidents
  - Google SRE Book: Postmortem Culture

Privacy:
  - no specific-company internal Kubernetes version, topology, hostname, IP, ticket, private dashboard, chat message, credential, personal contact, or non-public incident detail recorded
```

Source note: `sources/2026-05-12.md`

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
