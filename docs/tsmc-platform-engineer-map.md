# TSMC-scale platform engineer 技術地圖

> 狀態：初版研究地圖。  
> 來源：使用者提供的 TSMC IT Computing Platform Engineer/Manager 徵才方向 + 公開 Kubernetes / Platform Engineering 通用知識。  
> 注意：本文件不是 TSMC 官方文件，不描述任何未公開內部架構。

---

## 1) 這個角色的本質

```text
不是「會 kubectl」而已。

比較像：

  大型地端 compute platform 的 owner
  + Kubernetes / VM / bare metal 的深水區工程師
  + Day2 SRE operator
  + automation-first platform builder
  + 可帶團隊成長的技術負責人
```

---

## 2) 能力分層

```text
┌─────────────────────────────────────────────────────────────────────┐
│                         Business / Factory                          │
│  7x24 production, manufacturing constraints, local data center reality│
└───────────────────────────────┬─────────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────┐
│                         Platform Product                            │
│  Container service / VM service / SRE-friendly runtime platform       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────┐
│                         Kubernetes Layer                            │
│  API server / scheduler / controller / kubelet / CNI / CSI / CRI      │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────┐
│                         Compute Foundation                          │
│  Bare metal / Linux / kernel / network / storage / KVM / container    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3) Day 1 到 Day 2

```text
Day 1: Build
  - cluster bootstrap
  - node lifecycle
  - OS baseline
  - network/storage integration
  - security baseline
  - IaC / GitOps pipeline

Day 2: Run
  - monitoring / alerting
  - incident response
  - upgrade / rollback
  - capacity planning
  - performance tuning
  - failure testing
  - postmortem / continuous improvement
```

---

## 4) 技術領域拆解

### 4.1 Bare Metal / Linux / OS

```text
問題意識：
  Kubernetes 出事常常不是 Kubernetes 本身，
  而是 kernel、filesystem、network、disk、time、DNS、runtime、cgroup。

需要能回答：
  - node 為什麼 NotReady？
  - kubelet 為什麼壓力過高？
  - cgroup / CPU throttling / memory pressure 怎麼定位？
  - kernel / NIC / disk queue / conntrack 哪裡是瓶頸？
```

### 4.2 Kubernetes internals

```text
必懂元件：
  - kube-apiserver
  - etcd
  - kube-scheduler
  - kube-controller-manager
  - kubelet
  - kube-proxy / eBPF data plane
  - CNI
  - CSI
  - CRI / container runtime

要能做到：
  - 看懂 issue / PR / source code
  - 解釋 controller reconciliation loop
  - 理解 watch / cache / informer
  - 理解 API server scalability / etcd pressure
  - 理解 version skew / upgrade path
```

### 4.3 Network / BGP / ServiceMesh

```text
地端平台常見壓力：
  - east-west traffic
  - service discovery
  - ingress / egress control
  - load balancing
  - BGP route advertisement
  - pod-to-pod latency
  - DNS / conntrack / MTU / NAT 問題

可研究方向：
  - Cilium / Calico / kube-router
  - MetalLB / BGP mode
  - Envoy / Istio / Linkerd
  - eBPF observability
```

### 4.4 Storage / CSI

```text
要問的不是「可不可以掛 PVC」，而是：
  - failure domain 怎麼切？
  - attach/detach 慢怎麼查？
  - volume expansion / snapshot / backup policy？
  - stateful workload 的 SLO 是什麼？
  - etcd / database workload 是否應該跑在同一層平台？
```

### 4.5 Security / Supply Chain

```text
平台安全不是單點工具，而是一條鏈：

image provenance
  → registry policy
  → admission control
  → runtime isolation
  → secret handling
  → RBAC / audit
  → node hardening
  → incident evidence
```

### 4.6 IaC / Automation-first

```text
核心原則：
  所有操作都追求自動化。
  手動操作只能是 break-glass exception，不能是日常流程。

需要建立：
  - declarative config
  - reproducible bootstrap
  - upgrade pipeline
  - rollback plan
  - drift detection
  - runbook as code
```

---

## 5) SRE 運作模型

```text
┌────────────┐   signal    ┌────────────┐   response   ┌────────────┐
│ Telemetry  │───────────▶│ Alerting   │────────────▶│ Incident   │
└────────────┘            └────────────┘              └─────┬──────┘
                                                            │
                                                            ▼
┌────────────┐   improve   ┌────────────┐   learning   ┌────────────┐
│ Automation │◀───────────│ Postmortem │◀────────────│ Recovery   │
└────────────┘            └────────────┘              └────────────┘
```

### Incident Response

```text
要標準化：
  - severity definition
  - incident commander
  - comms channel
  - timeline
  - mitigation
  - customer / stakeholder update
  - postmortem owner
```

### Blameless Postmortem

```text
不問：誰犯錯？

改問：
  - 哪個防線沒有擋住？
  - 哪個訊號太晚出現？
  - 哪個手動步驟該自動化？
  - 哪個 runbook 不夠清楚？
  - 哪個測試沒有覆蓋？
```

---

## 6) AI 工具與 Agentic Workflow

```text
AI 不應只是「查答案」。

比較有價值的方向：
  - runbook draft / review
  - incident timeline summarization
  - log / metric triage assistant
  - upgrade impact checklist
  - PR / IaC diff review
  - chaos test scenario generation
  - postmortem action item tracking
```

安全邊界：

```text
AI agent 可以建議：
  - 查哪些指標
  - 跑哪些 read-only command
  - 產生 patch / PR
  - 草擬 postmortem

AI agent 不應無審核直接：
  - delete cluster resources
  - rotate production secrets
  - change BGP/network policy
  - perform irreversible storage operations
```

---

## 7) 能力成熟度

```text
L1 Operator
  會操作 kubectl、看 pod/log/event。

L2 Debugger
  能定位 node/network/storage/runtime 問題。

L3 Platform Builder
  能設計可重複部署、可升級、可觀測的平台。

L4 OSS-level Engineer
  能看 upstream source / issue / PR，理解底層實作。

L5 Technical Leader
  能建立標準、帶團隊、做容量/風險/投資決策。
```

---

## 8) 後續研究題目

```text
1. Kubernetes version / EOL / upgrade strategy
2. On-prem multi-cluster architecture
3. BGP + Kubernetes load balancing
4. CSI failure modes
5. Node pressure debugging playbook
6. Capacity planning model
7. Incident response template
8. AI-assisted SRE workflow safety model
```
