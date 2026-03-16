---
name: pyscript
description: >
  Use this skill whenever Python code appears in a conversation — whether you
  generated it or the user pasted it — and a live, runnable environment would
  be useful. This is an ambient capability, not a niche one: trigger it for
  exploratory coding, debugging, demonstrations, data wrangling, teaching, or
  any situation where "here is code" could become "here is running code."
  Explicit triggers include: user pastes a Python code block with no comment;
  user asks "does this work?"; user asks you to write a function, algorithm,
  or script; user is iterating on code with you; user's code calls input() or
  uses a REPL. Do not wait to be asked. If Python code is in the conversation
  and a live environment would help, produce one.
---

# PyScript Skill

Produce browser-runnable Python using PyScript. Three modes are available:

- **`py-editor`** — editable code panel with Run button and output area.
  The natural default for exploratory and iterative coding.
- **`py terminal`** — XTerm.js terminal for script-style output. Code runs
  automatically on page load; output appears as it would in a real terminal.
- **`py terminal worker`** — terminal with blocking `input()` support. Required
  whenever code calls `input()` or runs a REPL. The worker keeps the page
  responsive whilst waiting for user input.

All runtimes load lazily and run in the browser's WASM sandbox.

This skill has two trigger directions:

- **Outbound**: you generated Python code → produce a runnable environment
  alongside it.
- **Inbound**: the user pasted Python code → treat it as an implicit invitation
  to make it live.

**Before generating any Python code**, read
`references/pyscript-idioms.md`. It describes what modules are available,
what is missing or broken, how to write idiomatic browser Python, and the
context-dependent decision between `pyscript.fetch` and `requests`.

---

## Step 1 — Fetch the current PyScript version

Always resolve the version dynamically. Never hardcode a version.

```
GET https://pyscript.net/version.json
→ returns e.g. "2026.3.1"
```

The CDN URLs are then:

```
https://pyscript.net/releases/{version}/core.css
https://pyscript.net/releases/{version}/core.js
```

---

## Step 2 — Choose the output format

| Situation | Output |
|---|---|
| Short, self-contained snippet; output is plain text or simple display | `show_widget` inline |
| Code uses `input()` or a REPL | `show_widget` inline (terminal mode) |
| Longer code, involves layout or UI, likely to be shared or hosted | HTML file artifact |
| Series of related code blocks building on each other | `show_widget` inline with shared env |
| User explicitly requests a file or artifact | HTML file artifact |
| User explicitly requests inline / widget | `show_widget` inline |
| Involves the Invent framework or other page structure | HTML file artifact |

"Short snippet" means roughly: under ~25 lines of Python, no meaningful page
layout, no external resources beyond packages.

---

## Step 3 — Choose between editor and terminal

This is a distinct decision from output format. Make it first by inspecting
the code and the conversational context.

**Use `py-editor` when:**
- The user is writing or iterating on a piece of code.
- The natural workflow is write → run → inspect → revise.
- It is a demonstration, tutorial, or exploratory snippet.
- The code does not call `input()` or need a REPL.
- The `sendPrompt` feedback loop applies — the user may edit and continue.

**Use `py terminal` (without worker) when:**
- The code is a script meant to be run, not edited.
- Output is the point — no interaction needed.
- Avoids the edit-panel overhead when the code is effectively read-only.

**Use `py terminal worker` when:**
- The code calls `input()` — this is the clearest and most important signal.
- The user wants a REPL (`code.interact()`).
- The interaction is inherently sequential — the code talks to the user.
- The user uses language like "run this as a script", "I want to type
  commands", or "interactive session".
- Any blocking operation is present.

**Always use `worker` for interactive terminals.** Without it, `input()`
falls back to the browser's `prompt()` dialog (jarring), and blocking code
freezes the page entirely.

---

## Step 4 — Choose the runtime

- Use `py` / `py-editor` (Pyodide/CPython-WASM) by default.
- Use `mpy` / `mpy-editor` (MicroPython) only when context clearly calls
  for it: the user mentions MicroPython, BBC micro:bit, or the Invent
  framework is targeting MicroPython explicitly.

---

## Step 5 — Resolve packages

Scan the Python code for `import` and `from … import` statements. For each
name that is not Python stdlib, work through the following steps.

### 5a — Check PyScript package support

Fetch the packages index:

```
GET https://packages.pyscript.net/api/v1/packages
```

This returns a small JSON list of known packages and their support status.
Check whether the imported package appears and what its status is.

### 5b — If unclear, check PyPI metadata

