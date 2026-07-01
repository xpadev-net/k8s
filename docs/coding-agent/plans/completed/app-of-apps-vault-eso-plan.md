# Plan: App Of Apps Vault Eso

- status: done
- generated: 2026-07-01
- last_updated: 2026-07-01
- work_type: code

## Goal
- Add an initial app-of-apps layout for the osk and tky clusters.
- Manage HashiCorp Vault and External Secrets Operator through Argo CD child Applications per cluster.

## Definition of Done
- Root Applications exist for osk and tky and point at cluster-specific child Application directories.
- Each cluster has child Applications for Vault and External Secrets Operator.
- Kustomize can render each child Application directory.
- YAML manifests are syntactically valid.

## Scope / Non-goals
- Scope:
  - `argocd/root/{osk,tky}/application.yml`
  - `argocd/apps/{osk,tky}/kustomization.yaml`
  - `argocd/apps/{osk,tky}/{vault,external-secrets}.yml`
- Non-goals:
  - Vault production storage, seal, ingress, auth, and policy configuration.
  - ExternalSecret, SecretStore, or ClusterSecretStore resources.
  - Migrating existing top-level Applications into the root app.

## Context (workspace)
- Related files/areas:
  - `osk-metallb/application.yml`
  - `osk-signoz/application.yml`
  - `osk-cloudflared-tunnel/application.yml`
  - `tky-cloudflared-tunnel/application.yml`
- Existing patterns or references:
  - Argo CD Applications live in namespace `argocd`.
  - Existing apps use project `default` and in-cluster destination server.
  - Helm chart Applications pin `targetRevision`.
- Repo reference docs consulted:
  - No `docs/coding-agent/rules` suite exists in this repository.

## Open Questions
- Q1: Are osk and tky managed by separate in-cluster Argo CD instances, or one central Argo CD with multiple destination clusters?
- Q2: Which Vault storage/seal/auth settings should be used for production?
- Q3: Which ESO Vault auth and SecretStore strategy should be adopted?

## Assumptions
- A1: Each cluster can apply its own root app with `destination.server: https://kubernetes.default.svc`.
- A2: The first rollout should install charts only, leaving Vault/ESO integration resources for a later phase.
- A3: Chart versions are pinned to current Helm repo index entries: Vault `0.33.0`, External Secrets Operator `2.7.0`.

## Tasks

### Task_1: Add App Of Apps Manifests
- type: impl
- owns:
  - `argocd/root/**`
  - `argocd/apps/**`
- depends_on: []
- description: |
  Create root Applications for osk/tky and child Applications for Vault/External Secrets Operator.
- acceptance:
  - osk root Application points to `argocd/apps/osk`.
  - tky root Application points to `argocd/apps/tky`.
  - Each cluster directory contains Vault and External Secrets Operator child Applications.
  - Child Applications follow the repository's existing Argo CD Application conventions.
- validation:
  - kind: command
    required: true
    owner: orchestrator
    detail: "`rtk kustomize build argocd/apps/osk` and `rtk kustomize build argocd/apps/tky` pass"
  - kind: command
    required: true
    owner: orchestrator
    detail: "YAML parsing succeeds for added manifests"

### Task_2: Review Added Manifests
- type: review
- owns: []
- depends_on: [Task_1]
- description: |
  Review the new manifests against the requested app-of-apps structure and existing repo conventions.
- acceptance:
  - Review finds no blocking manifest or structure issues.
  - Validation evidence is present.
- validation:
  - kind: review
    required: true
    owner: orchestrator
    detail: "Diff reviewed manually against acceptance criteria"

## Task Waves
- Wave 1 (parallel): [Task_1]
- Wave 2 (parallel): [Task_2]

## Rollback / Safety
- Remove `argocd/root/**` and `argocd/apps/**` additions from this change.
- Do not apply root apps until the target cluster Argo CD instance is confirmed.

## Progress Log
- 2026-07-01 15:30 JST Wave 0 started:
  - Summary: Research completed; implementation approved by user continuation.
  - Validation evidence: pending.
  - Notes: Researcher used for read-only repo layout discovery.
- 2026-07-01 15:30 JST Wave 1 completed: [Task_1]
  - Summary: Added osk/tky root apps and Vault/External Secrets child apps.
  - Validation evidence: `rtk kustomize build argocd/apps/osk` passed; `rtk kustomize build argocd/apps/tky` passed; Ruby YAML parsing passed for all added manifests.
  - Notes: Chart versions pinned to Vault `0.33.0` and External Secrets Operator `2.7.0`.
- 2026-07-01 15:30 JST Wave 2 completed: [Task_2]
  - Summary: Orchestrator reviewed added manifests against request and existing conventions.
  - Validation evidence: Direct file review confirmed root paths, app names, namespaces, repo URLs, chart names, and sync policies.
  - Notes: Independent Reviewer waived because the available subagent runtime policy only permits new subagents when explicitly requested by the user.

## Decision Log
- 2026-07-01 15:30 JST Decision:
  - Trigger / new insight: Existing repository has no app-of-apps layout.
  - Plan delta: Introduce `argocd/root` and `argocd/apps` as the first app-of-apps structure.
  - Tradeoffs considered: Keep existing top-level apps unchanged to reduce rollout risk.
  - User approval: yes, via "plz continue".

## Notes
- Risks:
  - Root apps assume in-cluster Argo CD per location.
  - Vault default chart values are not production-complete.
  - ESO is installed without any provider store resources in this phase.
