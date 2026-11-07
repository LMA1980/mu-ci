# ADR-050: mu-ci-isolation 3-Layer Monotonic Privilege Attenuation & Sandbox Backends

**Status**: Accepted

## Context

Self-hosted CI/CD and continuous delivery agents running across the cluster (`node-a`, `node-b`, `worker-riscv64`) execute untrusted or semi-trusted compilation, verification, and hardware benchmark workloads. Relying on coarse OS service user permissions (`action-runner`) without per-repository and per-task privilege attenuation creates three unacceptable risks:

1. **Privilege Escalation via Repository Configuration**: A repository workflow could request write access outside its workspace, unrestricted network egress, or unbounded process/memory limits.
1. **System-Wide Hard Lockup via Unbounded `RLIMIT_MEMLOCK` (`AGENTS.md §5`)**: Setting `RLIMIT_MEMLOCK` to `unlimited` on the 32 GB SpacemiT K3 board (`worker-riscv64`) allows model or benchmark allocations to lock 100% of physical RAM, starving the Linux kernel slab allocator, page tables, and `sshd`.
1. **Heterogeneous Hardware Isolation vs. Zero-Jitter Requirements (`AGENTS.md §6`)**: Workloads pinning to SpacemiT K3 Cluster 1 (`A100` cores `8–15`, mask `0xFF00`) require direct access to `XSMTVDotII` matrix instructions (`smt.vpack.vv`, `smt.vmadot`), zero-jitter hardware PMU counters, and conditional registration with `/proc/set_ai_thread` on the `spacemit-hmp` kernel, whereas untrusted workloads require strong guest-kernel isolation via `/dev/kvm`.

## Decision

We adopt `mu-ci-isolation`'s **3-Layer Monotonic Privilege Attenuation Model** ($\\text{Task} \\subseteq \\text{Profile} \\subseteq \\text{Ceiling}$) with fail-closed `SecurityGrantDenied` semantics and dual isolation backends (`KernelSandbox` and `HypervisorSandbox`).

______________________________________________________________________

### 1. Three-Layer Monotonic Privilege Attenuation ($\\text{Task} \\subseteq \\text{Profile} \\subseteq \\text{Ceiling}$)

Effective sandbox privileges are computed as the strict intersection of three declarative layers:

$$\\text{EffectivePolicy} = \\text{HostCeiling} \\cap \\text{DesiredProfile} \\cap \\text{TaskOverride}$$

1. **Layer 1 — Operator Host Ceiling (`/etc/mu-isolation/ceiling.yml`)**:
   - Provisioned by the cluster operator on each physical host (`0640 root:action-runner`), immutable by CI/CD workflows.
   - Defines the absolute upper bound on allowed sandbox backends, normalized filesystem read/write/exec path prefixes, allowed IPv4 egress CIDRs, `max_memlock_kb` (hard-capped at `MAX_K3_MEMLOCK_KB = 29,360,128 KB` / 28 GiB), `max_cpu_cores`, `max_memory_mb`, `max_pids`, and `allow_spacemit_ai_thread`.
1. **Layer 2 — Repository Desired Security Profile (`.muci/security.yml` / `.actionrunner/security.yml`)**:
   - Versioned inside each repository; declares `default_profile` and named profiles (e.g., `default_ci`, `k3_hardware_bench`).
   - Every grant in a profile must be a subset of or equal to Layer 1 (`HostCeiling`).
1. **Layer 3 — Per-Task / Stage Attenuation Override (`.muci/mu-ci.yml` / `.actionrunner/ActionRunner.yml`)**:
   - Tasks select a named `sandbox_profile` from Layer 2 and may optionally supply `TaskSandboxConfig` overrides that **only tighten** (never expand) Layer 2 grants.

#### Fail-Closed Monotonic Intersection Law (`SecurityGrantDenied`)

`mu-ci-isolation::policy::intersect::resolve_effective_policy` validates every dimension before spawning a process or microVM. If Layer 2 exceeds Layer 1 or Layer 3 exceeds Layer 2 on any axis, execution aborts immediately prior to `fork`/`exec` or KVM VM creation with `IsolationError::SecurityGrantDenied { violated_layer, offending_field, requested_value, ceiling_value }`:

