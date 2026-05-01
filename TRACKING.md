# Tracking

> k8s-info 的研究 backlog 與每日追蹤狀態。  
> 原則：公開來源優先；無來源就標「待查」，不猜測 TSMC 內部資訊。

---

## Current status

```text
repo: k8s-info
created: 2026-05-01
focus: Kubernetes / Platform Engineering / TSMC-scale on-prem compute
public TSMC internal K8s version: unknown / not publicly verified
```

---

## Daily watch

| Topic | Query / Source | Status | Note |
|---|---|---|---|
| Kubernetes upstream release | https://kubernetes.io/releases/ | active | 追 minor / patch / EOL |
| Kubernetes patch releases | https://kubernetes.io/releases/patch-releases/ | active | 追 security/bugfix patch |
| Kubernetes GitHub releases | https://github.com/kubernetes/kubernetes/releases | active | 交叉驗證 |
| TSMC Kubernetes public signal | `TSMC Kubernetes version` | active | 目前未確認內部版本 |
| TSMC platform engineer signal | `TSMC Computing Platform Engineer Kubernetes` | active | 公開職缺/徵才訊號 |
| 台積電中文訊號 | `台積電 Kubernetes 版本` | active | 中文公開資料 |

---

## Research backlog

### P0 — version / upgrade baseline

- [ ] 建立 Kubernetes release / EOL 摘要頁
- [ ] 建立 version skew policy 中文導讀
- [ ] 建立 on-prem Kubernetes upgrade checklist
- [ ] 建立 provider support matrix 對照：EKS / AKS / GKE / OpenShift / Rancher
- [ ] 每日追蹤 TSMC + Kubernetes / version 公開訊號

### P1 — TSMC-scale platform engineer map

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
