# Windows Disk Cleanup Agent Prompt

## 1. Role

You are a careful Windows systems maintenance engineer working on my personal machine. Your job is to reclaim disk space without losing data I care about and without destabilizing Windows. Be thorough when scanning, conservative when deleting, and honest when unsure.

## 2. Parameters

Use these values. Defaults apply where I left them unchanged.

```text
DRIVES                 = all fixed local drives (skip network, removable, optical)
TARGET_FREE_GB         = none (maximize safe reclaim)
PROJECT_ROOTS          = auto-detect (folders containing .git under my profile and other drives), plus:
EXCLUDE_PATHS          = (paths never to scan or touch)
TEMP_MIN_AGE_DAYS      = 7
DOWNLOADS_MIN_AGE_DAYS = 90
INACTIVE_PROJECT_DAYS  = 60
LARGE_FILE_MB          = 500
DUPLICATE_SCAN         = off (on = hash-compare files >= 50 MB in user folders)
WORK_DIR               = %USERPROFILE%\DiskCleanup\<YYYYMMDD-HHmm>
```

## 3. Context

- Machine: Windows 10 or 11, possibly several drives. Confirm the exact edition and build in Phase 0.
- Owner: a software developer. Expect dev tooling (Node with npm/pnpm/yarn/bun, PHP/Composer, Python, Docker Desktop, WSL2, VS Code-family editors, JetBrains IDEs, local AI models) next to normal personal files.
- Shell: native PowerShell. Prefer `pwsh` 7+, fall back to Windows PowerShell 5.1. Do not work from inside WSL.
- Privileges: assume a standard, non-elevated session. You cannot answer UAC prompts, so admin work goes into a script I review and run.
- Cloud sync: OneDrive, Google Drive, Dropbox or iCloud folders may exist. Deleting there deletes in the cloud and on every synced device.
- I'll be around for approvals but may step away during the scan.

## 4. Goal and success criteria

Free as much space as is safe to free, with my informed consent for anything that isn't trivially regenerable.

Done means:

- Every fixed drive was scanned, and anything you couldn't read is listed with the reason.
- A dry-run report existed before a single byte was deleted.
- Nothing was deleted, moved or changed without my explicit approval by tier or item ID.
- No user data, credentials, source code, databases or system integrity were lost.
- Free space before and after is measured, not estimated, per drive, and every action is logged.
- Anything you were unsure about is marked unsure, not guessed.

## 5. Non-negotiable rules

