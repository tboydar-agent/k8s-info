# AI-assisted SRE / platform engineering workflow 安全導讀

> 狀態：公開來源整理，非官方建議。
> 範圍：Kubernetes / platform engineering 團隊如何把 Claude Code、GitHub Copilot / Copilot coding agent、OpenAI Codex 類工具放進低風險 SRE 工作流。
> 邊界：不保存、不推論、不處理任何特定公司內部 cluster 版本、hostnames、IP、ticket、incident、聊天紀錄、token 或 credentials。

---

## 1) 核心結論

AI coding agent / command-line agent 對 SRE 最有價值的地方，不是「直接操作 production」，而是把公開文件、sanitized logs、runbook、IaC diff、postmortem 草稿轉成更可審查的工程產物。

```text
Good fit:
  read-only / bounded / reviewable

  public docs + sanitized signals
          │
          ▼
  AI draft / checklist / diff review
          │
          ▼
  human SRE review + tests + approval gate
          │
          ▼
  normal GitOps / CI / change-management flow

Bad fit:
  unbounded agent + production credentials + direct cluster mutation
```

這份文件把 AI-assisted SRE 分成四層：

1. **Research / explain**：讀公開 docs、release notes、runbook，產生摘要與檢查表。
2. **Review / draft**：檢查 IaC / Helm / Kustomize / Terraform / YAML diff，提出風險與測試建議。
3. **Triage assistant**：根據 sanitized `kubectl describe` / Events / logs 摘要，產出下一步調查清單。
4. **Change execution**：只允許走既有 PR、CI、GitOps、change approval；不讓 agent 直接拿 production kubeconfig 自動改環境。

---

## 2) 來源驗證摘要

### 2.1 GitHub Copilot / coding agent

GitHub Docs 將 Copilot cloud agent / coding agent 放在「agent 可接任任務、追蹤 session、產生 pull request / reviewable output」的產品脈絡中，也包含 MCP、firewall、secrets / variables、network settings、activity / logs 等管理主題。

對 Kubernetes SRE 的可用解讀：

- 適合放在 **PR / issue / review** 工作流，而不是直接 production 操作。
- 若使用 agent 讀取 repo，需要明確限制：可讀哪些 repository、可連哪些 MCP、可用哪些 secrets / variables。
- 產物應該是 PR、patch、runbook 草稿、測試建議；交付後仍需要人類 review。

### 2.2 Anthropic Claude Code security

Claude Code security 文件明確強調安全防護與使用建議，例如：預設權限、執行命令 / 修改檔案前的 permission、敏感 code 的專案級設定、dev containers、credential 分離與縮小 blast radius。

對 Kubernetes SRE 的可用解讀：

- 使用 Claude Code 類工具時，應以 **least privilege + explicit approval** 為預設。
- 對敏感 IaC / production runbook repository，應使用 project-specific permissions。
- 對需要執行測試或命令的工作，應優先在 dev container / throwaway workspace 內執行。
- credential 不應混用；若必須使用憑證，也要限定用途、有效期與 blast radius。

### 2.3 OpenAI Codex

OpenAI Codex 開發者頁將 Codex 描述為可協助 debug、root-cause tracing、targeted fix、refactor、testing、migration、setup tasks 的 coding agent；文件索引也包含 permissions、sandboxing、shell / file tools、MCP / connectors、background mode 等主題。

對 Kubernetes SRE 的可用解讀：

- 適合用於 **debug reasoning + patch proposal + test automation**。
- 在平台工程場景中，Codex 類工具應被放在 sandbox / repo workspace，而不是 production bastion。
- 任何 `kubectl apply`、`helm upgrade`、node drain、CNI / CSI 變更都不應由 agent 自動執行，除非已有獨立審核與 change gate。

### 2.4 Kubernetes debugging / secrets docs

Kubernetes 官方 debugging 文件把 cluster / Pod debugging 建立在 `kubectl describe`、Events、logs、狀態檢查等可觀測訊號上；Secrets good practices 則提醒 secret 存取、RBAC、節點與 workload 暴露面的安全邊界。

