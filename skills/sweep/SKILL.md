---
name: sweep
description: Find abandoned Windows processes (leaked MCP servers, respawned duplicate helpers, hung CLI calls, stuck script trees), prove each one is idle with a live CPU sample, and kill only those. Also checks Desktop Window Manager for a memory leak. Windows only. Arguments - --dry-run.
argument-hint: "[--dry-run]"
disable-model-invocation: true
allowed-tools: Read, Glob, Grep
---

# freeram sweep

Developer machines that run for days pile up per-session helpers (MCP servers, language servers, headless browsers, CLI subprocesses) that their parent app failed to reap. Find them, prove each one is idle, kill only those, and leave every app in use alone.

## Rules

- Never infer "not in use" from age or process name. Sample CPU live (step 4) before every kill.
- Never kill a keep-list process itself. Its orphaned or duplicate helper children are fair game.
- Never kill a process that has a window title without asking first.
- Kill single PIDs with `Stop-Process -Id`. Never use `taskkill /T` and never kill by name, except the Explorer restart under "Known patterns", which needs a yes first.
- Never kill a PID you have not CPU-sampled in this run.
- When the arguments include `--dry-run`: run steps 1 to 4 and the DWM check, list what would be killed, kill nothing, launch no UAC prompt, and title the report "dry run, nothing killed".

## Before you start

- When the platform is not Windows, say freeram only runs on Windows and stop.
- Run every command with the PowerShell tool. When there is no PowerShell tool, tell the user to add `"CLAUDE_CODE_USE_POWERSHELL_TOOL": "1"` under `"env"` in `~/.claude/settings.json`, restart Claude Code, and run `/freeram:sweep` again. Then stop.
- `$PID` is a read-only PowerShell variable. Name loop variables `$procId`, never `$pid`.
- PowerShell tool calls do not share variables. Step 1 saves a CSV; later steps read it back.
- Tell the user in one line: each command asks for permission, the process snapshot is saved to `%TEMP%\freeram\`, and anything in a command line that looks like a token or password is masked before it reaches the conversation.
- Read `~/.claude/freeram/keep-list.txt` when it exists (one process name per line) and add those names to the keep-list.

## Keep-list

- `claude.exe`, `Claude.exe`: every Claude Code CLI session and the Claude desktop app. Killing one ends a live session.
- `codex.exe`, `ChatGPT.exe`
- Browsers: `chrome.exe`, `msedge.exe`, `msedgewebview2.exe`, `firefox.exe`, `brave.exe`
- Chat: `Slack.exe`, `ms-teams.exe`, `Teams.exe`, `Discord.exe`
- Editors: `Code.exe`, `Cursor.exe`, `Windsurf.exe`
- `WindowsTerminal.exe`, `explorer.exe`, `Docker Desktop.exe`
- Windows system processes and vendor services rooted at `services.exe`. Report large ones, never kill them.
- When an unfamiliar top-level app shows up with a window title or more than 500 MB private, ask whether to keep it. When the user says keep it, append its name to `~/.claude/freeram/keep-list.txt` with the Write or Edit tool.

## Step 1: snapshot every process

Run this once. It saves every process with its ancestry to `%TEMP%\freeram\snapshot.csv` and prints the totals.

```powershell
$dir = Join-Path $env:TEMP 'freeram'
New-Item -ItemType Directory -Force -Path $dir | Out-Null
$wmi = Get-CimInstance Win32_Process
$gp = Get-Process | Group-Object Id -AsHashTable -AsString
$byId = @{}
foreach ($p in $wmi) { $byId[[string]$p.ProcessId] = $p }

