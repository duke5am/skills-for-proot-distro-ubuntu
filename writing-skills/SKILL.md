---
name: writing-skills
description: How to author, validate, and sanity-check skills in this DeepSeek Harness environment. Covers the SKILL.md location and frontmatter format, naming rules, how to write the file without hitting FUSE/heredoc pitfalls, and a checklist to verify an existing skill is correct and discovered.
whenToUse: Use when creating a new skill, when asked to review or verify existing skills, or when a skill is not showing up in the session catalog.
---

# Writing and sanity-checking skills

## Where skills live
A skill is a directory containing a `SKILL.md`. Roots are scanned by the `skill-filesystem` provider:
- **User root:** `/root/.dsh/skills/<name>/SKILL.md` (DSH_HOME is `/root/.dsh`) — for environment- or user-wide skills.
- **Project roots:** `<projectRoot>/.dsh/skills/<name>/SKILL.md` and `<projectRoot>/.agents/skills/<name>/SKILL.md` — for a specific project.

## Required format
Markdown with YAML frontmatter. Only `name` and `description` are required; `whenToUse` is strongly recommended:

```yaml
---
name: my-skill
description: One or two sentences saying what the skill does and WHEN to use it.
whenToUse: Use when <condition>.
---

# Body — concrete, actionable instructions
```

## Naming rules
`name` must match `^[a-z0-9]+(?:-[a-z0-9]+)*$` — lowercase letters, digits and single hyphens (kebab-case); no underscores, spaces, or leading/trailing/double hyphens. The directory name must equal the `name`.

## What the loader enforces
- Frontmatter must parse as valid YAML.
- `name` and `description` must both be present and non-empty.
- `name` must match the kebab-case regex.
- On any violation the file is **silently ignored** (a warning is logged), so a typo = the skill never loads.

## Writing the file without pitfalls
- The harness `write` tool can EPERM on `/sdcard` (FUSE). Prefer Python `open('w')` or a bash heredoc (see the `shell-usage` skill).
- Bash heredoc: the delimiter must NOT appear in the content — an example that shows `<< 'EOF'` inside the skill will terminate an outer `<< 'EOF'` heredoc early. Use a unique delimiter.
- Python triple-quoted write: the outer delimiter (three single or three double quotes) must not appear in the content either.

## Sanity-checking an existing skill
For each skill verify:
1. `name` matches the kebab-case regex AND equals the directory name.
2. Frontmatter is valid YAML with non-empty `name` and `description`.
3. `description` states the trigger (when the model should load it), not just a title.
4. `whenToUse` is present (improves matching).
5. Body is concrete and environment-accurate — re-verify any environment claims (paths, commands, timeouts) against the real environment before trusting them.
6. Cross-references between skills are consistent (no contradictions, e.g. one says "do X" and another "avoid X").
7. The skill actually appears in the session catalog after creation.

Plain-Python frontmatter check (no pandas needed):
```python
import re
txt = open('/root/.dsh/skills/<name>/SKILL.md').read()
fm = txt.split('---', 2)[1] if txt.startswith('---') else ''
nm = re.search(r'^name:\s*([a-z0-9-]+)\s*$', fm, re.M)
ds = re.search(r'^description:\s*(.+)$', fm, re.M)
ok = bool(nm and ds and re.fullmatch(r'[a-z0-9]+(?:-[a-z0-9]+)*', nm.group(1)))
print('valid' if ok else 'INVALID')
```

To see what is registered, read the session's available-skills list (name + description). A skill that is valid but absent is not being discovered — check the directory name, frontmatter, and whether the catalog needs a refresh.
