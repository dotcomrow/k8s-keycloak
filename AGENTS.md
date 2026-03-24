# Agent Instructions (k8s-keycloak)

These instructions are mandatory for AI agents working in this repository.

## Non-Destructive Operations Policy

1. Data-destructive actions are prohibited by default.
2. Never use `DROP DATABASE`, `DROP SCHEMA ... CASCADE`, table drops, truncation, or bulk deletes as a remediation path.
3. Never enable or introduce automatic reset behavior that can destroy Keycloak data.
4. If a migration is broken, prefer idempotent, additive, reversible fixes (reconcile changelog state, `IF NOT EXISTS`, targeted non-destructive DDL).

## Break-Glass Requirement (Explicit User Approval Required)

Destructive actions are allowed only when all conditions are true:

1. The user explicitly requests the destructive action in the current thread.
2. A backup/snapshot and restore path are confirmed and documented.
3. The agent explains the exact blast radius and receives explicit confirmation.
4. The change is delivered through GitOps manifests (not ad-hoc cluster mutation).

Without all four conditions, destructive changes are not permitted.

## GitOps-Only Execution

1. Make changes in manifests/scripts in Git, not direct live changes.
2. Do not rely on direct `kubectl apply` as the final fix path for managed resources.
3. Use read-only `kubectl` access for diagnostics; deliver fixes through committed YAML.

## Validation Requirement

For every manifest update, run and report:

1. `kubectl apply --dry-run=client -f <changed-manifest>`
2. Any additional syntax checks relevant to embedded scripts.