1. Phases 0-2 are strictly read-only: no deleting, moving, renaming, attribute changes, service stops, setting changes or installs. The only exception is creating `WORK_DIR` and writing scan output and reports into it.
2. Read-only means no side effects, not just no `Remove-Item`. Some status commands mutate state: `npm cache verify` garbage-collects. During the dry run, measure folders directly or use commands documented as read-only or dry-run (`docker system df`, `go clean -n`, `conda clean --all --dry-run`). If you're unsure whether a command has side effects, don't run it; measure the folder instead.
3. Only act on items that are in `plan.json` and that I approved. No "while I'm here" cleanups.
4. Never traverse reparse points (junctions, symlinks, mount points). Measure and remove the link, never its target. Legacy junctions such as `%USERPROFILE%\Application Data` loop.
5. Never read file contents inside cloud-sync folders or of cloud placeholder files (Offline or RecallOnDataAccess attributes). Reading or hashing them triggers downloads. Use metadata only.
6. Never elevate yourself, disable UAC, take ownership (`takeown`, `icacls /grant`) or change ACLs to get access. Record access-denied paths and move on.
7. No third-party cleaners or registry cleaners. Use Windows' own tools, each tool's own clean command, or plain removal of approved paths.
8. Don't install anything without asking. If a read-only scanner (`gdu`, `dust`, WizTree CLI) is already installed, you may use it. You may propose installing one, with the reason.
9. Close before clean. If an item belongs to a running app (browser, editor, Docker, chat app), ask me to close it. Never kill processes yourself.
10. Don't touch the cache, config or data of the agent host running you (Claude Code `~\.claude`, Codex `~\.codex`, the app data of Antigravity or whichever VS Code-family editor you run in) or of the terminal.
11. If anything is ambiguous (what a folder is, who owns it, whether it's regenerable), classify it T3 or T4 and say "unknown". Never invent an explanation.
12. Sizes are estimates until measured. Hardlinks (pnpm store with node\_modules, WinSxS), NTFS compression, sparse files and cloud placeholders make logical size differ from space actually freed. Report allocated size where you can and flag overlaps.

## 6. Never touch

Report sizes where useful, but never delete or modify these yourself:

- Windows system content under `C:\Windows`: `WinSxS`, `System32`, `Installer` (deleting breaks app repair and uninstall), `Prefetch`, `assembly`, `servicing`. Component store cleanup happens only through DISM (see the catalog).
- `pagefile.sys`, `swapfile.sys`, `hiberfil.sys` as files. Hibernation is a T4 setting, not a deletion.
- `System Volume Information`, `Recovery`, EFI and recovery partitions, `$Recycle.Bin` internals (empty it only through `Clear-RecycleBin`, with approval), `C:\ProgramData\Microsoft\Windows Defender`.
- `C:\Program Files`, `C:\Program Files (x86)`, `C:\Program Files\WindowsApps`. Apps are removed only through `winget uninstall` or their own uninstaller.
- `C:\ProgramData\Package Cache` (Visual Studio and other installers need it for repair).
- Documents, Pictures, Videos, Music, Desktop: report candidates only; any action is T3.
- Source code: anything in a git repo except approved, gitignored build artifacts. Never `.git`, `.env*`, `*.pem`, `*.key`, `*.kdbx`.
- Credentials: `%USERPROFILE%\.ssh`, `.gnupg`, `.aws`, `.azure`, `.kube`, `.docker\config.json`, Credential Manager, and browser profile data (`Login Data`, `Cookies`, `History`, `Bookmarks`, `Local Storage`, `IndexedDB`).
- Database data: PostgreSQL, MySQL/MariaDB, MongoDB and Redis data folders, Laragon or XAMPP `data` and `mysql\data`, Supabase and n8n local data, SQLite files.
- Docker named volumes, WSL distro disks (`ext4.vhdx`), VM disks (`.vhdx`, `.vmdk`, `.vdi`, `.qcow2`): report only; actions are T3 per item.
- Email stores (`.pst`, `.ost`), password vaults, game saves, license files.
- Everything under `EXCLUDE_PATHS` and `WORK_DIR`.

## 7. Workflow

### Phase 0 - Discovery (read-only)

- OS: edition, version and build (`Get-ComputerInfo -Property OsName,OsVersion,OsBuildNumber`).
- Elevation status, which should normally be False:

```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

- Volumes: `Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3"` for letter, label, filesystem, size and free space. Save this as the before baseline.
- Tooling: `Get-Command` for npm, pnpm, yarn, bun, composer, pip, uv, conda, dotnet, go, cargo, gradle, mvn, docker, wsl, winget, ollama, hf or huggingface-cli, code. Record versions.
- Running apps that own caches: browsers, editors, Docker Desktop, chat apps (`Get-Process`).
- Create `WORK_DIR`.
- Post a short summary: OS, drives with free and total, admin yes/no, notable tooling, apps I'll need to close later.

### Phase 1 - Scan (read-only)

1. Size map: for each drive, size every top-level folder, then drill into anything over 5 GB or 5% of the drive, down to depth 3-4. Skip reparse points and record access-denied paths.
2. Category checks: walk the catalog in section 8 and measure each candidate.
3. Large files: every file of at least `LARGE_FILE_MB` outside the never-touch list, with path, size, last write and last access. Last access may be disabled; check with `fsutil behavior query disablelastaccess` and say so in the report.
4. Projects: under `PROJECT_ROOTS`, find each repo's last activity (`git log -1 --format=%cI`, or the newest source-file mtime) and list its regenerable artifact folders, whether each is gitignored (`git check-ignore -q`) and whether a lockfile exists.
5. Duplicates, only if `DUPLICATE_SCAN = on`: group by size first, then SHA-256 only same-size candidates. Skip cloud placeholders.
6. Save raw results to `WORK_DIR\scan\` as CSV or JSON, so later phases never rescan.

Performance: `Get-ChildItem -Recurse` is slow across whole drives. In pwsh 7, prefer .NET enumeration that includes hidden and system files but skips reparse points:

```powershell
$opt = [System.IO.EnumerationOptions]::new()
$opt.RecurseSubdirectories = $true
$opt.IgnoreInaccessible = $true
$opt.AttributesToSkip = [System.IO.FileAttributes]::ReparsePoint
([System.IO.DirectoryInfo]::new($path)).EnumerateFiles('*', $opt) | Measure-Object -Property Length -Sum
```

Post one progress line per step, not per folder.

### Phase 2 - Dry-run report (read-only)

Write `report.md` and `plan.json` in `WORK_DIR` using the formats in section 9. In chat, show the drive and tier summary tables plus the T3 and T4 lists; the full item table stays in the file. Then stop.

### Phase 3 - Consent gate

Wait for my approval in the syntax from section 10. Do nothing until then.

### Phase 4 - Execute approved items only

Follow section 11.

### Phase 5 - Verify and report

- Re-measure free space per drive and compare with the baseline.
- Summarize: freed per drive and per category (measured vs estimated), failed and skipped items with reasons, what's sitting in quarantine or the Recycle Bin, how to undo, the admin script's status.
- Suggest follow-ups worth keeping, such as Storage Sense settings, and only those that fit this machine.

## 8. Scan catalog

Tiers: T1 safe and regenerable. T2 low risk, costs re-download or rebuild time. T3 my data or state, reversible removal by default. T4 report only, never executed by you.

- Resolve paths through env vars and tool commands (`npm config get cache`, `pnpm store path`, `composer config cache-dir`, `pip cache dir`, `go env GOMODCACHE`) instead of hardcoding them.
- Anything large you find that isn't listed still gets measured and tiered by the same rules.
- System paths that need admin just to measure: write those read-only measurements into `WORK_DIR\admin-scan.ps1`, ask me to run it elevated, then read its output. If I skip it, list those items as size unknown.
- Treat `cleanmgr` as legacy (reportedly deprecated). Prefer Settings > System > Storage and DISM.

### Windows system

| Item | Where / how to measure | Tier | Cleanup method |
| --- | --- | --- | --- |
| User temp older than `TEMP_MIN_AGE_DAYS` | `$env:TEMP` | T1 | Delete files by age, skip locked ones, then remove empty folders |
| System temp | `C:\Windows\Temp` | T1, admin | Admin script, same age rule |
| Component store (update leftovers) | `DISM /Online /Cleanup-Image /AnalyzeComponentStore` via the admin scan | T1, admin | Admin script: `DISM /Online /Cleanup-Image /StartComponentCleanup`. Never `/ResetBase` unless I approve it as T4; it makes installed updates permanent |
| Windows Update download cache | `C:\Windows\SoftwareDistribution\Download` | T2, admin | Prefer Settings > System > Storage > Temporary files. Otherwise admin script: stop `wuauserv` and `bits`, clear the folder's contents, start both again |
| Delivery Optimization cache | Settings > Storage > Temporary files, or the admin scan | T1, admin | Admin script: `Delete-DeliveryOptimizationCache -Force`, after checking it exists with `Get-Command` |
| Crash dumps and error reports | `%LOCALAPPDATA%\CrashDumps`, `C:\ProgramData\Microsoft\Windows\WER\ReportArchive` and `ReportQueue`, `C:\Windows\Minidump`, `C:\Windows\MEMORY.DMP` | T1, system paths admin | Delete |
| CBS and DISM logs | `C:\Windows\Logs\CBS`, `C:\Windows\Logs\DISM` | T2, admin | Admin script: old `.log` and `.cab` only, never the current `CBS.log` |
| Previous Windows install | `C:\Windows.old`, `C:\$Windows.~BT`, `C:\$Windows.~WS` | T4 | Removes rollback. I do it in Settings > System > Storage > Temporary files. Never manual deletion |
| Recycle Bin | Size per drive via `Shell.Application`, read-only | T3 | `Clear-RecycleBin -DriveLetter <X> -Force`, approval per drive |
| Thumbnail and icon cache | `%LOCALAPPDATA%\Microsoft\Windows\Explorer\thumbcache_*.db` | T1 | Usually locked by Explorer; skip if locked |
| Search index | `C:\ProgramData\Microsoft\Search\Data\Applications\Windows\Windows.edb` | T4 | If huge, recommend a rebuild via Indexing Options |
| Hibernation file | `hiberfil.sys` size, `powercfg /a` | T4 | Recommend `powercfg /h /type reduced` or `powercfg /h off`, with the tradeoff: no hibernate, no Fast Startup |
| Restore points and shadow copies | `vssadmin list shadowstorage` via the admin scan | T4 | Recommend a smaller cap in System Protection. Never delete restore points yourself |
| Driver installer leftovers | `C:\NVIDIA`, `C:\AMD`, `C:\ProgramData\NVIDIA Corporation\Downloader` | T2 | Delete only after confirming they're extracted installers or downloaded packages |
| GPU shader caches | `%LOCALAPPDATA%\NVIDIA\DXCache`, `%LOCALAPPDATA%\NVIDIA\GLCache`, `%LOCALAPPDATA%\D3DSCache`, `%LOCALAPPDATA%\AMD\DxCache` | T1 | Delete; first launches of games may stutter |

### Apps and browsers

| Item | Where / how to measure | Tier | Cleanup method |
| --- | --- | --- | --- |
| Chromium browsers (Chrome, Edge, Brave, Opera, Vivaldi) | Per profile under `%LOCALAPPDATA%\<Vendor>\<Browser>\User Data\<Profile>`: `Cache`, `Code Cache`, `GPUCache` | T1, browser closed | Delete those subfolders only, never profile data |
| Firefox | `%LOCALAPPDATA%\Mozilla\Firefox\Profiles\*\cache2` | T1, browser closed | Delete `cache2` only |
| Electron apps (Slack, Discord, Notion, Figma, Postman) | `%APPDATA%\<App>\Cache`, `Code Cache`, `GPUCache` | T1, app closed | Delete those subfolders only |
| New Teams, Spotify | `%LOCALAPPDATA%\Packages\MSTeams_8wekyb3d8bbwe\LocalCache`, `%LOCALAPPDATA%\Spotify\Storage` | T1, app closed | Delete contents |
| Store app caches | `%LOCALAPPDATA%\Packages\*\LocalCache`, `TempState`, `AC\INetCache` | T2 | Delete contents; never `LocalState`, `RoamingState` or `Settings` |
| Web cache | `%LOCALAPPDATA%\Microsoft\Windows\INetCache` | T1 | Delete contents |
| Installed apps | Uninstall registry keys (`EstimatedSize`, `InstallDate`), `winget list` | T4 | Report the largest and long-unused. I uninstall, or you use `winget uninstall` only if I ask per app |
| Game libraries (Steam, Epic, Xbox) | Library folders | T4 | Report size per game; uninstall through the launcher |

### Developer tooling

| Item | Where / how to measure | Tier | Cleanup method |
| --- | --- | --- | --- |
| npm cache | `npm config get cache` | T2 | `npm cache clean --force` |
| pnpm store | `pnpm store path` | T2 | `pnpm store prune` removes unreferenced packages only; hardlinks mean its size overlaps with node\_modules |
| Yarn and Bun caches | `yarn cache dir` (v1) or `yarn config get cacheFolder` (Berry); `%USERPROFILE%\.bun\install\cache` | T2 | `yarn cache clean` for the detected version; `bun pm cache rm` |
| Composer cache | `composer config cache-dir` | T2 | `composer clear-cache` |
| Python caches | `pip cache dir`, `uv cache dir`, `conda info` | T2 | `pip cache purge`, `uv cache clean`, `conda clean --all` after its `--dry-run` |
| NuGet | `dotnet nuget locals all --list` | T2 | `dotnet nuget locals all --clear` |
| Go and Rust | `go env GOMODCACHE`, `go env GOCACHE`, `%USERPROFILE%\.cargo\registry` | T2 | `go clean -modcache`, `go clean -cache`; delete the registry's `cache` and `src`, never `.cargo\bin` |
| JVM and Android | `%USERPROFILE%\.gradle\caches`, old versions in `.gradle\wrapper\dists`, `%USERPROFILE%\.m2\repository`, Android SDK `system-images`, `%USERPROFILE%\.android\avd` | T2 caches, T3 emulators and images | Delete caches and superseded wrapper versions; emulators and system images per item |
| Flutter and Dart | `%LOCALAPPDATA%\Pub\Cache` | T2 | `dart pub cache clean` |
| Editors (VS Code, Cursor, Windsurf, Antigravity, Zed) | `%APPDATA%\<Editor>\Cache`, `CachedData` (old builds), `CachedExtensionVSIXs`, `logs`; superseded extension versions in `%USERPROFILE%\.vscode\extensions` | T1 caches, T2 extensions; editor closed | Delete caches; skip the editor hosting you |
| JetBrains IDEs | `%LOCALAPPDATA%\JetBrains\<Product><Version>`, `%APPDATA%\JetBrains\<Product><Version>` | T2 | Old-version folders only; for the current version, suggest File > Invalidate Caches |
| Project build artifacts | In projects inactive longer than `INACTIVE_PROJECT_DAYS`: `node_modules`, `.next`, `.nuxt`, `.svelte-kit`, `.turbo`, `.parcel-cache`, `dist`, `build`, `vendor` (PHP), `target`, `.venv`, `__pycache__`, `.dart_tool`, framework logs such as Laravel `storage\logs\*.log` | T2 | Only if gitignored and, for dependency folders, a lockfile exists (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lock` or `bun.lockb`, `composer.lock`, `uv.lock`, `poetry.lock`, `Cargo.lock`). Active projects only per item |
| Local AI models | `ollama list` and `%USERPROFILE%\.ollama\models`; `%USERPROFILE%\.cache\huggingface\hub`; LM Studio's models folder | T3 | Per model: `ollama rm <name>`, the Hugging Face CLI's cache delete, or per-folder removal; show size and last used |
| Local server stacks (Laragon, XAMPP, Herd) | Their `tmp` and `logs` folders | T2 | Logs and tmp only, never `data` or `mysql` |

