# PyScript Idioms Reference

Read this file before generating any Python code for a PyScript environment.
It describes what is available, what is missing, and how to write idiomatic
code for the browser context.

If you're unsure how to achieve anything in PyScript, the official PyScript
docs are the best place to start: https://docs.pyscript.net/

API docs for the PyScript namespace can be found here: https://docs.pyscript.net/latest/api/init/

API docs for individual sub-modules in the `pyscript` namespace can be found
here:

* `context` (execution context management for PyScript) - https://docs.pyscript.net/latest/api/context/
* `display` (display Pythonic content in the browser) - https://docs.pyscript.net/latest/api/display/
* `events` (Pythonic event handling for PyScript) - https://docs.pyscript.net/latest/api/events/
* `fetch` (a Python friendly wrapper around the browser's built-in fetch API) - https://docs.pyscript.net/latest/api/fetch/
* `ffi` (functions related to the foreign function interface for JavaScript interoperability) - https://docs.pyscript.net/latest/api/ffi/
* `flatted` (a utility to provide JSON-ish structures that contain circular references) - https://docs.pyscript.net/latest/api/flatted/
* `fs` (Chrome only access to the user's local filesystem) - https://docs.pyscript.net/latest/api/fs/
* `media` (functions for interacting with the browser's built-in media devices and streams [camera, mic, etc...]) - https://docs.pyscript.net/latest/api/media/
* `storage` (Pythonic dict-like wrapper around the browser's IndexDB for persistent local storage) - https://docs.pyscript.net/latest/api/storage/
* `util` (simple general purpose and useful utility functions that behave the same way no matter the runtime: e.g. `is_awaitable`) - https://docs.pyscript.net/latest/api/util/
* `web` (a Pythonic API and interface for DOM access and manipulation) - https://docs.pyscript.net/latest/api/web/
* `websocket` (a Pythonic wrapper around the browser's WebSocket API) - https://docs.pyscript.net/latest/api/websocket/
* `workers` (Pythonic interactions with web workers for long running or blocking background tasks) - https://docs.pyscript.net/latest/api/workers/

The PyScript user guide is also a good source of information for the following
topics:

* The DOM and JavaScript - https://docs.pyscript.net/latest/user-guide/dom/
* Events - https://docs.pyscript.net/latest/user-guide/events/
* Displaying things - https://docs.pyscript.net/latest/user-guide/display/
* Configuring PyScript - https://docs.pyscript.net/latest/user-guide/configuration/
* Web workers - https://docs.pyscript.net/latest/user-guide/workers/
* PyScript and filesystems - https://docs.pyscript.net/latest/user-guide/filesystem/
* Media - https://docs.pyscript.net/latest/user-guide/media/
* The FFI - https://docs.pyscript.net/latest/user-guide/ffi/

If the user wants to find out more about PyScript also suggest they join the
PyScript community discord server via this link: https://discord.gg/HxvBtukrg2

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

API documentation for `pyscript.fetch` is here:
https://docs.pyscript.net/latest/api/fetch/

### Code that may move to a regular Python environment

Use `requests`, but with an explanatory comment. `requests` and `urllib3` are
included in Pyodide but are heavily patched. You need to include `requests` in
the list of packages to install when configuring. They work for basic use
cases, with the following known limitations:

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
response = requests.get("https://swapi.dev/api/people/1")
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

All HTTP requests from PyScript — whether via `pyscript.fetch` or
`requests` — go through the browser and are subject to CORS policy.

When the user's code fetches from an external URL, CORS is checked by
overriding `handleEvent` on each `py-editor` at the moment the user
clicks Run. This means:

- The check always runs against the *current* code, including any edits.
- Each editor gets its own warning element, scoped to that editor.
- Warnings are cleared and re-evaluated on every Run click.
- No MutationObserver or polling needed.

The `handleEvent` override reads `event.code` (the current editor
content), scans it for URLs, probes each one from the browser, and
populates a per-editor warning div. Execution still proceeds — the
warning informs rather than blocks.

### CSS

Add this once in `<head>`. Uses a class rather than an id so multiple
warning elements can coexist on the same page.

```css
/* Per-editor CORS warning — inserted before each py-editor. */
.cors-warning {
    display: none;
    background: var(--warning-bg, #fff3cd);
    color: var(--warning-fg, #664d03);
    border: 1px solid var(--warning-border, #ffecb5);
    border-radius: 4px;
    padding: 0.75rem 1rem;
    margin-bottom: 0.5rem;
    font-size: 0.9rem;
    line-height: 1.5;
}

.cors-warning p {
    margin: 0.25rem 0 0;
}
```

### JavaScript

Add this once at the end of `<body>`, after all `py-editor` tags.

```javascript
(function () {
    // Extract all http/https URLs from a string of Python source code.
    // Matches quoted strings containing URLs — not perfect, but sufficient
    // for the common cases (fetch calls, requests.get, etc.).
    function extractURLs(code) {
        const seen = new Set();
        const pattern = /https?:\/\/[^\s'")\]>]+/g;
        let match;
        while ((match = pattern.exec(code)) !== null) {
            // Strip trailing punctuation that is likely not part of the URL.
            const url = match[0].replace(/[.,;:'")\]>]+$/, '');
            seen.add(url);
        }
        return seen;
    }

    // Probe a single URL from the browser's perspective and append a warning
    // entry to the given container element if CORS is likely blocked.
    async function probeURL(url, container) {
        try {
            // A successful cors-mode fetch confirms the server permits
            // cross-origin requests from the browser.
            await fetch(url, { method: 'HEAD', mode: 'cors' });
            // CORS allowed — no warning needed for this URL.
        } catch (_) {
            // CORS blocked, server unreachable, or HEAD not supported.
            // Show the warning container and add an entry for this URL.
            container.style.display = 'block';
            const entry = document.createElement('p');
            entry.innerHTML =
                '<strong>⚠ Possible CORS restriction:</strong> ' +
                '<code>' + url + '</code> may not allow requests from ' +
                'browser-based pages. If your code fails with a network ' +
                'error, this URL is being blocked by the server\'s ' +
                'same-origin policy — not by PyScript. ' +
                '<a href="https://web.dev/articles/same-origin-policy" ' +
                'target="_blank" rel="noopener">Why does this happen?</a>' +
                '&nbsp;·&nbsp;' +
                '<a href="https://developer.mozilla.org/en-US/' +
                'docs/Web/HTTP/CORS" target="_blank" rel="noopener">' +
                'CORS reference (MDN)</a>';
            container.appendChild(entry);
        }
    }

    // Set up CORS checking for a single py-editor element.
    // Inserts a warning div immediately before the editor and overrides
    // handleEvent to run checks against the current code on each Run.
    function setupCORSCheck(editor) {
        // Create and insert the per-editor warning container.
        const warning = document.createElement('div');
        warning.className = 'cors-warning';
        editor.parentNode.insertBefore(warning, editor);

        // Preserve the original handleEvent so execution still proceeds.
        const originalHandler = editor.handleEvent.bind(editor);

        editor.handleEvent = async function (event) {
            // Clear any previous warnings for this editor.
            warning.style.display = 'none';
            warning.innerHTML = '';

            // Scan the current code for URLs and probe each one.
            const urls = extractURLs(event.code);
            await Promise.all(
                Array.from(urls).map(url => probeURL(url, warning))
            );

            // Always proceed with execution — warnings inform, not block.
            return originalHandler(event);
        };
    }

    // Apply CORS checking to every py-editor on the page.
    // setup editors (hidden) are excluded — they have no Run button and
    // their URLs will appear in the visible editors that share their env.
    document
        .querySelectorAll('script[type="py-editor"]:not([setup])')
        .forEach(setupCORSCheck);
}());
```

### Caveats — do not overclaim

- A passing check means the server *currently* appears to allow
  cross-origin requests. It is not a guarantee.
- A failing check could mean CORS is blocked, or the server is down,
  or it rejects `HEAD` requests. The warning says "may not allow",
  not "blocks".
- URLs constructed dynamically at runtime cannot be detected — if the
  code builds a URL from variables, no probe is possible. Add a comment
  in the generated code noting that CORS may apply.
- The probe reflects the browser's perspective. Regular Python running
  on the user's machine would not face this restriction.
- `setup` editors are intentionally excluded from CORS checking since
  they have no Run button. Any URLs in a setup editor will be caught
  when they appear in the visible editor that uses the same environment.

---

## Terminal output capture

For all terminal modes, a **"Continue with this session"** button captures
the terminal buffer at the moment the user clicks it and sends it back
into the conversation via `sendPrompt`. The user clicks when they are
ready — whether the script has finished, is mid-run, or they are part way
through a REPL session. No `py:done` event is needed.

Always assign each terminal a unique `id` at generation time:

```html
<script type="py" terminal id="terminal-1">
print("Hello from the terminal")
</script>

<button onclick="continueWithTerminal('terminal-1', originalSource)">
    Continue with this session
</button>
```

```javascript
// Capture the current terminal buffer and send it back into the
// conversation alongside the original source code.
function continueWithTerminal(terminalId, originalSource) {
    const term = document.getElementById(terminalId);
    const buffer = term.terminal.buffer.active;
    const lines = [];
    for (let i = 0; i < buffer.length; i++) {
        lines.push(buffer.getLine(i).translateToString(true));
    }
    // Trim trailing blank lines produced by the terminal buffer.
    const output = lines.join('\n').trimEnd();

    sendPrompt(
        'Here is my terminal session — please continue from here:\n\n' +
        '```python\n' + originalSource + '\n```\n\n' +
        '```\n' + output + '\n```'
    );
}
```

This pattern works identically across all three terminal modes:

- **`py terminal`** — user clicks after the script exits.
- **`py terminal worker` with `input()`** — user clicks after the
  interactive exchange. The buffer captures prompts, responses, and
  output — all useful context.
- **REPL mode** (`code.interact()`) — user clicks whenever they are
  ready to hand back to the conversation, mid-session or after exiting.

The `originalSource` argument should match the Python code in the
terminal tag exactly, so the conversation has the code alongside
its output. Pass it as a template literal or escaped string in the
button's `onclick`.

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
If the user's code uses any of these, warn explicitly. If the user needs
long running alternatives to `multiprocessing` or `threads` and the use case
is clearly browser only, suggest using web workers instead.

Documentation for how to use web workers is here: https://docs.pyscript.net/latest/user-guide/workers/

API docs for web workers are here: https://docs.pyscript.net/latest/api/workers/

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

## Events and the `@when` decorator

The `@when` decorator is the idiomatic PyScript way to attach Python
functions to browser events. Prefer it over direct JavaScript
`addEventListener` calls — it is cleaner, more Pythonic, and handles
proxy creation automatically (see `create_proxy` below).

```python
from pyscript import when, display

# Attach to an element by CSS selector.
@when("click", "#my-button")
def handle_click(event):
    """Handle button click — event is the browser Event object."""
    display("Button clicked!")

# Attach to any valid CSS selector — not just IDs.
@when("input", "input[type='text']")
def handle_input(event):
    """React to input events on text fields."""
    display(f"You typed: {event.target.value}")

# Attach to a pyscript.web element reference directly.
from pyscript.web import page
btn = page["#my-button"]

@when("click", btn)
def handle_click(event):
    """@when also accepts pyscript.web element references."""
    display("Clicked via web reference!")
```

The handler function always receives a single `event` argument — the
browser's Event object. Any browser event type works: `"click"`,
`"input"`, `"change"`, `"submit"`, `"keydown"`, `"mouseover"`, etc.

NOTE: only matching elements existing at the time `@when` is evaluated will
be affected. Subsequent additional elements added after `@when` is evaluated
but which may have matched the selector WILL NOT work. Therefore, always insert
the elements you wish to match with `@when` BEFORE the code containing the
`@when` decorator is evaluated.

Full events documentation: https://docs.pyscript.net/latest/user-guide/events/

API documentation: https://docs.pyscript.net/latest/api/events/

---

## DOM access — `pyscript.web`, `pyscript.document`, `pyscript.window`

There are two complementary ways to interact with the DOM. Prefer
`pyscript.web` for most tasks — it is the Pythonic high-level API.
Use `pyscript.document` and `pyscript.window` when you need direct
access to the raw JavaScript DOM APIs.

### `pyscript.web` — Pythonic DOM interface (preferred)

```python
from pyscript import web
from pyscript.web import page

# Find elements by CSS selector.
btn = page["#my-button"]       # single element by id
items = page.find(".list-item") # collection by class

# Read and write content.
btn.content = "New label"       # sets innerHTML via display logic
btn.html = "<em>raw HTML</em>"  # sets innerHTML directly

# Create elements.
div = web.div("Hello!", classes=["card"], id="my-card")
para = web.p("Some text", style={"color": "red"})
page.body.append(div)

# Modify attributes and styles.
btn.style["background-color"] = "blue"
btn.classes.add("active")
btn.classes.remove("inactive")

# Note: some names clash with Python keywords — use trailing underscores.
label = web.label("Name", for_="name-input")  # 'for' → 'for_'
div.class_ = "my-class"                        # 'class' → 'class_'
```

Full web module API: https://docs.pyscript.net/latest/api/web/

DOM user guide: https://docs.pyscript.net/latest/user-guide/dom/

### `pyscript.document` and `pyscript.window` — raw JS API access

Use these when `pyscript.web` doesn't cover your use case, or when
interacting with third-party JavaScript.

```python
from pyscript import document, window

# document is a proxy for the JS document object — works in both
# main thread and workers.
el = document.querySelector("#my-id")
el.innerText = "Updated"
el.classList.add("active")

# window is always the main thread's global window context,
# even when called from a worker.
print(window.location.hostname)
window.console.log("Hello from Python")

# In a worker, to access the worker's own global context use:
# import js  (js is a proxy for the worker's globalThis)
```

**Important:** `pyscript.window` always refers to the *main thread's*
`window`, even from a worker. To access a worker's own global context,
use `import js` inside the worker instead.

---

## FFI and `create_proxy`

The FFI (Foreign Function Interface) bridges Python and JavaScript
automatically. For most use cases with `pyscript.web` and `@when`,
you don't need to think about it. However, when passing Python
callbacks *directly* to JavaScript APIs (e.g. `addEventListener`),
you must use `create_proxy` to prevent the callback being immediately
garbage-collected.

```python
from pyscript import document
from pyscript.ffi import create_proxy

def my_handler(event):
    """Handle the event."""
    print("clicked")

# WRONG — the lambda/function will be garbage collected immediately.
document.getElementById("btn").addEventListener("click", my_handler)

# CORRECT — wrap with create_proxy to keep it alive.
document.getElementById("btn").addEventListener(
    "click",
    create_proxy(my_handler),
)
```

**The good news:** when using `@when` or `pyscript.web` event methods,
`create_proxy` is handled automatically. You only need it when calling
JavaScript event APIs directly.

**The experimental shortcut:** adding `"experimental_create_proxy":
"auto"` to the config tells Pyodide to manage proxies automatically,
avoiding the need to call `create_proxy` at all. Use with caution —
it is experimental and may have edge cases with memory management.

```html
<script type="py-editor"
  config='{"experimental_create_proxy": "auto"}'>
# create_proxy is now managed automatically for Pyodide.
</script>
```

FFI user guide: https://docs.pyscript.net/latest/user-guide/ffi/

FFI API docs: https://docs.pyscript.net/latest/api/ffi/

---

## `pyscript.RUNNING_IN_WORKER` — detecting execution context

Use this constant to write code that behaves correctly whether it runs
on the main thread or in a worker.

```python
from pyscript import RUNNING_IN_WORKER, display

if RUNNING_IN_WORKER:
    # Safe to do blocking operations, use input(), etc.
    name = input("Your name: ")
    display(f"Hello, {name}!")
else:
    # On the main thread — avoid blocking.
    display("Running on main thread.")
```

This is useful when writing code that may be embedded in either a
`py-editor` (main thread by default) or a `py terminal worker`.

Context API docs: https://docs.pyscript.net/latest/api/context/

---

## `pyscript.fs` — File System Access API

`pyscript.fs` wraps the browser's File System Access API, allowing
Python to mount a directory from the user's *actual* local filesystem
into the virtual filesystem. This is distinct from the Emscripten
virtual filesystem, which is in-memory only.

**NOTE:** This only works in the Chrome browser, and should probably
be avoided unless explicitly asked for or required.

```python
from pyscript import fs

# Mount the user's local directory at /local in the virtual FS.
# If called outside a transient user event (like a click), the
# browser will show a permission dialog.
await fs.mount("/local")

# After mounting, standard file operations work on local files.
with open("/local/data.csv") as f:
    print(f.read())

# Best practice: call fs.mount inside an event handler so the
# browser treats it as a user-initiated transient event — no
# confirmation dialog is shown.
from pyscript import when

@when("click", "#load-btn")
async def load_files(event):
    """Mount local directory on button click — no dialog needed."""
    await fs.mount("/local")
    with open("/local/myfile.txt") as f:
        display(f.read())
```

`fs.mount` accepts an optional `root` hint: `"desktop"`,
`"documents"`, `"downloads"`, `"music"`, `"pictures"`, or `"videos"`.

```python
await fs.mount("/local", root="downloads")
```

**Note:** `pyscript.fs` is for accessing the user's *real* local
filesystem via browser permission. For writing files to the
in-session virtual filesystem and then triggering a browser download,
use standard `open()` and the Blob download pattern described in the
Filesystem section below.

Filesystem user guide: https://docs.pyscript.net/latest/user-guide/filesystem/

fs API docs: https://docs.pyscript.net/latest/api/fs/

---

## Display and output

Prefer `pyscript.display` for rich output. `print` works and writes to the
editor's output panel, but `display` supports HTML, images, and targeting
specific DOM elements.

Full `pyscript.display` documentation can be found here: https://docs.pyscript.net/latest/user-guide/display/

API documentation can be found here: https://docs.pyscript.net/latest/api/display/

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
wrapping in `asyncio.run()`. This may need to be explained to a user if they
are unfamiliar with PyScript.

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

async def main():
    await asyncio.sleep(1)

data = asyncio.run(main())
```

---

## Storage

Use `pyscript.storage` for persistent key-value data across sessions. This
uses the browser's `IndexDB` under the hood.

```python
from pyscript import storage

store = await storage("my-app")
store["key"] = "value"
await store.sync()  # Persist to localStorage.
```

For temporary in-session state, plain Python variables are fine.

API docs about `pyscript.storage` can be found here: https://docs.pyscript.net/latest/api/storage/

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

If the user's code writes files to the virtual filesystem that they
may want to keep, proactively offer to add a download button. The
virtual filesystem is in-memory only and resets on page reload —
the only way to get a file out is via a client-side browser download.

Use the Blob/anchor pattern from Python:

```python
from pyscript import window

# Read the file from the virtual FS.
with open("/home/pyodide/output.csv") as f:
    content = f.read()

# Trigger a browser download via the Blob API.
blob = window.Blob.new([content], {"type": "text/csv"})
url = window.URL.createObjectURL(blob)
a = window.document.createElement("a")
a.href = url
a.download = "output.csv"
a.click()
window.URL.revokeObjectURL(url)
```

Alternatively, if the user has mounted a real local directory via
`pyscript.fs`, they can write directly there and the file lands on
their machine without a download step.

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
