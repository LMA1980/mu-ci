# 3-Phase Configuration Migration Guide (`.muci/` & `.actionrunner/`)

## Location: `docs/migration.md`

### Overview

Because the Jero-Mu CI/CD fabric (`mu-ci`, `mu-ci-isolation`, `mu-cd`, `mu-registry`) dogfoods its own repositories and builds itself on the cluster (`node-a`, `node-b`, `worker-riscv64`), configuration schema evolution must **never** delete or break existing workflow files before the updated parser and worker daemons are deployed and verified in production (`AGENTS.md §4`).

This document codifies the mandatory **3-Phase Configuration Migration** from legacy `version: "2"` per-event workflow files (`.actionrunner/on-*.yml`) and `.actionrunner/` manifests to the modern `version: "3"` 3-tier declarative pipeline (`.muci/mu-ci.yml` with `.actionrunner/ActionRunner.yml` fallback) and `mu-ci-isolation` sandbox profile manifest (`.muci/security.yml` with `.actionrunner/security.yml` fallback).

______________________________________________________________________

## 1. Phase 1: Dual-Stack Introduction (Zero-Disruption Coexistence)

During Phase 1, repositories introduce the new `.muci/` declarative manifests **alongside** existing `.actionrunner/` workflow and security files. Neither legacy nor updated controller deployments fail during this coexistence window.

### Phase 1 Repository Requirements

1. **Author `.muci/mu-ci.yml` (`version: "3"`)**:
   - Primary workflow manifest evaluated first by `mu-ci-controller`.
   - Maintain `.actionrunner/ActionRunner.yml` for backwards compatibility until all worker nodes are upgraded.
   - Define `tasks` using only the 11 registered `mu-ci-tools` compile-time adapters (`cargo-fmt`, `cargo-clippy`, `cargo-test`, `cargo-bench`, `cargo-audit`, `cargo-llvm-cov`, `cargo-miri`, `cargo-publish`, `cargo-build`, `trunk`, `mdformat`).
   - Define `stages` with explicit `depends_on` DAG edges, `tags`, `threads`, `max_concurrent_per_node: 1`, and `exclusive: true` on hardware-sensitive benchmark stages.
   - Define `on_action` mappings for `push`, `pull_request`, `merge`, and `tag`.
1. **Author `.muci/security.yml`**:
   - Primary sandbox profile manifest evaluated first by `mu-ci-worker` and `mu-ci-isolation`.
   - Maintain `.actionrunner/security.yml` alongside for backwards compatibility.
   - Define `default_profile` and named `mu-ci-isolation` profiles (`default_ci`, `k3_hardware_bench`) with bounded `memlock_limit_kb` (never `unlimited`; $\\le 29360128\\text{ KB}$ on `worker-riscv64`), explicit Landlock `read_paths`, `write_paths`, `exec_paths`, and conditional SpacemiT K3 `require_set_ai_thread` / `cpu_affinity_mask` settings.
1. **Preserve Legacy Files Intact**:
   - Keep existing `.actionrunner/ActionRunner.yml`, `.actionrunner/security.yml`, and any remaining `.actionrunner/on-*.yml` files unmodified in version control so currently running production `mu-ci-controller` instances continue executing CI jobs without interruption.

### Phase 1 Per-Repository Verification Checklist

- [ ] `.muci/mu-ci.yml` parses and validates via `ActionRunnerWorkflow::from_yaml` (`version == "3"`, acyclic stage DAG, all task and stage references resolved).
- [ ] `.muci/security.yml` parses and validates via `DesiredSecurityFile::from_yaml_str` and satisfies monotonic privilege intersection ($\\text{Task} \\subseteq \\text{Profile} \\subseteq \\text{Ceiling}$) against `/etc/mu-isolation/ceiling.yml`.
- [ ] Both `.muci/` and legacy `.actionrunner/` directories remain tracked in Git (`git status` shows no unexpected deletions).

______________________________________________________________________

## 2. Phase 2: Production Rollout & Dual-Stack Verification

