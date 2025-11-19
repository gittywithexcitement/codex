# Windows Terminal Notifications (WSL 2) – Implementation Plan

## Context & Problem Statement
- The TUI emits desktop notifications only when it detects the terminal lost focus (`Tui::notify`, `tui/src/tui.rs:234-239`). When unfocused, it writes an OSC 9 escape sequence via `PostNotification` to ask the terminal to show a notification (`tui/src/tui.rs:486-505`).
- Windows Terminal (the default host for WSL 2) does **not** implement OSC 9 notifications today; it simply ignores `ESC ] 9 ; … BEL`. As a result, Codex appears silent even though notifications are technically “sent”.
- WSL users rely on Windows Terminal for focus changes, so both approval prompts and turn completions go unnoticed. There is no built-in fallback, and the `notify = […]` hook only fires for turn completion, not approvals, so it cannot cover this gap.

## Goals
1. Automatically surface desktop notifications for users running Codex in WSL 2 under Windows Terminal without requiring manual scripts.
2. Preserve existing OSC 9 behaviour for terminals that support it (iTerm2, WezTerm, kitty, etc.).
3. Maintain the existing `tui.notifications` filtering semantics (bool or per-type allowlist) and avoid double-notifying.

## Non-Goals
- Redesigning the external `notify = [...]` hook or adding approval notifications there (separate effort).
- Supporting arbitrary Windows shells (cmd.exe, PowerShell outside WSL). We only target “Linux Codex under WSL 2 inside Windows Terminal”.

## Proposed Fix Overview
We’ll introduce a *notification backend* abstraction inside the TUI:
- `Osc9` backend: current behaviour (default for terminals that advertise support).
- `WindowsToast` backend: a new fallback that spawns `powershell.exe` to raise a native Windows toast via WinRT APIs. This will run from WSL via interop.
- Backends are selected at startup: detect WSL (`cli/src/wsl_paths::is_wsl()`) **and** the Windows Terminal host (`WT_SESSION` env var is set). In that environment, default to `WindowsToast`; otherwise stay on `Osc9`.
- If `powershell.exe` is missing or toast invocation fails, log once and silently drop future notifications to avoid noisy retries.

## Detailed Implementation Steps
1. **Capability detection helper**
   - Add a small helper (e.g., `tui/src/notifications/mod.rs`) exposing `fn detect_backend() -> NotificationBackend`.
   - Reuse `cli::wsl_paths::is_wsl()` via a new `codex_core::env::is_wsl()` or move helper into a shared `utils` crate so the TUI crate can call it without depending on `cli`. (The TUI already depends on `codex_core`, so re-exporting a utility there is simplest.)
   - Detect Windows Terminal via `std::env::var_os("WT_SESSION")`. Document why this is sufficient (WT sets it for all panes, including WSL).

2. **Backend enum + trait**
   - Create `enum DesktopNotificationBackend { Osc9, WindowsToast(WindowsToastConfig) }` plus `impl DesktopNotificationBackend { fn notify(&self, message: &str) -> std::io::Result<()> }`.
   - `WindowsToastConfig` should carry precomputed command + arguments (e.g., resolved path to `powershell.exe` if available) and maybe a channel/app ID (“Codex”).
   - Keep focus tracking in `Tui::notify`; only swap the low-level send.

3. **Refactor `Tui::notify`**
   - Add a `notification_backend: Option<DesktopNotificationBackend>` field to `Tui` and initialize it in `Tui::new` using `detect_backend()` and `self.config.tui_notifications`.
   - Replace the hardcoded `execute!(…, PostNotification(..))` call with `backend.notify(message)`.
   - Preserve the `terminal_focused` gating; if a backend is `None`, return `false`.

4. **Implement OSC 9 backend**
   - Move the existing `PostNotification` command into the new module and have the backend call it; keep all behaviour unchanged.

5. **Implement Windows toast backend**
   - Spawn `powershell.exe -NoProfile -NoLogo -Command <script>` asynchronously (fire-and-forget). Use `std::process::Command::new("powershell.exe")` – WSL’s `wsl.exe` will resolve it.
   - Prefer passing the toast text through PowerShell’s `-ArgumentList` instead of embedding it in XML to avoid escaping headaches. Example command layout:
     ```
     powershell.exe
       -NoProfile
       -NoLogo
       -Command "& { param($title, $body)
         [Windows.UI.Notifications.ToastNotificationManager, Windows.UI.Notifications, ContentType = WindowsRuntime] | Out-Null
         $doc = New-Object Windows.Data.Xml.Dom.XmlDocument
         $doc.LoadXml(\"<toast activationType='protocol'><visual><binding template='ToastGeneric'><text>$title</text><text>$body</text></binding></visual></toast>\")
         $toast = [Windows.UI.Notifications.ToastNotification]::new($doc)
         [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier('Codex').Show($toast)
       }"
       -ArgumentList "Codex","Approval requested: rm -rf /tmp"
     ```
     The backend just supplies the two argument strings (title + body); PowerShell handles quoting, and the script constructs the XML internally.
   - Keep the process detached (no stdout/stderr) and guard against failures (log once using `tracing::warn!`).

6. **User messaging**
   - When switching to the toast backend, emit a one-time `tracing::info!` explaining that Codex detected Windows Terminal/WSL and is using Windows toasts because OSC 9 is unavailable.
   - If toast spawning fails permanently, fall back to OSC 9 (even if unsupported) and log the error via `tracing::error!` so users can see the failure in `RUST_LOG` output.

7. **Documentation updates**
   - Update `README.md` → “Notifications” to mention the automatic Windows toast fallback for WSL/Windows Terminal.
   - Update `../docs/config.md#notify` to clarify the difference between `tui.notifications` (now with Windows toast fallback) and the external `notify` hook.

8. **Testing**
   - Unit tests:
     - Verify backend detection logic under various env combinations (WSL vs non-WSL, WT_SESSION set vs unset).
     - Verify XML escaping helper to prevent invalid toast payloads.
   - Integration/manual:
     - In WSL, run `RUST_LOG=codex_tui=debug` and confirm log prints "using Windows toast notification backend".
     - Alt-tab away, trigger approval + turn completion, ensure Windows toast appears.
     - Re-test on macOS/Linux to make sure OSC 9 still works (no behaviour changes).

## Open Questions / Follow-ups
- Reuse the existing approval/turn-complete gating so toast notifications don’t need extra rate limiting.
- Longer-term: extend the `notify = [...]` hook to include approval events so power users can build their own Windows notification bridge.
