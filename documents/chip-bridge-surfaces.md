# Bridge surfaces and boundaries — what a chip can and cannot reach

Provenance: condensed from the endpoint handlers and header
documentation in `bridge.py` (the `/api/chips/*` and
`/api/workflows/*` routes, `execute_chip`, `run_workflow_chain`)
in the shipped Apex v0.6.9 tree (build stamps 2026-11-19.41,
seq 41).

## The HTTP surface a chip author meets

The bridge serves on port 8675. The chip endpoints:

- `POST /api/chips/run` — `{"query": "<name> <args>"}`.
  Executes a chip by invocation query and returns
  `{"command": "chip <slug>", "output": ..., "exit_code": ...}`.
  `/api/chat` runs chips the same way when a message invokes
  one.
- `POST /api/chips/create` — `{name, description, code,
  output, files, glyph, version, icon}`. Chip Studio's forge:
  builds the cartridge, writes `chips/<slug>.png`, registers a
  `local/<slug>` install record, returns the filename. Caps:
  name 120, description 500, code
  200,000, output 5,000, glyph 40,
  version 40 characters.
- `POST /api/chips/save` — `{file, name, description, code,
  output, files, glyph, version, icon}`. Re-forges an existing
  chip file in place with the edited file set. Saving over a
  GitHub-installed chip marks its record `locally_edited`.
- `GET /api/chips/source?name=<chip file>` — a local chip's
  code and full file set, for reopening it in the Studio.
- `POST /api/chips/install` / `/api/chips/update` /
  `/api/chips/delete` — `{full_name}`: the catalog lifecycle
  described in the cartridge document.
- `POST /api/chips/export` — `{file, version}`: builds the
  GitHub-ready repo ZIP (README, `navi-chip.json`, the files,
  the `.png`, `.gitignore`) in `chips-exports/`.
- `POST /api/workflows/create` — `{name, description,
  steps}`: a workflow needs a name and at least two chips from
  the folder. `POST /api/workflows/run` — `{id}`: runs the
  chain. Workflows are stored in `chips/workflows.json`.

## Configuration — what a chip can know

No configuration object is handed to chip code. There is no
`navi_internal` method that reads the seat's config, and a chip
must not go reading `config/navi.config.json` itself — that
file is the operator's, its shape is the bridge's business,
and a chip that depends on it breaks when the shape moves.
What a running chip legitimately knows:

- the **environment**: `NAVI_LLAMAFILE_ENDPOINT` and
  `NAVI_MODEL` (set by the bridge for the run), the inherited
  OS environment, and `PYTHONPATH` pointing at its sandbox;
- the **operator's name** as the stub's `get_user()` gives it;
- **its own file set**, importable by relative path or package
  path exactly as authored;
- the **working directory**: the shell's working directory on
  the seat — a chip that reads or writes relative paths does
  so there, in the open, where the operator can see it.

A chip that needs a preference asks for it — as an argument,
or by printing the question — it does not go digging.

## Cases and the Vault — the filing path

The stub exposes **no case or Vault API**. A chip cannot file
into a case directly, and `process_message` — the in-app route
that would hand text to Navi's pipeline — is unavailable in a
bridge chip run. The filing path that exists is the one the
whole system uses: **the chip prints a result worth filing
into the chat**, and from the chat it is filed — Navi files
conversationally into the open case under the print-first law,
and the latest terminal or chip output can be taken as a
finding from the case page. Design chips accordingly: make the
printed output a clean, self-contained record (what was asked,
what was found, what it means), because that text is what gets
filed.

## The Mesh and borrow — a hard boundary

The chip API has **no Mesh surface at all**: no borrowing, no
lending, no peer calls, no federation verbs. `llm_chat` reaches
only the seat's own configured model endpoint. A chip executes
as a local subprocess on the seat whose operator invoked it,
when they invoke it — nothing in the cartridge format lets a
chip arrive from the Mesh and run, and borrowed replies are
inert by doctrine: they never drive the borrower's machinery.
If a chip wants the fleet, that is Fleet Answers' job, not a
chip's.

## The trust wall — why install is an operator act

Be plain about what a chip **is** at run time: ordinary Python,
run with the operator's own OS privileges, in a temp directory,
under a 20-second budget, with its output captured. The temp
directory and the budget are courtesy, not a sandbox — chip
code can do anything the operator's account can do. That is
the deliberate shape of the system, and it is why:

- installing a chip is an **operator act**, never automatic;
- declared dependencies are **shown, never installed**;
- a chip's provenance rides its cartridge and its install
  record (`local/<slug>` or a GitHub full name);
- stdlib-first is the house rule: a chip that needs nothing
  beyond the standard library asks for no trust beyond its own
  code.

Author to that wall: no surprises in the code, no network
calls the description didn't promise, no writes outside what
the chip says it writes.

## Workflows — chips in a chain

A workflow is an ordered list of chip files. Running it calls
`execute_chip` for each step in turn, with **each chip's
output as the next chip's arguments text** — the way a
following chip in a pipeline reads what came before it. A step
whose chip is no longer in the folder stops the chain with
that reason; the steps that ran are still reported. A chip
that wants to serve in chains should read `arguments.text`
tolerantly: in a chain, that text is another chip's whole
output, not a person's tidy command line.