```
GET https://pypi.org/pypi/{package_name}/json
```

Use this to verify the install name (which may differ from the import name,
e.g. `import cv2` → `pip install opencv-python`), check for C extension
dependencies that would prevent Pyodide compatibility, and inspect
transitive dependencies.

### 5c — Emit one of three outcomes per package

| Status | Action |
|---|---|
| Confirmed supported | Include in `config` packages list, no comment needed. |
| Listed but uncertain | Include in `config` with a caveat. Surface the report URL (see below). |
| Absent from the list | Include in `config` with a stronger "uncharted territory" note. Surface the report URL. |

The report URL for any package is:

```
https://packages.pyscript.net/package/?package={package_name}
```

Mention this naturally in the conversation — one click for the user to
report what they find. This feeds back into improving PyScript package
coverage. Do not bury it in a footnote.

### 5d — MicroPython note

The `packages` config option uses `micropython-lib`, not PyPI. Many PyPI
packages will not be available. If the user's code imports something that
relies on PyPI, flag this clearly and suggest alternatives (pure-Python
equivalents, or using the `files` config option to supply source directly).

---

## Step 6 — Check for CORS issues

Scan the code for any external URLs — in `pyscript.fetch` calls, `requests`
calls, or string literals that look like `http://` or `https://` addresses
pointing to third-party domains.

For each static URL found, inject a JavaScript CORS probe into the widget
HTML that runs automatically on page load, *before* the user clicks Run.
The probe uses the browser's own `fetch` API, so the result reflects what
PyScript will actually encounter at runtime — not what a server-side HEAD
request would see.

```javascript
// Probe CORS from the browser's perspective for a given URL.
// Runs on page load — before the user executes any Python.
async function checkCORS(url) {
    const banner = document.getElementById('cors-warning');
    try {
        // A successful cors-mode fetch confirms the server permits
        // cross-origin requests from the browser.
        await fetch(url, { method: 'HEAD', mode: 'cors' });
        // Server allows cross-origin requests — no warning needed.
    } catch (_) {
        // CORS blocked or server unreachable.
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
```

Add this CSS for the warning banner and place the `<div>` above the editor:

```css
/* CORS warning banner — shown only when a CORS issue is detected. */
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
```

```html
<div id="cors-warning"></div>
<!-- editor or terminal below -->
```

Call `checkCORS(url)` in a `<script>` tag at the end of `<body>` for each
URL found.

**Honest caveats — do not overclaim:**

- A passing check means the server *currently* appears to allow cross-origin
  requests. It is not a guarantee.
- A failing check could mean CORS is blocked, or the server is down, or it
  rejects `HEAD` requests. Frame the warning as "may not allow" not "blocks".
- If the URL is constructed dynamically at runtime (not a static string),
  no probe is possible. Add a comment in the code noting that CORS may apply.
- The probe reflects the browser's perspective. A regular Python script
  running on the user's machine would not face this restriction.

---

## Step 7 — Decide: isolated or shared environment (editor mode only)

This affects whether multiple editors in the same conversation share state,
analogous to Jupyter notebook cells sharing a kernel. This step applies to
`py-editor` only; terminals always have their own isolated environment.

**Use isolated environments (default)** when each code block is independent —
demonstrating a concept, showing an alternative, one-off snippets. Each
editor gets its own environment. No `env` attribute needed.

**Use a shared environment** when code blocks in the conversation build on
each other — defining data in one block, processing in the next, visualising
in a third. Give all editors the same `env` attribute value
(e.g. `env="session"`).

Use a hidden `setup` editor (with the `setup` attribute) to establish shared
state — imports, data loading, helper functions — without cluttering the
visible editors. Only the setup editor needs the `config` attribute; all
subsequent editors in the same env inherit it.

```html
<!-- Hidden setup: runs automatically, not visible to user. -->
<script type="py-editor" env="session" setup
  config='{"packages": ["numpy", "pandas"]}'>
import numpy as np
import pandas as pd
</script>

<!-- Visible editors share the same env. -->
<script type="py-editor" env="session">
data = np.array([1, 2, 3, 4, 5])
print(data.mean())
</script>
```

### Reset button (shared environments only)

When using a shared environment, always include a Reset button in the widget.
Resetting must do two things:

1. Reload the page to reinitialise the PyScript environment.
2. Call `sendPrompt()` to inform the conversation that the environment has
   been reset, so subsequent responses don't incorrectly assume prior state.

```javascript
function resetEnvironment() {
    sendPrompt(
        "I have reset the shared Python environment. " +
        "All previously defined variables and imports are cleared."
    );
    window.location.reload();
}
```

