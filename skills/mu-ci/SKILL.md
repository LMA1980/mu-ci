______________________________________________________________________

## name: mu-ci description: "Orchestrate distributed CI execution, DAG stage scheduling, typed compile-time tool adapters, monotonic sandbox privilege attenuation, and Web UI monitoring across mu-ci-controller, mu-ci-worker, mu-ci-ipc, mu-ci-tools, mu-ci-isolation, and mu-ci-web."

# `mu-ci` Distributed Continuous Integration Workspace

## 1. Architecture Overview & Submodules

`mu-ci` is a modular, high-performance distributed continuous integration and verification fabric tailored for heterogeneous hardware architectures (including `x86_64` and SpacemiT K3 `riscv64gc`). The workspace integrates six dedicated crates:

- **`code/controller` (`mu-ci-controller`)**: Central ingress controller, GitHub webhook receiver, Directed Acyclic Graph (DAG) stage scheduler, active-standby HA leader election, job state persistence, and real-time SVG status scorecards.
- **`code/worker` (`mu-ci-worker`)**: High-performance stage execution agent running on worker nodes, managing slot leases, workspace synchronization, and sandbox lifecycle.
- **`code/ipc` (`mu-ci-ipc`)**: Shared zero-copy wire protocol and IPC serialization library implementing Apache Fory and Apache Arrow framing.
- **`code/tools` (`mu-ci-tools`)**: Modular `ToolAdapter` compile-time trait registry rejecting arbitrary shell execution and providing 11 typed tool wrappers (`cargo-fmt`, `cargo-clippy`, `cargo-test`, `cargo-bench`, `cargo-audit`, `cargo-llvm-cov`, `cargo-miri`, `cargo-publish`, `cargo-build`, `trunk`, `mdformat`).
- **`code/isolation` (`mu-ci-isolation`)**: 3-layer monotonic privilege attenuation engine and dual-backend sandboxing (`KernelSandbox` via Landlock/cgroups/namespaces and `HypervisorSandbox` via `/dev/kvm`).
- **`code/web` (`mu-ci-web`)**: Reactive Leptos WASM frontend for cluster queue monitoring, stage log streaming, and DAG drill-down visualization.

## 2. Single-Slot Stage Execution & Hardware Invariants

- **Single-Slot Scheduling (`max_slots = 1`)**: Every worker node enforces `max_slots = 1` (at most one active CI stage per node at a time) while tuning thread parallelism via thread budget variables (`CARGO_BUILD_JOBS`, `RAYON_NUM_THREADS`, `RUST_TEST_THREADS`: `2` on `node-a`, `6` on `node-b`, `6` on `worker-riscv64`).
- **Exclusive Stage Probe (`exclusive: true`)**: Benchmark and hardware-eval stages enforce node-level isolation ("drain before dispatch"), ensuring `active_slots == 0` and zero pending claims before stage activation.
- **SpacemiT K3 Heterogeneous Clustering (`worker-riscv64`)**:
  - Cluster 0 (`X100`, cores `0–7`, mask `0x00FF`): General purpose Out-of-Order cores (`VLEN = 256`), supports KVM hardware virtualization (`RISC-V H` extension).
  - Cluster 1 (`A100`, cores `8–15`, mask `0xFF00`): AI/Matrix In-Order cores (`VLEN = 1024`), supports `XSMTVDotII` matrix operations and requires conditional `/proc/set_ai_thread` registration.
- **Bounded Memlock Ceiling**: `RLIMIT_MEMLOCK` is strictly capped at `29360128 KB` (28 GiB on 32 GB RAM hardware), preserving safe headroom for kernel memory slab allocation and SSH responsiveness.

## 3. Monotonic Sandbox Privilege Attenuation

- Effective execution privileges are evaluated as the strict intersection of three layers:
  $$\\text{EffectivePolicy} = \\text{HostCeiling} \\cap \\text{DesiredProfile} \\cap \\text{TaskOverride}$$
- If any dimension in Layer 2 or Layer 3 requests access beyond the ceiling, execution fails closed with `SecurityGrantDenied`.
- Workloads select between `KernelSandbox` (zero-jitter bare-metal execution with Landlock LSM and cgroupv2 limits) and `HypervisorSandbox` (`/dev/kvm` microVMs for untrusted workloads).

## 4. Three-Phase Configuration Migration

- **Phase 1 (Dual-Stack Introduction)**: Repositories introduce `.muci/mu-ci.yml` and `.muci/security.yml` while maintaining backward-compatible `.actionrunner/ActionRunner.yml` and `.actionrunner/security.yml`.
- **Phase 2 (Production Verification)**: Workers and controllers resolve `.muci/` with precedence, falling back to `.actionrunner/` when absent.
- **Phase 3 (Decommissioning)**: Legacy `.actionrunner/` files and fallback parsers are removed once all services run on `.muci/`.