function Get-Ancestry([string]$procId) {
  $names = @(); $seen = @{}; $cur = $procId; $orphan = $true
  for ($i = 0; $i -lt 25; $i++) {
    if (-not $byId.ContainsKey($cur) -or $seen.ContainsKey($cur)) { break }
    $seen[$cur] = $true
    $p = $byId[$cur]; $names += $p.Name
    if ($p.Name -in @('explorer.exe', 'services.exe', 'svchost.exe')) { $orphan = $false; break }
    $parentId = [string]$p.ParentProcessId
    if ($parentId -eq $cur -or -not $byId.ContainsKey($parentId)) { break }
    if ($byId[$parentId].CreationDate -gt $p.CreationDate) { break }  # PID reused: the real parent is gone
    $cur = $parentId
  }
  [pscustomobject]@{ Root = $names[-1]; Chain = ($names -join ' < '); Orphan = $orphan }
}

function Hide-Secrets([string]$cmd) {
  if (-not $cmd) { return '' }
  $cmd = $cmd -replace '(?i)(?<k>(token|secret|password|passwd|pwd|api[-_]?key|apikey|auth|bearer|credential)s?["'']?\s*[=:\s]\s*["'']?)[^\s"'']+', '${k}***'
  $cmd -replace '(sk-[A-Za-z0-9_-]{16,}|gh[pousr]_[A-Za-z0-9]{20,}|AKIA[0-9A-Z]{16}|xox[abprs]-[A-Za-z0-9-]{10,}|eyJ[\w-]{10,}\.[\w-]{10,}\.[\w-]{10,})', '***'
}

$rows = foreach ($p in $wmi) {
  $a = Get-Ancestry ([string]$p.ProcessId)
  $g = $gp[[string]$p.ProcessId]
  [pscustomobject]@{
    PID = $p.ProcessId; PPID = $p.ParentProcessId; Name = $p.Name; Created = $p.CreationDate
    WS_MB = if ($g) { [math]::Round($g.WorkingSet64 / 1MB, 1) } else { 0 }
    Priv_MB = if ($g) { [math]::Round($g.PrivateMemorySize64 / 1MB, 1) } else { 0 }
    CPU = if ($g) { $g.CPU } else { 0 }
    Title = if ($g) { $g.MainWindowTitle } else { '' }
    Root = $a.Root; Orphan = $a.Orphan; Chain = $a.Chain
    Cmd = Hide-Secrets $p.CommandLine
  }
}
$rows | Export-Csv -Path (Join-Path $dir 'snapshot.csv') -NoTypeInformation -Encoding UTF8

