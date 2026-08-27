# DeepSeek Harness (dsh) Skills for PRoot-Distro Ubuntu on Android

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![dsh compatible](https://img.shields.io/badge/dsh-compatible-4D6BFE.svg)](https://github.com/deepseek-ai/dsh)
[![PRoot](https://img.shields.io/badge/PRoot-supported-green.svg)](https://github.com/termux/proot-distro)
[![Termux](https://img.shields.io/badge/Termux-ready-21b352.svg)](https://github.com/termux/termux-app)
[![Ubuntu 26.04 LTS](https://img.shields.io/badge/Ubuntu-26.04%20LTS-E95420.svg)](https://ubuntu.com)

> Production-tested [DeepSeek Harness](https://github.com/deepseek-ai/dsh) (dsh) skills that eliminate sub-agent execution failures, path-resolution errors, and `pip`/`apt` confusion when running Ubuntu 26.04 LTS under PRoot inside Termux on Android.

Reusable [DeepSeek Harness](https://github.com/deepseek-ai/dsh) skills for the **Ubuntu-under-PRoot-on-Android** environment (Termux + proot-distro). Each directory is one skill (`SKILL.md`) that teaches the model the environment-specific commands and gotchas a stock Linux model gets wrong — the root cause of most sub-agent timeouts, `EPERM` write failures, and "package not found" errors in a rootless mobile container.

## Quick install

One line, via `git clone`:

```bash
git clone --depth 1 https://github.com/duke5am/skills-for-proot-distro-ubuntu.git /tmp/dsh-skills && cp -r /tmp/dsh-skills/{package-installation,python-usage,shell-usage,writing-skills} ~/.dsh/skills/
```

Or via a `curl` tarball (no git needed):

```bash
curl -L https://github.com/duke5am/skills-for-proot-distro-ubuntu/archive/refs/heads/main.tar.gz -o /tmp/dsh-skills.tar.gz && tar xzf /tmp/dsh-skills.tar.gz -C /tmp && cp -r /tmp/skills-for-proot-distro-ubuntu-main/{package-installation,python-usage,shell-usage,writing-skills} ~/.dsh/skills/
```

(Adjust `~/.dsh` to wherever your harness `DSH_HOME` points — in this setup it's `/root/.dsh`.)

## The skills

| Skill | What it covers |
|---|---|
| `package-installation` | Install Python libs via **apt** (PEP 668 blocks pip), find package names, background big installs, recover half-installed packages. |
| `python-usage` | Run Python correctly here — write `.py` files not `-c`, file logging, no `tail` piping, background + monitor, numpy/OpenBLAS single-thread env vars. |
| `shell-usage` | Shell gotchas — the pipe-hang, writing files when the `write` tool EPERMs, heredoc delimiter collisions, background jobs, diagnostics. |
| `writing-skills` | Author/validate/sanity-check skills: `SKILL.md` location, frontmatter format, naming rules, FUSE/heredoc pitfalls, verification checklist. |

## Usage examples

### `package-installation` — install a Python library the right way

```bash
# ✗ refused by PEP 668 ("externally-managed-environment")
pip install pandas

# ✓ installs into /usr/lib/python3/dist-packages; import works immediately
apt install python3-pandas
```

### `python-usage` — stop numpy/pandas scripts from hanging

```bash
# ✗ can hang until the shell's ~240s timeout (pipe never sees EOF)
python3 script.py 2>&1 | tail -20

# ✓ redirect to a file, then read it in a separate command
python3 script.py > /tmp/out.log 2>&1
cat /tmp/out.log
```

```bash
# ✓ force single-threading before importing numpy/pandas
export OMP_NUM_THREADS=1 OPENBLAS_NUM_THREADS=1 MKL_NUM_THREADS=1 NUMEXPR_NUM_THREADS=1
python3 script.py
```

### `shell-usage` — write a file when the harness `write` tool EPERMs

```bash
cat > path/to/file << 'DELIM'
...content...
DELIM
```

### `writing-skills` — verify a skill will actually load

```python
import re
txt = open('/root/.dsh/skills/my-skill/SKILL.md').read()
fm = txt.split('---', 2)[1] if txt.startswith('---') else ''
nm = re.search(r'^name:\s*([a-z0-9-]+)\s*$', fm, re.M)
ds = re.search(r'^description:\s*(.+)$', fm, re.M)
print('valid' if (nm and ds and re.fullmatch(r'[a-z0-9]+(?:-[a-z0-9]+)*', nm.group(1))) else 'INVALID')
```

## Installing on a new phone

1. Run the one-line install above (or clone the repo and copy the four folders).
2. Restart/refresh the harness session so the skill catalog rescans.
3. Verify with the `writing-skills` checklist (or read the session's available-skills list and confirm the four names appear).

## Notes

- These skills are specific to Ubuntu-on-PRoot-on-Android. On a normal Linux box (or a venv-based setup), the pip/apt guidance differs — adjust accordingly.
- `writing-skills` documents how the loader validates skills, so if a copied skill doesn't show up, that's the one to consult first.

## License

MIT — see [LICENSE](LICENSE).
