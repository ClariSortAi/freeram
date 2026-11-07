# freeram

A Claude Code plugin for Windows that finds processes nobody is using any more, proves each one is idle, and kills only those. It also checks Desktop Window Manager (`dwm.exe`) for a memory leak.

Machines that run AI coding tools, MCP servers, editors and browser automation for days without a reboot collect leftovers. Each tool starts helper processes for a session or a task, and the parent app often fails to clean them up after a crash, a window reload or a finished task. Any one leak is small, but after a week of uptime they can add up to gigabytes. freeram finds them and leaves the apps you have open alone.

## Install

In Claude Code:

```
/plugin marketplace add ClariSortAi/freeram
/plugin install freeram@freeram
```

Then run `/reload-plugins` or restart Claude Code.

Requirements: Windows and Claude Code's PowerShell tool. The tool is on by default on Windows for claude.ai and Console accounts. On Amazon Bedrock, Google Cloud or Microsoft Foundry, or if you turned it off, add `"CLAUDE_CODE_USE_POWERSHELL_TOOL": "1"` under `"env"` in `~/.claude/settings.json` and restart Claude Code.

The plugin costs nothing until you run it. Its skill is user-invoked only, so its description never enters the context.

## Usage

```
/freeram:sweep              find, verify and kill abandoned processes
/freeram:sweep --dry-run    show what it would kill, kill nothing
```

On a machine you haven't run it on before, start with `--dry-run`.

## What it does

1. Takes one snapshot of every process with its parent chain, memory and command line, and saves it to `%TEMP%\freeram\snapshot.csv`.
2. Picks candidates: processes whose parent has exited, MCP servers and language servers their app left behind, headless browser instances nothing drives, and hung scripts or git calls.
3. Looks for helpers that should run once per app but run twice, and flags the older copies.
4. Samples each candidate's CPU time twice, 8 seconds apart. A process that used CPU in that window is in use and stays.
5. Kills the confirmed idle processes one at a time and asks you about anything less certain.
6. Checks how much private memory DWM holds and offers a restart when it has grown.
7. Reports what it killed, grouped by cause, what it left alone and why, and RAM used and commit before and after.

## What it leaves alone

- Claude Code sessions, the Claude desktop app, Codex, browsers, Slack, Teams, Discord, VS Code, Cursor, Windsurf, Windows Terminal, Explorer and Docker Desktop. These are on a built-in keep-list. Helper processes they left behind can still be cleaned up.
- Anything with a window title, unless you say yes.
- Windows system processes and vendor services, such as sync clients and updaters.
- Any process that used CPU during the sample. The exception is a runaway: its parent is gone and it keeps burning CPU and memory.
- Dev servers, databases, build watchers and local model servers. freeram asks about these even when they are idle.
- Helpers started in the last 30 minutes. They may be between tool calls in a live session, so freeram asks.

When freeram finds an unfamiliar app holding a lot of memory, it asks whether to keep it. If you say yes, the name goes into `~/.claude/freeram/keep-list.txt`, one process name per line, and later runs skip it. You can edit that file by hand too.

It kills by process ID, never by name, and never kills a whole tree in one command. The one exception is restarting Explorer, which it does only after you agree.

## Desktop Window Manager

`dwm.exe` draws your screen. On some machines it leaks memory over several days of uptime until the whole UI feels slow. freeram checks it on every run:

- Under 800 MB private, it reports the number and moves on.
- Between 800 and 1,500 MB, it offers a restart.
- Over 1,500 MB, it warns that the screen will go black for about a second and starts the restart. Open windows are not affected.

DWM can only be stopped with administrator rights, so the restart always goes through a Windows UAC prompt, and nothing happens unless you approve it. If nobody is at the keyboard, the prompt times out and the run carries on.

## Docker and WSL

Killing a process can't reclaim memory held by WSL (`vmmemWSL`) or Docker. freeram reports it and offers the fix for that layer: dropping the Linux page cache, `wsl --shutdown` when you don't need WSL, or a `memory=` cap in `.wslconfig`. It stops Docker or WSL only when you tell it to.

## Data and privacy

freeram has no code of its own. It is a set of instructions that Claude follows with its PowerShell tool, and it sends nothing anywhere by itself.

Claude reads:

- The process list: names, process IDs, parent IDs, start times, memory, CPU time, window titles and command lines.
- `~/.claude/freeram/keep-list.txt`, if it exists.

Command lines can contain secrets, because some programs take tokens or passwords as arguments. Before the snapshot is saved, freeram masks values that follow words like token, password, secret, api-key and bearer, plus common key formats such as `sk-...`, `ghp_...`, `AKIA...` and JSON web tokens. The masking works by pattern, so an unusual format can slip through.

The process details Claude reads go into your Claude Code conversation, like any other command output in a session.

It writes:

- `%TEMP%\freeram\snapshot.csv`, the process snapshot. Each run overwrites it, and you can delete it at any time.
- `~/.claude/freeram/keep-list.txt`, only when you ask it to keep an app.

## Things to know

- The skill pre-approves only Read, Glob and Grep. Every PowerShell command, including each kill, asks for permission first.
- A killed MCP server reconnects the next time its tools are used.
- RAM used often drops less than the memory freed, because Windows refills free RAM with file cache. The report shows commit and per-process private memory, which are better measures.
- Killing processes always carries some risk. freeram errs toward asking.

## Development

```
claude plugin validate .      manifest and skill checks
claude --plugin-dir .         load this checkout in a session
```

The whole plugin is `skills/sweep/SKILL.md`.

## License

MIT