對 AI-assisted SRE 的可用解讀：

- 可以提供給 AI 的是 **sanitized observation**，不是原始 kubeconfig、service account token、Secret manifest、完整內部 log dump。
- Debug summary 要保留「來源型態」與「時間範圍」，例如 `Events summary from namespace X, 20 minutes, redacted`。
- 對 Secrets / RBAC / production kubeconfig，預設不交給 AI agent。

---

## 3) 安全分層模型

```text
┌─────────────────────────────────────────────────────────────────┐
│ Layer 0: Public research                                         │
│ - official docs, release notes, public blog / talks              │
│ - no internal data                                               │
│ Risk: low                                                        │
└─────────────────────────────┬───────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 1: Repo-local review                                       │
│ - Markdown, runbooks, IaC diff, test plans                       │
│ - no secrets, no kubeconfig                                      │
│ Risk: low / medium                                               │
└─────────────────────────────┬───────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 2: Sanitized incident triage                               │
│ - redacted Events/log snippets/metrics summary                   │
│ - no hostnames/IPs/tickets/person names unless public + needed   │
│ Risk: medium                                                     │
└─────────────────────────────┬───────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 3: Change proposal                                         │
│ - AI proposes PR / patch / rollback plan                         │
│ - CI + peer review + change approval required                    │
│ Risk: medium / high                                              │
└─────────────────────────────┬───────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 4: Production mutation                                     │
│ - kubectl apply / helm upgrade / node drain / cloud API writes   │
│ - do not allow autonomous AI execution by default                │
│ Risk: high                                                       │
└─────────────────────────────────────────────────────────────────┘
```

建議預設：AI agent 可以到 Layer 3 的 **proposal**，不能自動跨到 Layer 4。

---

## 4) 三種 SRE workflow pattern

### Pattern A — IaC / YAML review assistant

適用：Helm chart、Kustomize patch、Terraform、Argo CD Application、NetworkPolicy、RBAC、PodDisruptionBudget、HPA、ResourceQuota、storage class 變更。

```text
Input:
  - PR diff
  - related public docs / internal sanitized runbook excerpt
  - expected blast radius

Ask AI to return:
  1. risk checklist
  2. missing tests
  3. rollback questions
  4. unclear assumptions

Do not provide:
  - production kubeconfig
  - secret values
  - private incident timeline unless redacted
  - raw customer / user data
```

審查重點：

- 是否增加 cluster-admin / wildcard RBAC？
- 是否變更 CNI / CSI / ingress / control-plane critical path？
- 是否缺 PDB、readiness probe、resource request / limit？
- 是否有跨 namespace / hostNetwork / privileged / hostPath 放大面？
- 是否有 upgrade / rollback / canary / drain 計畫？

### Pattern B — Incident summary assistant

適用：on-call 後整理初步 incident note 或 postmortem draft。

```text
Observation packet:
  - time window
  - affected surface in generic terms
  - sanitized Events summary
  - sanitized logs: only error class / count / representative line
  - metrics trend summary, not raw tenant data
  - actions already taken

AI output:
  - timeline draft
  - suspected contributing factors
  - follow-up questions
  - what evidence is missing
  - safe postmortem outline
```

禁止：

- 貼上完整 private dashboard URL。
- 貼上 customer / user identifiers。
- 貼上 kubeconfig、service account token、Secret、cloud credential。
- 讓 AI 直接寫「root cause 已確認」，除非證據已由人類 SRE 驗證。

### Pattern C — Runbook test / chaos tabletop assistant

適用：演練 node pressure、CoreDNS degraded、CSI attach 慢、LoadBalancer advertisement 失敗、API server latency、etcd alert。

```text
Runbook draft
      │
      ▼
AI generates tabletop scenario
      │
      ▼
Human SRE removes unsafe/private details
      │
      ▼
Team rehearsal / validation
      │
      ▼
Runbook PR update
```

AI 可以協助：

- 產生演練問題。
- 找 runbook 缺口。
- 轉成新人可讀的 checklist。
- 建議哪些 metrics / Events / logs 應該先看。

