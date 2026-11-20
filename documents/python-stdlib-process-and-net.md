# Python reference for chip authors — processes, the network, and house patterns

Written for this pack: a condensed, original reference.
Python 3.12, stdlib only. A chip run has a 20-second budget
and an operator watching — every pattern here is shaped by
that.

## subprocess — running other programs

How do I run a shell command safely from Python? With
`subprocess.run` and an argument **list** — never a shell
string. The list is the whole safety story, and it is spelled
out under the example.

```python
import subprocess

proc = subprocess.run(
    ["nmap", "-sV", target],
    capture_output=True, text=True, timeout=15,
)
out = proc.stdout or ""
if proc.returncode != 0:
    # name the failure; proc.stderr holds the tool's own words
    ...
```

- **The argument list is the safety.** `subprocess.run` with
  a list passes each element as one argument — no shell ever
  re-parses it, so an argument containing spaces, quotes, or
  `;` is data, not syntax. That is how the bridge runs chips
  themselves.
- **Never `shell=True` with text a person or a file
  supplied.** `shell=True` hands the string to a shell, and
  every metacharacter in it becomes live. If a command must
  be displayed, `shlex.join(argv)` renders the list safely;
  `shlex.split(s)` parses a typed line into a list (the
  `navi.py` shim's `get_parameters` is built on it).
- Always set `timeout=` — an unbounded child inside a
  20-second chip run is a timeout that reports nothing.
  Catch `subprocess.TimeoutExpired` and say what was stopped.
- `text=True` decodes output as text; add
  `errors="replace"` via `encoding=`/`errors=` kwargs when
  the tool's output encoding is not trustworthy.
- `check=True` raises `CalledProcessError` on nonzero exit —
  prefer reading `returncode` yourself when the output still
  matters on failure (scan tools report in stderr and exit
  nonzero for findings, not just failures).
- Long-running work does not belong in a chip run. Start
  nothing you cannot finish and reap inside the budget.

## urllib and http.client — talking HTTP

The standard library's client is `urllib.request`. A JSON
POST — the same shape the shipped `navi_internal` stub uses
to reach the model server:

```python
import json, urllib.request

body = json.dumps({"model": model, "messages": messages}).encode()
req = urllib.request.Request(
    url, data=body, headers={"Content-Type": "application/json"})
with urllib.request.urlopen(req, timeout=10) as resp:
    data = json.loads(resp.read().decode("utf-8", "replace"))
```

- `data=` makes it a POST; no `data` is a GET. Query strings
  belong in the URL — build them with
  `urllib.parse.urlencode` and quote path parts with
  `urllib.parse.quote(part, safe="")`, never by concatenation
  of raw user text.
- **Always pass `timeout=`.** Failures arrive as
  `urllib.error.HTTPError` (the server answered, with an
  error status — it carries `.code` and a readable body) and
  `urllib.error.URLError` (nothing answered). Name which one
  happened; they have different fixes and the operator
  deserves to know which.
- `http.client` is the lower-level door — reach for it when
  `urllib` cannot express the request, which for chip work
  is rare.
- A chip's network reach is the operator's machine's reach,
  under the trust wall: the chip's description must promise
  every call the code makes. Local-first is the house
  posture — the model endpoint is local, the Mesh is not a
  chip surface, and an outside call is a declared behavior,
  never a surprise.

## threading — usually the wrong tool in a chip

- `threading.Thread(target=fn, args=(...), daemon=True)`,
  `.start()`, `.join(timeout)`. Shared state needs a
  `threading.Lock` around it; hand results back through
  `queue.Queue`, not shared variables.
- A chip run is one-shot and budgeted: a thread that outlives
  `run()` is killed with the subprocess and reports nothing.
  Prefer doing less, sequentially, inside the budget. When
  several independent fetches must happen, threads plus a
  join-with-timeout can fit — but the total still answers to
  the same 20 seconds, and partial results must be reported
  as partial.

## House patterns — how Apex's own code writes it

Patterns lifted from the shipped codebase's habits, worth
imitating in any chip:

- **Coerce at the border.** `str(x or "")`, `int(x or 0)`,
  `list(x or [])` — outside data (JSON, payloads, arguments)
  is normalized the moment it enters, so the body of the
  code can trust its types.
- **Refuse in plain words.** A refusal is a `ValueError`
  whose message is a complete sentence naming the problem
  and the fix — "the repo holds several Python files (...)
  and none is the chip's entry — name the entry file
  root.py, or add a navi-chip.json manifest naming it."
  A chip's error output deserves the same craft.
- **Degrade to the honest fallback.** When real execution
  fails, the bridge shows the declared output *with the
  failure noted* — never a silent substitution, never a
  swallowed error. Catch at boundaries, keep the tail of
  the evidence (the bridge keeps the last 1,200 characters),
  and say what happened.
- **Guard the filesystem reads.** `try: ... except
  Exception: rows = []` around stores that may not exist
  yet — absence is a state, not a crash. But a *write* that
  fails is news: let it raise, or report it by name.
- **Comment the why above the function.** The bridge's
  functions carry a block comment stating what the function
  is for, what it refuses, and what it returns — the next
  reader (often Navi herself) navigates by those. Write
  yours to that standard.
- **Print is the interface.** In a chip, output the operator
  reads is the product: complete sentences, the numbers that
  matter, nothing that requires the code to interpret.
