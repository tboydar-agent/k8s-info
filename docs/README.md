# docs/

Kubernetes / Platform Engineering 技術文件（zh-TW）。

## 建議從這裡開始

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
