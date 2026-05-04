# Kubernetes provider support matrix watch

> 更新時間：2026-05-04
> 目的：把 upstream Kubernetes support window 與 managed / vendor Kubernetes support window 分開追蹤。
> 邊界：只整理公開官方文件；不追蹤、保存或推論任何特定公司內部 Kubernetes 版本或架構。

---

## 0) 為什麼要分開看？

```text
Kubernetes upstream:
  cares about project release branches and patch support.

Cloud provider / vendor:
  cares about product availability, support channel, rollout region, extension policy,
  OS image, addon, CNI/CSI/ingress/runtime integration.

大型平台真正要管的是：
  upstream EOL
    + provider/vendor support window
    + addon compatibility
    + node OS / kernel / runtime
    + rollout / rollback / freeze window
```

---

## 1) 2026-05-03 快照

```text
┌──────────────────────┐
│ Upstream Kubernetes   │
└───────────┬──────────┘
            │  latest-three maintenance baseline
            ▼
  1.36 / 1.35 / 1.34

┌──────────────────────┐
│ Patch train           │
└───────────┬──────────┘
            │  latest GitHub/changelog cross-check
            ▼
  1.36.0 / 1.35.4 / 1.34.7 / 1.33.11

┌──────────────────────┐
│ Provider / vendor     │
└───────────┬──────────┘
            │  product lifecycle can differ
            ▼
  EKS / AKS / GKE / OpenShift / Rancher / RKE2 / Tanzu
```

---

## 2) Upstream baseline

```text
Source:
  - https://kubernetes.io/releases/
  - https://kubernetes.io/releases/patch-releases/
  - https://github.com/kubernetes/kubernetes/releases

Observed on 2026-05-03:
  - Latest GitHub release observed: v1.36.0
  - Maintained upstream minor branches observed: 1.36, 1.35, 1.34
  - 1.36 release date: 2026-04-22
  - 1.36 EOL: 2027-06-28
  - Latest patch cross-check from GitHub/changelog:
      1.36.0 / 1.35.4 / 1.34.7 / 1.33.11
  - Next monthly patch target observed: 2026-05-12
  - Branch-specific next patch targets observed 2026-05-04:
      1.36.1 / 1.35.4 / 1.34.7
  - Caveat observed 2026-05-04:
      release-summary page rows may lag GitHub tags and raw changelogs
```

Implication:

```text
If a fleet still depends on 1.33 assumptions, treat it as an upgrade-readiness signal.
Do not infer any specific company is using 1.33; this is a generic planning warning only.
```

---

## 3) Provider / vendor watch

### EKS

```text
Source:
  https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html

Observed on 2026-05-02:
  standard support:
    - 1.35
    - 1.34
    - 1.33

  extended support:
    - 1.32
    - 1.31
    - 1.30

Lifecycle model observed:
  - 14 months standard support
  - then 12 months extended support

2026-05-03 re-check:
  - No material lifecycle-model change observed.
  - EKS still differs from upstream by product release and extended-support policy.
```

Planning note:

```text
EKS extended support is not the same as upstream latest-three maintenance.
It may be useful for risk buy-down, but should not become a reason to skip upgrade discipline.
```

### AKS

```text
Source:
  https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions?tabs=azure-cli

Observed on 2026-05-02:
  - AKS lifecycle table lists 1.33, 1.34, 1.35, 1.36.
  - Roadmap text observed for 1.36:
      upstream release: Apr 2026
      AKS preview: May 2026
      AKS GA: Jun 2026
      EOL: around Jun 2027
      LTS EOL: around Jun 2028
```

Planning note:

```text
AKS production planning should separate:
  upstream release date
  AKS GA date
  region availability
  platform support date
  node image / addon compatibility
```

### GKE

```text
Sources:
  - https://cloud.google.com/kubernetes-engine/docs/release-notes
  - https://cloud.google.com/kubernetes-engine/versioning

Observed on 2026-05-02:
  - GKE release notes contain 1.36.0-gke.* entries.
  - GKE release notes contain 1.35.3-gke.* recommended / auto-upgrade target entries.
  - GKE versioning docs describe:
      standard support: month 1 through month 14
      extended support: month 15 through month 24, where enabled / available

2026-05-04 re-check:
  - GKE versioning page last-updated marker observed: 2026-05-01 UTC.
  - Key planning rule unchanged: x.y.z-gke.N is provider-specific and channel/zone rollout matters.
```

Planning note:

```text
GKE versions roll out by release channel and region.
A release-note entry is a signal, not a substitute for checking the target channel/region.
```

### OpenShift

```text
Source:
  https://access.redhat.com/support/policy/updates/openshift

Observed on 2026-05-02:
  - Official lifecycle page available.
  - Use Red Hat lifecycle policy as the source of truth for OpenShift support windows.
  - OpenShift 4 lifecycle is phased by minor release; even-numbered minor releases are EUS releases.
```

Planning note:

```text
OpenShift maps Kubernetes into a broader platform lifecycle:
  Kubernetes + RHCOS + operators + CNI + ingress + registry + monitoring.
Do not map upstream Kubernetes EOL directly to OpenShift EOL without checking Red Hat docs.
```

### SUSE Rancher / RKE2

```text
Source:
  https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/

Observed on 2026-05-03:
  - Rancher Manager v2.14.1 certified Kubernetes platform range: v1.33 through v1.35.
  - RKE2 downstream support: 1.35 / 1.34 / 1.33.
  - K3s downstream support: 1.35 / 1.34 / 1.33.
```

Planning note:

```text
For on-prem fleets, track:
  Rancher Manager version
  RKE2/K3s Kubernetes version
  OS/kernel support
  CNI/CSI support
  air-gapped upgrade path
```

### VMware Tanzu

```text
Source family:
  Broadcom / VMware Tanzu official documentation

Observed on 2026-05-03:
  - Live Broadcom TKG v2.5.x release notes page observed.
  - TKG v2.5.4 is documented as the final enterprise release of TKG.
  - TKG v2.5.4 supports Kubernetes workload cluster versions v1.33.1 through v1.27.16.
  - Workload clusters cannot skip Kubernetes minor versions during upgrade.
```

Planning note:

```text
Tanzu / VKS planning must be treated as a vendor lifecycle and migration topic,
not as a direct upstream Kubernetes version-only decision.
```

---

## 4) Upgrade-readiness checklist for provider drift

```text
Before deciding target version:

  [ ] upstream version is still maintained
  [ ] provider/vendor supports target version in required region/channel
  [ ] provider/vendor supports current source version long enough for rollout
  [ ] node OS image lifecycle is compatible
  [ ] CNI / CSI / ingress / service mesh matrix is compatible
  [ ] autoscaler / GitOps / backup tooling matrix is compatible
  [ ] observability metric rename/deprecation diff is reviewed
  [ ] rollback version and data-path risks are documented
  [ ] freeze window and exception path are approved
```

---

## 5) Sources

```text
Kubernetes:
  https://kubernetes.io/releases/
  https://kubernetes.io/releases/patch-releases/
  https://kubernetes.io/releases/version-skew-policy/
  https://github.com/kubernetes/kubernetes/releases

Providers:
  https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html
  https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions?tabs=azure-cli
  https://cloud.google.com/kubernetes-engine/docs/release-notes
  https://cloud.google.com/kubernetes-engine/versioning

Vendors:
  https://access.redhat.com/support/policy/updates/openshift
  https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/
  https://techdocs.broadcom.com/us/en/vmware-tanzu/standalone-components/tanzu-kubernetes-grid/2-5/tkg/mgmt-release-notes.html
```
