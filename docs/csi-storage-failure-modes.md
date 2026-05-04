# CSI / Kubernetes storage failure modes 導讀

> 狀態：非官方、source-grounded 維運筆記。
> 適用範圍：Kubernetes PersistentVolume / PVC / StorageClass / CSI driver / snapshot 的 Day 2 初判與升級前檢查。
> 邊界：不可直接套用到任何特定公司內部環境；實際 storage backend、CSI driver 版本、OS baseline、topology、備份政策與 RPO/RTO 必須依公開 vendor 文件與現場設定驗證。
> 隱私：本文不保存內部 cluster 名稱、hostname、IP、ticket、事件細節、個人聯絡方式、私訊或非公開資料。

---

## 1. 一句話摘要

Kubernetes storage 故障不要只看「PVC 有沒有 Bound」。更可靠的初判方式是把資料路徑拆成：

```text
Application Pod
  │
  ▼
PVC request ── StorageClass policy ── PV binding
  │                    │                 │
  │                    ▼                 ▼
  │              CSI controller      backend volume
  │                    │                 │
  ▼                    ▼                 ▼
kubelet ── CSI node plugin ── mount / filesystem / device
```

常見事故通常落在五個面向：

1. provisioning / binding：PVC 等不到合適 PV 或動態建立失敗。
2. attach / mount：VolumeAttachment、CSI node plugin、kubelet mount 流程卡住。
3. topology / scheduling：volume zone、node label、`WaitForFirstConsumer` 與 Pod placement 不一致。
4. expansion / snapshot：PVC resize、snapshot CRD / controller / driver capability 不完整。
5. reclaim / deletion：`Delete` / `Retain`、finalizer、backend volume 刪除邊界不清楚。

---

## 2. 來源與驗證基線