$os = Get-CimInstance Win32_OperatingSystem
"RAM used: {0:N1} GB of {1:N1} GB" -f (($os.TotalVisibleMemorySize - $os.FreePhysicalMemory) / 1MB), ($os.TotalVisibleMemorySize / 1MB)
"Commit: {0:N1} GB of {1:N1} GB" -f (($os.TotalVirtualMemorySize - $os.FreeVirtualMemory) / 1MB), ($os.TotalVirtualMemorySize / 1MB)
"Uptime: " + ((Get-Date) - $os.LastBootUpTime).ToString("d' d 'h' h'")
$rows | Group-Object Name | ForEach-Object {
  [pscustomobject]@{ Name = $_.Name; Count = $_.Count; Priv_MB = [math]::Round(($_.Group | Measure-Object Priv_MB -Sum).Sum) }
} | Sort-Object Priv_MB -Descending | Select-Object -First 25 | Format-Table -AutoSize | Out-String -Width 200
```

- Record the RAM used and commit lines as the "before" numbers.
- Judge memory by private MB. After days of uptime, leaked memory is paged out and working set hides it.

## Step 2: list candidates

```powershell
$rows = Import-Csv (Join-Path $env:TEMP 'freeram\snapshot.csv')
$keep = 'claude.exe', 'codex.exe', 'ChatGPT.exe', 'chrome.exe', 'msedge.exe', 'msedgewebview2.exe', 'firefox.exe', 'brave.exe', 'Slack.exe', 'ms-teams.exe', 'Teams.exe', 'Discord.exe', 'Code.exe', 'Cursor.exe', 'Windsurf.exe', 'WindowsTerminal.exe', 'Docker Desktop.exe'
$kf = Join-Path $HOME '.claude\freeram\keep-list.txt'
if (Test-Path $kf) { $keep += Get-Content $kf | ForEach-Object { $_.Trim() } | Where-Object { $_ } }
$system = 'System', 'System Idle Process', 'Registry', 'Memory Compression', 'smss.exe', 'csrss.exe', 'wininit.exe', 'winlogon.exe', 'services.exe', 'lsass.exe', 'fontdrvhost.exe', 'dwm.exe', 'svchost.exe', 'explorer.exe', 'conhost.exe'
$hostRx = '^(node|python|pythonw|bash|sh|cmd|powershell|pwsh|uv|uvx|deno|bun|git|jq|tail|grep|nohup|gcloud)\.exe$'
$helperRx = '(?i)mcp|language-?server|tsserver|pylance|rust-analyzer|gopls|eslint_d|--headless|--remote-debugging-p(ort|ipe)|playwright|next dev|vite|webpack|\btsx\b'
$rows | Where-Object {
    $_.Name -notin $system -and -not $_.Title -and $_.Cmd -notmatch '--type=' -and
    ($_.Name -notin $keep -or $_.Cmd -match '--headless|--remote-debugging-p(ort|ipe)') -and
    ($_.Name -match $hostRx -or $_.Orphan -eq 'True' -or $_.Cmd -match $helperRx)
  } |
  Sort-Object { [double]$_.Priv_MB } -Descending | Select-Object -First 80 |
  Select-Object PID, PPID, Name, Created, Priv_MB, Orphan, Chain, @{ n = 'Cmd'; e = { if ($_.Cmd.Length -gt 160) { $_.Cmd.Substring(0, 160) } else { $_.Cmd } } } |
  Format-Table -AutoSize -Wrap | Out-String -Width 400
```

The list leaves out keep-list apps (except automation-only browsers), anything with a window title, and browser or Electron `--type=` children. Treat a row as a candidate when any of these hold:

- `Orphan` is True: the parent chain dead-ends, or the parent PID was reused by a newer process. Many desktop apps and vendor services (sync clients, updaters, tray tools) also show a dead launcher; those are apps, not leaks. Report a large one, never kill it unasked.
- The command line is a per-session helper: `npx ... mcp`, `*-mcp`, `mcp-remote`, `mcp-proxy`, `uvx ...`, `server.mjs`, and similar. A live parent does not clear it: the leak is a parent that forgot a finished child.
- It is an IDE language server or debug adapter (`tsserver.js`, `pylance`, `rust-analyzer`, `gopls`, `eslint_d`, `copilot-language-server`, `vscode-*-server`) left under a live `Code.exe` or `Cursor.exe` after its workspace closed.
- It is an automation-only browser child (`--headless`, `--remote-debugging-port`, no window title) that nothing drives any more. The browser keep-list protects the user's own browser windows, not these.
- It is a one-off script call (`npx tsx some-script.ts`) that has sat for hours.
- It is a hung `git diff --quiet ...` or `git ls-files --others ...` call, or a statusline `jq.exe` call, with a dead parent.

Ask instead of killing, even at zero CPU:

- Dev servers (`next dev`, `vite`, `webpack serve`), databases, build watchers, language servers for an open workspace.
- Local model servers such as `ollama serve`. Run `ollama ps` first; an empty list means no model is loaded.
- `tail -f <log> | grep --line-buffered ...` pairs left by finished sessions. They poll, so they never read as fully idle.
- An unfamiliar desktop app with no window title and a dead parent.

## Step 3: find duplicate singletons

Some helpers should run once per parent (an extension host, a workspace language server, an MCP stdio server). A parent that respawns one after a crash or reload often leaves the old copy running.

```powershell
$rows = Import-Csv (Join-Path $env:TEMP 'freeram\snapshot.csv')
function Normalize-Cmd($cmd) {
  if (-not $cmd) { return '' }
  ($cmd -replace '[A-Fa-f0-9]{8,}', '#hash#' -replace '\b\d{3,}\b', '#num#')
}
$noise = 'RuntimeBroker\.exe|conhost\.exe|--type=(renderer|utility|gpu-process)|/factory,\{|^\s*"?[^"]*\\(claude|Cursor|Code|codex)\.exe"?\s*$'
$dupes = $rows | Where-Object { $_.Cmd -and $_.Cmd -notmatch $noise } |
  Group-Object Root, @{ Expression = { Normalize-Cmd $_.Cmd } } |
  Where-Object { $_.Count -gt 1 }
