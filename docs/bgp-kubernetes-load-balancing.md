# BGP / Kubernetes LoadBalancer 導讀

> 日期：2026-05-11
> 範圍：大型地端 / bare-metal Kubernetes 的 `Service type=LoadBalancer`、L2 announcement、BGP advertisement 與 CNI/LoadBalancer 實作選型。
> 風格：source-grounded、低風險、可驗證；本文件不是任何特定公司內部網路設計。

---

## TL;DR

在公有雲以外的 Kubernetes，`Service type=LoadBalancer` 不會自動等於「有雲端 LB 可用」。大型地端平台通常要自己回答三個問題：

```text
Service VIP 從哪裡來？
  └─ IP pool / LB IPAM / service external IP range

誰對外宣告這個 VIP？
  ├─ L2: ARP / NDP announcement
  └─ L3: BGP route advertisement

流量進來後怎麼落到 Pod？
  ├─ kube-proxy / eBPF / CNI data path
  ├─ externalTrafficPolicy: Cluster / Local
  └─ ECMP、node selector、failure detection、source IP preservation
```

最保守的結論：

- Kubernetes 官方文件定義 `LoadBalancer` Service API，但裸機環境必須有額外 controller / CNI / network integration 才能真的分配與宣告外部 IP。
- MetalLB 是裸機 LoadBalancer 的常見實作：先用 `IPAddressPool` 分配 IP，再用 L2 或 BGP 對外宣告。
- Cilium 可以用 BGP Control Plane 對 connected routers 宣告 routes，也提供 L2 Announcements（文件標示 Beta）讓 service 在 local area network 可見。
- Calico 可以透過 BGP 宣告 service ClusterIP / ExternalIP / LoadBalancer IP；官方文件特別提醒 ECMP、ToR、`externalTrafficPolicy` 與 source IP preservation 的邊界。
- 選型時不要只問「哪個元件能宣告 IP」，要一起檢查：失敗偵測時間、ECMP/route policy、節點下線流程、VIP ownership、IP pool 變更流程、CNI 版本相容性與 rollback。

---

## 1. Kubernetes 官方邊界：Service 是抽象，不是裸機 LB 實作

Kubernetes `Service` 是把一組 Pod endpoints 暴露成網路服務的 API 抽象。`type=LoadBalancer` 會在支援的環境中建立外部負載平衡器；但在 bare-metal/on-prem 環境，是否能取得外部 IP、如何宣告該 IP、流量如何回到節點，都取決於額外元件。

```text
Kubernetes Service API
        │
        ▼
Service type=LoadBalancer
        │
        ├─ cloud provider: 通常由 provider controller 建 LB
        │
        └─ bare metal: 需要 MetalLB / Cilium / Calico / appliance / 自研 integration
```

### on-prem 檢查點

- 是否有明確的 VIP / LoadBalancer IP pool？
- IP pool 是否與資料中心 routing / firewall / IPAM 流程一致？
- `externalTrafficPolicy` 設為 `Cluster` 或 `Local`？是否需要 source IP preservation？
- Service 變更是否會觸發 BGP/L2 宣告變更？誰審核？誰 rollback？

---

## 2. L2 announcement vs BGP advertisement

### L2：ARP/NDP ownership 模式

L2 announcement 的核心是：讓區域網路內的其他 host/router 透過 ARP/NDP 找到目前持有 VIP 的節點。

```text
client / router
      │ ARP/NDP: who has VIP?
      ▼
selected node owns VIP
      │
      ▼
Kubernetes service data path
      │
      ▼
Pod endpoints
```

適合：

- 小到中型 cluster、單一 L2 domain、希望快速上手。
- 網路團隊不希望或暫時不能讓每個節點與 ToR 建 BGP peering。

風險與邊界：

- L2 domain 擴大後，廣播/鄰居表/故障切換行為要更仔細測試。
- 一個 VIP 通常會有 leader/owner 概念；故障切換時間取決於 lease / ARP cache / implementation。
- Cilium L2 Announcements 官方文件目前標示 Beta，且提醒與 `externalTrafficPolicy: Local` 不相容，可能造成無 Pod 節點宣告 IP 後掉流量。

