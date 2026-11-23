# Refactoring Tapir: what the size actually is, and what to do about it

A proposal for discussion, not a decision. Everything below is measured on `main` at `0bb1e0f`
(4,333 files, ~778k counted lines, excluding binaries); the numbers are what the argument rests on,
so they are easy to re-check with the commands in the last section. Section 7 lists the open pull
requests that touch the same ground, and the order this plan waits for them in.

## 1. The repo is not fat. It is tall.

There is almost no duplication to delete: the whole tree has **two** byte-identical file pairs. The size
comes from one property — *the unit of this repository is a command, and every command has to appear in
8–11 places by hand*.

There are **804 commands in the published reference** (831 in the sources). `ModifyKeynoteItems` alone
touches seven areas: the C++ (schema + implementation), an example, a check, a Grasshopper component,
the Python client, the command reference, plus an icon. `CreateColumns` touches sixteen check files.

| area | files | lines | nature |
| --- | --- | --- | --- |
| `archicad-addon/Sources` | 178 | 247,942 | product logic — **~69k of it is JSON schema literals inside C++** |
| `grasshopper-plugin/…/Components` | 868 | 114,663 | one class per command, median 95 non-blank lines, **~53% scaffolding** |
| `docs/archicad-addon` | 11 | 75,493 | **generated, committed** |
| `archicad-addon/Test/checks` | 380 | 69,250 | hand-written tests — the real suite, keep |
| `marketplace` + `builtin-scripts` | 128 | 33,321 | user-facing scripts, keep |
| `tools` | 64 | 16,590 | tooling, keep |
| `archicad-addon/Examples` | 855 | 15,037 | one script per command, documentation content |
| `archicad-addon/Test/Scripts` | 170 | 12,703 | golden scripts |
| `python-client` | 9 | 12,061 | **generated** `commands.py` is 11,425 of it |
| GH icons | 753 PNG + 289 SVG | — | about 280 of the PNGs have an SVG source; the rest exist only as PNG |
| everything else | ~150 | ~30,000 | site pages, branding, installer |

## 2. Three things are already half-migrated, and finishing them is the whole job

**Schemas are half data.** Shared definitions already live in a data file —
`Sources/RFIX/Images/CommonSchemaDefinitions.json` (1.3 MB), compiled into the add-on as a `FILE`
resource (`ID_COMMON_SCHEMA_DEFINITIONS_FILE`) and read once with `RSLoadResource`. But the **per-command**
schemas — 729 `GetInputParametersSchema` and 826 response-schema implementations — are JSON written as
raw string literals inside 74 `.cpp` files. A third of the C++ is JSON that happens to be spelled in C++.

**Grasshopper components are data spelled as code.** 868 hand-written C# classes, one per command, with
~53% of their non-blank lines being boilerplate (`using`, class header, `In*`/`Out*` calls with a
description). The proof that this is data wearing a costume: `tools/generate_gh_docs.py` is a 700-line
**regex parser over the C#** whose only job is to read those inputs back out, and it documents that a
name built at run time "is left out rather than guessed".

**Generated output is committed.** The command reference (75k lines) and the Python client (11.4k) are
produced by `tools/generate_addon_docs.py` and `python-client/generate.py`, and both are checked in.
Every release rewrites 75k lines that no one reviews, which is why a drift gate is worth having.

## 3. The moves, in the order that pays

The first four shrink the tree; the fifth shrinks what a pull request costs to verify.

### Move 1 — finish the schema migration (enabler; no line count won, everything else unlocked)

