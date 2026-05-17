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

## 5.1) 2026-05-03 material update

```text
Upstream / patch train:
## 5.1) 2026-05-03 material update

```text
Upstream / patch train:
  - Kubernetes v1.36.0 remains latest upstream release.
  - GitHub/changelog cross-check records patch baselines:
      v1.35.4 / v1.34.7 / v1.33.11
  - next monthly patch target observed: 2026-05-12

Provider / vendor:
  - EKS: standard 1.35 / 1.34 / 1.33; extended 1.32 / 1.31 / 1.30
  - AKS: lifecycle table includes 1.36 preview May 2026, GA Jun 2026, EOL Jun 2027, LTS EOL Jun 2028
  - GKE: standard/extended support model confirmed; x.y.z-gke.N rollout remains channel/zone-specific
  - Rancher v2.14.1: certified v1.33 through v1.35
  - Tanzu TKG v2.5.4: documented as final enterprise TKG release; Kubernetes workload cluster versions v1.33.1 through v1.27.16

Privacy:
  - no specific-company internal Kubernetes version, architecture, contact, private link, or non-public data recorded
```

## 5.2) 2026-05-04 material update

```text
Upstream / patch train:
  - no newer stable Kubernetes release observed after v1.36.0
  - maintained upstream minor branches remain 1.36 / 1.35 / 1.34
  - GitHub/changelog patch baseline remains:
      v1.36.0 / v1.35.4 / v1.34.7 / v1.33.11
  - official patch page next target remains 2026-05-12
  - branch-specific next patch targets observed:
      v1.36.1 / v1.35.4 / v1.34.7
  - release-summary page may lag patch rows; cross-check GitHub tags and raw changelogs

v1.36 upgrade-action themes:
  - metric renames: volume_operation_total_errors, etcd_bookmark_counts
  - scheduler PreBind plugin API / parallel execution changes
  - DRA ResourceClaim granular status authorization RBAC
  - kubeadm FlexVolume support removal
  - git-repo volume plugin disabled and in-tree Portworx plugin removed

Provider / vendor:
  - EKS: standard 1.35 / 1.34 / 1.33; extended 1.32 / 1.31 / 1.30
  - AKS: 1.36 roadmap remains preview May 2026, GA Jun 2026, standard EOL Jun 2027, LTS EOL Jun 2028
  - GKE: versioning page last-updated marker observed as 2026-05-01 UTC
  - Rancher v2.14.1: certified v1.33 through v1.35 remains visible baseline
  - Tanzu TKG v2.5.x: release notes last-updated marker observed as 2026-04-30

Large-scale platform signal:
  - CNCF 2026-04-30 article frames AI-era sandboxing as a Kubernetes security/isolation design problem
  - CNCF KubeCon EU recap reinforces platform engineering as socio-technical work: self-service APIs, onboarding, feedback loops, and inclusive design

Privacy:
  - no specific-company internal Kubernetes version, architecture, contact, private link, or non-public data recorded
```

## 5.3) 2026-05-06 material update

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

Privacy:
  - No specific-company internal Kubernetes version, architecture, contact, private link, or non-public data recorded.

Note: Patch baselines have been corrected. The previous record of v1.35.4 / v1.34.7 / v1.33.11 has been updated to reflect actual release status.
```

Source note: `../sources/2026-05-06.md`

## 5.4) 2026-05-11 material update

```text
Upstream / patch train:
  - v1.36.0 remains latest upstream minor baseline.
  - Official Kubernetes patch-release page and GitHub releases now both verify:
      v1.35.4 / v1.34.7 / v1.33.11
  - Official patch rows show release cut 2026-04-10 and release date 2026-04-14 for those patches.
  - GitHub release publication timestamps are 2026-04-15 UTC.
  - v1.36.1 remains a next patch target / not observed as released in official rows.

Provider / vendor:
  - EKS remains standard 1.35 / 1.34 / 1.33; extended 1.32 / 1.31 / 1.30.
  - AKS lifecycle row includes 1.36: upstream Apr 2026, preview May 2026,
    GA Jun 2026, standard EOL Jun 2027, platform support until 1.40 GA.
  - GKE release schedule includes 1.36 milestones and page last-updated marker 2026-05-08 UTC.
  - OpenShift/Tanzu exact docs URLs need follow-up because automated checks hit HTTP 403/404; no new vendor support claim recorded from those failed fetches.

Large-scale platform signal:
  - no new material version-specific large-scale platform signal recorded today.
  - operational takeaway: keep upstream patch baseline, provider support windows, and internal upgrade sequencing separate.

Privacy:
  - no specific-company internal Kubernetes version, architecture, contact, private link, token, or non-public data recorded.
```

Source note: `../sources/2026-05-11.md`

## 5.5) 2026-05-12 material update

```text
Upstream / patch train:
  - No newer stable upstream release observed today.
  - v1.36.0 remains latest upstream minor baseline.
  - Released patch baseline remains:
      v1.35.4 / v1.34.7 / v1.33.11
  - Official patch page now exposes next active-branch targets:
      v1.36.1 / v1.35.5 / v1.34.8
  - GitHub release API did not show those next targets as published stable tags in the fetched page.

Provider / vendor:
  - EKS remains standard 1.35 / 1.34 / 1.33; extended 1.32 / 1.31 / 1.30.
  - AKS 1.36 lifecycle row remains visible: preview May 2026, GA Jun 2026,
    standard EOL Jun 2027, platform support until 1.40 GA.
  - GKE 1.36 release schedule milestones remain visible; page last-updated marker 2026-05-08 UTC.
  - SUSE/Rancher matrix was reachable and exposes RKE2/K3s entries including 1.35 / 1.34 / 1.33;
    no new exact Rancher Manager certification claim was added without per-row validation.

Large-scale platform signal:
  - no new material version-specific large-scale platform signal recorded today.
  - operational takeaway: released tags are the patch baseline; next-target rows are watch items.

Privacy:
  - no specific-company internal Kubernetes version, architecture, contact, private link, token, or non-public data recorded.
```

