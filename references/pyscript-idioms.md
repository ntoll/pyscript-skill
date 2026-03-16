# PyScript Idioms Reference

Read this file before generating any Python code for a PyScript environment.
It describes what is available, what is missing, and how to write idiomatic
code for the browser context.

---

## HTTP requests — context-dependent decision

This is the most important idiom decision. The right choice depends on whether
the code is browser-only or may move to a regular Python environment.

### Code intended only for the browser

Use `pyscript.fetch`. It maps directly onto the browser's `fetch` API, works
natively in workers, and has no patching surprises.

```python
from pyscript import fetch

# Simple GET.
response = await fetch("https://api.example.com/data")
data = await response.json()

# POST with JSON body.
response = await fetch(
    "https://api.example.com/submit",
    method="POST",
    headers={"Content-Type": "application/json"},
    body='{"key": "value"}',
)
```

### Code that may move to a regular Python environment

Use `requests`, but with an explanatory comment. `requests` and `urllib3` are
included in Pyodide but are heavily patched. They work for basic use cases,
with the following known limitations:

- **Streaming downloads** only work when Pyodide is running in a web worker
  on a cross-origin isolated page. Otherwise the full body is downloaded
  before the call returns.
- **No control over certificates, timeouts, proxies** — all network calls go
  via the browser, so these settings are ignored or raise errors.
- **CORS applies** — requests are subject to browser CORS policy just like
  any JavaScript `fetch`.
- **Synchronous on the main thread freezes the UI** — use a worker or
  `asyncio` where possible.

```python
import requests

# NOTE: This uses Pyodide's patched requests, which works for basic
# GET/POST but has browser-imposed limitations (no streaming outside
# workers, no timeout/proxy control, CORS applies). Replace with
# pyscript.fetch if targeting browser-only use.
response = requests.get("https://api.example.com/data")
data = response.json()
```

### Guidance summary

| Context | Use |
|---|---|
| Browser-only widget or demo | `pyscript.fetch` |
| Script likely to run on regular Python too | `requests` with explanatory comment |
| Ambiguous | Ask the user, or default to `requests` with comment |

**TODO / needs testing**: Verify which specific `requests` behaviours break
under Pyodide's patch in current PyScript versions. Test: streaming,
timeouts, sessions, auth, redirects.

---

## CORS — browser network constraints

All HTTP requests from PyScript — whether via `pyscript.fetch` or `requests`
— go through the browser and are subject to CORS policy.

When the user's code fetches from an external URL, inject a JavaScript CORS
probe into the widget that runs automatically on page load. Display a clear
warning banner if the check fails, *before* the user clicks Run.

