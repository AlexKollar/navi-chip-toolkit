# A minimal chip, end to end — Word Count

Provenance: this example was authored for this pack and
**verified against the shipped Apex v0.6.9 tree** at
generation time (build stamps 2026-11-19.41, seq 41): its
code passes Chip Studio's four structure checks, it forges
with the shipped `build_cartridge_png`, it parses back with
the shipped `parse_cartridge_png` with every field intact,
and it runs under the shipped `execute_chip`. The generator
re-runs all of that on every regeneration; a failure stops
the build, so what you are reading is what the shipped code
accepts.

## What the Word Count chip does

Word Count takes a line of text and reports its word and
character counts. One entry file, one helper module, no
declared output — it computes. It uses only `print_message`
from navi_internal and one stdlib-free helper, so it runs
anywhere.

## root.py — the entry

```python
import navi_internal

navi_instance = navi_internal.navi_instance

command = "wordcount"
use = "Count the words and characters in a line of text."
aliases = ["wc", "countwords"]
params = "wordcount <text> - counts the words and characters in <text>."

from texttools import count_words


def run(arguments):
    text = (arguments.text or "").strip()
    if not text:
        navi_instance.print_message(
            "Give me a line of text and I will count it: wordcount <text>")
        return
    words, chars = count_words(text)
    navi_instance.print_message(
        str(words) + " words, " + str(chars) + " characters.")```

The four declarations sit at module level where the Studio's
checks look for them: `command`, `use`, `aliases`, `params`.
`run(arguments)` reads `arguments.text`, refuses an empty ask
by name — in plain words, the way the bridge itself refuses —
and answers through `print_message`.

## texttools.py — the helper module

```python
"""Helpers for the Word Count chip."""


def count_words(text):
    """Return (word count, character count) for a line of text."""
    return len(text.split()), len(text)```

A plain module at the file set's root. `root.py` imports it
with `import` semantics — `from texttools import count_words`
— and the runner places it beside `root.py` in the sandbox,
so the import resolves exactly as authored. No package mode
is triggered: no relative imports, no `/` in any path.

## The manifest — navi-chip.json

What the GitHub-ready export writes for this chip:

```json
{
  "format": "navi-chip/1",
  "name": "Word Count",
  "description": "Counts the words and characters in a line of text.",
  "version": "0.1.0",
  "entry": "root.py",
  "files": [
    "root.py",
    "texttools.py"
  ],
  "output": "",
  "glyph": "text",
  "icon": [],
  "generatedBy": "Apex - Chip Studio chip export"
}```

`entry` names `root.py`; `files` lists the whole set,
`root.py` first; `glyph` is `text` — a real glyph name, so
the face wears it without the keyword heuristic; `icon` is
empty (no author art); `output` is empty because the chip
computes instead of declaring.

## The cartridge payload

Forged into the PNG's `navi-chip` tEXt chunk, the payload is
the same facts in the cartridge's own shape: `name`,
`description`, `code` (the root.py text above), `output`
(empty), `files` (`root.py` synthesized first from `code`,
then `texttools.py`), `glyph`, `version`, `icon` (empty
list). The version — `0.1.0` — also rides the
drawn face.

## What happens when it runs

Someone types `./wordcount` — no: the invocation resolves by
the chip's **name and slug**, so `./word-count the quick
brown fox` (or `/chip word-count ...`, or "run the Word
Count chip"). The bridge:

1. reads the cartridge out of `chips/word-count.png` with
   `parse_cartridge_png`;
2. sees `def run` in the code, so the chip executes (a chip
   without it would show its declared output instead);
3. writes the sandbox: stub `navi_internal.py`, the `navi.py`
   shim, `root.py`, `chip_module.py`, `texttools.py`;
4. runs the runner with the argument text
   `the quick brown fox`, inside the 20-second budget;
5. captures the output — `4 words, 19 characters.` — and
   lands it in the chat as the chip's result.

Run it with no text and it answers with its own instructions
instead of an error. That is the whole shape: declare, define
`run()`, print what the operator should see, finish fast.
