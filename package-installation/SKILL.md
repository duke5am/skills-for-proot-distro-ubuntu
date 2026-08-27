---
name: package-installation
description: Install Python libraries and system packages in this environment (Ubuntu 26.04 LTS running under PRoot on Android). Use this whenever a task needs a Python module (pandas, requests, pypdf, numpy, etc.) or any apt package — `pip install` is refused by PEP 668, so the correct command is `apt install python3-<module>`.
whenToUse: Use when installing packages, when `pip install` fails with "externally-managed-environment", or when a task requires a Python library that is not yet importable.
---

# Package installation in this environment

## Environment at a glance
- **OS:** Ubuntu 26.04 LTS (aarch64), running under **PRoot** on Android. `HOME=/root`; the workspace is mounted at `/sdcard` (a FUSE filesystem).
- **Python:** `/usr/bin/python3` (3.14.4). `pip3` 25.1.1 is present, but system site-packages are **externally managed (PEP 668)**.
- **No sudo needed:** the shell already runs as root.

## Rule of thumb
Install Python libraries with **apt**, not pip:

```bash
apt install python3-<module>
```

Debian/Ubuntu package most Python modules as `python3-<name>`:

```bash
apt install python3-pypdf python3-requests python3-pandas python3-numpy
```

apt installs into `/usr/lib/python3/dist-packages/`, which is already on `sys.path`, so `import <module>` works immediately — no venv, no pip.

## Large installs can exceed the 240s shell timeout
Heavy packages (e.g. `python3-pandas`, which pulls numpy) can take longer than the shell's ~240s timeout and be killed mid-install, leaving packages half-installed. Run them in the background; if interrupted, finish with:

```bash
dpkg --configure -a
```

A package left in `iU`/`iHR` state (`dpkg -l`) often still imports and works — verify with a real import before reinstalling.

## Find the right package name
Before installing, confirm the module is packaged and get its exact name:

```bash
apt-cache search --names-only "^python3-<name>$"
apt-cache policy python3-<name>
```

If the search is empty or stale, refresh the index first:

```bash
apt-get update
```

## Why pip fails (do not keep retrying it)
`pip3 install <pkg>` and `pip3 install --user <pkg>` are **refused immediately** (under a second) with:

```
error: externally-managed-environment
```

This is PEP 668 and happens before any network access. (An earlier `pip … | tail` that appeared to "hang" was the shell pipe, not pip — see the `shell-usage` skill.)

Only after `--break-system-packages` does pip actually reach PyPI, and that network fetch can be slow in this PRoot environment.

So the reliable path is apt.

## If you genuinely need a package that is not apt-packaged
Try, in order:

1. `apt-cache search <term>` — a differently named apt package may already cover it.
2. A virtualenv — venv is available and its pip is not PEP-668-restricted (still needs PyPI network):
   ```bash
   python3 -m venv .venv && source .venv/bin/activate && pip install <pkg>
   ```
3. Last resort for a system-wide install (bypasses PEP 668, then hits PyPI network):
   ```bash
   pip3 install --break-system-packages <pkg>
   ```

## System (non-Python) packages
Use apt/apt-get directly, e.g. `apt install jq unzip git`. Run `apt-get update` first if the index is stale.

## Related gotcha: writing files
The harness `write` tool can fail with `EPERM` on `/sdcard` (FUSE does not support its atomic rename/hard-link). Write files via bash heredoc or Python instead (see the `shell-usage` skill for heredoc delimiter rules).