| Policy Dimension        | Monotonic Attenuation Rule ($\\text{Child} \\subseteq \\text{Parent}$)                                                                   |
| :---------------------- | :--------------------------------------------------------------------------------------------------------------------------------------- |
| `backend`               | $\\text{profile.backend} \\in \\text{ceiling.allowed_backends}$ and $\\text{task.backend} \\in {\\text{None}, \\text{profile.backend}}$  |
| `read_paths`            | Every normalized path (rejecting `..`) must be equal to or a subdirectory of an allowed parent prefix.                                   |
| `write_paths`           | Every normalized path (rejecting `..`) must be equal to or a subdirectory of an allowed parent write prefix.                             |
| `exec_paths`            | Every normalized path (rejecting `..`) must be equal to or a subdirectory of an allowed parent exec prefix.                              |
| `network_egress_cidrs`  | `Offline` $\\subseteq$ `RestrictedCidrs(child)` $\\subseteq$ `allowed_egress_cidrs(parent)` via bitwise IPv4 CIDR subnet containment.    |
| `memlock_limit_kb`      | $0 < \\text{task} \\le \\text{profile} \\le \\min(\\text{ceiling.max_memlock_kb}, 29360128)$. `0` and `u64::MAX` (`unlimited`) rejected. |
| `max_pids`              | $0 < \\text{task} \\le \\text{profile} \\le \\text{ceiling.max_pids}$.                                                                   |
| `max_memory_mb`         | $0 < \\text{task} \\le \\text{profile} \\le \\text{ceiling.max_memory_mb}$.                                                              |
| `cpu_affinity_mask`     | Popcount $\\le \\text{ceiling.max_cpu_cores}$, mask $\\ne 0$, and $(\\text{task_mask} \\mathbin{&} \\sim\\text{profile_mask}) == 0$.     |
| `require_set_ai_thread` | $\\text{task} \\implies \\text{profile} \\implies \\text{ceiling.allow_spacemit_ai_thread}$.                                             |

______________________________________________________________________

### 2. Canonical `/etc/mu-isolation/ceiling.yml` Host Templates

#### A. `worker-riscv64` (SpacemiT K3 `riscv64gc`, 16 Cores, 32 GB RAM)

```yaml
node_id: "worker-riscv64"
arch: "riscv64gc"
allowed_backends:
  - Kernel
  - Hypervisor
allowed_read_prefixes:
  - /usr
  - /bin
  - /lib
  - /etc/ssl
  - /var/lib/action-runner
  - /var/lib/k3-store
  - /home/runner/repository
  - /usr/local/lib/ollama/.ollama
  - /opt/llvm-24-spacemit-x100
  - /opt/gcc-17-spacemit-x100
allowed_write_prefixes:
  - /var/lib/action-runner/workspaces
  - /var/lib/k3-store
  - /home/runner/repository/jero-runner/code/target
  - /tmp
allowed_exec_prefixes:
  - /usr/bin
  - /usr/local/bin
  - /bin
  - /var/lib/action-runner/.cargo/bin
  - /var/lib/action-runner/.rustup
  - /var/lib/action-runner/.local/bin
  - /opt/llvm-24-spacemit-x100/bin
  - /opt/gcc-17-spacemit-x100/bin
allowed_egress_cidrs:
  - "192.0.2.0/24"
max_memlock_kb: 29360128
max_cpu_cores: 16
max_memory_mb: 28672
max_pids: 512
allow_spacemit_ai_thread: true
```

#### B. `node-a` (Intel N100 Alder Lake `x86_64`, 4 Cores, Control Plane + Light Worker)

```yaml
node_id: "node-a"
arch: "x86_64"
allowed_backends:
  - Kernel
  - Hypervisor
allowed_read_prefixes:
  - /usr
  - /bin
  - /lib
  - /lib64
  - /etc/ssl
  - /var/lib/action-runner
allowed_write_prefixes:
  - /var/lib/action-runner/workspaces
  - /tmp
allowed_exec_prefixes:
  - /usr/bin
  - /usr/local/bin
  - /bin
  - /var/lib/action-runner/.cargo/bin
  - /var/lib/action-runner/.rustup
  - /var/lib/action-runner/.local/bin
allowed_egress_cidrs:
  - "192.0.2.0/24"
max_memlock_kb: 2097152
max_cpu_cores: 2
max_memory_mb: 6144
max_pids: 256
allow_spacemit_ai_thread: false
```

#### C. `node-b` (Intel Xeon E3 `x86_64`, 8 Threads, Primary `x86_64` Build/Test Worker)

