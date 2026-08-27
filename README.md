# skills-for-proot-distro-ubuntu

Reusable [DeepSeek Harness](https://github.com/deepseek-ai/dsh) skills for the
**Ubuntu-under-PRoot-on-Android** environment (e.g. Termux + proot-distro).

Each directory is one skill (`SKILL.md`). Copy them into the harness user skills
root so they're auto-discovered:

```bash
cp -r package-installation python-usage shell-usage writing-skills \
  ~/.dsh/skills/
```

(Adjust `~/.dsh` to wherever your harness `DSH_HOME` points — in this setup it's
`/root/.dsh`.)

## The skills

| Skill | What it covers |
|---|---|
| `package-installation` | Install Python libs via **apt** (PEP 668 blocks pip), find package names, background big installs, recover half-installed packages. |
| `python-usage` | Run Python correctly here — write `.py` files not `-c`, file logging, no `tail` piping, background + monitor, numpy/OpenBLAS single-thread env vars. |
| `shell-usage` | Shell gotchas — the pipe-hang, writing files when the `write` tool EPERMs, heredoc delimiter collisions, background jobs, diagnostics. |
| `writing-skills` | Author/validate/sanity-check skills: `SKILL.md` location, frontmatter format, naming rules, FUSE/heredoc pitfalls, verification checklist. |

## Installing on a new phone

1. Clone this repo (or just copy the four folders).
2. `cp -r */ ~/.dsh/skills/` (or `/root/.dsh/skills/`).
3. Restart/refresh the harness session so the skill catalog rescans.
4. Verify with the `writing-skills` checklist (or just read the session's
   available-skills list and confirm the four names appear).

## Notes

- These skills are specific to Ubuntu-on-PRoot-on-Android. On a normal Linux
  box (or a venv-based setup), the pip/apt guidance differs — adjust accordingly.
- `writing-skills` documents how the loader validates skills, so if a copied
  skill doesn't show up, that's the one to consult first.