During Phase 2, the updated `mu-ci-controller` (`node-a`) and `mu-ci-worker` daemons (`node-a`, `node-b`, `worker-riscv64`) are deployed with dual-stack configuration resolution.

### Phase 2 Resolution Precedence

1. **Primary Resolution (`.muci/mu-ci.yml` & `.muci/security.yml`)**:
   - On receiving a GitHub webhook (`push`, `pull_request`, `merge`, `release` / `tag`), `mu-ci-controller` first checks for `.muci/mu-ci.yml` in the checked-out commit.
   - When present, `mu-ci-controller` extracts the minimal event-specific stage DAG via `subgraph_for_event(event)`, resolves `.muci/security.yml` (falling back to `.actionrunner/security.yml`) against the target worker's `/etc/mu-isolation/ceiling.yml`, and dispatches typed `ToolInvocation` tasks to `mu-ci-worker`.
1. **Secondary Resolution (`.actionrunner/ActionRunner.yml`)**:
   - If `.muci/mu-ci.yml` is absent, `mu-ci-controller` checks for `.actionrunner/ActionRunner.yml` and resolves `.actionrunner/security.yml`.
1. **Backward-Compatible Fallback (`.actionrunner/on-<event>.yml`)**:
   - If no declarative v3 workflow is present in an unmigrated branch, `mu-ci-controller` falls back to `.actionrunner/on-<event>.yml` and emits a structured deprecation warning (`WARN legacy_workflow_fallback=true repo=<repo> file=.actionrunner/on-<event>.yml`) to the telemetry aggregator (`observe.ci.example.internal:9090` or `127.0.0.1:4318`).

### Phase 2 Per-Repository Verification Checklist

- [ ] Production `mu-ci-controller` on `node-a` and `mu-ci-worker` instances on `node-a`, `node-b`, and `worker-riscv64` are running the v3-capable release binary installed atomically via `sudo install -m 755`.
- [ ] `/etc/mu-isolation/ceiling.yml` is deployed on `node-a`, `node-b`, and `worker-riscv64` with permissions `0640 root:action-runner`.
- [ ] At least one `push` and `pull_request` pipeline run for the repository completes end-to-end using `.muci/mu-ci.yml` and `.muci/security.yml` with zero `legacy_workflow_fallback` warnings.
- [ ] Exclusive SpacemiT K3 benchmark stages (`exclusive: true`) on `worker-riscv64` verify `active_slots == 0` before execution and enforce bounded `RLIMIT_MEMLOCK` (`29360128 KB`).

______________________________________________________________________

## 3. Phase 3: Removal of Legacy `.actionrunner/` Fallback

Phase 3 begins **only after** Phase 2 is verified in production across all repositories (`jero-runner`, `k3-*` submodules, `mu-ci-*`, `mu-*`).

### Phase 3 Decommissioning Actions

1. **Remove Legacy `.actionrunner/on-*.yml` and Deprecated Files**:
   - Delete `.actionrunner/on-push.yml`, `.actionrunner/on-pullrequest.yml`, `.actionrunner/on-merge.yml`, and `.actionrunner/on-tag.yml` via dedicated cleanup Pull Requests.
   - In repositories fully converted to `.muci/`, archive and remove redundant `.actionrunner/` files once `.muci/` handles 100% of pipeline executions.
1. **Remove Legacy Fallback Parser & Shell Execution Path**:
   - Delete legacy fallback resolution in `mu-ci-controller` and remove any legacy shell-string stage execution paths in `mu-ci-worker`, making `.muci/mu-ci.yml` + `mu-ci-tools` the sole declarative execution path.

### Phase 3 Per-Repository Verification Checklist

- [ ] Zero `legacy_workflow_fallback=true` log events observed across the cluster over a 7-day verification window.
- [ ] Legacy `.actionrunner/on-*.yml` files removed via reviewed PR.
- [ ] Post-removal CI pipeline triggers (`push`, `pull_request`, `merge`, `tag`) succeed exclusively through `.muci/mu-ci.yml`.
