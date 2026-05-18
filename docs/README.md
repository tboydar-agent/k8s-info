# docs/

Kubernetes / Platform Engineering 技術文件（zh-TW）。

## 建議從這裡開始

- [Kubernetes version skew 中文導讀](kubernetes-version-skew.md)：把 upstream skew policy 轉成 on-prem control plane / node pool 升級檢查語言。
- [Linux node pressure debugging playbook](node-pressure-debugging.md)：整理 MemoryPressure / DiskPressure / PIDPressure、kubelet eviction signals 與 on-call 初判流程。
- [CSI / Kubernetes storage failure modes 導讀](csi-storage-failure-modes.md)：把 PVC/PV/StorageClass/CSI sidecars/snapshot 的常見故障拆成 on-call 初判流程。
- [BGP / Kubernetes LoadBalancer 導讀](bgp-kubernetes-load-balancing.md)：整理 bare-metal `Service type=LoadBalancer`、L2 announcement、BGP advertisement、MetalLB / Cilium / Calico 選型與 Day-2 檢查。
- [Kubernetes observability / incident response 導讀](kubernetes-observability-incident-response.md)：把 Events、logs、metrics、probes 與 SRE incident/postmortem 原則整理成 on-call 初判與安全 postmortem 模板。
- [Multi-cluster / multi-region Kubernetes architecture 導讀](multi-cluster-architecture.md)：整理 cluster-local baseline、MCS `ServiceExport` / `ServiceImport`、provider-managed fleet/global ingress、hub-and-spoke inventory 與 on-prem 檢查表。
- [AI-assisted SRE / platform engineering workflow 安全導讀](ai-assisted-sre-workflows.md)：把 Claude Code、GitHub Copilot coding agent、OpenAI Codex 類工具放進研究、IaC review、sanitized incident summary、runbook tabletop 的安全流程。
- [Kubernetes capacity planning / performance tuning 導讀](kubernetes-capacity-planning-performance-tuning.md)：把 requests/limits、node allocatable、quota/LimitRange、HPA、node pressure 與 control-plane metrics 整理成低風險容量規劃流程。
- [Kubernetes release intelligence（近一年，2025-04 到 2026-04）](kubernetes-release-intelligence-2025-2026.md)：用 1–5 顆星整理近一年 Kubernetes release 對全球級資料中心 SRE / platform engineering 的影響。
- [大型地端平台工程師技術地圖](large-scale-platform-engineer-map.md)：把大型地端運算平台所需能力拆成可學、可追、可驗證的領域。
- [Kubernetes version watch](kubernetes-version-watch.md)：追蹤 Kubernetes upstream version、support window、供應商支援矩陣，以及公開大型平台/K8s 訊號。
- [Kubernetes provider support matrix watch](kubernetes-provider-support-matrix.md)：分開追蹤 upstream 與 EKS / AKS / GKE / OpenShift / Rancher / Tanzu 等 provider/vendor support window。

## 寫作原則

```text
優先：ASCII art / 純文字圖
避免：未驗證內部資訊、個資、私訊、憑空猜測
標示：公開來源 / 待查 / 推論 / 通用建議
```

## 主題範圍

- Kubernetes release / patch / EOL / version skew
- Bare Metal / OS / KVM / Container runtime
- Network / BGP / ServiceMesh / Load balancing
- Storage / CSI / data path
- Security / supply chain / policy / secret handling
- IaC / automation / GitOps
- Observability / incident response / blameless postmortem
- Capacity planning / performance tuning
- AI agent / coding assistant for SRE and platform engineering