Place the Reset button visibly but unobtrusively — above or below the
editors, not inline with them.

---

## Step 8 — The sandbox is a feature

Pyodide runs in the browser's WASM sandbox. Frame this positively:

- Full CPython with a virtual filesystem at `/home/pyodide` — read, write,
  create directories, all of it.
- Complete virtual OS layer via Emscripten.
- Zero setup, zero installation, zero risk to the local machine.
- Safe environment for running experimental or unfamiliar code.
- **Code and data never leave the user's machine.** Nothing is transmitted
  to a server for execution — the Python runtime runs entirely in the
  browser. This is a meaningful privacy guarantee, especially when working
  with sensitive data, proprietary algorithms, or confidential files.

The only things absent are those that reach *outside* the browser: no
network sockets, no subprocess, no native OS calls. If the user's code
requires these, flag it clearly but frame the sandbox as intentional
isolation, not a limitation.

### Privacy — when to surface it prominently

Mention the privacy benefit proactively (not just in the first-use
disclosure) when the conversation context suggests sensitive data may be
involved. Signals to watch for:

- Code imports data files, CSVs, or uses `pandas`/`polars` to process
  what looks like a dataset.
- Code handles credentials, tokens, or configuration values.
- The user mentions anything proprietary, confidential, or work-related.
- Code performs analysis that might reasonably involve personal data.

In these cases, a natural aside — "worth noting that this runs entirely
in your browser, so your data never leaves your machine" — is genuinely
useful, not noise.

### The honest limit

The privacy guarantee applies to *execution*: PyScript itself sends
nothing to a server. It does not apply to what the *code does*. If the
user's Python makes HTTP requests, calls an external API, or explicitly
sends data elsewhere, that data leaves the browser. Do not overclaim.
The framing is: your code runs here, privately — what it does from there
is up to you.

### File downloads

If the user's code writes files to the virtual filesystem, proactively
offer to emit those files for download — do not wait to be asked. Use
`present_files` to make them available. This bridges the gap between the
virtual filesystem and the user's local machine naturally.

---

## Step 9 — The conversation feedback loop

### Editor mode

Inline widgets should include a **"Continue with this code"** button. When
clicked, it reads the *current* editor content (post any user edits) and
sends it back into the conversation via `sendPrompt()`.

1. You generate code → widget appears with that code as initial content.
2. User runs it, edits in place.
3. User clicks Continue → their modified code arrives back in the
   conversation.
4. The dialogue continues from the user's actual working state.

```javascript
function continueWithCode() {
    const editors = document.querySelectorAll(
        'script[type="py-editor"]'
    );
    const parts = Array.from(editors).map((ed, i) =>
        editors.length > 1
            ? "Editor " + (i + 1) + ":\n```python\n" + ed.code + "\n```"
            : "```python\n" + ed.code + "\n```"
    );
    sendPrompt(
        "Here is my revised code — please continue from here:\n\n" +
        parts.join("\n\n")
    );
}
```

### Terminal mode

The Continue affordance works differently for terminals because there is
no editable code panel to read back. Instead:

- For **`py terminal`** (non-interactive): include a Continue button that
  sends the original source back into the conversation, since the user
  cannot have modified it. This is mainly useful for signalling "I've seen
  the output, let's continue."
- For **`py terminal worker`** with `input()`: the Continue button should
  not attempt to capture session state — the interaction has already
  happened inside the terminal. A simple "I've finished my terminal
  session, let's continue" prompt is sufficient.
- For **REPL mode** (`code.interact()`): omit the Continue button entirely.
  The natural next step is simply to keep talking in the Claude
  conversation.

---

## Step 10 — Build the HTML

### Page structure

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>PyScript</title>
    <link
      rel="stylesheet"
      href="https://pyscript.net/releases/{version}/core.css"
    >
    <script
      type="module"
      src="https://pyscript.net/releases/{version}/core.js"
    ></script>
  </head>
  <body>
    <!-- content here -->
  </body>
