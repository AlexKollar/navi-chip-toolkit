# Python reference for chip authors — the core language

Written for this pack: a condensed, original quick reference
to the Python 3.12 a chip author actually uses. Apex's chips
run on the seat's Python 3.12, stdlib-first — if it isn't in
the standard library, a chip shouldn't need it. House idioms
(the way the shipped Apex code itself writes Python) are
marked as such.

## Values and types

- Core types: `int`, `float`, `str`, `bytes`, `bool`, `None`,
  `list`, `tuple`, `dict`, `set`. `bool` is an `int` subclass;
  `True`/`False` are singletons — compare with `is` only for
  `None`, `==` for everything else.
- Truthiness: `0`, `0.0`, `""`, `[]`, `{}`, `set()`, `None`
  are false; everything else is true. `x or default` returns
  `default` when `x` is falsy — the codebase's favorite
  coercion is built on it: `str(x or "")`, `int(x or 0)`.
- Strings are immutable Unicode. `bytes` are raw; decode
  explicitly: `raw.decode("utf-8", "replace")` — the
  `"replace"` error mode keeps bad input from killing a run.
- Numbers: `/` is true division, `//` floors, `%` is modulo,
  `**` is power. Integers are arbitrary precision.

## Strings

- f-strings interpolate: `f"{name} has {n} words"`. Format
  specs ride after a colon: `f"{x:.2f}"`, `f"{n:>6}"`,
  `f"{ts:%Y-%m-%d}"` (datetime). The shipped bridge code more
  often concatenates with `+` after `str()` — house style in
  bridge.py; either is correct, concatenation is what the
  codebase around a chip does.
- Workhorses: `s.strip()`, `s.split()` (any whitespace runs),
  `s.split(",")`, `",".join(parts)`, `s.lower()`, `s.startswith(p)`,
  `s.endswith(p)`, `s.replace(a, b)`, `s.find(sub)` (-1 when
  absent), `sub in s`, `s.count(sub)`.
- Slicing: `s[a:b]`, `s[:n]`, `s[-n:]`, `s[::-1]` reverses.
  Slices never raise for out-of-range indexes.

## Containers

- Lists: ordered, mutable. `append`, `extend`, `pop()`,
  `sort(key=..., reverse=...)`, `len`, indexing. Build with
  comprehensions (below), not append-loops, when the shape is
  a simple map/filter.
- Dicts: insertion-ordered. `d.get(k, default)` never raises;
  `d[k]` raises `KeyError` when absent — prefer `.get` for
  data that came from outside (JSON, payloads, manifests).
  `d.items()`, `d.keys()`, `d.values()`; merge with `{**a, **b}`.
- Sets: uniqueness and membership. `a | b`, `a & b`, `a - b`.
- Tuples: immutable sequences; the shape of `(text, status)`
  returns — unpack them: `text, status = pair`.
- Unpacking everywhere: `a, b = b, a`; `head, *rest = items`;
  `for i, item in enumerate(items)`; `for k, v in d.items()`.

## Comprehensions

- `[f(x) for x in xs if keep(x)]`, `{k: v for k, v in pairs}`,
  `{x for x in xs}`. Generator expressions in parens stream
  instead of building: `sum(len(x) for x in xs)`,
  `any(...)`, `all(...)`, `next((x for x in xs if ok(x)), None)`.
- Keep a comprehension to one idea. A three-line loop that a
  reader can follow beats a nested comprehension nobody can.

## Control flow

- `if / elif / else`; the conditional expression
  `a if cond else b` for values, statements for actions.
- `for` over any iterable; `while` for condition loops;
  `break`, `continue`, and the loop `else` (runs when no
  `break` happened — handy for search loops).
- `match` (3.10+) for structural dispatch on shapes — useful
  on parsed JSON: `match payload: case {"name": str(n)}: ...`.
  Reach for it when the alternative is a ladder of
  `isinstance` checks.
- Iteration helpers: `range(n)`, `enumerate`, `zip(a, b)`,
  `sorted`, `reversed`, `min`/`max` with `key=`.

## Functions

- `def f(a, b=default, *args, key_only=None, **kwargs)`.
  Defaults are evaluated once, at definition — never use a
  mutable default (`[]`, `{}`); use `None` and fill inside.
- Return early and flat over nesting deep. Multiple returns
  are fine; return a tuple for a value-plus-status pair, the
  shape `llm_chat` itself uses.
- Lambdas are for one-expression callbacks (`key=lambda p:
  p["name"]`), not for named logic.
- A function's contract in this codebase is stated in the
  comment block above it — why it exists, what it refuses,
  what it returns. Write yours the same way.

## Exceptions

- `try: ... except SomeError as e: ... else: ... finally: ...`.
  Catch the narrowest exception that names the failure:
  `json.JSONDecodeError`, `OSError`, `KeyError`,
  `UnicodeDecodeError`, `subprocess.TimeoutExpired`.
- `except Exception` is the codebase's boundary idiom at
  system edges (a handler must not die on one bad chip) — it
  belongs at boundaries, not around every call.
- Raising: `raise ValueError("plain words naming the
  problem")`. The bridge's refusals are `ValueError`s whose
  message is the explanation the operator reads — imitate
  that: an error message is a sentence a person acts on.
- Chain deliberately: a bare `raise` inside `except`
  re-raises; `raise NewError(...) from e` keeps the cause.
- Never swallow silently at a surface the operator sees:
  catch, name what failed, degrade to the honest fallback
  (the declared output, the refusal, the empty shelf).

## Context managers

- `with open(path, encoding="utf-8") as f:` — the file closes
  on exit, exception or not. Always open text files with an
  explicit `encoding="utf-8"`.
- Modes: `"r"` read, `"w"` write (truncates), `"a"` append,
  `"x"` exclusive create; add `"b"` for binary
  (`"rb"`, `"wb"`). PNG work is always binary.
- Write your own with `contextlib.contextmanager` when a
  chip has a setup/teardown pair worth naming.

## Classes, lightly — and dataclasses

- Chips rarely need classes. When state groups naturally,
  a `@dataclass` is the house-shaped tool:

```python
from dataclasses import dataclass, field

@dataclass
class Finding:
    title: str
    severity: str = "info"
    tags: list = field(default_factory=list)
```

- Dataclasses give `__init__`, `__repr__` and equality for
  free; `field(default_factory=...)` is how mutable defaults
  stay safe.
- Plain classes: `class C:` with `__init__(self, ...)`
  assigning to `self.x`. Methods take `self` first. Keep
  chip classes shallow — data and one job.

## Typing hints, as the codebase uses them

- Annotate signatures, not every local: `def load(path: str)
  -> dict:`. Built-in generics are the 3.12 way:
  `list[str]`, `dict[str, int]`, `tuple[str, int]`,
  `set[str]`, `str | None` for optionals.
- Hints are documentation the interpreter ignores (mostly) —
  they cost nothing at run time and tell the next reader the
  shape. `typing.Any` is the honest escape hatch for JSON-
  shaped data.

## Modules and imports

- `import json`, `from pathlib import Path`. Imports go at
  the top of the file, stdlib first. A chip's own modules
  import by their file-set path: `from texttools import
  count_words` for a sibling, `from . import helpers` only
  inside package mode (see the runtime document).
- `if __name__ == "__main__":` guards code that should run
  only when a file is executed directly — inside a chip,
  `run()` is the entry; module level should declare and
  define, not act.
