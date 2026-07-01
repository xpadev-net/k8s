# Plan: Vault Mutual Transit

- status: done
- generated: 2026-07-01
- last_updated: 2026-07-01
- work_type: code

## Goal
- Prepare osk and tky Vault Applications for HA Raft and mutual transit auto-unseal.
- Prepare Vault Applications to auto-sync once peer transit tokens and stable peer addresses are ready.

## Definition of Done
- Cluster-specific Vault Helm values exist for osk and tky.
- Vault Applications reference the values through Argo CD multi-source.
- Vault Applications keep auto-sync enabled.
- Runbook documents transit bootstrap and break-glass requirements.
- Kustomize renders both app directories.

## Scope / Non-goals
- Scope:
  - `argocd/apps/{osk,tky}/vault.yml`
  - `argocd/apps/{osk,tky}/vault-values.yaml`
  - `docs/vault-mutual-transit.md`
- Non-goals:
  - Creating transit tokens in Git.
  - Migrating the live standalone file-storage Vault to Raft automatically.
  - Publishing DNS records or Cloudflare tunnel routes.

## Assumptions
- osk Vault transit endpoint will be reachable at `http://10.154.11.20:8200`.
- tky Vault transit endpoint will be reachable at `http://10.153.11.20:8200`.
- Each cluster will store the peer transit token in a local Secret named `vault-transit-seal`.
- Full dual-site outage recovery is manual break-glass by Shamir unseal on the chosen first site.

## Tasks

### Task_1: Add HA/Transit Values
- type: impl
- owns:
  - `argocd/apps/osk/vault-values.yaml`
  - `argocd/apps/tky/vault-values.yaml`
- depends_on: []
- acceptance:
  - Each values file enables HA Raft.
  - Each values file configures transit seal to the peer site's active Vault endpoint.
  - Transit token is sourced from a Kubernetes Secret, not Git.
  - UI/API service is exposed through a stable MetalLB IP.
- validation:
  - kind: command
    required: true
    owner: orchestrator
    detail: "`rtk kustomize build argocd/apps/osk` and `rtk kustomize build argocd/apps/tky` pass"

### Task_2: Gate Vault Application Sync
- type: impl
- owns:
  - `argocd/apps/osk/vault.yml`
  - `argocd/apps/tky/vault.yml`
- depends_on: [Task_1]
- acceptance:
  - Vault Applications use Argo CD multi-source to load Git-hosted values.
  - Vault Applications auto-sync the HA/transit configuration.
  - Injector webhook CA bundle drift is ignored.
- validation:
  - kind: command
    required: true
    owner: orchestrator
    detail: "YAML parsing succeeds for changed manifests"

### Task_3: Document Operations
- type: docs
- owns:
  - `docs/vault-mutual-transit.md`
- depends_on: [Task_1, Task_2]
- acceptance:
  - Runbook documents Secret names and key names.
  - Runbook documents bootstrap order.
  - Runbook documents full-cycle break-glass recovery.
- validation:
  - kind: review
    required: true
    owner: orchestrator
    detail: "Runbook reviewed against configured values"

## Task Waves
- Wave 1 (parallel): [Task_1]
- Wave 2 (parallel): [Task_2, Task_3]

## Progress Log
- 2026-07-01 17:02 JST Wave 1 completed: [Task_1]
  - Summary: Added osk/tky Vault values for HA Raft and mutual transit.
  - Validation evidence: `rtk helm template osk-vault-chart ... -f argocd/apps/osk/vault-values.yaml` passed; `rtk helm template tky-vault-chart ... -f argocd/apps/tky/vault-values.yaml` passed.
  - Notes: Peer endpoints use MetalLB IPs.
- 2026-07-01 17:02 JST Wave 2 completed: [Task_2, Task_3]
  - Summary: Gated Vault Application auto-sync and documented bootstrap/break-glass operations.
  - Validation evidence: `rtk kustomize build argocd/apps/osk` passed; `rtk kustomize build argocd/apps/tky` passed; YAML parse passed for Vault Application and values files.
  - Notes: Live migration remains manual.

## Decision Log
- 2026-07-01 17:02 JST Decision:
  - Trigger / new insight: Mutual transit requires stable peer Vault endpoints and peer tokens.
  - Plan delta: Use active Vault UI/API LoadBalancer service per site, with tokens in `vault-transit-seal`.
  - Tradeoffs considered: Cloudflared has no ingress routes today; MetalLB is already available per site.
  - User approval: yes, via "相互transitの方向で進めて".

## Notes
- Applying these values can replace the existing empty standalone osk Vault state.