### BGP：把 VIP 當 route 對外宣告

BGP 模式的核心是：Kubernetes node / speaker 對 ToR 或 route reflector 宣告 service VIP route，讓外部網路用 routing table 找到 cluster 入口。

```text
          ┌───────────────┐
          │ external net   │
          └───────┬───────┘
                  │ route to VIP
                  ▼
        ┌───────────────────┐
        │ ToR / RR / router  │
        └───┬───────────┬───┘
            │ BGP       │ BGP
            ▼           ▼
        node A       node B
        speaker      speaker
            │           │
            └──── service data path ────► pods
```

適合：

- 大型 bare-metal / multi-rack 環境，需要 routing policy、ECMP、failover 可觀測性。
- 網路團隊已管理 BGP、AS number、ToR/route reflector、BFD/health/failure detection。

風險與邊界：

- BGP session、AS number、route policy、community、prefix length、node selector 都要被變更管理。
- 如果 ToR 沒有 ECMP / multipath，可能不是所有可宣告節點都會平均承載流量。
- `externalTrafficPolicy: Local` 可以幫助保留 source IP，但必須避免無 local endpoint 的節點宣告 VIP。

---

## 3. 實作選項對照

| 選項 | 官方文件觀察 | 適合情境 | 主要待查 |
|---|---|---|---|
| Kubernetes Service `LoadBalancer` | 官方 API 抽象，裸機需要額外實作 | 定義 Service 暴露方式 | 誰分配外部 IP？誰宣告？誰 health-check？ |
| MetalLB L2 | MetalLB 文件提供 `IPAddressPool` + `L2Advertisement` | 單一 L2 domain、簡化起步 | VIP failover、ARP/NDP cache、leader 選舉、節點下線流程 |
| MetalLB BGP | MetalLB 文件提供 `BGPPeer` + `BGPAdvertisement`，可做 peer/node/VRF/source address 等進階設定 | bare-metal rack/ToR BGP integration | AS/peer 設計、BFD、route policy、community、ECMP |
| Cilium BGP Control Plane | 官方文件：用 BGP 對 connected routers 宣告 routes；需啟用 `bgpControlPlane` | 已採 Cilium/eBPF datapath，需要 CNI-native BGP | Cilium 版本、CRD 狀態、BGP troubleshooting、升級/降級流程 |
| Cilium L2 Announcements | 官方文件標示 Beta；用 policy 控制 service/node/interface 宣告 | Cilium 環境中的 L2 LB | Beta 風險、lease QPS、與 `externalTrafficPolicy: Local` 不相容 |
| Calico BGP service IP advertisement | 官方文件支援宣告 ClusterIP / ExternalIP / LoadBalancer IP；提醒 ToR ECMP 與 BGP multipath | 已採 Calico BGP routing 的 cluster | `serviceClusterIPs`/`serviceLoadBalancerIPs` 範圍、route reflector、source IP preservation |

---

## 4. 低風險設計清單

### A. IP 與宣告邊界

- [ ] LoadBalancer IP pool 是否只包含網路團隊允許的 CIDR？
- [ ] 是否分開：測試池、內部服務池、對外服務池、特殊專線池？
- [ ] 是否禁止 application team 隨意指定任意 `loadBalancerIP` / external IP？
- [ ] 是否有 audit：哪個 Service 佔用哪個 VIP？

### B. Routing / ToR / ECMP

- [ ] BGP peer 是每 node 對 ToR、每 rack route reflector，還是集中 speaker？
- [ ] 是否啟用 multipath/ECMP？最大 path 數量是否足夠？
- [ ] prefix 長度是單一 VIP `/32`/`/128`，還是 aggregation prefix？
- [ ] 是否需要 BGP community、local preference、NO_ADVERTISE 或 VRF？
- [ ] 是否需要 BFD 或其他快速故障偵測？

