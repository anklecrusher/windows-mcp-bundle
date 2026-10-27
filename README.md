# windows-mcp-bundle

A configuration-only [DeepSeek Harness](https://github.com/deepseek-ai) (DSH)
bundle that bridges [CursorTouch/Windows-MCP](https://github.com/CursorTouch/Windows-MCP)
into the harness using the shipped `@deepseek-ai/dsh-mcp-client`, giving the
model a real computer-use loop on Windows.

No JavaScript, no build step, no dependencies in this repository — one patch
file plus a Python environment that lives outside it.

## What it adds

Thirteen tools, namespaced `mcp__winmcp__*`:

| Tool | Purpose |
| --- | --- |
| `Screenshot` | Fast screenshot-first capture with cursor and window summaries |
| `Snapshot` | Full desktop state including the interactive UI tree and browser DOM |
| `Click` | Click by pixel coordinate or by element label |
| `Type` | Type text, optionally at a location, with clear / press-enter |
| `Scroll` | Scroll vertically or horizontally at a location |
| `Move` | Move the cursor, optionally dragging from one point to another |
| `Shortcut` | Send key combinations |
| `Wait` | Sleep for a fixed interval |
| `WaitFor` | Block until text or a window appears |
| `App` | Launch, resize, or switch applications |
| `DisplayInventory` | Enumerate monitors and their geometry |
| `Clipboard` | Read and write the clipboard |
| `ControlStatus` | Control-ownership readout (force-kept by the server) |

## Why the native MCP client, not `dsh-mcp-lazy`

`dsh-mcp-lazy` renders every image block as one line of text:

```
[image: image/png, 710946 bytes — this gateway returns text only, so the pixels are not forwarded]
```

Enabling its `directTools` promotion does not help — the promoted tools still
call `renderToolResult()`, whose output schema is fixed to `{ type: 'string' }`.
**Screenshot is the core of computer use**, so routing this server through that
gateway would leave a text-only stub of the plugin.

The native client instead runs the full admission chain —
`containsImage` → `decodeImage` → `resolveImageAdmission` →
`attachments.saveImages()` — which is what puts actual pixels into the
conversation. `decodeImage` requires a declared media type in
PNG/JPEG/WebP/GIF and canonical base64, both of which Windows-MCP satisfies.

`resolveImageAdmission` additionally requires the active model route to declare
image input. The `desktop` profile declares `input: [text, image]` for
`deepseek-v4.1-flash`, so admission passes. On a text-only route the image
degrades to a text diagnostic rather than failing the call.

## Why a dedicated Python 3.14 environment

`windows-mcp` 0.8.8 requires **Python >= 3.14**; 0.8.6 dropped 3.13. Version
0.8.5 is the last release that runs on 3.13, and it is missing the entire
control-takeover subsystem (`desktop/control.py`, `control_hooks.py`,
`control_overlay.py`, `control_ledger.py`) — meaning the model can move your
mouse and **you cannot take it back**. That subsystem is the reason to be on
0.8.8, so the environment is built for it.

All 17 dependencies ship `cp314` wheels, so nothing compiles at install time:

```
conda create -p E:\conda-envs\wmcp314 --override-channels -c conda-forge "python=3.14" -y
& E:\conda-envs\wmcp314\python.exe -m pip install "windows-mcp==0.8.8"
```

`--override-channels -c conda-forge` avoids the Anaconda channel entirely,
which sidesteps the interactive Terms-of-Service prompt that otherwise hangs
`conda create` in a non-interactive shell.

The interpreter is invoked by absolute path and needs no activation: pywin32
registers its DLLs through `pywin32.pth`, which is independent of `PATH` and of
`activate`. Verified with an empty environment (`env -i`).

## Install

```powershell
& "D:\Users\ankle crusher\AppData\Local\Programs\DeepSeek Harness\resources\runtime\cli\bin\dsh.cmd" `
    plugin --profile desktop add "@local/windows-mcp@link:E:/GitHubWorkspace/own/opensource/windows-mcp-bundle"
```

Then capture the profile mirror, read `git diff`, and restart the desktop app.
`dsh.profile.bundles` is read at boot.

## Tool allowlist

The twelve-tool allowlist is passed to the upstream server's own `--tools`
switch, so the excluded tools are never described to the model at all:

| Dropped | Why |
| --- | --- |
| `PowerShell` | The harness already ships `bash` and `pwsh` |
| `Registry` | Not needed, and whole-machine |
| `FileSystem` | The harness already ships `read`/`write`/`edit`/`glob`/`grep` |
| `Process` | The harness already covers process inspection |
| `Scrape` | The harness already ships web fetch/search |

Registering a second, far more powerful path for capabilities the harness
already has does not add reach — it makes the model pick the wrong tool.
Windows-MCP's own README describes `PowerShell` as "full system access and can
perform irreversible operations", which is not something worth exposing twice.

`ControlStatus` cannot be removed: the server force-retains it
(`keep.add("ControlStatus")`), which is why the allowlist yields thirteen tools
rather than twelve.

## Measured cost

Measured on this machine — Windows-MCP 0.8.8, Python 3.14.8, 2 × 2560×1440:

| | |
| --- | --- |
| Tool schemas, 13 tools | 11,036 B ≈ **2,759 tokens/request** |
| Tool schemas, all 21 tools | 16,297 B ≈ 4,074 tokens/request |
| Cold start | 2.27 s |
| `Screenshot` payload | ~227 KB PNG |

### End-to-end image admission: confirmed

The claim this repository exists to make — that pixels reach the model rather
than a text stub — was verified on a live session, not inferred from reading
the admission chain. A `Screenshot` call returned the desktop as a genuinely
legible image, and the harness persisted it as an attachment at

```
C:\Users\ankle crusher\.dsh\attachments\v1\objects\a6\a6d2bced…8a5f4
```

That object only exists if `resolveImageAdmission` passed and
`attachments.saveImages()` ran, so the full chain is confirmed working in the
`desktop` profile. The result also carried a sha256 for the object plus a
normalized read-only copy, which is what the native client produces and what
`dsh-mcp-lazy` cannot produce at all.

### Screenshot sizing

`windows_mcp/desktop/service.py` applies

```python
scale = min(WINDOWS_MCP_SCREENSHOT_SCALE, 1920 / width, 1080 / height)
```

against a hardcoded `MAX_IMAGE_WIDTH, MAX_IMAGE_HEIGHT = 1920, 1080`. On a
4480×1440 virtual desktop the ceiling is `1080/1440 = 0.75`, so **any value at
or above 0.75 is clamped to the same result**. Measured:

| `WINDOWS_MCP_SCREENSHOT_SCALE` | PNG size |
| --- | --- |
| unset (1.0) | 627 KB |
| 1.0 | 628 KB |
| 0.5 | 625 KB |
| **0.25** | **230 KB** |

0.5 and 1.0 are indistinguishable because both are clamped; 0.25 is the only
setting that measurably reduces the payload. Raise it if small UI text stops
being legible — the response includes a `Coordinate Scale` line telling the
model what to multiply image pixels by, so clicks stay accurate at any scale.

## Notes

- **`ANONYMIZED_TELEMETRY: 'false'`** — upstream defaults this **on** and posts
  to PostHog.
- **`WINDOWS_MCP_DISABLE_FLASH: '1'`** — without it every capture flashes an
  orange-red border across the screen.
- **`PYTHONUTF8: '1'`** — forces UTF-8 stdio regardless of the machine's ANSI
  code page (936 here). Window titles and UI-tree text are Chinese on this
  machine, and the UI tree reads them correctly with this set.
- **Screenshot backend is `dxcam` for a region, `pillow` for the whole virtual
  desktop.** `dxcam` (DXGI desktop duplication) needs a single-display capture
  region, so a whole-virtual-desktop grab spanning two monitors necessarily
  falls back to `pillow`. Confirmed both ways: a full grab reported
  `Screenshot Backend: pillow`, a `region=[0,0,320,200]` grab reported
  `Screenshot Backend: dxcam`. This is expected, not a missing dependency.
  Check the `Screenshot Backend:` line in the result whenever capture feels slow.
- **A `region` capture skips window enumeration.** Observed directly: the
  `region` call above reported `Focused Window: No active window found` and
  `Opened Windows: No windows found` while the pixels themselves were correct.
  Prefer `Screenshot` without `region`, or `Snapshot`, when the window list is
  what you actually need.
- **`Coordinate Scale` confirms the scale setting is live.** The same
  `region=[0,0,320,200]` capture reported `Screenshot Coordinate Scale: 4.0`,
  i.e. exactly `1 / 0.25`, which is the configured
  `WINDOWS_MCP_SCREENSHOT_SCALE`. The response tells the model to multiply
  image pixels by it, so clicks stay accurate at any scale.
- **`toolCallTimeoutMs: 120000`** — a `Snapshot` with `use_ui_tree=True` can take
  several seconds on a busy desktop; the package default of 60 s is left
  deliberately generous rather than tight.

## Known limitations

- **Privileges.** Even with the five dangerous tools dropped, this plugin can
  move your mouse, type into any window, and read your screen. Combined with a
  `danger-full-access` file policy there is no buffer.
- **No sandbox.** Everything runs as your user, in your session.
- **`Snapshot` output is large.** A UI tree on a busy desktop measured 32,919
  characters in a single result. The native client has no output-size guard
  (unlike `dsh-mcp-lazy`'s 50 KB cap), so that text enters the context directly.
- **The takeover guard gates on real user input.** While the user was typing at
  the machine, `ControlStatus` returned `state: user` and `DisplayInventory` was
  refused outright with `USER_CONTROL`. That is the 0.8.8 protection working as
  designed, not a fault: the desktop goes back to `state: ready` after roughly
  ten seconds of no physical input. Expect agent calls to fail with
  `USER_CONTROL` for as long as the user keeps working — and note that this is
  the mechanism that lets the user reclaim the mouse, which 0.8.5 does not have.

## License

MIT — see [LICENSE](LICENSE). Windows-MCP itself is MIT, Copyright (c) 2025
CursorTouch.