AI 不應該：

- 自行 invent production topology。
- 自行產生真實 hostnames / IP / ticket。
- 直接對 cluster 執行破壞性測試。

---

## 5) Prompt templates

### 5.1 IaC review prompt

```text
You are reviewing a Kubernetes infrastructure PR.
Use only the diff and public Kubernetes docs I provide.
Do not infer private environment details.
Return:
1. high-risk changes,
2. missing tests,
3. rollback questions,
4. assumptions to verify,
5. safe reviewer checklist.
Do not suggest applying this directly to production.
```

### 5.2 Sanitized incident summary prompt

```text
You are helping draft a sanitized Kubernetes incident note.
Input has been redacted and may be incomplete.
Do not invent root cause.
Separate:
- observed facts,
- hypotheses,
- missing evidence,
- suggested next checks,
- safe postmortem wording.
Do not include secrets, people names, private URLs, IPs, hostnames, or customer identifiers.
```

### 5.3 Runbook gap review prompt

```text
Review this Kubernetes runbook as a safety reviewer.
Find unclear steps, missing prechecks, missing rollback criteria,
and steps that should require human approval.
Assume no direct production access is available to you.
Return a checklist and questions, not executable commands.
```

---

## 6) 導入檢查表

### Before enabling an AI agent

- [ ] 定義 agent 的可讀 repo / directory 範圍。
- [ ] 定義 agent 是否可執行 shell command；預設需要 approval。
- [ ] 禁止讀取 `.kube/`、cloud credentials、Secret dumps、private log archives。
- [ ] 若有 MCP / connector，逐一列出資料來源與權限。
- [ ] 將 agent output 限制為 PR / patch / Markdown / checklist。
- [ ] CI / lint / unit / policy checks 必須由既有 pipeline 執行。
- [ ] Production mutation 仍走 GitOps / change approval。

### During use

- [ ] 只給最小必要上下文。
- [ ] 先 redaction，再貼給 AI。
- [ ] 要求 AI 標示 assumptions / unknowns。
- [ ] 要求 AI 不要產生不可驗證結論。
- [ ] 人類 reviewer 對每個高風險建議簽核。

### After use

- [ ] 保存的是 PR / runbook / checklist，不保存 raw private context。
- [ ] 檢查是否包含 email、token、private URL、IP、hostname、customer identifiers。
- [ ] 對錯誤建議建立 correction note，避免下次重複。
- [ ] 若 agent 產生 incident/postmortem 草稿，標示為 draft until reviewed。

---

## 7) 待查事項

- Copilot / Claude Code / Codex 在 enterprise policy、audit logs、network egress、MCP connector governance 的細節會隨產品更新；需要定期回看官方文件。
- 不同組織的資料分類、法務、資安規範不同；這份文件只提供通用安全框架，不可直接替代內部 policy。
- 若未來要把 AI agent 接進 ChatOps / ticket / CI/CD，需要另寫「connector 權限矩陣」與「break-glass 禁止條款」。

---

## 8) Sources

- GitHub Docs — GitHub Copilot cloud agent: <https://docs.github.com/en/copilot/how-tos/agents/copilot-coding-agent>
- Anthropic Docs — Claude Code Security: <https://docs.anthropic.com/en/docs/claude-code/security>
- OpenAI Developers — Codex: <https://developers.openai.com/codex/>
- Kubernetes Docs — Troubleshooting Clusters: <https://kubernetes.io/docs/tasks/debug/debug-cluster/>
- Kubernetes Docs — Debug Running Pods: <https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/>
- Kubernetes Docs — Good practices for Kubernetes Secrets: <https://kubernetes.io/docs/concepts/security/secrets-good-practices/>

---

## 9) Privacy / safety note

本文只整理公開產品文件與 Kubernetes 官方文件。本文不包含：

- 特定公司內部 Kubernetes 版本。
- 內部 cluster topology、hostnames、IP、ticket、incident timeline。
- 個人 email / phone / private chat。
- kubeconfig、service account token、cloud credential、Secret 值。
