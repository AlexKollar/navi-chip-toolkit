# The Navi Chip-Building Toolkit — a guide to this pack

This document is GENERATED (by generate.py) — it is the pack's own map of itself. The API documents are condensed from the shipped tree with provenance in their headers; the Python reference is written for this pack.

## What this pack is

The toolkit Navi authors chips with, and only that: **her API** (the chip cartridge format, the navi_internal runtime, the bridge surfaces and their boundaries, one verified worked example) and **a Python reference** (the core language and the stdlib modules chips actually use, stdlib-first). It exists so she can build chips fluidly, from the shipped truth, without the whole source tree in the room — that is the development library pack's job; this one is the workbench, not the warehouse.

## The version lock

- Generated from: the shipped Apex **v0.6.9** tree (VERSION file: `0.6.9 edge`).
- Build stamps: **2026-11-19.41** / seq **41** — the same stamps the running bridge reports about itself.
- Regeneration rule: this pack is regenerated from the shipped tree at every release and its version moves with the build. An API reference that does not match the running build teaches dead APIs, so an out-of-date copy of this pack should be updated, not trusted.

## Quick facts — the questions chip authoring asks

How does a chip register a verb? In this build it doesn't — a chip registers no new bridge routes or verbs (that machinery is Chips Expanded, a later era, not in v0.6.9). What a chip registers is its invocation surface: its cartridge name and slug, its declared `command` and `aliases` in root.py, and its place on the shelf and in Workflows. The runtime document has the full answer.

What goes in a chip cartridge manifest? A navi-chip.json manifest carries: format (`navi-chip/1`), name, description, version, entry (`root.py`), files (the whole file set, root.py first), output (the declared output, or empty), glyph, and icon. The cartridge document has every field and its caps.

How big can a chip be? The entry code and each file's content are truncated at 200,000 characters at forge and at install; the declared output caps at 5,000 characters at forge. Keep chips lean — the cap is a cliff, not a target.

How long can a chip run? Twenty seconds. execute_chip runs a chip in a subprocess with a 20-second budget; a chip that runs longer is stopped, and its declared output is shown with the reason when it has one.

What does llm_chat return? A `(text, status)` tuple — the model's reply text and 200 on success; on any failure the text is a plain-words reason and the status is 0. Unpack it and check the status before trusting the text.

Which Python does a chip get? The seat's Python 3.12, stdlib-first: a chip that needs nothing beyond the standard library asks for no trust beyond its own code. Declared requirements are shown on the chip's card and never installed automatically.

Which build is this toolkit locked to? Apex v0.6.9, build stamps 2026-11-19.41, seq 41.

## How the pack is organised

Eight documents: the API first (cartridge, runtime, surfaces, the worked example), then the Python reference (core language, files and data, processes and the network). Documents:

- `documents/00-pack-guide.md` — This guide: what the pack is, the version lock, and the quick-facts answers to the questions chip authoring asks.
- `documents/chip-cartridge-format.md` — The cartridge: the navi-chip tEXt payload field by field, the navi-chip/1 repo manifest, entry resolution, glyphs and icon art, requirements.
- `documents/chip-runtime-api.md` — The runtime: navi_internal method by method as the shipped stub behaves, root.py's declarations, the Studio's four structure checks, invocation, and the 20-second execution model.
- `documents/chip-bridge-surfaces.md` — The surfaces and boundaries: the /api/chips endpoints, what a chip can know of config, the case/Vault filing path, the Mesh boundary, the trust wall, workflows.
- `documents/chip-worked-example.md` — One minimal chip end to end — Word Count: root.py, its helper, its manifest, its payload, and what the bridge does when it runs. Verified against the shipped code at generation time.
- `documents/python-core-language.md` — Python 3.12, the core language: types, strings, containers, comprehensions, functions, exceptions, context managers, dataclasses, typing, imports.
- `documents/python-stdlib-files-and-data.md` — Python stdlib for data work: json, pathlib/os, re, datetime, hashlib, zipfile, struct (with the PNG chunk walk), sqlite3.
- `documents/python-stdlib-process-and-net.md` — Python stdlib for acting: subprocess with its safety rules, urllib/http.client, threading, and the house patterns of the shipped codebase.

## What is NOT in here

The shipped source itself (that is the development library pack), the operator's data, and any API that does not exist in the shipped build — chip registration of routes and verbs (#298) is named here only to say plainly that it is not in this build.
