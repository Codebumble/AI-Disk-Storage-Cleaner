# Disk Cleanup Agent Prompts

Nov 4, 2026 · @Codebumble Inc

![Image](https://i.ibb.co.com/Tx6pKSxm/dfg.jpg)

## How to use

Copy the tab for the machine you're cleaning, edit its Parameters block, and paste it as the first message of a fresh agent session.

- Run the agent natively on that machine: PowerShell 7 (`pwsh`) on Windows, Terminal on macOS.
- On Windows, don't run it inside WSL. It only sees `/mnt/c` and misses Windows tooling.
- Start the session in your home folder, not inside a project repo.
- Prefer a file? Export the tab to Markdown, save it as `cleanup-prompt.md`, and tell the agent "Follow ./cleanup-prompt.md".
- Keep approvals on for the whole run. The prompt has its own consent gate; the agent's permission prompts are the second lock.

| Agent | Settings for this task | Avoid |
| --- | --- | --- |
| Claude Code | Default permission mode, so it asks before commands and edits. Allowlisting read-only scan commands (`du`, `find`, `Get-ChildItem`) is fine. | `--dangerously-skip-permissions`, allowlisting `rm` or `Remove-Item` |
| Codex CLI | `codex --sandbox read-only --ask-for-approval on-request`, started from your home folder. It will ask before writing the report folder too; approve that. | `--full-auto`, `--dangerously-bypass-approvals-and-sandbox`, `-a never` |
| Antigravity | Terminal Command Auto Execution and Artifact Review Policy both on Request Review. Open your home folder as the workspace, or allow non-workspace file access for this session only. | Turbo, Always Proceed, the Agent-driven preset |

Agents can't reliably answer UAC or sudo password prompts, so both prompts make the agent write a separate admin script that you review and run yourself.

Setting names shift between versions. If one doesn't match, check `codex --help` or the [Antigravity settings docs](https://www.antigravity.google/docs/ide/settings).

## What to expect

The agent works in six phases and stops twice for you: after the dry-run report, and before execution.

1. Discovery and scan, read-only. Large drives can take a while.
2. Dry-run report in `DiskCleanup/<timestamp>/` under your home folder: `report.md` (summary and every item) and `plan.json` (machine-readable plan). The agent stops here.
3. You approve by tier or item ID (syntax below).
4. The agent echoes the final action list and waits for you to type `EXECUTE`.
5. Execution: safest first, every path re-checked against the scan, every action logged. Admin items go into `admin-actions.ps1` or `admin-actions.sh` for you to run.
6. Final report with measured before/after free space and how to undo.

```text
APPROVE T1 ALL
APPROVE T2 T2-004 T2-007
APPROVE T3 T3-002 QUARANTINE   (macOS: TRASH)
APPROVE T3 T3-005 PURGE
SKIP REST
```

| Tier | What it covers | What the agent does |
| --- | --- | --- |
| T1 Safe | Regenerable caches, temp files, crash dumps | Deletes, or runs the tool's own clean command |
| T2 Low risk | Dev caches and build artifacts that cost re-download or rebuild time | Tool's own clean command; build folders only if gitignored and a lockfile exists |
| T3 Review | Your files and state: old downloads, installers, VMs, phone backups, stopped containers, local AI models | Moves to quarantine (Windows) or Trash (macOS) unless you say PURGE; space frees only after purge |
| T4 Report only | System-level tradeoffs: hibernation, restore points, snapshots, app uninstalls, cloud offload | Never executes; if you approve one, it goes into the admin script commented out |

Vague replies like "ok, go" approve nothing. The agent will ask you to use the syntax.
