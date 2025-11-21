# Canonicalize Writable Roots for Sandbox

Read the ExecPlan implementation guide at
[ExecPlan_implementor_instructions.md](/home/dank/repo/codex/codex-rs/ExecPlan_implementor_instructions.md)
now. Follow its instructions and keep this document updated as work proceeds.
Also read [sandbox-writable-root.md](/home/dank/repo/codex/codex-rs/sandbox-writable-root.md)
for additional context on the underlying issue and current behavior.


## Purpose / Big Picture

Users running in workspace-write sandbox mode supply extra writable roots (e.g. a game Mods folder on WSL/DrvFS). Today those paths are passed to Landlock without canonicalization, so the rule silently fails to install and writes are denied unless the command is re-run with escalated permissions. After this change, a workspace-write session should be able to write to configured extra roots without escalation; the behavior is proved by a failing test that passes once the canonicalization fix lands.

## Progress

- [ ] Read ExecPlan_implementor_instructions.md (see link above) and follow its instructions carefully as you implement this ExecPlan.
- [ ] Map current sandbox writable-root handling in `core`, `protocol`, and `linux-sandbox`, noting where paths flow and where canonicalization is missing.
- [ ] Add a unit/integration test that uses a non-canonical writable root and demonstrates the sandbox denying writes before the fix.
- [ ] Implement canonicalization (and any logging) so the sandbox allowlist includes the configured root; update docs if APIs change.
- [ ] Run formatting, linting, and targeted tests; capture results and update this plan with timestamps.

## Surprises & Discoveries

- Observation: _None yet; record unexpected behaviors or error outputs as they occur._

## Decision Log

- Decision: _To be recorded when design or implementation choices are made._

  Rationale: _Pending._

  Date/Author: _Pending._

## Outcomes & Retrospective

Summarize the end state once work completes: which scenarios now succeed, remaining gaps, and lessons for future sandbox changes.

## Context and Orientation

The sandbox writable-root bug arises in workspace-write mode when users configure extra writable roots. `core/src/config/mod.rs::ConfigToml::derive_sandbox_policy` builds a `SandboxPolicy::WorkspaceWrite` using `sandbox_workspace_write.writable_roots` directly. The policy is later consumed by Landlock in `linux-sandbox/src/landlock.rs::apply_sandbox_policy_to_current_thread`, which calls `SandboxPolicy::get_writable_roots_with_cwd` (defined in `protocol/src/protocol.rs`) and passes those roots to `landlock::path_beneath_rules`. On WSL/DrvFS, `path_beneath_rules` needs canonical, openable paths; otherwise the rule fails to install and writes are denied. CLI overrides (`additional_writable_roots`) are canonicalized in `Config::load_from_base_config_with_overrides`, but config-defined `writable_roots` are not, causing the mismatch.

Key terms: Landlock is a Linux security module that restricts file system access; a “writable root” is a directory where writes are allowed. “Canonicalization” resolves symlinks and removes `..` or relative segments to yield a stable path string that Landlock can open on all filesystems, including DrvFS.

## Plan of Work

First, trace the existing path flow: confirm how `sandbox_workspace_write.writable_roots` from the config reach `SandboxPolicy`, how `additional_writable_roots` are canonicalized, and how `get_writable_roots_with_cwd` augments the list (cwd, /tmp, TMPDIR). Next, design a deterministic repro using a non-canonical absolute path (e.g. a symlink or `..` segment) added as a writable root so Landlock receives a path that needs canonicalization; add a test in `linux-sandbox/tests/suite/landlock.rs` or a focused unit test in `core/src/config/mod.rs` that asserts the policy stores a canonical path and that sandboxed commands can write to it. Then change `ConfigToml::derive_sandbox_policy` (or the appropriate construction point) to canonicalize config-provided `writable_roots`, mirroring the treatment of `additional_writable_roots`; on failure, fall back to the original path so behavior remains permissive. Optionally add logging around Landlock rule installation to surface skipped roots. Update documentation in `docs/` if the user-facing behavior or configuration expectations change. Finally, run `just fmt`, targeted `just fix -p` commands for touched crates, and the relevant cargo tests to prove the regression is fixed.

## Concrete Steps

Work in `/home/dank/repo/codex/codex-rs`.

1. Inspect sandbox policy construction and writable-root expansion:
   - `rg "derive_sandbox_policy" core/src/config/mod.rs` then read the function body.
   - `rg "get_writable_roots_with_cwd" protocol/src/protocol.rs` to see default additions and structure.
   - `rg "path_beneath_rules" -n linux-sandbox/src/landlock.rs` to note how roots are applied.
