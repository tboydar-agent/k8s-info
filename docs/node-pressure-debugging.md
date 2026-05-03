# Linux node pressure debugging playbook

> 狀態：公開來源整理；非官方 runbook。
> 語言：繁體中文（zh-TW）。
> 適用範圍：Linux Kubernetes worker / control-plane node 的 `MemoryPressure`、`DiskPressure`、`PIDPressure` 與 kubelet node-pressure eviction 初步判讀。
> 邊界：不可直接套用到任何特定公司內部環境；實際門檻、OS baseline、container runtime、CNI/CSI 與監控命名需依叢集發行版與現場設定驗證。

---

## 1. 先把問題分成三層

```text
使用者看到的症狀
  │
  ├─ Pod 層：Pending / Evicted / OOMKilled / CrashLoopBackOff / probe failed
  │
  ├─ Node 層：Ready=False/Unknown, MemoryPressure, DiskPressure, PIDPressure
  │
  └─ OS/runtime 層：cgroup memory, filesystem/inode, process count, kubelet/runtime logs
```

這份筆記的重點是第二層與第三層：先用 Kubernetes API 判斷 node condition 與 eviction 訊號，再回到 Linux / kubelet / container runtime 找可驗證原因。

---

## 2. 官方來源摘要

| 來源 | 本筆記使用方式 |
|---|---|
| [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/) | kubelet 如何依 memory / disk / inode / PID 訊號主動終止 Pod |
| [Node Status](https://kubernetes.io/docs/reference/node/node-status/) | `Ready`、`MemoryPressure`、`DiskPressure`、`PIDPressure`、capacity / allocatable 與 nodeInfo 的判讀 |
| [Troubleshooting Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/) | 從 `kubectl get nodes`、`kubectl describe node`、node YAML 與 kubelet logs 開始排查 |
| [Resource Metrics Pipeline](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/) | `kubectl top` / Metrics API 只能提供基本 CPU、memory 視角，不等於完整 node pressure 根因 |
| [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) | requests / limits / cgroups / memory OOM 與 ephemeral-storage 的基礎語意 |
| [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) | node condition 轉成 taint 後對排程與既有 Pod 的影響 |

---

## 3. Node pressure 不是單一錯誤碼

Kubernetes `Node.status.conditions` 至少提供這幾個與 pressure 相關的 condition：

```text
MemoryPressure   node memory is low
DiskPressure     disk capacity is low
PIDPressure      too many processes on the node
Ready            node 是否健康且可接受 Pod；Unknown 常代表 kubelet 停止回報
```

kubelet 的 node-pressure eviction 會主動終止 Pod 以回收 node 資源。官方文件列出的主要 eviction signal 包含：

```text
memory.available
nodefs.available
nodefs.inodesFree
imagefs.available
imagefs.inodesFree
containerfs.available
containerfs.inodesFree
pid.available
```

重要差異：

```text
API-initiated eviction：通常由 controller / drain / eviction API 觸發
node-pressure eviction：kubelet 在 node 資源壓力下本地決策
```

對 SRE 的實務含義：看到 `Evicted` 不應只問「誰刪了 Pod」，也要問「kubelet 當時看到哪個 node signal 低於門檻」。

---

## 4. 10 分鐘初判流程

```text
Step 0: freeze scope
  - 只處理公開 / 通用資訊；不要把內部 hostname、IP、ticket、私訊貼進公開文件

Step 1: node list
  kubectl get nodes -o wide

Step 2: condition + taint
  kubectl describe node <node>
  kubectl get node <node> -o yaml

Step 3: pod blast radius
  kubectl get pods -A --field-selector spec.nodeName=<node> -o wide
  kubectl get events -A --sort-by=.lastTimestamp

Step 4: basic metrics, but do not over-trust them
  kubectl top node <node>
  kubectl top pod -A --containers

Step 5: node-local evidence
  journalctl -u kubelet --since "2 hours ago"
  crictl ps -a
  crictl stats
  df -h
  df -i
  free -m
  ps -eLf | wc -l
```

`kubectl top` 來自 Metrics API / metrics-server，官方定位是 autoscaling 所需的基本 CPU / memory pipeline；它不會完整回答 inode、imagefs、containerfs、PID、runtime log、kubelet threshold 等問題。

---

## 5. MemoryPressure 判讀

官方 node-pressure eviction 文件提醒：Linux 上 kubelet 的 `memory.available` 來自 cgroupfs 計算，不等同於直接看 `free -m`。kubelet 會排除它認為在壓力下可回收的 `inactive_file`。

```text
常見現象：
  - Node condition: MemoryPressure=True
  - Pod status/reason: Evicted 或 container OOMKilled
  - kubelet log: eviction manager / memory.available threshold

先驗證：
  - 是否有 BestEffort / Burstable Pod 遠超 request？
  - 是否有 DaemonSet / sidecar / log agent 無限制吃記憶體？
  - system-reserved / kube-reserved / eviction-hard 是否與 OS baseline 對齊？
  - 最近是否升級 kernel、container runtime、cgroup v2 或 kubelet config？
```

不要只用「node 還有 cache」判斷安全；kubelet 的 eviction 視角與一般 shell 工具可能不同。

---

## 6. DiskPressure / inode pressure 判讀

官方文件把 filesystem 訊號拆成 `nodefs`、`imagefs`、`containerfs`：

```text
nodefs       node 主要 filesystem；常涵蓋 /var/lib/kubelet、logs、local volumes、非 memory emptyDir
imagefs      container runtime image / read-only layer filesystem
containerfs  writable layer、logs、local disk volume、ephemeral storage 等分離 filesystem
```

排查順序：

```text
1. df -h：容量是否低於 kubelet threshold？
2. df -i：inode 是否耗盡？
3. container runtime：image / writable layer / dead container 是否堆積？
4. /var/log 與 Pod logs：是否有單一 workload 快速寫爆？
5. emptyDir / ephemeral-storage：是否有 request/limit 或 quota 缺口？
6. 是否使用分離 imagefs/containerfs；runtime 是否支援對應 metrics？
```

地端平台常見陷阱：把 root filesystem、container images、Pod logs、kubelet state 全放在同一顆磁碟，會讓 nodefs 壓力同時影響 system 與 workload。

---

## 7. PIDPressure 判讀

`PIDPressure=True` 表示 node 上 process 數量過多。官方 eviction signal 是：

```text
pid.available = node.stats.rlimit.maxpid - node.stats.rlimit.curproc
```

先查：

```text
- 是否有 workload fork bomb / zombie process？
- 是否有 hostPID Pod 或 privileged agent 擴大影響？
- kubelet / runtime / CNI plugin 是否反覆 spawn process？
- OS pid_max / cgroup pids limit / container runtime 設定是否有 baseline？
```

PIDPressure 的處理通常不能只刪 Pod；需要定位是哪個 workload 或 node agent 製造 process churn。

---

## 8. Eviction 與排程影響

node condition 可能對應到 taint，taint 會影響排程與既有 Pod。概念上：

```text
Condition=True / Ready=False / Ready=Unknown
      │
      ▼
control plane / kubelet 對 node 加上對應 taint
      │
      ├─ NoSchedule：新 Pod 通常不排上去，除非有 toleration
      └─ NoExecute：既有 Pod 也可能被驅逐，除非有 toleration / tolerationSeconds
```

注意：node-pressure eviction 下 kubelet 可能不尊重 PodDisruptionBudget；hard threshold 下也可能用 0 秒 grace period。因此高優先級系統 Pod、PDB、drain policy 不能完全取代 node resource headroom。

---

## 9. 建議收斂成 incident checklist

```text
[ ] A. 判斷 pressure 類型：Memory / Disk / inode / PID / Ready Unknown
[ ] B. 保存公開安全的證據摘要：condition、event reason、kubelet log 關鍵字、時間窗
[ ] C. 確認影響範圍：單 node、node pool、zone、cluster-wide
[ ] D. 找最大貢獻者：Pod、DaemonSet、runtime image、log、emptyDir、process tree
[ ] E. 先恢復服務：cordon / drain / delete offending Pod / clean image/log / 擴容
[ ] F. 再修 baseline：requests/limits、ephemeral-storage、system-reserved、disk layout、alert
[ ] G. 回寫 postmortem：只寫通用事實，不寫內部 IP、hostname、ticket、私訊或個資
```

---

## 10. 適用邊界與待查事項

適用：

- Linux node 的 kubelet pressure / eviction 初步排查。
- 大型地端或混合雲平台的通用 SRE runbook 草案。
- 把官方 Kubernetes behavior 轉成 on-call 可讀 checklist。

不適用 / 不應直接套用：

- 特定公司內部 node pool、hostname、IP、incident、版本與監控命名。
- 未驗證的 OS tuning、kernel parameter、runtime GC policy。
- 把 `kubectl top` 當成完整 pressure 根因分析。

待查：

- cgroup v1 / v2 在各發行版與 kubelet 版本上的 memory.available 差異。
- containerd / CRI-O 對 imagefs / containerfs metrics 與 GC 行為的實務差異。
- OpenShift / RKE2 / GKE / EKS / AKS 對 eviction threshold、node allocatable 與 system reservation 的額外預設。
- 如何把 node pressure evidence 轉成 Prometheus alert 與 postmortem template。

---

## 11. 最小 ASCII 心智圖

```text
NodePressure
├─ MemoryPressure
│  ├─ cgroup memory.available
│  ├─ OOMKilled vs Evicted
│  └─ requests/limits + reserved headroom
├─ DiskPressure
│  ├─ nodefs / imagefs / containerfs
│  ├─ capacity + inode
│  └─ logs / images / emptyDir / writable layers
├─ PIDPressure
│  ├─ pid.available
│  ├─ fork / zombie / agent churn
│  └─ pids limit baseline
└─ Ready Unknown / False
   ├─ kubelet heartbeat
   ├─ network/runtime/systemd
   └─ taints + pod eviction impact
```