Extract each command's input and response schema from the C++ to
`archicad-addon/Sources/Schemas/<Command>.json` (`{ "input": …, "response": … }`), bundle the folder
into one `FILE` resource at build time, and have `CommandBase` look a command's schemas up by its name.
That is the mechanism the shared definitions already use, so nothing new is loaded from disk at run
time, and no generated C++ header is needed (a header of ~1,500 large string literals would also run
into MSVC's limit on string literal size).

*Mostly scriptable, not entirely*: about 1,456 schemas are plain raw literals, already valid JSON, none
of them inside a version check, so they can be extracted by script. **62 are built by code**
(`BuildLibraryPartBasedSchema ("objectsData", "Object", …)` and the like) and need a template or a
shared definition first; they stay in the C++ until then.

*Five readers move with it*, in the same pull request as the first extracted family, or the tooling
breaks: `tools/generate_addon_docs.py` (a C++ tokenizer today), `tools/new_command.py`,
`tools/check_examples.py`, `tools/check_tool_settings.py`, and `tools/check_contracts.py` if #3022 has
merged. So do the CLAUDE.md sections the issue bot follows ("Schemas", "Adding or changing a command"):
otherwise it keeps writing schemas into the C++.

The pilot is the Keynotes family: **seven** commands (`GetKeynoteTree`, `GetKeynoteAutoTexts`,
`CreateKeynoteFolders`, `CreateKeynoteItems`, `ModifyKeynoteFolders`, `ModifyKeynoteItems`,
`DeleteKeynoteFolders`), whose implementations exist only from Archicad 28, so the pilot also proves the
schemas load on 25–27. Its proof is `tools/expand_schemas.py --compare` reporting no difference for any
command, a byte-identical `command_definitions.js`, and all twelve builds green.

Payoff: a new command becomes **schema + implementation** instead of ten places; docs, client,
Grasshopper, checks skeletons and LLM files all read one input. Effort L, risk M, and it is the
prerequisite for Move 2.

### Move 2 — generate the Grasshopper components (−~95k lines, 12% of the repo)

Today each component is a 95-line class encoding three things: the command name, a per-parameter
mapping from JSON to Grasshopper inputs (type, order, label), and boilerplate. Move 1 makes the first
mechanical; the second is a real decision that should be **~10 lines of JSON per command**
(`components/<Command>.json`), and the third should not be written by hand at all.

A generator plus a small runtime (the `ArchicadExecutorComponent` base already exists) turns 114k lines
of C# into ~8k lines of mapping data + ~2k of generator. And `generate_gh_docs.py` stops regex-parsing
C# — it reads the same mapping data.

Migratable command by command, with a golden comparison of the 868 components' metadata (name, group,
inputs, outputs) before and after. Effort L, risk M–H.

Three limits on it:

- **It must not undercut the tool components.** CLAUDE.md's target architecture replaces the element
  components with one component per Archicad tool (Walls first), labelled from `ToolSettings/`, and
  retires the old per-command ones after one release. Generating those per-command components would be
  work on what is meant to go away, so Move 2 covers the commands that no tool component will.
- **Saved definitions must keep working.** A `.gh` file finds a component by its GUID and its
  parameters by position, so the mapping data carries each component's GUID and keeps the parameter
  order exactly.
- **Measure first.** ~53% scaffolding means ~47% is real code. How many components are pure
  pass-through, with no logic of their own, is not measured yet, and that count, not 868, is what
  decides whether this saves 95k lines or much less.

### Move 3 — stop committing generated output (−87k lines, 11%)

`docs/archicad-addon` and `python-client/commands.py` become build artifacts: generated in the Pages
workflow and in the release, published to a branch the way `api-coverage` already is, with a `--check`
in CI for the pull requests that touch schemas. It removes 75k lines from every release diff, and it
ends the up-to-a-week lag between a merged command and its docs on `main`.

Effort M, not S, for three reasons:

- **The site is served from `docs/` on `main`**; there is no Pages deploy workflow. Publishing the
  generated files means switching Pages to deploy from Actions, a repository setting only a maintainer
  can change.
- **There are more generated files than the two above.** `llms.txt`, `llms-full.txt`,
  `commands.json`/`.md`, each page's `index.md` and `sitemap.xml` come from the same sources and are
  committed too; they move with the docs.