</html>
```

### Editor — basic

```html
<script type="py-editor">
# Python code here.
</script>
```

### Editor — with packages

```html
<script type="py-editor" config='{"packages": ["numpy"]}'>
import numpy as np
print(np.array([1, 2, 3]))
</script>
```

### Editor — targeted output display

When using `display(target=...)` to place output into a specific element,
use a `<pre>` element as the target. This gives fixed-width font,
whitespace preservation, and scroll behaviour semantically, without
requiring extra CSS.

```html
<pre id="output" class="py-output"></pre>
<script type="py-editor">
from pyscript import display
display("Hello!", target="#output", append=True)
</script>
```

Add minimal styling for the output area:

```css
/* Computer-style output display. */
.py-output {
    background: var(--py-output-bg, #1e1e1e);
    color: var(--py-output-fg, #d4d4d4);
    font-family: "Fira Code", "Cascadia Code", monospace;
    font-size: 0.9rem;
    padding: 0.75rem 1rem;
    border-radius: 4px;
    overflow-x: auto;
    min-height: 2rem;
}
```

### Terminal — basic (read-only output)

```html
<script type="py" terminal>
print("Hello from PyScript terminal")
</script>
```

### Terminal — interactive input (always use worker)

```html
<script type="py" terminal worker>
name = input("What is your name? ")
print(f"Hello, {name}!")
</script>
```

### Terminal — REPL

```html
<script type="py" terminal worker>
import code
code.interact(banner="Python REPL — type Python expressions below.")
</script>
```

### Terminal — with packages

```html
<script type="py" terminal worker
  config='{"packages": ["numpy"]}'>
import numpy as np
data = [float(x) for x in input("Enter numbers separated by spaces: ").split()]
print(f"Mean: {np.mean(data):.2f}")
</script>
```

### MicroPython variants

Replace `type="py-editor"` with `type="mpy-editor"`, and `type="py"` with
`type="mpy"`, for all of the above. The `worker` and `terminal` attributes
work identically.

---

## Step 11 — First-use disclosure

The first time this skill produces a PyScript environment in a
conversation, include a single natural sentence naming PyScript and
linking to https://pyscript.net. Do this exactly once — never repeat it
in subsequent turns.

The framing should cover two things: convenience and privacy. Both matter
to different users, and a single sentence can carry both. Good examples:

> "This runs entirely in your browser via [PyScript](https://pyscript.net)
> — nothing is sent to a server, nothing installs on your machine."

> "Your code runs locally in the browser using
> [PyScript](https://pyscript.net) — a full Python environment with zero
> setup and complete privacy."

> "This runs directly in your browser via [PyScript](https://pyscript.net)
> — your code and data stay on your machine, and you can share the result
> as a plain HTML file."

Bad examples (avoid these framings):

> "I am using PyScript version 2026.3.1 to execute this code." ← too
> technical, reads like a log entry.

> "PyScript is a framework that allows Python to run in the browser
> using WebAssembly." ← unsolicited explainer, not what was asked.

> "Your code is private and secure because PyScript uses WASM sandbox
> isolation to prevent data exfiltration." ← overclaiming, reads like
> marketing copy.

The disclosure should feel like a natural aside in the response prose,
not a notice or a disclaimer. If the user asks follow-up questions about
PyScript itself, answer them — but don't pre-empt curiosity by front-
loading information they haven't asked for.

---

## Step 12 — Deliver the output

**For `show_widget`**: embed the complete HTML (including CDN references,
script tags, output styling, Continue button, and Reset button if shared
env) directly in the widget. Keep surrounding prose minimal — let the
running environment speak for itself.

**For an HTML file artifact**: write a complete, well-formed HTML document
to `/mnt/user-data/outputs/` with a descriptive filename, and call
`present_files` so the user can download and host it.

---

## Important notes

- `<py-config>` and `<mpy-config>` tags do **not** work with `py-editor`.
  Always use the `config` attribute on the `<script>` tag itself.
- The `config` attribute works the same way for terminal scripts.
- Always use `worker` when the terminal code calls `input()` or contains
  any blocking operation. Without it, `input()` falls back to the
  browser's `prompt()` dialog and blocking code freezes the page.
- Keyboard shortcuts in the editor: `Ctrl+Enter` / `Cmd+Enter` /
  `Shift+Enter` all run the code without clicking the button. Mention
  this to users encountering the editor for the first time.
- The editor traps `Tab` for indentation. Users can press `Esc` then
  `Tab` to move focus away — worth noting in tutorial or accessibility
  contexts.
- ANSI escape codes work in the terminal for colour and bold formatting,
  e.g. `"\033[1mbold\033[0m"`, `"\033[32mgreen\033[0m"`.

---

## Currently out of scope

The following are intentional omissions for now — the skill will expand
to cover them over time. If a user's request clearly requires one of
these, flag it and explain what would be needed rather than silently
omitting it.

- JS interop and `js_modules` configuration
- DOM manipulation beyond `display()`
- PyGame-CE