```yaml
node_id: "node-b"
arch: "x86_64"
allowed_backends:
  - Kernel
  - Hypervisor
allowed_read_prefixes:
  - /usr
  - /bin
  - /lib
  - /lib64
  - /etc/ssl
  - /var/lib/action-runner
allowed_write_prefixes:
  - /var/lib/action-runner/workspaces
  - /tmp
allowed_exec_prefixes:
  - /usr/bin
  - /usr/local/bin
  - /bin
  - /var/lib/action-runner/.cargo/bin
  - /var/lib/action-runner/.rustup
  - /var/lib/action-runner/.local/bin
allowed_egress_cidrs:
  - "192.0.2.0/24"
max_memlock_kb: 8388608
max_cpu_cores: 6
max_memory_mb: 24576
max_pids: 512
allow_spacemit_ai_thread: false
```

______________________________________________________________________

### 3. Selection Rules: `KernelSandbox` vs. `HypervisorSandbox`

All three nodes expose `/dev/kvm` (`worker-riscv64` via the RISC-V Hypervisor `H` extension on Cluster 0 `X100` cores `0–7`; `node-a` and `node-b` via Intel VT-x). `mu-ci-isolation` selects the execution backend according to workload hardware invariants:

| Criterion                                                       | `KernelSandbox` (`backend: Kernel`)                                                                                                                                                                                                                                                                                        | `HypervisorSandbox` (`backend: Hypervisor`)                                                                                                       |
| :-------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Isolation Primitives**                                        | Landlock LSM (V3+ filesystem ruleset), `cgroupv2` (`memory.max`, `pids.max`, `cpuset.cpus`), Linux namespaces (`CLONE_NEWUSER`, `CLONE_NEWNS`, `CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_NEWIPC`), `PR_SET_NO_NEW_PRIVS`, and seccomp-BPF.                                                                                    | Hardware-virtualized `/dev/kvm` microVM with dedicated guest kernel, virtio-fs workspace mount, and guest memory ballooning.                      |
| **SpacemiT K3 Cluster 0 (`X100`, cores `0–7`, mask `0x00FF`)**  | Supported (`VLEN = 256`, `vlenb = 32`, Out-of-Order `2.15 GHz`).                                                                                                                                                                                                                                                           | Supported (`RISC-V H` extension hardware virtualization is implemented on Cluster 0 `X100` cores `0–7`).                                          |
| **SpacemiT K3 Cluster 1 (`A100`, cores `8–15`, mask `0xFF00`)** | **Mandatory (`KernelSandbox` only)**: In-Order AI/Matrix cores (`1.80 GHz`, `VLEN = 1024`, `vlenb = 128`) with `XSMTVDotII` (`smt.vpack.vv` `.word 0x6704102b`, `smt.vmadot` `.word 0xe304302b`).                                                                                                                          | **Not Used for A100 Matrix Workloads**: `X100` cores trap with `SIGILL` on `XSMTVDotII`, and A100 scheduling requires host `/proc/set_ai_thread`. |
| **Conditional `/proc/set_ai_thread` Registration**              | When `require_set_ai_thread: true` (or `cpu_affinity_mask & 0xFF00 != 0`) and `/proc/set_ai_thread` exists on the `spacemit-hmp` kernel, `KernelSandbox` writes `SYS_gettid` to `/proc/set_ai_thread` **before** calling `sched_setaffinity`. Gracefully skipped on vanilla kernels where `/proc/set_ai_thread` is absent. | N/A (`HypervisorSandbox` pins vCPU threads exclusively to `X100` mask `0x00FF` on `worker-riscv64` or VT-x cores on `x86_64`).                    |
| **Primary Use Cases**                                           | Deterministic Rust builds (`cargo-fmt`, `cargo-clippy`, `cargo-test`), `cargo-bench` hardware micro-benchmarks, and SpacemiT K3 `X100`/`A100` RVV & IME kernel evaluation requiring zero virtualization jitter.                                                                                                            | High-isolation multi-tenant or external plugin verification requiring guest-kernel boundary separation on `X100` (`0x00FF`) or Intel VT-x.        |

## Consequences

- Every task executed by `mu-ci-worker` or `mu-cd-agent` is mathematically bounded by `/etc/mu-isolation/ceiling.yml`.
- `worker-riscv64` is protected against `RLIMIT_MEMLOCK` exhaustion (`max_memlock_kb: 29360128`), guaranteeing ~3.3 GiB of unpinned physical RAM for the headless Linux kernel and SSH control plane.
- SpacemiT K3 A100 matrix workloads receive deterministic bare-metal `KernelSandbox` execution with automatic conditional `/proc/set_ai_thread` registration.
