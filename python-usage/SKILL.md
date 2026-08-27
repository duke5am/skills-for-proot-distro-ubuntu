---
name: python-usage
description: How to run Python correctly in this environment (Ubuntu 26.04 PRoot on Android) — write .py files instead of inline -c, use file logging, avoid piping output to tail/head, run in the background and monitor, and set single-thread env vars to avoid numpy/OpenBLAS stalls. Load when a Python script hangs, needs monitoring, or uses numpy/pandas.
whenToUse: Use when writing or running Python scripts, when a Python command hangs or times out, when using numpy/pandas, or when the harness write tool fails with EPERM.
---

# Using Python correctly in this environment

## Environment
- Ubuntu 26.04 LTS (aarch64) under PRoot on Android; `python3` = 3.14.4 at `/usr/bin/python3`.
- Install libraries with **apt** (`apt install python3-<name>`), not pip — see the `package-installation` skill (PEP 668 blocks pip).
- `pandas`/`numpy` take ~5s to import; that is normal here.

## Two things make Python "hang" — fix both

1. **Piping output into `tail`/`head` (the common 240s-timeout cause).** A numpy/pandas process keeps the pipe's write end open, so the reader never sees EOF and the pipeline blocks until the shell timeout. Redirect to a file and read it in a separate command:

```bash
python3 script.py > /tmp/out.log 2>&1
cat /tmp/out.log    # separate command
```

2. **OpenBLAS thread stalls.** numpy/OpenBLAS can spin up threads that stall under PRoot, making operations hang intermittently. Force single-threading before importing numpy/pandas:

```bash
export OMP_NUM_THREADS=1 OPENBLAS_NUM_THREADS=1 MKL_NUM_THREADS=1 NUMEXPR_NUM_THREADS=1
python3 script.py
```

```python
# top of script.py, BEFORE importing numpy/pandas
import os
for v in ('OMP_NUM_THREADS','OPENBLAS_NUM_THREADS','MKL_NUM_THREADS','NUMEXPR_NUM_THREADS'):
    os.environ[v] = '1'
import pandas as pd, numpy as np
```

With both fixes, a pandas analysis that previously timed out completes in ~0.2s.

## Write a .py file — do not use `python3 -c`
Inline `-c` is opaque when it hangs. Write a real script so you can log, run in background, and re-run:

```bash
cat > script.py << 'DELIM'
...code...
DELIM
python3 script.py
```

(The harness `write` tool fails with `EPERM` on `/sdcard` — FUSE does not support its atomic rename — so create files via `bash` heredoc or Python `open('w')`. Pick a heredoc delimiter that does not appear in the content.)

## Log to a file with a `mark()` helper
```python
import logging, time
logging.basicConfig(filename='/tmp/script.log', level=logging.INFO, format='%(asctime)s %(message)s')
log = logging.getLogger(); t0 = time.time()
def mark(s): log.info(f"[+{time.time()-t0:5.1f}s] {s}")

mark("start")
# ... one mark() after each step ...
mark("done")
```
Then inspect `cat /tmp/script.log` to see exactly which step hung.

## Run in the background and monitor
For anything that may run long, use the harness `run_in_background: true` and read the job with `job_output`. Avoid detached `( ... & )` subshells — proot-distro may not reap them cleanly (see the `shell-usage` skill).

## Other notes
- Read CSVs that may carry a BOM (e.g. a Curve export) with `encoding='utf-8-sig'`.
- If `apt install` was interrupted (SIGTERM), packages can be left `iU`/`iHR` (half-installed). The library often still imports and works — verify with a real operation before reinstalling. Clean up with `dpkg --configure -a`.
- Diagnose a hang with a minimal probe script first (import → one op → read a small file), logging each step, before assuming a library is broken.
