# Codex sandbox writable_roots bug (WSL / Landlock)

## What we observed
- In workspace-write mode, writes to the configured extra writable root `/mnt/g/Steam/steamapps/common/Stardew Valley/Mods` are denied by the sandbox (Permission denied / Landlock) even though OS perms are 777.
- The same write succeeds when the command is retried with `with_escalated_permissions: true` (which skips the sandbox). Example: `touch "/mnt/g/Steam/steamapps/common/Stardew Valley/Mods/.codex-write-test-<ts>"` fails normally, succeeds with escalation.
- Mod build with `/p:EnableModDeploy=true` fails at deploy step for the same reason (cannot copy to that folder).

## Where the bug likely is in codex/codex-rs
- Landlock rules are built in `linux-sandbox/src/landlock.rs::apply_sandbox_policy_to_current_thread`, using `SandboxPolicy::get_writable_roots_with_cwd` and feeding them to `install_filesystem_landlock_rules_on_current_thread`.
- `SandboxPolicy` is constructed in `core/src/config/mod.rs::derive_sandbox_policy`. For workspace-write, it copies `[sandbox_workspace_write].writable_roots` **without canonicalizing**.
- CLI `additional_writable_roots` are canonicalized later in `Config::load_from_base_config_with_overrides` (~lines 1000+), but the config-provided `writable_roots` are not.
- On WSL/DrvFS, Landlock often needs canonical, openable paths for `path_beneath_rules`; non-canonical DrvFS paths can fail silently to add the rule. Result: the extra root never makes it into the allowlist, so writes are denied; escalation works because it skips Landlock.

## Proposed fix
1) Canonicalize config `sandbox_workspace_write.writable_roots` when building the `SandboxPolicy` (same treatment as `additional_writable_roots`). If canonicalization fails, fall back to the original path.
2) Optional: log or surface failures from `landlock::path_beneath_rules` when adding writable roots so missing rules aren’t silent.
3) Add regression test: with a temp dir outside cwd, create `SandboxPolicy::WorkspaceWrite { writable_roots: [tempdir], … }`, apply Landlock, then `touch tempdir/ok`; assert success. Candidate locations: `linux-sandbox/tests/suite/landlock.rs` or `exec/tests/suite/sandbox.rs`.

## Repro steps
1) Config in `~/.codex/config.toml`:
   ```toml
   sandbox_mode = "workspace-write"
   [sandbox_workspace_write]
   writable_roots = ["/mnt/g/Steam/steamapps/common/Stardew Valley/Mods"]
   ```
2) Run (without escalation):
   ```bash
   touch "/mnt/g/Steam/steamapps/common/Stardew Valley/Mods/.codex-write-test-$(date +%s)"
   ```
   => Permission denied (Codex sandbox).
3) Run with escalation (`with_escalated_permissions=true`): same command succeeds.

## Rationale
- Execpolicy is not the culprit: it only decides retries without sandbox after a denial. The denial happens because the writable root is missing from the Landlock allowlist.
- Fixing canonicalization should ensure the root is included; logging will catch future silent misses.

## Pointers to relevant code
- `core/src/config/mod.rs::derive_sandbox_policy`
- `core/src/config/mod.rs::load_from_base_config_with_overrides` (canonicalizes `additional_writable_roots`)
- `linux-sandbox/src/landlock.rs`
- Optional test spot: `linux-sandbox/tests/suite/landlock.rs` or `exec/tests/suite/sandbox.rs`
