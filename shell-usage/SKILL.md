---
name: shell-usage
description: How to run shell commands correctly in this environment (Ubuntu 26.04 under proot-distro on Android, /sdcard on FUSE). Covers the pipe-hang gotcha, writing files when the write tool EPERMs, heredoc delimiter collisions, background execution, and diagnostics. Load when a bash command hangs/times out, or when writing files fails.
whenToUse: Use when writing or running bash commands, when a command hangs or hits the 240s timeout, when the write tool fails with EPERM, or when creating files via heredoc.
---

# Using the shell correctly in this environment

## Environment
- `bash` running as root (`HOME=/root`); the workspace is usually `/sdcard/...` (a FUSE mount on Android, via proot-distro).
- Every command has a ~240s hard timeout (`settings.yaml` -> `shell.timeoutMs`). Anything longer must run in the background.
- The file sandbox is `danger-full-access` and approval prompts are disabled, so commands run unconfined.

## The #1 gotcha: pipes hang in proot-distro
Piping a program that spawns threads or forks (numpy/OpenBLAS, `dpkg`, `apt`) into `tail`/`head`/`grep` can hang forever: the child keeps the pipe's write end open, the reader never sees EOF, and the pipeline blocks until the 240s timeout.

**Don't:**
```bash
python3 script.py 2>&1 | tail -20     # can hang forever
```

**Do — redirect to a file, then read it in a separate command:**
```bash
python3 script.py > /tmp/out.log 2>&1
cat /tmp/out.log
```

The same rule applies to `apt`/`dpkg`/`pip` output — redirect to a log file rather than piping to `tail`.

## Avoid detached subshells `( ... & )`
proot-distro does not always reap `( cmd & )` subshells cleanly — they can linger and hold locks (e.g. the dpkg lock). Prefer the harness `run_in_background: true` + `job_output`, or run in the foreground with a file redirect. If you must background in-shell, redirect output so nothing keeps a tty/pipe open.

## Writing files (the `write` tool fails on /sdcard)
The harness `write` tool returns `EPERM` on `/sdcard` (FUSE does not support its atomic rename/hard-link). Write files with a bash heredoc or Python instead:

```bash
cat > path/to/file << 'DELIM'
...content...
DELIM
```

**Pick a delimiter that does NOT appear anywhere in the content.** If the content itself contains `DELIM`, the heredoc ends early and the remaining lines are executed as shell (syntax errors). `EOF` and `PYEOF` are common pitfalls when the content itself shows heredoc examples. Use a unique token.

Python is often safer for multi-line content with quotes/backticks:
```bash
python3 << 'UNIQUE'
open('path/to/file','w',encoding='utf-8').write("...content...")
UNIQUE
```

## Keep commands simple
One purpose per call. Avoid long `&&`/`;` chains that couple a hanging command to the rest — you lose the ability to see which step blocked. Run one step, inspect, then proceed.

## Diagnostics
- `ps aux | grep <name>` — find lingering processes (empty result = none).
- `dpkg -l | grep <name>` — package state: `ii` = installed, `iU`/`iHR` = half-installed (the library may still import and work; finish with `dpkg --configure -a` in the background).
- `ls -la /var/lib/dpkg/lock*` — check for a held dpkg lock.
- `cat /tmp/<name>.log` — inspect a script's log (see the `python-usage` skill for the logging pattern).

## Tools present vs absent
Present: `ls`, `cat`, `grep`, `find`, `head`, `tail`, `wc`, `sort`, `python3`, `apt`, `apt-get`, `dpkg`, `ps`.
Absent: `xxd`, `pdftotext`, `jq` (unless installed). Use Python for hexdumps, PDF text extraction, and JSON handling.
