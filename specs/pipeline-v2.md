# Spec: 3-Tier Declarative Execution Pipeline (`mu-ci.yml` / `ActionRunner.yml` v3)

## Location: `specs/pipeline-v2.md`

### Overview

Defines the `version: "3"` 3-tier declarative CI/CD workflow specification (`.muci/mu-ci.yml`, with backward-compatible `.actionrunner/ActionRunner.yml` fallback) enforced by `mu-ci-controller`, `mu-ci-worker`, and `mu-ci-tools`. Replaces legacy per-event `version: "2"` shell-script files (`on-push.yml`, `on-pullrequest.yml`, `on-merge.yml`, `on-tag.yml`) via the mandatory 3-Phase Configuration Migration (`docs/migration.md`).

______________________________________________________________________

### 1. Tier 1: `tasks` — Typed Compile-Time Tool Invocations

Each entry in `tasks` defines a reusable, immutable execution unit mapped to a compile-time `ToolAdapter` registered in `mu-ci-tools::ToolRegistry`. Arbitrary shell execution (`sh -c`, `bash -c`, inline scripts, command chaining `;`, `&&`, `||`, pipes `|`, redirection `>`, `<`, and subshell expansion `$()`, `` ` ``) is rejected at parse and validation time (`ToolValidationError`).

#### Registered `mu-ci-tools` Adapters (11 Built-In Adapters)

| Tool Identifier  | Canonical Binary / Subcommand | Purpose & Validation Rules                                                                                           |
| :--------------- | :---------------------------- | :------------------------------------------------------------------------------------------------------------------- |
| `cargo-fmt`      | `cargo fmt`                   | Rust formatting check (`--all`, `--check`, `-p <pkg>`).                                                              |
| `cargo-clippy`   | `cargo clippy`                | Static analysis and lint gate (`--workspace`, `--all-targets`, `-- -D warnings`).                                    |
| `cargo-test`     | `cargo test`                  | Unit, integration, and doc test execution (`--workspace`, `--lib`, `--bins`, `--tests`, `-- --test-threads`).        |
| `cargo-bench`    | `cargo bench`                 | Criterion / native micro-benchmark execution (`--workspace`, `--benches`, `--no-run`).                               |
| `cargo-audit`    | `cargo audit`                 | RustSec advisory database verification (`--db /var/lib/action-runner/.cargo/advisory-db`, `--no-fetch`, `-D`).       |
| `cargo-llvm-cov` | `cargo llvm-cov`              | LLVM source-based code coverage (`--workspace`, `--summary-only`, `--lcov`, `--fail-under-lines`).                   |
| `cargo-miri`     | `cargo +nightly miri test`    | Undefined-behavior and stacked-borrows verification (`-p <pkg>`, `--lib`).                                           |
| `cargo-publish`  | `cargo publish`               | Internal crate publishing restricted exclusively to `--registry mu-registry`.                                        |
| `cargo-build`    | `cargo build`                 | Workspace and target compilation (`--workspace`, `--release`, `--target`).                                           |
| `trunk`          | `trunk build`                 | WebAssembly SPA asset compilation and content-hash bundling (`build --release`, `--dist`).                           |
| `mdformat`       | `mdformat`                    | CommonMark + GFM specification formatting (`--check`, `--wrap keep`, relative path validation within `working_dir`). |

#### `TaskConfig` Schema

```yaml
tasks:
  <task-id>:
    tool: "<one of the 11 registered mu-ci-tools adapters>"
    args: ["<validated-arg-1>", "<validated-arg-2>"]
    working_dir: "code"                # Optional relative subdirectory (rejects '..' traversal)
    env:                               # Optional environment overrides (denylists LD_PRELOAD, PATH, etc.)
      KEY: "VALUE"
    timeout_seconds: 900               # Bounded execution timeout
    sandbox_profile: "k3_hardware_bench" # References a profile in .muci/security.yml or .actionrunner/security.yml
```

______________________________________________________________________

### 2. Tier 2: `stages` — Directed Acyclic Graph (DAG) & Node Scheduling

Each stage in `stages` groups an ordered sequence of `tasks`, declares upstream stage dependencies (`depends_on`), specifies worker node routing selectors (`arch`, `tags`, `toolchain`), and enforces node-level concurrency invariants.

#### `StageConfig` Schema & Scheduling Invariants

- **`id` & `name`**: Unique stage identifier and human-readable display name.
- **`tasks`**: Non-empty ordered list of `<task-id>` keys defined in Tier 1. Tasks within a stage execute sequentially in the same workspace; the stage aborts immediately on the first non-zero task exit code.
- **`depends_on`**: Upstream `<stage-id>` edges forming a strict Directed Acyclic Graph (DAG). `mu-ci-controller` validates acyclicity and computes topological ordering via Kahn's algorithm before scheduling.
- **Target Node Selectors (`tags`, `arch`, `toolchain`)**:
  - `arch: x86_64 | riscv64` (or capability `tags` such as `["k3"]`, `["docs"]`, `["fmt"]`, `["clippy"]`, `["audit"]`) routes stages to matching worker nodes (`node-a`, `node-b`, `worker-riscv64`).
  - `toolchain`: Optional toolchain selector (`llvm-24`, `gcc-17`, `gcc-17-spacemit-x100`).
- **Single-Slot Stage Execution & Exclusive Hardware Isolation**:
  - `max_concurrent_per_node: 1` (default `1`): Enforces `max_slots = 1` per worker node while bounding intra-stage parallelism via `threads` (`CARGO_BUILD_JOBS`, `RAYON_NUM_THREADS`, `RUST_TEST_THREADS`: `2` on `node-a`, `6` on `node-b`, `6` on `worker-riscv64`).
  - `exclusive: true` (alias `isolate_node: true`): Guarantees sole hardware residency on the target worker node before stage dispatch, preventing cross-job cache, memory-bandwidth, or PMU counter contention during `cargo-bench` and SpacemiT K3 (`X100` / `A100`) evaluations.

______________________________________________________________________

### 3. Tier 3: `on_action` — Event Triggers & Subgraph Projection

`on_action` maps repository lifecycle events (`push`, `pull_request`, `merge`, `tag` / `release`) to the list of target stage IDs required for that event:

```yaml
on_action:
  push: ["docs-format", "fmt-check", "clippy-check", "audit-check", "coverage-check", "miri-check", "bench-run"]
  pull_request: ["docs-format", "fmt-check", "clippy-check", "audit-check", "coverage-check", "miri-check", "bench-run"]
  merge: ["docs-format", "fmt-check", "clippy-check", "audit-check", "coverage-check", "miri-check", "bench-run", "build-release"]
  tag: ["build-release"]
```

When an event arrives, `ActionRunnerWorkflow::subgraph_for_event(event)` performs a backward breadth-first traversal from the listed target stages across `depends_on` edges, pruning unreferenced stages and tasks to produce the minimal self-contained execution DAG.
