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
- [ ] 補 Linux node pressure debugging playbook
- [ ] 補 BGP + Kubernetes load balancing 導讀
- [ ] 補 CSI / storage failure mode 導讀
- [ ] 補 multi-cluster / multi-region 架構筆記

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
