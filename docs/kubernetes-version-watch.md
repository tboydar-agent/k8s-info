# Kubernetes version watch

> 狀態：每日追蹤入口。
> 目標：追蹤 Kubernetes upstream version / support window / patch release，以及大型地端平台 / platform engineering 相關公開訊號。
> 重要邊界：不追蹤、推論或揭露任何特定公司內部 Kubernetes 版本、內部架構或非公開資訊。

---

## 1) 為什麼要追 Kubernetes version？

大型地端平台的 K8s version 不是「升到最新版」這麼簡單，而是：

```text
upstream release cadence
  + security patch window
  + version skew policy
  + CNI / CSI / runtime compatibility
  + OS / kernel / driver constraints
  + app team migration cost
  + manufacturing 7x24 reliability pressure
```

---

## 2) 追蹤模型

```text
┌──────────────────────┐
│ Kubernetes upstream   │
│ releases / patches    │
└───────────┬──────────┘
            │
            ▼
┌──────────────────────┐
│ Provider matrices     │
│ EKS / AKS / GKE / RKE │
└───────────┬──────────┘
            │
            ▼
┌──────────────────────┐
│ On-prem implication   │
│ upgrade / EOL risk    │
└───────────┬──────────┘
            │
            ▼
┌──────────────────────┐
│ Public large-scale platform signals   │
│ jobs / talks / docs   │
└──────────────────────┘
```

---

## 3) 每日追蹤查詢

```text
large-scale Kubernetes version
large-scale container platform Kubernetes
large-scale Computing Platform Engineer Kubernetes
大型地端平台 Kubernetes 版本
大型地端平台 container platform Kubernetes
large enterprise Kubernetes careers
large enterprise Kubernetes public source
site:ithelp.ithome.com.tw Kubernetes 地端 平台
site:kubernetessummit.ithome.com.tw Kubernetes 地端 平台
```

---

## 4) 可信來源優先序

```text
A. 官方/一手來源
  - Kubernetes official releases
  - Kubernetes GitHub releases
  - large enterprise official careers pages
  - large-scale platform engineering / blog / conference pages

B. 高可信二手來源
  - CNCF / KubeCon / iThome Kubernetes Summit speaker pages
  - cloud provider official support matrix: EKS / AKS / GKE
  - vendor docs: Red Hat OpenShift / Rancher / VMware Tanzu

C. 只作線索
  - LinkedIn jobs
  - Glassdoor / Cake / job mirrors
  - forum posts / social posts
  - conference slides not hosted by organizer or speaker
```

---

## 5) 初始公開訊號（2026-05-01）

目前初步搜尋可見：

```text
有公開職缺/搜尋結果顯示大型組織與 Kubernetes / Platform Engineer / Principal Software Engineer (Kubernetes) 相關。

本 repo 不追蹤任何特定公司內部 Kubernetes 版本。
```

因此本 repo 目前只能追蹤：

```text
- Kubernetes upstream 版本與 EOL
- 供應商支援週期作為比較基準
- 大型地端平台公開職缺和公開技術訊號
- 對大型地端平台的通用 upgrade 風險分析
```

不能寫成：

```text
某特定公司目前使用 Kubernetes vX.Y
```

除非未來有可信公開來源。

---

## 6) 版本追蹤欄位

```text
date:
source:
source_type: official | provider | conference | job | secondary
signal:
version_mentioned:
confidence: high | medium | low
impact:
action:
```

範例：

```text
date: 2026-05-01
source: Kubernetes official releases
source_type: official
signal: upstream release / patch / support window
version_mentioned: <to be checked by daily job>
confidence: high
impact: affects upgrade planning baseline
action: update TRACKING.md if new minor/patch/EOL appears
```

---

## 7) Daily job 行為

每日一次：

```text
1. 檢查 Kubernetes upstream releases / patch releases
2. 搜尋 large-scale platform + Kubernetes / container platform / platform engineer 公開訊號
3. 檢查 provider/vendor support window：EKS / AKS / GKE / OpenShift / Rancher / Tanzu
4. 若有新可信資訊：
   - 更新 docs/kubernetes-version-watch.md 或新增 sources/YYYY-MM-DD.md
   - 更新 TRACKING.md
   - commit 到 k8s-info
   - 若變更可驗證且低風險，push 到 origin/main
5. 若沒有新資訊：
   - 回報 no material update
   - 不製造空 commit
```

---

## 8) 升級風險 checklist

```text
Before version bump:
  [ ] API deprecations
  [ ] kubelet / control-plane skew
  [ ] CNI compatibility
  [ ] CSI compatibility
  [ ] CRI / container runtime compatibility
  [ ] OS / kernel requirement
  [ ] admission controller / webhook compatibility
  [ ] observability stack compatibility
  [ ] backup / restore tested
  [ ] rollback path documented
  [ ] load / chaos test complete
```
