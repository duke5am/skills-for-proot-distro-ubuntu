---
name: browser-usage
description: Drive a headless Chromium browser (Playwright) in this environment to load pages, extract rendered text/HTML/markdown, click/fill forms, capture screenshots/PDFs, and read images back through a DeepSeek vision MCP. Use whenever a task must visit a URL, read JS-rendered content, interact with a page, or "see" a screenshot.
whenToUse: Use when the task needs a real browser (rendered or JavaScript-heavy content, form interaction, screenshots, PDFs) or needs to understand the visual content of an image/screenshot.
---

# Browser + vision in this environment

There is no native `browser` tool, but a working headless Chromium and a vision MCP are installed. Drive them through `bash` (browser) and the `mcp__vision__*` tools (image reading).

## Browser CLI
```
/root/browser-tool/venv/bin/python /root/browser-tool/browse.py <cmd> ...
```
Everything lives in `/root/browser-tool/`: `browse.py`, `ui_review.py`, `vision_mcp.py`, `venv/` (Playwright 1.62 + `mcp` + Pillow), `profile/` (cookies/localStorage), `state.json` (last URL/title).

### Commands
- `fetch  [--url U] [--mode text|html|markdown] [--selector S] [--wait MS]`
- `title  [--url U]`
- `links  [--url U]`  — JSON list of {text, href}
- `boxes  <selector> [--url U] [--limit N]`  — JSON list of matches {index, tag, text, x, y, w, h}
- `eval   '<js expr>' [--url U]`  — runs JS, prints JSON
- `screenshot <out.png> [--url U] [--full-page] [--scale N] [--selector S] [--index I]`
  - `--scale 2` = retina (2x) capture; `--selector S [--index I]` = capture just the I-th matching element
- `pdf    <out.pdf> [--url U]`
- `click <selector>` / `fill <selector> <value>` / `type <selector> <text>` / `press <key>` — act on the current page
- `state` / `reset`

`--url` defaults to the persisted current page, so multi-step flows work across separate calls.

## Vision — reading images (first-class MCP tools)
The text-only model "sees" through two tools (wired via `~/.dsh/profiles/web/cordis.patch.yml`):
- `mcp__vision__describe_image(path_or_url, prompt, mode, json, crop, verify)`
- `mcp__vision__analyze_image(path_or_url, mode, tile_size, overlap, prompt)`

### The one rule that matters
`deepseek-v4-flash-vision-exp` downscales any image over ~640k px on ingest, so **small text is illegible in a full-page shot regardless of screenshot scale**. To read text: **crop** (`crop="x,y,w,h"`) a region, or **tile** (`analyze_image`). Geometry/layout analysis does NOT need this — use `mode="layout"`.

### describe_image
- `mode`: `auto` (general) | `layout` (geometry/spacing/alignment only, no OCR) | `ocr` (transcribe) | `qa` (direct answer + confidence).
- `crop`: `"x,y,w,h"` region zoom — mandatory for reading specific small text.
- `json`: return structured `{confidence, observations, inferences}`.
- `verify`: two independent OCR passes → `{reading, second_reading, agree}`. `agree:false` = the read is unreliable (a hallucinating model misreads inconsistently). `agree:true` ≠ guaranteed correct — still cross-check against source.

### analyze_image
Tiles a large image into overlapping ≤768px tiles (each under the downscale budget), analyzes each, returns `{image, grid, tiles:[{row,col,x,y,w,h,text}], full_text}`. Use for full-page TEXT (OCR). For geometry use `describe_image(mode="layout")` — layout survives downscaling, so tiling is unnecessary and slower.

## One-call UI review
```
/root/browser-tool/venv/bin/python /root/browser-tool/ui_review.py <url> [--selector S] [--scale N] [--elements K] [--full-page] [--ocr-full]
```
One browser session → DOM text (`body.innerText`) + screenshot + `boxes` → one `describe_image(mode=layout)` → per-element `describe_image(ocr, verify)`. Prints one JSON report. DOM text is exact and free, so vision OCR is only needed for text NOT in the DOM. `--ocr-full` adds a slow tiled `analyze_image(ocr)` over the whole screenshot (opt-in).

## Rules (read before using)
1. Always use the venv python path above. The system `playwright`/`python3-playwright` is Debian's `+ds` repack whose node driver is missing — broken.
2. Root, so `--no-sandbox --disable-gpu --disable-dev-shm-usage` are already passed — do not remove.
3. Never pipe `browse.py` output into `tail`/`head` (Chromium keeps the pipe open → 240s hang). Redirect to a file, then read it.
4. Browser calls share one profile dir — run sequentially (profile lock). Vision calls are stateless and can run in parallel.
5. Screenshots/PDFs go to files; text/markdown/JSON go to stdout.
6. Navigation timeout is 60s (`--timeout MS`); use `--wait 2000` on slow JS-heavy sites.
7. `analyze_image` caps at 64 tiles; on very tall pages use `crop`/`--selector` to target regions instead.
