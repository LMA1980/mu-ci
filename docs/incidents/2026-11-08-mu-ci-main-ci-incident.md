# Incident Report: `LMA1980/mu-ci` `main` CI Failure in `test-check` (`job-1794100779-c5ce7375`)

- **Date**: 2026-11-08
- **Status**: Resolved (Fix PR opened on `LMA1980/mu-ci-worker` #50 and `LMA1980/mu-ci`)
- **Affected Repository**: `LMA1980/mu-ci` (`main` branch, commit `c5ce737527fe90d22695cf17e17eb8828f91d812`)
- **Failed Job ID**: `job-1794100779-c5ce7375` (`test-check` stage)

______________________________________________________________________

## 1. Incident Summary & Impact

On 2026-11-08 at 01:23:15 UTC, the `main` branch CI pipeline for `LMA1980/mu-ci` (`job-1794100779-c5ce7375`, commit `c5ce7375`) failed during the `test-check` stage (`cargo test --workspace`) dispatched to a Linux worker node with error:

```text
thread 'test_scenario_c_outbound_write_timeout_on_stalled_socket' (515195) panicked at worker/tests/tunnel_liveness_test.rs:362:5:
Expected worker to fail with write timeout error, but got Ok(())
```

### Impact

- Automated Continuous Integration on `mu-ci`'s `main` branch entered a `failure` state following the merge of PR #1 (`feat(m8): bootstrap mu-ci Cargo workspace and pipeline specifications`).
- Downstream stages (`coverage-check` and `miri-check`) remained `Pending` and could not execute on `main`.

______________________________________________________________________

## 2. Root Cause Analysis

1. **Kernel TCP Socket Buffer Autotuning vs. Mock Registration Payload**:
   In `worker/tests/tunnel_liveness_test.rs`, `test_scenario_c_outbound_write_timeout_on_stalled_socket` validates that when the coordinator's TCP window stalls, outbound frame transmission times out within `write_timeout` (2 seconds). The test set `config.tags = vec!["x".repeat(3_000_000)]` (3 MB). On Linux hosts, the kernel's dynamic TCP autotuning (`tcp_wmem`) can allocate up to `4,194,304` bytes (4.19 MB) for `SO_SNDBUF`. Under full workspace concurrent test execution (`cargo test --workspace`), the 3 MB frame fit into the expanded send buffer, allowing `ws_sender.send` to complete immediately without blocking.

1. **Premature Mock Server Disconnection**:
   The mock server task slept for a fixed duration of 15 seconds before terminating and dropping the WebSocket stream. Because the registration send succeeded, the worker entered its inbound receive loop. When the mock server exited at 15 seconds and closed the TCP connection, `ws_receiver.next()` yielded `None` (EOF), causing the worker to exit cleanly with `Ok(())`. The test asserted that `res.is_err()`, which failed and triggered a panic.

1. **Submodule Branch Tracking in `.gitmodules`**:
   The parent `mu-ci` repository's `.gitmodules` file tracked `branch = feat/m8-naming-alignment` for all submodules (`code/controller`, `code/worker`, `code/ipc`, `code/tools`, `code/isolation`, `code/web`). Once the individual submodule pull requests merged into `main`, the submodule pointers in `mu-ci` remained pinned to pre-merge feature commits.

1. **How the Regression Reached `main`**:
   PR #1 in `mu-ci` was merged after submodule alignment. While isolated test runs on macOS passed because Darwin loopback socket buffers default to 128 KB, Linux execution with larger TCP window limits exposed the payload size insufficiency during full multi-threaded workspace runs on `main`.

______________________________________________________________________

## 3. Remediation & Preventive Controls

### Immediate Remediation

1. **Worker Outbound Write Timeout Hardening (`LMA1980/mu-ci-worker#50`)**:
   - Increased the mock registration payload in `test_scenario_c_outbound_write_timeout_on_stalled_socket` from 3 MB to 10 MB (`10_000_000` bytes), safely exceeding the maximum Linux kernel `tcp_wmem` buffer ceiling (~4.19 MB) to guarantee that socket writes block under all system configurations.
   - Increased the mock server sleep duration to 30 seconds to prevent socket teardown prior to write timeout expiration.
   - Added dual environment variable resolution (`MU_CI_WRITE_TIMEOUT_SECS` / `ACTION_RUNNER_WRITE_TIMEOUT_SECS`) in `WorkerAgent`.
1. **Submodule Alignment & Default Branch Tracking (`LMA1980/mu-ci`)**:
   - Updated `.gitmodules` across all 6 submodules to track default branch `main`.
   - Advanced submodule pointers to the latest commits including the write timeout fix.
1. **Branching & PR Discipline**:
   - Created feature branch `feat/fix-main-ci-test-timeout` strictly avoiding direct commits to `main`.
   - Executed the full Pre-PR Local CI Gate Sequence (`fmt` -> `docs` -> `clippy` -> `test` -> `miri`).
   - Submitted a Pull Request against `main` with full metadata and assigned `LMA1980`.

### Preventive Controls Against Recurrence

1. **Deterministic Stalled Socket Testing**:
   - Ensure socket stall tests write payloads at least 2.5x larger than standard Linux `tcp_wmem` maximums (>= 10 MB) or set explicit `SO_SNDBUF` ceilings to avoid dependence on platform-specific TCP buffer tuning.
1. **Submodule Default Branch Tracking**:
   - Ensure repository `.gitmodules` files reference default branch `main` once bootstrapping feature branches are merged.
