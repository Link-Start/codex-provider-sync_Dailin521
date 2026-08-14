# AI / Agent Operator Guide

This file is for AI assistants and automation working in this repository. User-facing setup and usage belong in [README.md](README.md) and `docs/`.

## Goal

Build an independent tool for diagnosing, repairing, backing up, and migrating local Codex session data. The current implemented focus is restoring session visibility after `model_provider` changes by keeping rollout metadata, project paths, and the resolved SQLite thread index aligned.

Provider synchronization is one repair capability, not the long-term product boundary. Do not treat the project as an authentication, account-management, or provider-routing tool, and do not describe planned storage or cross-device migration as already implemented. Product scope and sequencing are recorded in [docs/PRODUCT_DIRECTION_ZH.md](docs/PRODUCT_DIRECTION_ZH.md).

## Choose the interface

- Prefer the Windows GUI for users who want a double-click tool and do not want Node.js.
- Prefer the Local Web UI for browser-based or cross-platform use: `codex-provider web`.
- Use the CLI for explicit command requests, automation, diagnostics, WSL paths, or when a GUI is unavailable.
- Use `CodexProviderSync.Automation.exe` only for repository development or explicit Automation work. It ships with the v0.4 Windows Release, but protocol 0.4 is experimental and is not a stable public API or a production GUI control port.

## Safe operating flow

1. Inspect with the UI status action or `codex-provider status`.
2. Confirm the current Provider, effective SQLite Home/database, and rollout/SQLite Provider distributions.
3. Choose `sync`, `switch`, or `restore` from the rules below.
4. Execute once; do not manually edit rollout files or SQLite when the tool can perform the operation.
5. Report the final Provider alignment, backup location, and any skipped or blocked data.

## Choose the operation

- `sync`: the user already changed Provider/account with CCSwitch or another tool, and `config.toml` already contains the intended root `model_provider`.
- `switch <provider-id>`: the user explicitly wants this tool to change the root Provider and synchronize history. Custom providers must already exist in `config.toml`; built-in `openai` is always valid.
- `switch <provider-id> --keep-root-model`: preserve the root `model`.
- `switch <provider-id> --model <name>`: explicitly set the root `model`.
- `restore <backup-dir>`: roll back a mistaken operation. Cross-SQLite-Home restore requires an explicit target, relocation confirmation, and no config restore.
- `prune-backups --keep <n>`: remove only older managed backups.

`sync` uses the current root `model_provider`, falling back to `openai` when it is absent. Sync and switch create a backup before writing and prune only tool-managed backups according to retention.

## Storage and path rules

Resolve SQLite Home in this order:

1. explicit CLI, desktop GUI, or Web profile override
2. root `sqlite_home` in `config.toml`
3. `CODEX_SQLITE_HOME`
4. `<Codex Home>/sqlite`

Only the default layout may fall back to legacy `<Codex Home>/state_5.sqlite`. A missing explicit, config, or environment SQLite Home is an error; never silently fall back elsewhere.

On Windows, `\\wsl.localhost\...` and `\\wsl$\...` SQLite Homes are diagnostic-only. Run SQLite operations inside the matching WSL distribution, using Linux paths. Metadata v2 backups record `sqliteHome` and `sqliteDbFiles`; do not bypass relocation checks.

## Safety boundaries

- Never read, copy, log, or modify `auth.json`, credentials, or tokens. Do not copy, log, modify, or expose message bodies outside the Web UI's explicit read-only History view.
- Do not change thread `updated_at` or reorder history to force visibility.
- Preserve backup-first, locking, transaction, rollback, WSL, and path-boundary behavior.
- Rollout/SQLite counts may differ briefly because of an active session; Provider distributions are the alignment signal.
- Metadata synchronization restores visibility only. Another Provider/account may be unable to decrypt existing `encrypted_content`; advise the user to return to the original Provider/account or start a new session if continuation or compact fails.
- Tests and reproduction scripts must use temporary directories or fixtures, never a real user Codex Home.

## Handle common outcomes

- SQLite in use: stop before rollout mutation; ask the user to close Codex CLI, Codex App, app-server, and retry.
- Skipped locked rollout files: classify as partial success. List the skipped files and recommend another sync after the active session ends.
- Missing custom Provider: define it in `config.toml` or switch with the user's normal Provider tool, then run `sync`.
- Missing explicit SQLite database: keep the explicit path authoritative and report the error; do not use the legacy database.
- WSL UNC diagnostic: run the CLI in WSL with the Windows Codex Home under `/mnt/<drive>/...` and a Linux SQLite Home such as `/home/...`.

## Engineering direction

- Node CLI and Web UI share the service layer in `src/`; do not duplicate sync logic in the browser.
- Make the CLI/Core the single long-term source of truth for discovery, diagnosis, planning, repair, backup, restore, and migration.
- Expose a versioned machine-readable CLI protocol before moving desktop behavior onto it. Keep human CLI output separate from the machine contract.
- Evolve the Windows GUI into a lightweight client over that contract; presentation, native integration, and process lifecycle belong in the desktop layer, not duplicated data logic.
- Today the Windows GUI still routes through the .NET Application/Core layers and macOS calls .NET Core directly. Preserve their safety behavior until each client is migrated; do not document the target architecture as if it were already complete.
- Add focused tests for behavior changes. Prefer controller tests over new reflection-based WinForms business tests.

## Reporting

State the current Provider, whether rollout and SQLite metadata are aligned, the resolved database path, the backup created by a write operation, and whether the result was complete, partial, or blocked. Distinguish automated tests from real-machine validation and list anything not run.