Source note: `../sources/2026-05-12.md`

## 5.6) 2026-05-13 material update

```text
Upstream / patch train:
  - Kubernetes GitHub releases now show the 2026-05 patch train as published stable tags:
      v1.36.1 / v1.35.5 / v1.34.8 / v1.33.12
  - Raw upstream changelogs also expose matching entries:
      CHANGELOG-1.36.md -> v1.36.1
      CHANGELOG-1.35.md -> v1.35.5
      CHANGELOG-1.34.md -> v1.34.8
      CHANGELOG-1.33.md -> v1.33.12
  - v1.36 remains newest upstream minor baseline.
  - Official release landing page still describes maintained minor branches as 1.36 / 1.35 / 1.34.
  - Official patch-release page did not surface explicit new patch rows in the fetched HTML snapshot; keep watching for row-level cut/date details.

Provider / vendor:
  - EKS lifecycle observation remains standard 1.35 / 1.34 / 1.33; extended 1.32 / 1.31 / 1.30.
  - AKS 1.36 lifecycle row remains preview May 2026, GA Jun 2026, standard EOL Jun 2027, platform support until 1.40 GA.
  - GKE release schedule page last-updated marker advanced to 2026-05-12 UTC.
  - SUSE/Rancher support matrix remained reachable; no new exact certification claim added.

Large-scale platform signal:
  - no new material version-specific large-scale platform signal recorded today.
  - operational takeaway: upstream patch tags, provider support windows, and internal rollout sequencing are separate control points.

Privacy:
  - no specific-company internal Kubernetes version, architecture, contact, private link, token, or non-public data recorded.
```

Source note: `../sources/2026-05-13.md`

## 5.7) 2026-05-14 material update

```text
Upstream / patch train:
  - Kubernetes official patch-release page now exposes May 2026 patch rows:
      v1.36.1 / v1.35.5 / v1.34.8 / v1.33.12
  - Official row details observed:
      1.36.1  cherry-pick deadline 2026-05-08, target date 2026-05-13
      1.35.5  cherry-pick deadline 2026-05-08, target date 2026-05-12
      1.34.8  cherry-pick deadline 2026-05-08, target date 2026-05-12
      1.33.12 cherry-pick deadline 2026-05-08, target date 2026-05-12
  - Next official active-branch watch targets are now:
      1.36.2 / 1.35.6 / 1.34.9
  - v1.36 remains newest upstream minor baseline.
  - v1.33 remains a maintenance / near-EOL line; official EOL date remains 2026-06-28.

Provider / vendor:
  - EKS lifecycle page remains reachable; support model remains standard support followed by extended support.
  - AKS 1.36 lifecycle row remains preview May 2026, GA Jun 2026,
    standard EOL Jun 2027, platform support until 1.40 GA.
  - GKE release schedule page last-updated marker advanced to 2026-05-13 UTC.
  - SUSE/Rancher support matrix remained reachable; no new exact certification claim added.

Large-scale platform signal:
  - no new material version-specific large-scale platform signal recorded today.
  - operational takeaway: the May patch train is now confirmed by official patch rows,
    GitHub releases, and raw changelogs; internal rollout sequencing is not inferred.

Privacy:
  - no specific-company internal Kubernetes version, architecture, contact, private link, token, or non-public data recorded.
```

Source note: `../sources/2026-05-14.md`

## 5.8) 2026-05-16 material update

```text
Upstream / patch train:
  - Kubernetes official releases landing page now lists the May patch train
    as latest releases for active maintained lines:
      1.36 latest: 1.36.1, released 2026-05-13
      1.35 latest: 1.35.5, released 2026-05-12
      1.34 latest: 1.34.8, released 2026-05-12
  - GitHub releases and official patch-release page remain aligned:
      v1.36.1 / v1.35.5 / v1.34.8 / v1.33.12
  - Next active-branch patch watch targets remain:
      1.36.2 / 1.35.6 / 1.34.9
  - v1.36 remains newest upstream minor baseline.
  - v1.33 remains a maintenance / near-EOL line; official EOL date remains 2026-06-28.

Provider / vendor:
  - EKS lifecycle page remained reachable; no EKS 1.36 availability claim recorded.
  - AKS 1.36 lifecycle row remains preview May 2026, GA Jun 2026,
    standard EOL Jun 2027, platform support until 1.40 GA.
  - GKE release schedule page last-updated marker advanced to 2026-05-14 UTC.
  - SUSE/Rancher matrix remained reachable; no new exact certification claim added.
  - Red Hat OpenShift lifecycle page was reachable; no new Kubernetes-version mapping claim added.

Large-scale platform signal:
  - CNCF blog front page exposed a 2026-05-13 public platform-engineering post
    about scaling from GitOps toward intent-based developer experience.
  - Treat this as a direction signal for platform UX / self-service abstraction,
    not as any organization-specific Kubernetes version disclosure.

Privacy:
  - no specific-company internal Kubernetes version, architecture, contact,
    private link, token, or non-public data recorded.
```

Source note: `../sources/2026-05-16.md`

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