2. Create a failing test capturing the missing canonicalization:
   - Option A (integration): in `linux-sandbox/tests/suite/landlock.rs`, create a temp dir, a symlinked path such as `tempdir/path/../alias`, pass the non-canonical path in `writable_roots`, run a sandboxed `echo > <real file>` using `run_cmd`, and assert it currently panics because writes are denied.
   - Option B (unit): in `core/src/config/mod.rs` tests, build a `ConfigToml` with `sandbox_workspace_write.writable_roots` containing a symlinked/relative absolute path and assert `derive_sandbox_policy` returns a `SandboxPolicy::WorkspaceWrite` whose roots are canonicalized; this should fail before the fix.
   - Choose the option that deterministically fails on Linux without WSL-specific behavior; document the choice in the Decision Log.
3. Implement the fix:
   - In `ConfigToml::derive_sandbox_policy`, canonicalize each configured writable root similarly to how `additional_writable_roots` are handled (absolute path resolution, `std::fs::canonicalize` with fallback to original on error).
   - Ensure new roots are deduplicated where appropriate and keep existing defaults (cwd, /tmp, TMPDIR) unchanged.
   - If adding logging for Landlock rule installation, thread a sensible log message when `path_beneath_rules` fails for a root without aborting the whole install.
4. Update tests/documentation:
   - Adjust or add snapshots/expectations if the test location requires it; prefer `pretty_assertions::assert_eq` in unit tests.
   - Note any doc changes in `docs/` if configuration semantics or guidance shifts.
5. Run maintenance commands:
   - `just fmt` (required after Rust edits).
   - `just fix -p codex-core` and/or `just fix -p codex-linux-sandbox` depending on touched crates.
   - Run the new failing test’s crate: e.g. `cargo test -p codex-linux-sandbox` or `cargo test -p codex-core -- tests::name`.
   - If core/protocol/common changes are made, ask before running `cargo test --all-features`; capture outputs for Acceptance.

## Validation and Acceptance

Acceptance hinges on demonstrating that a workspace-write sandbox can write to an extra writable root without escalation. The new test must fail before the fix and pass after. Run the crate-level test suite where the test lives; expect all tests to pass. If logging is added, optionally confirm a run shows no warnings about skipped writable roots when the path is canonicalizable. Document the exact commands and their success outputs in this file.

## Idempotence and Recovery

Steps are safe to rerun: canonicalizing paths is pure, and tests create temporary directories. If a test run leaves temp files, rerun `cargo clean -p codex-linux-sandbox` only if necessary. If canonicalization causes a path to disappear due to a transient filesystem issue, fall back to the original path as implemented; rerunning will reuse the fallback. No destructive migrations are involved.

## Code Pointers

- core/src/config/mod.rs:780-850 — `ConfigToml::derive_sandbox_policy` builds `SandboxPolicy` from config, currently copies `writable_roots` verbatim.
- core/src/config/mod.rs:1000-1050 — `Config::load_from_base_config_with_overrides` canonicalizes `additional_writable_roots`, a template for the fix.
- protocol/src/protocol.rs:362-419 — `SandboxPolicy::get_writable_roots_with_cwd` augments and shapes writable roots before Landlock sees them.
- linux-sandbox/src/landlock.rs:30-90 — `apply_sandbox_policy_to_current_thread` and `install_filesystem_landlock_rules_on_current_thread` apply Landlock rules using provided roots.
- linux-sandbox/tests/suite/landlock.rs:1-200 — existing sandbox integration tests and helper `run_cmd` suitable for reproducing the write-denied failure.

## Artifacts and Notes

- Reference bug description: writes to `/mnt/g/Steam/steamapps/common/Stardew Valley/Mods` are denied in workspace-write mode unless escalated, implying the extra writable root is absent from the Landlock allowlist.
- Capture future command outputs (test failures/passes) here once executed.

## Interfaces and Dependencies

Public-facing types: `codex_core::config::ConfigToml`, `codex_core::protocol::SandboxPolicy`, and `codex_linux_sandbox` integration with Landlock. After changes, `SandboxPolicy::WorkspaceWrite` should contain canonicalized writable roots when sourced from config. Any logging should use existing tracing facilities. Tests rely on `tokio`, `tempfile`, and `landlock` via crate dependencies already present. No new external dependencies are expected.