- **Some tools read the committed output.** `tools/check_contracts.py` (#3022) reads
  `docs/archicad-addon/command_definitions.js`; it would generate the file first instead.

It builds on two open pull requests rather than replacing them: #3021 fails a pull request whose
committed generated files are stale (it keeps them honest until they stop being committed, and the
Grasshopper reference is already stale on `main`), and #3001 regenerates and publishes the Python
client to PyPI with each release (the release can keep publishing it once `commands.py` is no longer
committed).

### Move 4 — shrink the per-command ritual and the icons (−1,000 files)

The PNG icons that have an SVG source (about 280 of the 753) should be built, not committed (or
replaced by the SVG at runtime). The other ~475 exist only as PNG and stay committed until someone draws
their SVG, so this saves about 280 files, not 750. The
`Test/checks` skeleton and the `Examples` stub can start from the schema the way `tools/new_command.py`
already scaffolds the rest.

### Move 5 — thin the build matrix on pull requests (CI cost, not files)

`devkits.json` lists six Archicad versions (25–30), and a pull request that builds builds all six on
Windows and macOS: **12 builds**, the largest thing a pull request triggers.

Most of the saving is already in place: `.github/actions/build-gate` skips every build for a pull request
that only changes docs, tests, examples, workflows, scripts, branding or Markdown. What is left is the
pull requests that change `tools/`, `.github/`, the installer or the Grasshopper plugin, which still build
the whole add-on matrix without touching the add-on. Those could build two versions per OS (the oldest
and the newest) and leave the full matrix to the push to `main`. Effort S, risk L.

The full matrix must stay on every pull request that changes `archicad-addon/Sources`. Version-specific
breakage is common: #2968 failed on Windows AC25 and AC26 only, and passed on 27–30. And the issue bot's
pull requests are merged automatically once every build check is green, so a nightly-only matrix would
let a bot change that breaks an older Archicad merge with no person looking.

### What I would *not* do

- **Do not split into separate repositories yet.** The schema is the shared dependency; splitting first
  gives five repos with the same replication and a versioning problem between them. Split after Move 1,
  if at all.
- **Do not touch `Test/checks`.** 380 files, 69k lines, and it is the only part of the repo that
  actually checks the product.
- **Do not shrink the marketplace and built-in scripts.** They are user-facing product, not overhead.
- **Do not touch the per-version `RFIX` resources.** 1.4 MB in total; not the problem.

## 4. Where it lands

| | now | after |
| --- | --- | --- |
| files | 4,333 | ~3,300 (icons: only the ~280 with an SVG source) |
| lines | ~778k | ~590k at most; Move 2's share waits on the pass-through count |
| C++ that is JSON | ~69k | 0 |
| Grasshopper C# | 114,663 | ~10k (data + generator) if most components are pass-through, less saved otherwise |
| generated output in git | 87k | 0 |
| places to touch for a new command | 8–11 | **2** (schema, implementation) |
| CI per pull request | 12 builds, or none for docs/test-only changes | 12 for add-on changes, 4 for tooling changes, none for docs/test-only |

The line counts are the smaller half of the payoff. The real one is that adding a command stops being a
ten-file ritual that only a person with the whole convention in their head can do correctly — which is
also why a new command is so easy to get partially right: nothing in the repository says which of the
ten places is the source and which are copies.

## 5. Start here

Move 1 is the enabler. The first pull request I would open is the **Keynotes pilot**: seven commands,
schemas extracted to JSON and bundled into a `FILE` resource, the five readers and CLAUDE.md updated in
the same pull request, built on all twelve targets, with `expand_schemas.py --compare` showing nothing
changed — a small, reviewable slice that proves the whole pattern before 800 commands follow it.

It waits for the pull requests in section 7 that change the same files. Then Move 3 (once a maintainer
agrees to deploy Pages from Actions), the pass-through count before Move 2, and Move 4 for the icons that
have an SVG.

## 6. How to re-check the numbers

```bash
git ls-files | wc -l                                             # 4,333 files
wc -l archicad-addon/Sources/*.cpp | tail -1                     # 201,380 lines of C++
git ls-files grasshopper-plugin/TapirGrasshopperPlugin/Components | wc -l    # 868 components
grep -c "GetInputParametersSchema" archicad-addon/Sources/*.cpp | awk -F: '{s+=$2} END {print s}'   # 729
ls grasshopper-plugin/TapirGrasshopperPlugin/Resources/*.png | wc -l        # 753 icons
```

The schema share of the C++ (~34%) is the count of lines in those files that are JSON member
declarations (`"$ref"`, `"type"`, `"description"`, `"properties"`, braces and brackets) — an
approximation, and the only number here that is one.

## 7. Open pull requests that touch the same ground

None of them does any of the moves, and none should be stopped for this plan; the plan is sequenced
around them.

| pull request | what it does | relation to the plan |
| --- | --- | --- |
| #3023 | unit tests for `tools/expand_schemas.py` | **merge before the pilot**: the pilot's proof is that comparison |
| #2520 | hands Archicad the `$ref`s in the form it resolves, so shared definitions are validated | **merge or close before the pilot**: it changes `CommandBase`, `SchemaDefinitions.cpp` and `CommonSchemaDefinitions.json`, the files the pilot changes; the pilot keeps whatever `$ref` form `main` then uses |
| #2755 | refuses unknown keys inside shared definitions, for every command (#2623) | **merge or close before the pilot**: same files |
| #3022 | checks the test steps against the command schemas | if merged first, `check_contracts.py` is a fifth reader the pilot updates, and Move 3 must generate `command_definitions.js` for it |
| #3021 | fails a pull request whose committed generated files are stale | Move 3 builds on it |
| #3001 | regenerates the Python client with each release and publishes it to PyPI | Move 3 keeps the publishing, drops the commit |