### C. Kubernetes Service 行為

- [ ] `externalTrafficPolicy` 預設策略是什麼？
- [ ] 如果用 `Local`，只有有 local endpoints 的 node 會宣告嗎？
- [ ] 若使用 NodePort allocation，是否真的需要 NodePort？是否能關閉不必要 allocation？
- [ ] kube-proxy / eBPF datapath / CNI policy 是否與 Service exposure 設計一致？

### D. Day-2 Operations

- [ ] 節點 drain / cordon / reboot 前，route/VIP 宣告如何撤除？
- [ ] CNI/MetalLB/Calico/Cilium controller 升級時，是否會短暫撤 route？
- [ ] 觀測指標有哪些：BGP session state、route count、VIP owner、endpoint readiness、drop/error counter？
- [ ] 演練項目是否包含：speaker crash、node network down、ToR down、endpoint 全部消失、IP pool 誤設定？

---

## 5. 故障初判流程

```text
Symptom: Service LoadBalancer IP 無法連線

1) Kubernetes API 層
   ├─ kubectl get svc -A -o wide
   ├─ LoadBalancer IP 是否已分配？
   └─ endpoints / EndpointSlice 是否有 Ready endpoints？

2) 宣告層
   ├─ L2: VIP 目前由哪個 node 回 ARP/NDP？
   ├─ BGP: VIP route 是否在 ToR/RR 出現？next-hop 是否正確？
   └─ speaker/controller logs 是否有 IP pool / peer / policy error？

3) 節點資料路徑
   ├─ externalTrafficPolicy: Cluster 或 Local？
   ├─ local endpoint 是否存在？
   ├─ kube-proxy/eBPF service map 是否更新？
   └─ NetworkPolicy / host firewall / rp_filter / MTU 是否阻擋？

4) 回程路徑
   ├─ source IP preservation 是否影響 policy？
   ├─ return route 是否走錯 VRF / gateway？
   └─ ECMP hash 是否把流量導到沒有 endpoint 的節點？
```

---

## 6. 適用範圍與不可直接套用的邊界

適用：

- 大型地端、裸機、private cloud、lab cluster 的 Kubernetes service exposure 研究。
- 設計 review、on-call runbook、PoC checklist、平台 team 與網路 team 對齊詞彙。

不適用 / 待查：

- 不能直接當成任何特定公司內部網路架構或 routing policy。
- 不能替代實際 ToR/RR/vendor router 設定審查。
- 不能替代 CNI/MetalLB/Calico/Cilium 的版本相容性矩陣；正式升級前仍需查當前版本 release notes。
- 不宣稱哪個方案「最佳」；選型取決於既有 CNI、網路團隊 BGP 成熟度、failure-domain 設計與操作能力。

---

## Sources

- Kubernetes Docs — Service: <https://kubernetes.io/docs/concepts/services-networking/service/>
- MetalLB — Concepts: <https://metallb.io/concepts/>
- MetalLB — Configuration: <https://metallb.io/configuration/>
- MetalLB — Advanced BGP configuration: <https://metallb.io/configuration/_advanced_bgp_configuration/>
- Cilium — BGP Control Plane: <https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane/>
- Cilium — L2 Announcements / L2 Aware LB: <https://docs.cilium.io/en/stable/network/l2-announcements/>
- Calico — Advertise Kubernetes service IP addresses: <https://docs.tigera.io/calico/latest/networking/configuring/advertise-service-ips>
- Calico — Configure BGP peering: <https://docs.tigera.io/calico/latest/networking/configuring/bgp>

---

## 後續研究

- 補一份「BGP LoadBalancer lab checklist」：kind/bare-metal lab、FRR/GoBGP/ToR 模擬、failure injection、觀測指標。
- 補 CNI-specific compatibility watch：MetalLB + Calico/Cilium、Cilium BGP + kube-proxy replacement、Calico route reflector upgrade。
- 補 multi-rack design：per-rack IP pool、rack-local announcement、failure domain 與 traffic locality。