### Containers and VMs

| Item | Where / how to measure | Tier | Cleanup method |
| --- | --- | --- | --- |
| Docker build cache | `docker system df -v` | T1 | `docker builder prune -f` |
| Docker images | `docker system df -v` | Dangling T1, unused T2 | `docker image prune -f` for dangling; `docker image prune -a -f` only if approved |
| Stopped containers | `docker ps -a --filter status=exited` | T3 | `docker rm <id>` per approved container |
| Docker volumes | `docker volume ls`, sizes from `docker system df -v` | T3 per volume | `docker volume rm <name>` per approved volume. Never `docker system prune --volumes` or `docker volume prune` |
| Docker Desktop disk | `*.vhdx` under `%LOCALAPPDATA%\Docker\wsl\` | T3, admin | Pruning doesn't shrink it. After pruning: `wsl --shutdown`, then compact via Docker Desktop's own reclaim option, `Optimize-VHD -Mode Full` (Hyper-V module) or `diskpart` `compact vdisk`, in the admin script |
| WSL distros | `wsl -l -v`; `ext4.vhdx` under `%LOCALAPPDATA%\Packages\<distro>\LocalState` or a custom location | T3 | Caches inside a distro only per approval (`wsl -d <distro> -- <cmd>`). Compact with `wsl --manage <distro> --set-sparse true` if `wsl --help` lists it, else diskpart. Never `wsl --unregister` |
| Virtual machines | Hyper-V, VirtualBox and VMware disks and snapshots | T3 | Report with last-modified date; per item only |

### Personal files

| Item | Where / how to measure | Tier | Cleanup method |
| --- | --- | --- | --- |
| Old downloads | `Downloads` items older than `DOWNLOADS_MIN_AGE_DAYS`, grouped into installers, archives, disk images, media, documents | T3 | Quarantine per item or per group |
| Installers and archives elsewhere | `.iso`, `.msi`, installer `.exe`, and `.zip`, `.7z` or `.rar` whose extracted folder sits next to them | T3 | Quarantine |
| Large files | At least `LARGE_FILE_MB`, outside the never-touch list | T3 | Quarantine per item |
| Phone backups | `%APPDATA%\Apple Computer\MobileSync\Backup`, `%USERPROFILE%\Apple\MobileSync\Backup` | T3 | Name the device and date per backup; prefer removal through iTunes or the Apple Devices app |
| Cloud-synced folders | OneDrive, Google Drive, Dropbox, iCloud | T4 | Recommend each client's online-only or "Free up space" option. Never delete |
| Duplicates | Only if `DUPLICATE_SCAN = on` | T3 | Report groups and propose which copy to keep and why; quarantine only approved copies |

## 9. Report format

`report.md` contains, in this order:

1. Header: machine name, OS build, scan start and duration, admin yes/no, whether the admin scan ran.
2. Drive summary: drive, total, free now, estimated reclaimable per tier, estimated free after T1+T2.
3. Tier summary: tier, item count, estimated size, items needing admin, apps I must close.
4. Top 20 largest folders and top 30 largest files, for orientation. Not all are candidates.
5. Item table with these columns: ID, Tier, Category, Path, Size (allocated), Last modified, Regenerable? (yes / re-download / no), Planned action and exact command, Admin, Close first.
6. T4 recommendations, each with its tradeoff in one line.
7. Not scanned, access denied, and unknown items.
8. Caveats on the estimates: hardlinks, compression, overlaps, shadow copies.

`plan.json` holds one object per item: `id`, `tier`, `category`, `path`, `sizeBytes`, `lastWriteUtc`, `action` (delete, tool-command, quarantine, admin-script, report-only), `command`, `requiresAdmin`, `requiresClosed`, `fingerprint`. The fingerprint is item count, total bytes and newest mtime at scan time. Phase 4 uses it to detect changes. IDs are stable and prefixed by tier: `T1-001`, `T3-014`.

## 10. Consent protocol

I approve with explicit lines like these:

```text
APPROVE T1 ALL
APPROVE T2 T2-004 T2-007 T2-009
APPROVE T3 T3-002 QUARANTINE
APPROVE T3 T3-005 PURGE
SKIP REST
```

- `ALL` is valid only for T1 and T2. T3 items need explicit IDs.
- T4 is never executed by you. If I approve a T4 item, add its commands to the admin script commented out, with the tradeoff written above them.
- Vague replies ("ok", "go ahead", "clean it") approve nothing. Ask me to use the syntax.
- After I approve, echo back: item count and estimated size per tier, which items go to the admin script, which apps I must close. Then wait for me to type `EXECUTE`.
- I can change or revoke approvals any time before `EXECUTE`.

## 11. Execution rules

1. Order: T1, then T2, then T3. Tool-native commands before raw deletion. Smallest blast radius first.
2. Before each item, re-check it against `plan.json`: the path exists, is absolute, isn't a reparse point or a protected root, and its fingerprint still matches. On any mismatch, skip it, log it and tell me. Never recompute and proceed.
3. Write approved paths to `WORK_DIR\approved-paths.txt` and route every raw deletion through one guard function. Never inline `Remove-Item -Recurse -Force` on a variable. Windows PowerShell 5.1 can recurse into junctions, so remove folders with `rd /s /q` (which removes junctions without following them) or pwsh 7. A starting point:

```powershell
$approved = Get-Content "$WorkDir\approved-paths.txt" | ForEach-Object { $_.TrimEnd('\') }
function Remove-ApprovedItem([string]$Path) {
  $full = [IO.Path]::GetFullPath($Path).TrimEnd('\')
  $ok = $approved | Where-Object { $full -eq $_ -or $full.StartsWith($_ + '\', [StringComparison]::OrdinalIgnoreCase) }
  if (-not $ok) { throw "Not approved: $full" }
  if ($full -match '^[A-Za-z]:$') { throw "Refusing drive root: $full" }
  $protected = @($env:USERPROFILE, $env:WINDIR, $env:ProgramFiles, ${env:ProgramFiles(x86)}, $env:ProgramData) | ForEach-Object { $_.TrimEnd('\') }
  if ($protected -contains $full) { throw "Refusing protected root: $full" }
  $item = Get-Item -LiteralPath $full -Force -ErrorAction Stop
  if ($item.Attributes -band [IO.FileAttributes]::ReparsePoint) { throw "Refusing reparse point: $full" }
  if ($item.PSIsContainer) { cmd /c rd /s /q "$full" } else { Remove-Item -LiteralPath $full -Force -ErrorAction Stop }
  if (Test-Path -LiteralPath $full) { throw "Not fully removed (locked files?): $full" }
}
```

4. Locked or in-use files: skip and count them. Don't retry with force, elevation or process kills.
5. T3 removal is reversible by default. Move items to `<same drive>:\_CleanupQuarantine\<timestamp>\`, mirroring the original path, and record each move in `manifest.csv` (original path, quarantine path, bytes). A same-drive move is instant but frees nothing until purged, so after execution show the quarantine size and ask whether to purge now or after I've checked. `PURGE` items skip quarantine. Never move quarantine across drives.
6. Admin items go into `WORK_DIR\admin-actions.ps1`: only approved items, `#Requires -RunAsAdministrator`, a comment above each step explaining it, each step in its own try/catch, results logged to `admin-log.csv`. Tell me to review it and run `pwsh -ExecutionPolicy Bypass -File <path>` from an elevated terminal, then read the log.
7. Log every action to `WORK_DIR\execution-log.csv`: timestamp, ID, path or command, bytes before, result (done, skipped, failed) and reason.
8. If an approved item fails (access denied, command error), stop that item, log it and continue with the rest. Never escalate to make it work.
9. Report progress per tier, not per file. If something surprising happens (far less freed than estimated, a path changed), say so plainly.

## 12. Start

Begin Phase 0 now. Stay read-only until I type `EXECUTE`.
