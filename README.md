# mu-ci

<p align="center">
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-ci/status.svg" alt="CI Status"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-ci/score.svg" alt="Quality Score"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-ci/coverage.svg" alt="Coverage"/></a>
  <a href="https://hook.lma1980.duckdns.org/jobs"><img src="https://hook.lma1980.duckdns.org/badges/LMA1980/mu-ci/benchmark.svg" alt="Benchmark"/></a>
</p>

______________________________________________________________________

Distributed Continuous Integration (CI) and verification fabric for the K3 ecosystem. `mu-ci` orchestrates declarative 3-tier execution pipelines (`.muci/mu-ci.yml` / `.actionrunner/ActionRunner.yml`), enforces Kahn DAG stage topological scheduling, manages typed compile-time tool invocations without arbitrary shell execution, applies 3-layer monotonic privilege attenuation sandboxing, and delivers real-time execution monitoring across heterogeneous `x86_64` and SpacemiT K3 (`riscv64gc`) worker clusters.

See the key design documents:

- [3-Tier Declarative Pipeline Specification (`specs/pipeline-v2.md`)](specs/pipeline-v2.md)
- [3-Phase Configuration Migration Guide (`docs/migration.md`)](docs/migration.md)
- [ADR-050: Privilege Attenuation & Sandboxing (`docs/adr/ADR-050-mu-isolation-privilege-attenuation.md`)](docs/adr/ADR-050-mu-isolation-privilege-attenuation.md)

______________________________________________________________________

## Workspace Submodules

| Submodule                             | Target                 | License                                       | Responsibility                                                                                        |
| ------------------------------------- | ---------------------- | --------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| [`mu-ci-controller`](code/controller) | native                 | AGPL-3.0-only                                 | WebAPI ingress, GitHub webhooks, DAG stage scheduler, HA failover, job store, and SVG scorecards      |
| [`mu-ci-worker`](code/worker)         | native                 | AGPL-3.0-only                                 | Distributed stage execution agent, slot lease manager, and stage runner daemon                        |
| [`mu-ci-ipc`](code/ipc)               | native                 | LGPL-3.0-only WITH LGPL-3.0-linking-exception | Zero-copy framed IPC protocol library using Apache Fory, Apache Arrow, and zerocopy                   |
| [`mu-ci-tools`](code/tools)           | native                 | LGPL-3.0-only WITH LGPL-3.0-linking-exception | Compile-time modular `ToolAdapter` registry and 11 typed tool wrappers (rejects arbitrary shell exec) |
| [`mu-ci-isolation`](code/isolation)   | native                 | LGPL-3.0-only WITH LGPL-3.0-linking-exception | 3-layer monotonic privilege attenuation engine and dual-backend `KernelSandbox`/`HypervisorSandbox`   |
| [`mu-ci-web`](code/web)               | wasm32-unknown-unknown | AGPL-3.0-only                                 | Leptos WASM Queue Monitor & Job Drill-Down Web UI for SpacemiT K3                                     |

## Architecture & Lifecycle

```text
[ Webhook Event (Push / PR / Tag) ]
   │
   ▼
[ Ingress Controller: Resolve Declarative Workflow (.muci/mu-ci.yml) ]
   │
   ▼
[ DAG Projection (subgraph_for_event) & Topological Kahn Scheduling ]
   │
   ▼
[ Discover Idle Worker Slot (max_slots = 1) ] ──(Busy / Exclusive Wait)──> [ Scheduler Requeue ]
   │
   ▼
[ Acquire Worker Slot Lease ]
   │
   ▼
[ Monotonic Sandbox Attenuation (Task ⊆ Profile ⊆ Ceiling) ]
   │
   ▼
[ Execute Typed Tool Invocations (mu-ci-tools Registry) ]
   ├── Stage Success ──> [ Advance Downstream DAG Stages ]
   └── Stage Failure ──> [ Abort Dependent Stages & Emit Failure Scorecard ]
   │
   ▼
[ Complete Job & Notify Telemetry Hub ]
```

## Three-Phase Configuration Migration

Workflow configuration adheres to the platform 3-phase configuration migration policy:

- **Phase 1 (Dual-Stack Introduction)**: Support primary `.muci/mu-ci.yml` and `.muci/security.yml` alongside legacy `.actionrunner/ActionRunner.yml` and `.actionrunner/security.yml`.
- **Phase 2 (Production Rollout & Verification)**: Deploy updated controllers and worker daemons across cluster nodes (`node-a`, `node-b`, `worker-riscv64`), resolving `.muci/` with precedence and falling back to `.actionrunner/`.
- **Phase 3 (Decommissioning)**: Decommission legacy `.actionrunner/` files and fallback parsing paths once all repositories run on `.muci/`.

## Building & Verification

```bash
# Pre-PR local verification gate
(cd code && cargo fmt --all -- --check)
./scripts/format_docs.sh --check
(cd code && cargo clippy --workspace --all-targets -- -D warnings)
(cd code && cargo test --workspace)
cargo agents plugin validate Symposium.toml --no-check-crates
```

## License

`mu-ci` (workspace root, controller, worker, and web frontend) is licensed under the GNU Affero General Public License v3.0 (**AGPL-3.0-only**). See [LICENSE](LICENSE).

In accordance with the K3 platform Two-Tier Licensing Policy:

- Core libraries, IPC crates, tool registries, and sandboxing engines ([`mu-ci-ipc`](code/ipc), [`mu-ci-tools`](code/tools), and [`mu-ci-isolation`](code/isolation)) are licensed under the GNU Lesser General Public License v3.0 with the LGPL-3.0 Linking Exception (**LGPL-3.0-only WITH LGPL-3.0-linking-exception**).
- Autonomous daemons, controllers, and web frontends ([`mu-ci-controller`](code/controller), [`mu-ci-worker`](code/worker), [`mu-ci-web`](code/web)) are licensed under **AGPL-3.0-only**.
