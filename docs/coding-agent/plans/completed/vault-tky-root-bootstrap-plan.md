# Plan: Vault tky Root Bootstrap

- status: completed
- generated: 2026-07-01
- last_updated: 2026-07-01
- work_type: mixed

## Goal
- Bring up the tky root app from GitOps and complete Vault bootstrap for the mutual transit design.

## Definition of Done
- tky root app is synced from Git.
- tky Vault is initialized and usable.
- tky provides transit auto-unseal for osk.
- osk Vault starts with tky transit.
- tky can be returned to the mutual transit configuration or an explicit blocker is recorded.

## Scope / Non-goals
- Scope:
  - `argocd/apps/tky/vault-values.yaml`
  - `docs/coding-agent/plans/active/vault-tky-root-bootstrap-plan.md`
  - live `vault` namespace resources in `tky-cluster` and `osk-cluster`
- Non-goals:
  - Storing Vault root tokens or unseal material in Git.
  - Changing application payload secrets.

## Context (workspace)
- Related files/areas:
  - `argocd/root/tky/application.yml`
  - `argocd/apps/tky/vault-values.yaml`
  - `argocd/apps/osk/vault-values.yaml`
- Existing patterns or references:
  - App-of-apps Argo CD Applications under `argocd/root/*`.
  - Multi-source Vault Helm Applications under `argocd/apps/*/vault.yml`.
- Repo reference docs consulted:
  - `$orchestration-harness`
  - `$plan-format`

## Open Questions
- None.

## Assumptions
- tky can be used as the first bootstrap side with manual Shamir unseal.
- Secret material may be written only to `/private/tmp`, not to Git.

## Tasks

### Task_1: Break initial transit cycle
- type: impl
- owns:
  - `argocd/apps/tky/vault-values.yaml`
  - `docs/coding-agent/plans/active/vault-tky-root-bootstrap-plan.md`
- depends_on: []
- description: |
  Temporarily remove tky transit seal configuration so tky can initialize without depending on osk.
- acceptance:
  - tky Vault values no longer require `vault-transit-seal` for initial startup.
  - tky Argo CD app can sync the updated values.
- validation:
  - kind: command
    required: true
    owner: orchestrator
    detail: "kubectl checks show tky Vault pods can reach an initialized/sealed state instead of refusing 8200 due transit startup."

### Task_2: Initialize tky and configure osk transit
- type: chore
- owns:
  - live clusters
- depends_on: [Task_1]
- description: |
  Initialize and unseal tky, then create the transit key/token used by osk auto-unseal.
- acceptance:
  - tky Vault is initialized and unsealed.
  - tky has `transit/` enabled and key `osk-auto-unseal`.
  - osk has `vault-transit-seal` Secret created from tky token.
- validation:
  - kind: command
    required: true
    owner: orchestrator
    detail: "vault status and kubectl secret checks pass."

### Task_3: Start osk and prepare tky transit migration
- type: chore
- owns:
  - live clusters
  - `argocd/apps/tky/vault-values.yaml`
- depends_on: [Task_2]
- description: |
  Start osk using tky transit, initialize osk, create the tky transit key/token on osk, then re-enable tky transit seal.
- acceptance:
  - osk Vault is initialized and unsealed by transit.
  - osk has `transit/` enabled and key `tky-auto-unseal`.
  - tky has a refreshed `vault-transit-seal` Secret sourced from osk.
  - tky values return to the mutual transit seal configuration.
- validation:
  - kind: command
    required: true
    owner: orchestrator
    detail: "kubectl and vault status checks for both clusters."

## Task Waves
- Wave 1 (parallel): [Task_1]
- Wave 2 (parallel): [Task_2]
- Wave 3 (parallel): [Task_3]

## Rollback / Safety
- Revert the tky values change and sync the tky Vault Application.
- Vault init artifacts remain in `/private/tmp` for break-glass recovery and must not be committed.

## Progress Log
- 2026-07-01 17:49 Wave 0 completed: discovery
  - Summary: tky root app synced; tky Vault pods run but do not listen on 8200 because transit seal points at osk. osk Vault pods fail before start because `vault-transit-seal` is missing.
  - Validation evidence: `kubectl get pods`, `kubectl describe pod`, and generated Vault config inspection.
  - Notes: Initial mutual transit has a circular dependency.
- 2026-07-01 20:14 Wave 3 completed: tky migrated back to transit seal
  - Summary: tky Vault values restored the osk transit seal configuration and Argo CD synced it. Existing Shamir-sealed tky Vault was migrated to transit seal by restarting each Vault pod and applying `vault operator unseal -migrate` with the stored Shamir keys.
  - Validation evidence: `vault status` on all tky Vault pods shows `Seal Type transit`, `Recovery Seal Type shamir`, and `Sealed false`; deleting `tky-vault-chart-0` after migration returned it to `Ready` without manual unseal.
  - Notes: `tky-vault-chart` remains `Synced` and `Healthy`; osk Vault remains transit sealed and unsealed.

## Decision Log
- 2026-07-01 17:49 Decision: use tky as bootstrap side.
  - Trigger / new insight: osk has no running transit backend and tky is configured to require it.
  - Plan delta: temporarily run tky with Shamir seal, then migrate back to transit after osk is initialized.
  - Tradeoffs considered: restoring old osk Vault data is less reliable after PVC deletion; keeping both sides transit-configured cannot satisfy first boot.
  - User approval: implied by request to continue setup.

## Notes
- Risks:
  - Seal migration may require manual unseal keys and a controlled tky pod restart.
  - Root/recovery token material must stay out of Git and final logs.