```javascript
// Probe whether the target URL allows browser-based requests.
// This runs from the browser, so the result reflects what PyScript
// will actually see at runtime.
async function checkCORS(url) {
    const banner = document.getElementById('cors-warning');
    try {
        // A successful cors-mode fetch confirms the server permits
        // cross-origin requests from the browser.
        await fetch(url, { method: 'HEAD', mode: 'cors' });
        // CORS allowed — no warning needed.
    } catch (_) {
        // CORS blocked or server unreachable — show warning.
        banner.style.display = 'block';
        banner.innerHTML =
            '<strong>⚠ Possible CORS restriction</strong><br>' +
            'The server at <code>' + url + '</code> may not allow ' +
            'requests from browser-based pages. If your Python code ' +
            'fails with a network error when fetching this URL, it is ' +
            'being blocked by the server\'s same-origin policy — not ' +
            'by PyScript. This affects all browser-based code, not ' +
            'just PyScript. ' +
            'Options: use a public CORS proxy, fetch the data outside ' +
            'the browser and load it as a file, or check whether the ' +
            'API offers a browser-friendly endpoint. ' +
            '<a href="https://web.dev/articles/same-origin-policy" ' +
            'target="_blank" rel="noopener">Why does this happen?</a> ' +
            '&nbsp;·&nbsp; ' +
            '<a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS" ' +
            'target="_blank" rel="noopener">CORS reference (MDN)</a>';
    }
}

// Style for the warning banner.
const style = `
    #cors-warning {
        display: none;
        background: var(--warning-bg, #fff3cd);
        color: var(--warning-fg, #664d03);
        border: 1px solid var(--warning-border, #ffecb5);
        border-radius: 4px;
        padding: 0.75rem 1rem;
        margin-bottom: 1rem;
        font-size: 0.9rem;
    }
`;
```

Place the banner `<div id="cors-warning"></div>` above the editor or terminal,
and call `checkCORS(url)` for each external URL detected in the code.

**Honest caveats to understand:**
- A passing check means the server *currently* allows CORS from an arbitrary
  origin. A failing check means it likely doesn't, but could also mean the
  server is down or rejects HEAD requests. Frame warnings accordingly.
- The check result reflects the browser's perspective, not a server-side
  proxy or a regular Python environment.
- Do not overclaim: tell the user what *may* fail, not what *will* fail.

---

## Removed modules

The following stdlib modules are entirely absent from Pyodide. If the user's
code imports any of these, flag it immediately and explain the constraint.

```
curses       dbm          ensurepip    fcntl
grp          idlelib      lib2to3      msvcrt
pwd          resource     syslog       termios
tkinter      turtle       turtledemo   venv
winreg       winsound
```

These cannot be worked around within the browser. If the user needs
equivalent functionality, suggest alternatives:

| Module | Browser alternative |
|---|---|
| `tkinter` | DOM manipulation via `pyscript.document`, or an Invent widget |
| `curses` | PyScript terminal (`py terminal`) |
| `venv` | Not applicable — Pyodide manages its own environment |
| `turtle` | Consider `<canvas>` via JS interop, or matplotlib |

---

## Importable but non-functional modules

These modules can be imported without error but do not work:

- `multiprocessing` — no process spawning in WASM
- `threading` — no threads unless Pyodide is built with pthreads support
- `sockets` — no raw socket access in the browser

These will *not* raise `ImportError`, which makes them silently dangerous.
If the user's code uses any of these, warn explicitly.

**Detecting thread support at runtime:**

```python
import sys
import platform

def can_start_thread() -> bool:
    """Check whether the current platform supports threads."""
    if sys.platform == "emscripten":
        return sys._emscripten_info.pthreads
    return platform.machine() not in ("wasm32", "wasm64")

if not can_start_thread():
    n_threads = 1  # Fall back to single-threaded operation.
```

**Cannot be imported at all** (depend on removed `termios`):
- `pty`
- `tty`

---

## Modules requiring explicit loading

These stdlib modules are unvendored by default to reduce download size.
They must be loaded before import, either via config or programmatically.

```python
# Via config attribute (preferred).
# config='{"packages": ["hashlib"]}'

# Or programmatically in a setup editor.
import micropip
await micropip.install("hashlib")
```

Modules commonly needed that require explicit loading: `hashlib` (OpenSSL
algorithms only — basic hashing works without loading), `decimal` (C
implementation; pure Python `_pydecimal` needs `micropip.install("pydecimal")`).

---

## Unvendored stdlib modules (need fullStdLib or explicit load)

The following are present but not loaded by default. Most users won't need
them, but flag this if relevant:

`antigravity`, `cgi`, `cgitb`, `chunk`, `crypt`, `imaplib`, `mailbox`,
`msilib`, `nntplib`, `ossaudiodev`, `pipes`, `smtpd`, `spwd`, `sunau`,
`telnetlib`, `uu`, `xdrlib`

---

## Display and output

Prefer `pyscript.display` for rich output. `print` works and writes to the
editor's output panel, but `display` supports HTML, images, and targeting
specific DOM elements.

```python
from pyscript import display

# Print-style text output.
display("Hello, world!")

# Display a matplotlib figure inline.
import matplotlib.pyplot as plt
fig, ax = plt.subplots()
ax.plot([1, 2, 3], [4, 5, 6])
display(fig, target="#output")

# Append vs replace.
display("Line 1", append=True)
display("Line 2", append=True)
```

---

## Async patterns

The browser event loop is fundamentally async. `pyscript.fetch` is
naturally awaitable. In `py-editor`, top-level `await` works without
wrapping in `asyncio.run()`.

```python
# This works in py-editor — top-level await is fine.
from pyscript import fetch
response = await fetch("https://api.example.com/data")
data = await response.json()
display(data)
```

In regular Python outside the browser, wrap with `asyncio.run()`:

```python
import asyncio
import requests

async def main():
    response = requests.get("https://api.example.com/data")
    return response.json()

data = asyncio.run(main())
```

---

## Storage

Use `pyscript.storage` for persistent key-value data across sessions. This
uses the browser's `localStorage` under the hood.

```python
from pyscript import storage

store = await storage("my-app")
store["key"] = "value"
await store.sync()  # Persist to localStorage.
```

For temporary in-session state, plain Python variables are fine.

---

## Filesystem

Pyodide provides a virtual filesystem via Emscripten. It is fully functional
within a session but does not persist between page loads.

```python
# Write a file.
with open("/home/pyodide/output.csv", "w") as f:
    f.write("name,value\nalpha,1\nbeta,2\n")

# Read it back.
with open("/home/pyodide/output.csv") as f:
    print(f.read())
```

If the user writes files they may want to keep, proactively offer to emit
them for download via `present_files`. The virtual filesystem resets on
page reload.

---

## SSL and certificates

The `ssl` module is replaced with a stub. Methods depending on OpenSSL will
not work or will raise `NotImplementedError`. HTTPS works fine for normal
browser requests (the browser handles TLS), but low-level certificate
manipulation is not available.

---

## `webbrowser`

The `webbrowser` module is replaced with browser-specific stubs. Only
`open()`, `open_new()`, and `open_new_tab()` work, and they call the
browser's native tab/window opening behaviour.

---

## `zoneinfo` and timezones

`zoneinfo` requires timezone data, which must be installed explicitly:

```python
# Via config.
# config='{"packages": ["tzdata"]}'
import zoneinfo
```

---

## `hashlib`

Basic hash algorithms work without loading anything extra. OpenSSL-dependent
algorithms (`sha3_*`, `blake2*`, etc.) require explicitly loading `hashlib`:

```python
# Via config if OpenSSL algorithms needed.
# config='{"packages": ["hashlib"]}'
import hashlib
```