| 來源 | 本文使用方式 |
|---|---|
| [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) | PV/PVC lifecycle、binding、dynamic provisioning、reclaim、storage object protection |
| [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/) | `provisioner`、`reclaimPolicy`、`allowVolumeExpansion`、`volumeBindingMode`、`allowedTopologies` |
| [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) | volume 與 Pod lifecycle、ephemeral vs persistent 的邊界 |
| [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) | snapshot CRD、snapshot controller、`csi-snapshotter`、PVC source protection |
| [Kubernetes CSI developer docs](https://kubernetes-csi.github.io/docs/) | CSI driver 與 kubelet / control-plane 的互動模型 |
| [CSI sidecar containers](https://kubernetes-csi.github.io/docs/sidecar-containers.html) | standard sidecars 如何 watch Kubernetes API 並呼叫 CSI driver |
| [external-provisioner](https://kubernetes-csi.github.io/docs/external-provisioner.html) | PVC -> `CreateVolume` -> PV；delete reclaim policy -> `DeleteVolume` |
| [external-attacher](https://kubernetes-csi.github.io/docs/external-attacher.html) | `VolumeAttachment` -> `ControllerPublishVolume` / `ControllerUnpublishVolume` |
| [external-resizer](https://kubernetes-csi.github.io/docs/external-resizer.html) | PVC edit -> `ControllerExpandVolume` |
| [node-driver-registrar](https://kubernetes-csi.github.io/docs/node-driver-registrar.html) | CSI node plugin 如何向 kubelet 註冊 socket |

---

## 3. Failure mode 1：PVC Pending / provisioning 失敗

### Grounded facts

- PV 是 cluster-level storage resource；PVC 是 namespaced storage request。
- PVC binding 是一對一、exclusive；如果沒有符合條件的 PV，PVC 會維持 unbound，直到合適 PV 出現。
- 動態 provisioning 由 StorageClass 的 `provisioner` 驅動；CSI `external-provisioner` watch PVC，呼叫 CSI driver 的 `CreateVolume`，成功後建立 PV。
- PVC 若明確設定 `storageClassName: ""`，表示不要用 default StorageClass 做動態 provisioning。

### 初判 checklist

```text
[ ] PVC 是否 Pending？
[ ] PVC 的 namespace 是否與 Pod 相同？
[ ] storageClassName 是缺省、指定 class，還是明確 ""？
[ ] cluster 是否有且只有預期的 default StorageClass？
[ ] StorageClass.provisioner 是否對應到已部署的 CSI driver？
[ ] external-provisioner Pod 是否健康、leader election 是否正常？
[ ] CSI controller log 是否有 CreateVolume / permission / quota / parameter 錯誤？
[ ] backend 是否真的有足夠容量與對應 storage pool？
```

### 低風險解讀

PVC Pending 不是單一錯誤；它可能是「沒有 class」、「class 指到不存在的 provisioner」、「backend 容量不足」、「topology 無法滿足」或「controller sidecar 無法建立 PV」。不要直接刪 PVC 重建，除非已確認沒有 data-loss 風險。

---

## 4. Failure mode 2：Pod 卡在 attach / mount

### Grounded facts

- Kubernetes control-plane 不直接用 Unix socket 跟 CSI driver 溝通；需要控制面操作的 CSI driver 必須 watch Kubernetes API 並觸發 CSI operation。
- `external-attacher` watch `VolumeAttachment`，再呼叫 CSI endpoint 的 `ControllerPublishVolume` / `ControllerUnpublishVolume`。
- kubelet 直接對 node 上的 CSI driver 發出 `NodeStageVolume`、`NodePublishVolume` 等 mount/unmount 相關 calls。
- `node-driver-registrar` 會用 kubelet plugin registration mechanism，把 CSI driver socket 註冊給 kubelet。

### 初判 checklist

```text
[ ] Pod event 是否顯示 FailedAttachVolume / FailedMount？
[ ] VolumeAttachment 物件是否存在？是否卡在 attachError？
[ ] CSI controller Pod：external-attacher / driver container 是否健康？
[ ] 目標 node 上 CSI node plugin DaemonSet 是否 Running/Ready？
[ ] node-driver-registrar 是否成功註冊 driver？
[ ] kubelet log 是否有 NodeStageVolume / NodePublishVolume timeout？
[ ] backend 是否允許同一 volume attach 到目標 node？access mode 是否相容？
[ ] node OS 是否缺 device path、mount helper、filesystem tool 或 kernel module？
```

### ASCII 故障定位圖

```text
PVC Bound
  │
  ├─ attach failed?
  │    ├─ check VolumeAttachment
  │    ├─ check external-attacher
  │    └─ check backend attach state
  │
  └─ mount failed?
       ├─ check CSI node plugin DaemonSet
       ├─ check node-driver-registrar / kubelet plugin socket
       ├─ check kubelet events/logs
       └─ check OS mount/filesystem dependency
```

---

## 5. Failure mode 3：topology / zone / scheduling 不一致

### Grounded facts

StorageClass 可以設定：

- `volumeBindingMode`
- `allowedTopologies`
- provider-specific `parameters`

`WaitForFirstConsumer` 常用來延後 binding/provisioning，讓 Pod scheduling constraints、node topology 與 volume topology 一起被考慮。

### 初判 checklist

```text
[ ] StorageClass.volumeBindingMode 是 Immediate 還是 WaitForFirstConsumer？
[ ] StorageClass.allowedTopologies 是否限制 zone / rack / node pool？
[ ] Pod nodeSelector / affinity / topology spread 是否與 storage topology 衝突？
[ ] StatefulSet 是否在多 zone / 多 rack 中使用只支援單 zone attach 的 volume？
[ ] PV nodeAffinity 是否把 Pod 限在不存在或不可用的 node set？
[ ] scheduler event 是否顯示 volume node affinity conflict？
```

### 低風險解讀

如果 storage backend 有 zone / rack / failure-domain 邊界，`Immediate` binding 可能太早選定 storage 位置；`WaitForFirstConsumer` 能降低「PV 已建立但 Pod 排不到可用 node」的機率。不過這不是 universal fix，仍需確認 driver 與 backend 支援的 topology semantics。

---

## 6. Failure mode 4：resize / snapshot 能力落差

### Resize

StorageClass 的 `allowVolumeExpansion: true` 只代表 Kubernetes 允許使用者編輯 PVC 要求更大容量；CSI driver 還需要對應 volume expansion capability，並搭配 `external-resizer` watch PVC edit，再呼叫 `ControllerExpandVolume`。

```text
[ ] StorageClass.allowVolumeExpansion 是否為 true？
[ ] CSI driver 是否宣告 VolumeExpansion capability？
[ ] external-resizer 是否部署且健康？
[ ] PVC condition 是否顯示 resizing / filesystem resize pending？
[ ] 檔案系統是否需要 Pod restart 或 node-side expansion？
[ ] backend 是否真的支援線上擴容？
```

### Snapshot

VolumeSnapshot 不是 core API，而是 CRD；snapshot 支援只適用於 CSI drivers，且 driver 可能不支援 snapshot。Kubernetes snapshot 流程需要 snapshot controller、`csi-snapshotter` sidecar，以及通常由 distribution 安裝的 CRD / validating webhook。

```text
[ ] VolumeSnapshot / VolumeSnapshotContent / VolumeSnapshotClass CRD 是否已安裝？
[ ] snapshot controller 是否由 cluster distribution 管理且健康？
[ ] CSI driver 是否支援 snapshot？
[ ] csi-snapshotter sidecar 是否部署？
[ ] 正在 snapshot 的 source PVC 是否被保護、未被強制刪除？
[ ] snapshot 是否已驗證可 restore，而不只是建立成功？
```

---

## 7. Failure mode 5：reclaim / deletion / finalizer 邊界

### Grounded facts

- StorageClass `reclaimPolicy` 影響 dynamically provisioned PV 的回收行為；預設是 `Delete`。
- PV/PVC 有 storage object in-use protection；使用中的 PVC 或 bound PV 被刪除時，可能進入 `Terminating`，等待不再被使用或不再 bound。
- CSI `external-provisioner` 在 PVC 刪除且 PV reclaim policy 為 delete 時，會呼叫 `DeleteVolume`，成功後刪 Kubernetes PV object。

### 初判 checklist

```text
[ ] PV reclaimPolicy 是 Delete 還是 Retain？
[ ] PVC / PV 是否卡 finalizer？finalizer 是保護資料還是 controller 失效？
[ ] PVC 是否仍被 Pod object reference？
[ ] backend volume 是否已刪、保留，或處於 deleting 狀態？
[ ] 是否有 snapshot / clone / backup dependency 阻擋刪除？
[ ] Retain 流程是否有人工交接：誰清資料、誰重綁 PV、誰刪 backend volume？
```

### 危險動作提醒

不要把「移除 finalizer」當成第一個修復步驟。finalizer 可能正在保護 in-use volume 或等待 backend operation 完成；強制移除前必須先確認資料保護、備份、restore drill、backend state 與 owner approval。

---

## 8. On-call 快速分流流程

```text
Start: Pod cannot use persistent storage
  │
  ├─ PVC not Bound
  │    └─ provisioning / binding / StorageClass / quota / backend capacity
  │
  ├─ PVC Bound but Pod Pending
  │    └─ topology / PV nodeAffinity / access mode / scheduler event
  │
  ├─ Pod scheduled but container not starting
  │    └─ attach / mount / kubelet / CSI node plugin / OS dependency
  │
  ├─ running workload has IO error or stale data
  │    └─ backend health / filesystem / node path / application semantics
  │
  └─ delete / resize / snapshot stuck
       └─ sidecar controller / finalizer / driver capability / backend operation
```

---

## 9. 升級前 storage readiness 檢查

```text
[ ] Kubernetes minor 目標版本與 CSI driver / sidecar support matrix 是否相容？
[ ] CSI controller Deployment / node DaemonSet image tag 是否有明確 release notes？
[ ] external-provisioner / external-attacher / external-resizer / csi-snapshotter / node-driver-registrar 是否一起盤點？
[ ] StorageClass reclaimPolicy / volumeBindingMode / allowedTopologies 是否符合現行 workload？
[ ] StatefulSet / database / queue workload 是否有 restore drill，而不只是 snapshot policy？
[ ] PDB、drain policy、volume detach timeout、node reboot SOP 是否已在 staging 驗證？
[ ] monitoring 是否涵蓋 PVC Pending、VolumeAttachment error、CSI sidecar error、kubelet mount error？
[ ] 是否避免保存任何內部 hostname、IP、ticket、客戶資料或私訊到公開 repo？
```

---

## 10. 待查事項

- 各主流 CSI driver（例如 cloud block storage、Ceph/Rook、NFS、vSphere、local PV）對 Kubernetes 1.35/1.36 的 sidecar 建議版本。
- `VolumeAttributesClass` 與 driver-specific mutable attributes 的 production readiness 與風險邊界。
- Stateful workload restore drill 模板：從 snapshot / backup 到 application-level consistency 驗證。
- 多 zone / 多 rack / bare-metal storage topology 的公開案例與 KubeCon / SIG Storage 資料。

---

## 11. 不可直接套用的邊界

本文是公開資料整理，不代表任何特定環境的支援矩陣或操作授權。以下資訊必須在實際環境另外驗證：

- CSI driver vendor support matrix 與 image digest。
- storage backend firmware / OS / kernel / multipath / filesystem toolchain。
- RPO / RTO、備份保留、加密、權限與合規要求。
- drain / reboot / upgrade 的變更窗口與 rollback plan。
- 任何涉及資料刪除、finalizer 移除、volume detach/attach 強制操作的授權流程。
