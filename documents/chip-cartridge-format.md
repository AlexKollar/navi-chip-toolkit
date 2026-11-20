# The chip cartridge format — how a chip is stored

Provenance: condensed from `chips/README.md` and the cartridge
machinery in `bridge.py` (`build_cartridge_png`,
`parse_cartridge_png`, `_valid_chip_files`, `fetch_chip_payload`,
`build_chip_repo_files`, the `/api/chips/create` and
`/api/chips/save` handlers) in the shipped Apex v0.6.9 tree
(build stamps 2026-11-19.41, seq 41). Where this document and
the code ever disagree, the code is the truth.

## What a chip is, physically

A chip is a single `.png` file. The visible image is a drawn
battle-chip face (528x616, drawn at forge time). The chip's entire
codebase rides inside the PNG in a **tEXt chunk with the keyword
`navi-chip`**: the chunk data is the keyword, a NUL byte, then the
payload as UTF-8 JSON. Share the image and you share the chip —
there is no separate code file. The forge inserts the tEXt chunk
immediately after the IHDR chunk, ahead of the image data
(`build_cartridge_png` in bridge.py).

Chips live in the install's `chips/` folder. They survive updates,
reinstalls, and even a `--fresh` install. A cartridge whose
payload will not parse is still listed — as unreadable, visible
and deletable, never silently dropped — but it cannot run.

## The payload — every field

The payload is one JSON object. `parse_cartridge_png` reads these
fields; the forge handlers cap them as shown:

- `name` — **required.** A payload that is not a JSON object, or
  has no `name`, does not parse at all. At forge time the name is
  trimmed and capped at 120 characters. The chip's file
  name is the slug of the name (lowercase, non-alphanumerics
  folded to `-`) plus `.png`.
- `description` — one line on what the chip does; capped at
  500 characters at forge time. It rides the
  chip's card, its catalog entry, and the shelf reply in chat.
- `code` — the entry code: the content of `root.py`, the file
  Navi runs. Truncated at 200,000 characters at forge time
  and again at install time — a longer file is cut, so keep the
  entry well under the cap.
- `output` — the chip's **declared output**: what a chip that
  defines no `run()` shows instead of executing. Capped at
  5,000 characters at forge time. A declarative chip is
  a legitimate chip — declare the output and define no `run()`.
- `files` — the chip's full file set, a list of
  `{"path": ..., "content": ...}`. Validation
  (`_valid_chip_files`) synthesizes `root.py` first, holding
  `code`; each further entry is kept only when its path is
  relative, ends in `.py`, has no `..` segment, does not start
  with `/`, and is not a duplicate. Unsafe or duplicate entries
  are dropped silently. At install, each file's content is
  truncated at 200,000 characters, like the code.
- `glyph` — the face's built-in glyph name, capped at
  40 characters. It is used only when it names a real
  glyph (see the list below); otherwise a keyword read of the
  chip's name and description picks one, and if nothing matches,
  the face falls back to the legacy name-seeded pattern.
- `icon` — the author's own face art: exactly 20 strings of
  exactly 20 characters over `.` (empty), `#` (accent) and `o`
  (shade). Art in the wrong shape **refuses a forge or an
  install in plain words**; art that arrives damaged inside a
  picture parses as no icon at all — a bad picture never bricks
  a cartridge, the face falls back to the glyph.
- `version` — the chip's own version, capped at 40
  characters. It rides inside the picture: drawn on the face,
  named in the install receipt, shown on the ChipDeck sleeve.
  Older pictures simply have none, and no surface invents one.
- `readme` — optional: an operator-edited README kept inside the
  cartridge, used in place of the generated page on export.
- `requirements` — not authored in the cartridge. At install
  time the bridge reads the repo's `chip-requirements.txt` (see
  below) and stores the declared dependencies on the record.

The built-in glyphs a chip can name (`GLYPH_PIXELS` in
bridge.py): `key`, `lock`, `hash`, `text`, `swap`, `gear`, `network`, `image`, `music`, `dice`, `book`, `terminal`, `shield`, `clock`, `chart`, `star`. The keyword heuristic that picks one
when no glyph is named reads, in order: cryptex/cipher/encrypt
words earn `key`; hash words earn `hash`; password/vault words
earn `lock`; then swap, music, image, dice, book, network,
shield, terminal, clock, chart, text, gear, star.

## The repo form — navi-chip/1

A chip published for everyone lives in a public GitHub repo
carrying the **navi-chips** topic. The GitHub-ready export
(`build_chip_repo_files`) writes: a README page, a
`navi-chip.json` manifest, the chip's files at the repo root
(`root.py` is the entry; the rest are its importable modules),
the forged `<slug>.png`, and a `.gitignore`. The manifest:

```json
{
  "format": "navi-chip/1",
  "name": "<chip name>",
  "description": "<one line>",
  "version": "<version>",
  "entry": "root.py",
  "files": ["root.py", "<module>.py"],
  "output": "<declared output, or empty>",
  "glyph": "<glyph name, or empty>",
  "icon": [],
  "generatedBy": "Apex - Chip Studio chip export"
}
```

What goes in a chip cartridge manifest is exactly those fields:
format, name, description, version, entry, files, output,
glyph, icon. At install (`fetch_chip_payload`) the bridge walks
the repo **recursively** — package folders keep their relative
paths — and picks the entry file in this order: the manifest's
`entry`, then `root.py`, then the repo name minus a `navi-`
prefix plus `.py`, then the repo name plus `.py`, then
`chip.py`, then `main.py`, then the repo's only Python file if
there is exactly one. Several Python files and no determinable
entry is a refusal that names the fix. Every Python file comes
down at its relative path: the entry becomes `code`, the rest
ride as `files`.

A repo may declare dependencies in `chip-requirements.txt` at
its root: non-comment lines, at most 50 kept. They are shown on
the chip's card, plainly — **they are never installed
automatically.** A chip that needs a third-party package must
say so and degrade honestly without it; stdlib-first code needs
no such conversation.

Installs are recorded in `chips/installed.json`: a GitHub full
name for catalog installs, a `local/<slug>` record for
Studio-forged chips. Saving an edit to a GitHub-installed chip
marks it `locally_edited`, and a catalog update then refuses
rather than overwriting the edits.
