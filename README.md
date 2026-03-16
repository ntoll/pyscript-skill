# PyScript Skill for Claude

A [Claude skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
that makes Python code in a conversation immediately runnable in the browser,
using [PyScript](https://pyscript.net).

This is an **ambient** capability - it activates whenever Python code appears
in a conversation, whether Claude generated it or you pasted it in. You don't
need to ask for it explicitly.

---

## What it does

- Produces a live, editable Python environment directly in the Claude
  conversation using PyScript's `py-editor`.
- Chooses between an inline widget (for short snippets) and a downloadable
  HTML file (for longer code or anything you want to share or host).
- Switches to a terminal environment automatically when your code calls
  `input()` or uses a REPL.
- Verifies third-party package support against
  [packages.pyscript.net](https://packages.pyscript.net) before including
  them, and surfaces a link for you to report findings back to the community.
- Probes for CORS restrictions before you run code that fetches from external
  URLs, with a clear explanation and links to further reading if a restriction
  is detected.
- Runs entirely in your browser - your code and data never leave your machine.

---

## Installation

Download the latest `pyscript.skill` file from
[Releases](../../releases/latest) and install it via Claude's skill settings.

---

## Developing

Clone the repo and edit the files directly:

```
git clone https://github.com/ntoll/pyscript-skill.git
```

The skill has two main files:

- **`SKILL.md`** — the main instructions Claude reads when the skill triggers.
- **`references/pyscript-idioms.md`** — a reference file covering idiomatic
  PyScript patterns, Pyodide constraints, and the CORS probe implementation.
  Claude reads this before generating any Python code.

To package a `.skill` file locally for testing:

```bash
zip -r pyscript.skill pyscript-skill/
```

---

## Contributing

Contributions welcome - particularly:

- Corrections or additions to `references/pyscript-idioms.md` (idioms,
  Pyodide constraints, package compatibility notes).
- Test cases and reports of skill behaviour in practice.
- Verification of `requests` behaviour under Pyodide.

Please open an issue before starting significant work so we can discuss
the approach first.

---

## Licence

[MIT](LICENSE.md)
