# Tabnine CLI support for RTK — Design

**Date**: 2026-04-09
**Status**: Approved for implementation

## Goal

Add first-class Tabnine CLI integration to RTK alongside the existing Claude Code,
Gemini CLI, Codex, Copilot, and Cursor integrations. Users should be able to run
`rtk init -g --tabnine` and get a BeforeTool hook that rewrites shell commands to
their RTK equivalents, delivering the same 60-90% token savings Tabnine CLI users
currently don't benefit from.

## Context

[Tabnine CLI](https://docs.tabnine.com/main/getting-started/tabnine-cli) is a fork
of Gemini CLI. Verified facts (from both the docs and direct inspection of the
installed binary at `~/.local/bin/tabnine` and bundled source at
`~/.tabnine/agent/.bundles/<version>/tabnine.mjs`):

- **Binary name**: `tabnine` (installed to `~/.local/bin/tabnine` on Unix)
- **Global config directory**: `~/.tabnine/agent/`
- **Settings file**: `~/.tabnine/agent/settings.json`
- **Context file**: `AGENTS.md` by default, at `~/.tabnine/agent/AGENTS.md`.
  Configurable via the `contextFileName` setting (string or array of strings).
  Source proof in the bundle: `rPn="AGENTS.md"` and
  `URo(t){if(t.contextFileName)... else return["AGENTS.md"]}`.
- **Hook wire format**: identical to Gemini CLI
  - Settings shape: `hooks.BeforeTool[]` with `matcher: "run_shell_command"`,
    each entry containing `hooks: [{type: "command", command: "..."}]`
  - stdin JSON: `tool_name` + `tool_input.command`
  - Output: `{"decision":"deny","reason":"..."}` to block,
    `{"decision":"allow","hookSpecificOutput":{"tool_input":{"command":"..."}}}`
    to rewrite

Because the hook wire format is byte-identical to Gemini CLI's, RTK's existing
`run_gemini` hook processor handles Tabnine's input/output correctly as-is. The
integration work is almost entirely install plumbing.

## Non-goals

- Installing RTK commands/skills into Tabnine's `~/.tabnine/agent/commands/` or
  `/skills/` directories — out of scope for this change.
- Supporting Tabnine-specific features (subagents, MCP server config) — RTK does
  not manage these for any other CLI either.
- Project-local (non-global) Tabnine install — Gemini is also global-only today;
  same constraint applies here for symmetry and to match how Tabnine's
  `~/.tabnine/agent/settings.json` is a user-scoped file.
- Custom `contextFileName` detection — we install to the default `AGENTS.md`.
  Users who customized this can add the `@RTK.md` reference manually.

## Design

### User-facing CLI surface

```
rtk init -g --tabnine               # install (asks before patching settings.json)
rtk init -g --tabnine --auto-patch  # install without prompt
rtk init -g --tabnine --no-patch    # install hook + RTK.md, print manual steps
rtk init -g --tabnine --hook-only   # install hook only, skip AGENTS.md patching
rtk init -g --tabnine --uninstall   # remove all Tabnine artifacts
rtk hook tabnine                    # hook processor (stdin JSON → stdout JSON)
```

Flag is mutually exclusive with `--gemini`, `--codex`, `--copilot` at the same
level as those existing flags (same switch-style dispatch in `main.rs`).

### Install flow (`run_tabnine`)

Mirrors `run_gemini` exactly, with the target directory changed from
`~/.gemini/` to `~/.tabnine/agent/` and the context file changed from
`GEMINI.md` to `AGENTS.md`:

1. Resolve `~/.tabnine/agent/` (fail with a clear error if `$HOME` is unset).
   Create the directory if missing.
2. Create `~/.tabnine/agent/hooks/` if missing.
3. Write `~/.tabnine/agent/hooks/rtk-hook-tabnine.sh` containing
   `#!/bin/bash\nexec rtk hook tabnine\n`, chmod 0o755 on Unix.
4. Unless `--hook-only`: write the existing `RTK_SLIM` constant into
   `~/.tabnine/agent/AGENTS.md` via `write_if_changed`. This is the direct
   Gemini-equivalent (`write_if_changed(..., RTK_SLIM, GEMINI_MD, ...)` →
   `write_if_changed(..., RTK_SLIM, AGENTS_MD, ...)`).
5. Patch `~/.tabnine/agent/settings.json` via the new `patch_tabnine_settings`
   helper (logic copied from `patch_gemini_settings`, only the directory path
   differs):
   - Respect `PatchMode::Ask` / `Auto` / `Skip` consistently with Gemini.
   - Detect an existing RTK hook entry (command string contains `rtk`) and skip
     if present.
   - Atomic write via `tempfile::NamedTempFile::persist`.
6. Print a success message ending with "Restart Tabnine CLI. Test with: git status".

**Trade-off note:** `write_if_changed` overwrites the target file entirely if
content differs, so any user-authored content in `~/.tabnine/agent/AGENTS.md`
will be replaced — including Tabnine's own `## Tabnine Added Memories` section
if present. This matches how Gemini support currently treats `~/.gemini/GEMINI.md`,
where the file is considered RTK-owned once `--gemini` has been run. Users who
want to preserve existing AGENTS.md content should use `--hook-only`.

### Uninstall flow (`uninstall_tabnine`)

Mirrors `uninstall_gemini` line-for-line, with Tabnine paths:

1. Remove `~/.tabnine/agent/hooks/rtk-hook-tabnine.sh` if present.
2. Remove `~/.tabnine/agent/AGENTS.md` if present. (Gemini-style treats the
   context file as RTK-owned — same as `uninstall_gemini` removes `GEMINI.md`.)
3. Remove the RTK hook entry from `~/.tabnine/agent/settings.json` by filtering
   out entries whose first hook command contains `rtk`, then rewriting the
   file. Reuses the exact matching logic from `uninstall_gemini`.
4. Print the removed items list, or "RTK Tabnine support was not installed" if
   nothing was removed.

### Hook processor (`rtk hook tabnine`)

The existing `run_gemini` handler in `src/hooks/hook_cmd.rs` already produces
the correct output for Tabnine because the wire format is identical. To keep a
clean seam for future divergence and clear logs, we add a dedicated
`HookCommands::Tabnine` variant and a dedicated `run_tabnine()` function, but
implement it by calling a shared `run_gemini_format_hook()` helper.

Refactor:

```rust
// src/hooks/hook_cmd.rs
pub fn run_gemini() -> Result<()> {
    run_gemini_format_hook()
}

pub fn run_tabnine() -> Result<()> {
    run_gemini_format_hook()
}

fn run_gemini_format_hook() -> Result<()> {
    // existing body of run_gemini
}
```

This is a zero-behavior-change refactor for Gemini. Tests for Gemini continue
to pass unchanged; new tests verify `run_tabnine` returns equivalent output.

### Code changes

| File | Change |
|------|--------|
| `src/hooks/constants.rs` | Add `TABNINE_HOOK_FILE = "rtk-hook-tabnine.sh"` |
| `src/hooks/hook_check.rs` | Add Tabnine to `other_integration_installed` test helper + `test_other_integration_tabnine` |
| `src/hooks/init.rs` | Add `resolve_tabnine_agent_dir`, `run_tabnine`, `uninstall_tabnine`, `patch_tabnine_settings`, `TABNINE_HOOK_SCRIPT` const. Wire `--tabnine --uninstall` into `uninstall()` dispatch. |
| `src/hooks/hook_cmd.rs` | Extract `run_gemini_format_hook()`, add `run_tabnine()`. Add tests for tabnine parity. |
| `src/main.rs` | Add `tabnine: bool` field to `Commands::Init`, add `HookCommands::Tabnine` variant, add dispatch branches for install + uninstall + hook |
| `docs/superpowers/specs/2026-04-09-tabnine-cli-support-design.md` | This document |

No new crate dependencies. No changes to `src/cmds/`, filters, or the TOML DSL.

### Testing strategy

Following RTK's established patterns (see `.claude/rules/cli-testing.md`):

1. **Unit tests in `hook_cmd.rs`**:
   - `run_tabnine` allows non-shell tools (same as Gemini)
   - `run_tabnine` rewrites `git status` → `rtk git status`
   - `run_tabnine` denies blocked commands
   - `run_tabnine` respects `excluded_commands` config

2. **Settings patching test**:
   - Build a synthetic `settings.json` with existing non-RTK `BeforeTool` hooks,
     run `patch_tabnine_settings`, assert the new hook is appended without
     clobbering the existing entry.
   - Idempotency test: running twice is a no-op.

3. **Uninstall test**:
   - After install + uninstall, verify no RTK-owned file remains and AGENTS.md
     preserves non-RTK content.

4. **Manual smoke test** (documented in the PR, not automated):
   - `cargo install --path .`
   - `rtk init -g --tabnine --auto-patch`
   - Inspect `~/.tabnine/agent/settings.json` and
     `~/.tabnine/agent/hooks/rtk-hook-tabnine.sh`
   - Launch `tabnine` and issue a shell command via its tool — verify it gets
     rewritten to the `rtk` equivalent.

### Error handling

- Follow RTK's fallback-on-failure rule: if reading/patching settings.json fails,
  print a clear warning telling the user how to patch it manually (matching the
  Gemini pattern at `patch_gemini_settings` when `PatchMode::Skip`).
- Return `anyhow::Result` with `.context()` on every I/O call, per
  `.claude/rules/rust-patterns.md`.
- Never `unwrap()` outside tests.

### Risks & mitigations

- **Risk**: Tabnine CLI may evolve its hook format (it's a fork, not a mirror).
  **Mitigation**: the dedicated `run_tabnine()` function gives us an isolated
  seam to add Tabnine-specific handling without touching `run_gemini`.
- **Risk**: Gemini-style install overwrites any existing
  `~/.tabnine/agent/AGENTS.md`, including Tabnine's own memory section and any
  notes authored by other tools that target AGENTS.md.
  **Mitigation**: (a) the success message clearly states the AGENTS.md path was
  written, so users know their prior file was replaced; (b) `--hook-only`
  install is available for users who want to keep full control of AGENTS.md;
  (c) this behavior exactly matches `--gemini`, so users familiar with that
  integration will find it consistent.
- **Risk**: Tabnine users who customized `contextFileName` to something other
  than `AGENTS.md` won't benefit from the installed file.
  **Mitigation**: `--hook-only` install works regardless; the printed success
  message notes the AGENTS.md path so users can adapt manually.
