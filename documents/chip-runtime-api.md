# The chip runtime API — navi_internal, run(), and how a chip executes

Provenance: condensed from the "Navi Chip Functions" and Chips
pages of the shipped in-app documentation (`app/index.html`) and
the execution machinery in `bridge.py` (`NAVI_INTERNAL_STUB`,
`NAVI_PY_SHIM`, `execute_chip`, `chip_invocation`,
`resolve_chip_invocation`) in the shipped Apex v0.6.9
tree (build stamps 2026-11-19.41, seq 41). The stub source
quoted below is the shipped text itself.

## The doorway

A chip does not run off on its own — it runs inside Navi, and
everything it does with her goes through one doorway:

```python
import navi_internal

navi_instance = navi_internal.navi_instance
```

Take hold of `navi_instance` at the top of `root.py` or at the
top of `run()`. Every function a chip can call is a method on
that instance.

## What root.py declares

A loadable chip declares, as module-level assignments in
`root.py`:

- `command` — the word people type to run it.
- `use` — one short line saying what it does.
- `aliases` — other names it answers to (a list).
- `params` — a description of its inputs; this is what `-help`
  shows.

And it defines `run(arguments)` — the part Navi calls.
`arguments.text` is the words typed after the chip's name.

Chip Studio checks four things live and underlines what is
missing (the checks are regular expressions over root.py, from
the Studio's `rootChecksFail` in app/index.html):

1. **imports navi_internal** — a line matching `import
   navi_internal`.
2. **declares command** — a line starting `command =` or
   `command :`.
3. **declares use** — a line starting `use =` or `use :`.
4. **defines run(arguments)** — a line starting `def run(`.

A chip whose code contains no `def run` at all is
**declarative**: the bridge never executes it; its declared
`output` is its result. (`execute_chip` decides on the plain
test `"def run" in code`.)

## How a chip is invoked

From chat: `./<name>` with arguments after it (`./navi-cryptex
hex -e -t "Hello world!"`), `/chip <name>`, "run the <name>
chip", or the chip's name on its own. The `./` resolver widens
in rings: the exact slug or name; then the name with a leading
`navi-` forgiven on either side (`./cryptex` finds
navi-cryptex); then a unique prefix. More than one fit returns
the candidates instead of guessing. A bare `./` or `/chip`
answers with the shelf — every installed chip and a line on
what it does. Under the CLI latch (`$`), `./` is the shell's
own again. Programmatically the bridge offers
`POST /api/chips/run` with `{"query": "<name> <args>"}`.

## How does a chip register a verb?

In this build, it doesn't — not in the bridge sense. A v0.6.9
chip registers no new routes, no new bridge verbs, no nav items:
that registration machinery belongs to Chips Expanded (#298), a
later era, and **is not in this build**. What a chip registers
today is its **invocation surface**, and it is complete:

- its **name and slug** (from the cartridge), which the `./`,
  `/chip` and name-alone invocations resolve;
- its **declared `command` and `aliases`** in root.py — the
  vocabulary the Studio checks, `-help` speaks, and the chip's
  own argument handling keys off;
- its **place on the shelf**: the Chips screens, the ChipDeck,
  and Workflows, where its output can feed the next chip.

Write the declarations as if they were the registration,
because for everything a chip can do in this build, they are.

## Execution — exactly what the bridge does

`execute_chip` runs the chip the way Navi does:

- **In a subprocess**: `[sys.executable, runner.py, <sandbox>,
  <args text>]`, with a **20-second budget**. Output capture is
  stdout and stderr concatenated and stripped.
- **In a fresh temp sandbox** (prefix `navi-chip-`), always
  deleted afterwards. The sandbox holds: the stub
  `navi_internal.py`; the legacy `navi.py` shim (unless the chip
  ships its own `navi.py` — the chip's always wins); the entry
  code as both `root.py` and `chip_module.py`; and every file
  in `files` at its own path, re-validated on the way in
  (relative, `.py`, no `..`, never `navi_internal.py` or
  `root.py` — those names are taken by the machinery).
- **Package mode**: when the entry code uses relative imports
  (`from . import ...`) or any file path contains a `/`, the
  entry is also written as `chip_pkg/chip_entry.py` inside a
  synthetic package with the chip's package folders beside it,
  and imported as `chip_pkg.chip_entry`, so relative imports
  resolve and a plain `import <package>` finds the package too.
- **The runner** builds `args = SimpleNamespace(text=<the
  argument string>)` and calls `chip_module.run(args)`.
- **Environment**: the operator's environment, plus
  `NAVI_LLAMAFILE_ENDPOINT` (the seat's configured model
  endpoint, default `http://localhost:8080/v1`), `NAVI_MODEL`
  (default `navi`), and `PYTHONPATH` set to the sandbox. The
  working directory is the shell's working directory.
- **Results**: exit code 0 returns the captured output, or
  `(chip ran - no output)`. A nonzero exit returns the reason
  and the last 1,200 characters of output — or, when the chip
  declares an `output`, the declared output with the failure
  noted, never a swallowed error. A timeout returns "The chip
  ran longer than 20 seconds and was stopped." — again with
  the declared output shown when there is one.

Design for the budget: a chip that cannot finish in 20 seconds
should do less per run, cache nothing it cannot rebuild, and
say what it did finish.

## navi_internal — the shipped stub, verbatim

When the bridge runs a chip, this is the `navi_internal.py`
written into its sandbox (`NAVI_INTERNAL_STUB` in bridge.py):

```python
import os, json, urllib.request

class _NaviInstance:
    def print_message(self, text, include_ai_name=True):
        print(text)
    def llm_chat(self, user_message, called_from_app=False):
        endpoint = os.environ.get("NAVI_LLAMAFILE_ENDPOINT", "http://localhost:8080/v1")
        model = os.environ.get("NAVI_MODEL", "navi")
        if not str(user_message or "").strip():
            # (#224) An empty ask is refused here, by name — the
            # model server rejects empty content outright, and a
            # chip deserves the reason, not the raw refusal.
            return ("(llm_chat was given nothing to ask — the message was empty.)", 0)
        body = json.dumps({"model": model, "messages": [{"role": "user", "content": user_message}], "stream": False, "max_tokens": 512}).encode()
        req = urllib.request.Request(endpoint + "/chat/completions", data=body, headers={"Content-Type": "application/json"})
        try:
            with urllib.request.urlopen(req, timeout=30) as resp:
                data = json.loads(resp.read().decode())
            return data["choices"][0]["message"]["content"], 200
        except Exception as e:
            return ("(llm_chat could not reach the model server: " + str(e) + ")", 0)
    def process_message(self, user_message):
        print("(process_message is not available during a bridge chip run)")
    def get_user(self):
        return os.environ.get("USER") or os.environ.get("USERNAME") or "operator"
    def set_user(self, sys_user):
        pass
    def clear_terminal(self):
        pass
    def chat_with_navi(self):
        print("(chat_with_navi is not available during a bridge chip run)")
    def setup_history(self):
        pass
    def setup_navi_vocab(self):
        pass

navi_instance = _NaviInstance()
NaviApp = _NaviInstance```

What each method does **in a bridge chip run** — the stub's
behavior, which is what shipped code can rely on:

- `print_message(text, include_ai_name=True)` — prints the
  text. Print output and `print_message` both land in the
  chip's captured output, which lands in the chat as a
  shell-style result. This is the main way a chip talks:
  results, progress, errors — if the operator should see it,
  it goes through here (or `print()`).
- `llm_chat(user_message, called_from_app=False)` — asks her
  model. It POSTs one user message to
  `<endpoint>/chat/completions` (endpoint from
  `NAVI_LLAMAFILE_ENDPOINT`, model from `NAVI_MODEL`,
  `max_tokens` 512, 30-second timeout) and returns a
  **`(text, status)` tuple**: the reply text and `200` on
  success. An empty message is refused by name, returning
  `("(llm_chat was given nothing to ask — the message was
  empty.)", 0)`. Any failure returns `("(llm_chat could not reach the model
  server: <error>)", 0)`. Unpack it:
  `text, status = navi_instance.llm_chat("...")` and check
  `status == 200` before trusting the text. The published
  navi-nmap chip uses it to turn raw scan output into a summary
  a person can read.
- `process_message(user_message)` — in the full app this runs
  a message through Navi's whole pipeline, as if typed. **In a
  bridge chip run it is not available**: it prints
  `(process_message is not available during a bridge chip run)`
  and returns None. Do not build a chip that depends on it.
- `chat_with_navi()` — likewise not available in a bridge run;
  it prints its unavailability and returns None.
- `get_user()` — returns the OS user's name (`USER`, else
  `USERNAME`, else `"operator"`) in the stub. (The in-app
  documentation describes the full NaviApp semantics — the
  operator name from Settings; a bridge-run chip gets the stub
  behavior above.)
- `set_user(sys_user)`, `clear_terminal()`, `setup_history()`,
  `setup_navi_vocab()` — present so code written against the
  full API imports and runs; in the stub they are no-ops.

The difference between `llm_chat` and `process_message`
matters: `llm_chat` is a question and an answer;
`process_message` is handing her the keyboard — and in this
build's bridge runs, only the question exists.

What does llm_chat return? A `(text, status)` tuple: the
model's reply text with status `200` when the model answered;
a plain-words reason as the text with status `0` when the ask
was empty or the model server could not be reached. Always
unpack both halves — `text, status = navi_instance.llm_chat(...)`
— and trust the text only when the status is 200.

## The legacy navi.py shim

Chips written for the original Navi import `navi` and call
`get_parameters(text)`. Every sandbox carries this shim unless
the chip ships its own `navi.py` (`NAVI_PY_SHIM` in bridge.py):

```python
import shlex
from navi_internal import navi_instance, NaviApp

def get_parameters(text):
    try:
        return shlex.split(text or "")
    except Exception:
        return (text or "").split()```

Note the old semantics exactly: `get_parameters` returns the
shlex-split token list **with the command name included** —
the chip's `run()` pops it, the way the old chips were written.
On a shlex error it falls back to a plain whitespace split.
