# Navi Chip-Building Toolkit

Her chip-building toolkit — the chip cartridge format and the navi_internal runtime API exactly as shipped in Apex v0.6.9 (stamps 2026-11-19.41), a verified worked example, and a stdlib-first Python 3.12 reference, so Navi can author chips fluidly from the shipped truth.

A **knowledge pack** for Apex — Navi's answers can draw on
these documents, and cite the pack and document they came from.
The pack format is `navi-rag/1` (see `navi-rag.json`).

## The version lock

This pack is **generated from the shipped Apex v0.6.9 tree** (build stamps 2026-11-19.41, seq 41) by `generate.py`: its API documents are condensed from that tree and its code values extracted from it and asserted at generation time. The Python reference is authored for the pack. The pack is regenerated from the shipped tree at every release; its version is the build's version. If the pack's version does not match the running build, update the pack before trusting it.

## Documents

- `documents/00-pack-guide.md` — This guide: what the pack is, the version lock, and the quick-facts answers to the questions chip authoring asks.
- `documents/chip-cartridge-format.md` — The cartridge: the navi-chip tEXt payload field by field, the navi-chip/1 repo manifest, entry resolution, glyphs and icon art, requirements.
- `documents/chip-runtime-api.md` — The runtime: navi_internal method by method as the shipped stub behaves, root.py's declarations, the Studio's four structure checks, invocation, and the 20-second execution model.
- `documents/chip-bridge-surfaces.md` — The surfaces and boundaries: the /api/chips endpoints, what a chip can know of config, the case/Vault filing path, the Mesh boundary, the trust wall, workflows.
- `documents/chip-worked-example.md` — One minimal chip end to end — Word Count: root.py, its helper, its manifest, its payload, and what the bridge does when it runs. Verified against the shipped code at generation time.
- `documents/python-core-language.md` — Python 3.12, the core language: types, strings, containers, comprehensions, functions, exceptions, context managers, dataclasses, typing, imports.
- `documents/python-stdlib-files-and-data.md` — Python stdlib for data work: json, pathlib/os, re, datetime, hashlib, zipfile, struct (with the PNG chunk walk), sqlite3.
- `documents/python-stdlib-process-and-net.md` — Python stdlib for acting: subprocess with its safety rules, urllib/http.client, threading, and the house patterns of the shipped codebase.

## Install it in Navi

1. Open Navi's **RAG** tab, then **Packs**.
2. **Search GitHub** — every public repo tagged `navi-rag` is listed.
3. **Install** this pack. Navi downloads the documents at the latest
   release and builds the pack's index on your machine from them.

Only **active** packs feed her answers; toggle the pack off in the
same tab whenever you like.

## Layout

- `README.md` — this page. Navi's installer skips a root README, so
  it describes the pack without ever being indexed as a document.
- `navi-rag.json` — the pack manifest (name, version, document list).
- `index.json` — the pack's built index, shipped as a reference copy;
  installing Navi rebuilds it locally from the documents.
- `documents/` — the pack's documents (`.md` / `.txt`).
- `generate.py` + `pack_content.py` — the generator and its authored
  content (stdlib only). Neither is a document; neither is indexed.
- `BUILD-NOTE` — the build note for this generation, extensionless
  on purpose: the installer collects only .md/.txt/.pdf, so the note
  rides the repo without ever becoming a document.

## Publishing an update

Regenerate with `generate.py` against the new release's shipped tree,
push, and publish a new release — the version tag is created with
that publish. Keep the `navi-rag` topic on the repo — that is how
Navi finds it.