$dupes | ForEach-Object {
  "=== $($_.Name) ($($_.Count) instances) ==="
  $_.Group | Select-Object PID, PPID, Created, Priv_MB | Sort-Object Created | Format-Table -AutoSize | Out-String
}
```

- The noise filter drops groups that fan out by design: `RuntimeBroker.exe`, `conhost.exe`, browser and Electron `--type=` children, COM surrogates, and top-level CLI and editor sessions. When a new by-design group shows up, extend `$noise` for this run instead of killing it.
- A duplicate group is a prior, not a verdict. Normalizing can merge two different services, such as two dev servers on different ports. Confirm the arguments describe the same singleton.
- In a real duplicate group the newest instance is usually the live one. Older ones are leaked respawns; CPU-check them in step 4 anyway.
- Idle CPU does not tell a leaked singleton from the live one, because a live one also idles between requests. The grouping does that work.
- The Claude desktop app starts two `uv ... run windows-mcp` trees a few seconds apart at every launch. That pair is by design; never kill one as a duplicate.
- Each `claude.exe` CLI session runs its own MCP servers. That is by design while the session lives.
- Codex desktop (`codex.exe ... app-server` under `ChatGPT.exe`) can start a full MCP set every 12 to 13 minutes during a recurring task without reaping the old sets. Kill sets older than 30 minutes at zero CPU and keep the newest few.

## Step 4: prove idleness

```powershell
$targetPids = 1111, 2222, 3333  # candidates from steps 2 and 3
$before = @{}
foreach ($procId in $targetPids) { $g = Get-Process -Id $procId -ErrorAction SilentlyContinue; if ($g) { $before[$procId] = $g.CPU } }
Start-Sleep -Seconds 8
$out = foreach ($procId in $targetPids) {
  $g = Get-Process -Id $procId -ErrorAction SilentlyContinue
  if (-not $g) { [pscustomobject]@{ PID = $procId; Alive = $false } }
  else {
    $delta = if ($before.ContainsKey($procId)) { [math]::Round($g.CPU - $before[$procId], 3) } else { $null }
    [pscustomobject]@{ PID = $procId; Name = $g.ProcessName; Alive = $true; DeltaCPU = $delta; Priv_MB = [math]::Round($g.PrivateMemorySize64 / 1MB, 1) }
  }
}
$out | Sort-Object DeltaCPU -Descending | Format-Table -AutoSize | Out-String
```

- When DeltaCPU is about 0, the process is idle and can be killed, with three exceptions:
  - An interactive shell (`powershell.exe`, `cmd.exe`, `bash.exe` with no script argument) whose chain ends at a live terminal host such as `WindowsTerminal.exe` is an open tab. Kill it only when its chain is broken or the terminal is gone.
  - A helper created in the last 30 minutes may be between tool calls in an active session. Ask instead of killing.
  - A persistent service someone would expect to stay up (see "Ask instead of killing" in step 2).
- When DeltaCPU is above 0, something is using it. Do not kill; list it as in use.
- When DeltaCPU is above 0, the parent is dead, and private memory is large or growing against the step 1 snapshot, it is a runaway (for example a CLI call that should take seconds but has burned a core for hours). Kill it and its orphaned shell parent unless it is on the keep-list.

## Step 5: kill

- Kill without asking only when every PID on the list is a script host or helper (a `$hostRx` name from step 2, or an automation-only browser), is not keep-listed, has no window title, read DeltaCPU about 0, is older than 30 minutes, and, for duplicates, is confirmed to be the same singleton.
- List everything else separately and ask with AskUserQuestion before touching it.
- When killing a tree, kill children before parents. Skip a child created before its parent (PID reuse) and any keep-list name.

```powershell
$killList = 1111, 2222  # confirmed idle
$freedMB = 0
$results = foreach ($procId in $killList) {
  $g = Get-Process -Id $procId -ErrorAction SilentlyContinue
  if (-not $g) { [pscustomobject]@{ PID = $procId; Name = '(already gone)'; Priv_MB = 0; Killed = $false; Error = '' }; continue }
  $mb = [math]::Round($g.PrivateMemorySize64 / 1MB, 1)
  try {
    Stop-Process -Id $procId -Force -ErrorAction Stop
    $freedMB += $mb
    [pscustomobject]@{ PID = $procId; Name = $g.ProcessName; Priv_MB = $mb; Killed = $true; Error = '' }
  } catch {
    [pscustomobject]@{ PID = $procId; Name = $g.ProcessName; Priv_MB = $mb; Killed = $false; Error = $_.Exception.Message }
  }
}
$results | Sort-Object Priv_MB -Descending | Format-Table -AutoSize | Out-String -Width 200
"Freed about $([math]::Round($freedMB, 1)) MB private across $(@($results | Where-Object Killed).Count) processes"
```

## Step 5b: check Desktop Window Manager

`dwm.exe` can leak over multi-day uptime, and a bloated DWM makes the whole UI sluggish. Steps 2 to 5 never catch it because it is neither an app nor an orphan. Check it on every run.

```powershell
$mySession = (Get-Process -Id $PID).SessionId
$dwm = Get-Process dwm | Where-Object SessionId -eq $mySession
[pscustomobject]@{
  PID = $dwm.Id
  Private_MB = [math]::Round($dwm.PrivateMemorySize64 / 1MB)
  WS_MB = [math]::Round($dwm.WorkingSet64 / 1MB)
  Uptime = ((Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime).ToString("d' d 'h' h'")
} | Format-Table -AutoSize | Out-String
```

- When Private_MB is under 800: healthy. Report the number and move on.
- When Private_MB is 800 to 1500: report it and offer the restart. Do not launch it unasked.
- When Private_MB is over 1500: tell the user the screen will go black for about a second and open windows are unaffected, then launch the restart below. The Windows UAC prompt is the user's approval.
- Never `Stop-Process dwm` from the tool shell. It runs without elevation and gets Access Denied. Always go through `-Verb RunAs`.
- Never suggest `Win+Ctrl+Shift+B` to reclaim memory. It only resets the graphics driver; DWM keeps its PID and its memory.

Restart (shows a UAC prompt; Windows relaunches DWM within about a second):

```powershell
$mySession = (Get-Process -Id $PID).SessionId
$old = Get-Process dwm | Where-Object SessionId -eq $mySession
$oldPrivMB = [math]::Round($old.PrivateMemorySize64 / 1MB)
try {
  Start-Process -FilePath "$env:SystemRoot\System32\taskkill.exe" -ArgumentList '/f', '/pid', $old.Id -Verb RunAs -WindowStyle Hidden -Wait -ErrorAction Stop
} catch {
  "DWM restart not run (UAC declined or timed out): $($_.Exception.Message)"
  return
}
$new = $null
for ($i = 0; $i -lt 15; $i++) {
  Start-Sleep -Seconds 1
  $new = Get-Process dwm -ErrorAction SilentlyContinue | Where-Object { $_.SessionId -eq $mySession -and $_.Id -ne $old.Id }
  if ($new) { break }
}
if (-not $new) { "No new DWM after 15 s. Tell the user to press Win+Ctrl+Shift+B or Ctrl+Alt+Del to recover the display."; return }
Start-Sleep -Seconds 5
$new = Get-Process -Id $new.Id
"DWM restarted: PID $($old.Id) -> $($new.Id), private $oldPrivMB MB -> $([math]::Round($new.PrivateMemorySize64 / 1MB)) MB"
```

- When nobody is at the keyboard, the UAC prompt times out and the catch branch reports it. Report that and continue with step 6.
- Expect RAM used to drop far less than DWM's freed memory, because file cache refills the space. Report DWM private MB and commit as the proof.
- When DWM is back above about 1 GB within a day of a restart, something feeds the leak. Recommend updating the graphics driver or closing GPU overlay tools (such as NVIDIA Broadcast) as a test, instead of restarting DWM again and again.

## Docker and WSL

`Stop-Process` is the wrong tool for VM-backed memory: `vmmem`, `vmmemWSL`, `wslhost.exe`, `com.docker.backend.exe`.

- Wrap every `wsl.exe` and `docker` call in `Start-Job` with `Wait-Job -Timeout 75`, then `Receive-Job`. Strip NUL characters from `wsl.exe` output with `-replace "`0", ''`.
- Run `docker ps` first and report the running containers. Stop Docker only when the user says so.
- Never `docker stop` a container whose restart policy is `unless-stopped` to prepare a shutdown: it will not come back on the next start.
- A Linux distro keeps its page cache, which shows up as `vmmemWSL` private memory. Offer `wsl -d <distro> -u root -e sh -c "sync; echo 3 > /proc/sys/vm/drop_caches"`. Windows takes the memory back over about 2 minutes; re-measure after that before calling it a failure.
- Run `wsl --shutdown` only when the user says WSL is not needed. `wsl shutdown` without dashes runs Linux `shutdown` inside the distro and changes nothing.
- When `vmmemWSL` keeps growing, suggest a `memory=` cap in `%USERPROFILE%\.wslconfig`, applied by `wsl --shutdown`.
- When Docker Desktop hangs after a restart (frontends still running, engine API returning 500), and the user agrees: `Stop-Process -Force` on `Docker Desktop`, `com.docker.backend` and `com.docker.build`, then start `C:\Program Files\Docker\Docker\Docker Desktop.exe`.

## Known patterns

- Explorer can hold several GB after days with many folder windows open. Ask first, because a restart closes every folder window. `Stop-Process -Name explorer -Force`; the shell comes back by itself.
- Vendor services under `services.exe` (for example `AppHelperCap.exe` or `OverlayHelper.exe`) can be large but are not orphans. Report them; do not kill them.
- A local model server started twice by an agent (for example `ollama serve` on two ports, each launched through `nohup.exe`) holds memory with no model loaded. Ask, then stop both servers and their `nohup.exe` parents.

## Step 6: report

```powershell
$os = Get-CimInstance Win32_OperatingSystem
"RAM used: {0:N1} GB of {1:N1} GB" -f (($os.TotalVisibleMemorySize - $os.FreePhysicalMemory) / 1MB), ($os.TotalVisibleMemorySize / 1MB)
"Commit: {0:N1} GB of {1:N1} GB" -f (($os.TotalVirtualMemorySize - $os.FreeVirtualMemory) / 1MB), ($os.TotalVirtualMemorySize / 1MB)
```

- Group what was killed by cause (runaway, stuck script, leaked MCP server, duplicate respawn, hung git call), with private MB per group. No raw PID dump.
- List what was left alone and why: keep-list, window title, nonzero CPU, or waiting on the user's answer.
- Show RAM used and commit before and after, and DWM private MB before and after when it was restarted.
- When a killed process was an MCP server, say that its tools reconnect on next use.
- On `--dry-run`, title the report "dry run, nothing killed" and show the planned kills with their private MB.
