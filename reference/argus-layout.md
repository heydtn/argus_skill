# Argus: where things are

Argus is the hex package `argus_beam` (renamed from `panoptes` at 0.20;
modules keep the `Argus` namespace). Its source and docs are at
`github.com/QuinnWilton/argus`, and paths below are relative to that
repo's root. A project that depends on it gets `lib/` and `priv/` under
`deps/argus_beam`, but not `docs/` or `AGENTS.md`. Read those in a
clone:

```sh
gh repo clone QuinnWilton/argus /tmp/argus_src -- --depth 1
```

From the release after 0.20.1, `docs/analyses/` and `docs/design/` also
ship on hexdocs.pm/argus_beam, beside `docs/bug-classes.md`.

## Docs

| Path | What it is |
|---|---|
| `AGENTS.md` | Setup and checks, where to edit, and the correctness, cache, query and test constraints. Read first. |
| `docs/design/rule-style.md` | How a rule reads and changes: the three layers, names and comments, suppressions, refactors. Short and binding. |
| `docs/design/analysis-model.md` | The shared model: facts to findings, which reach component answers which question, process identity, requests, startup and repeated execution, uncertainty and priors. |
| `docs/bug-classes.md` | The catalog: the analyses, what severity means, how to review a finding. |
| `docs/analyses/<analysis>.md` | Each analysis's findings, the model behind them and their limits, with links to its rules and finding builder. |
| `CHANGELOG.md` | What each release changed, including the schema changes a custom Datalog consumer must account for. |

Up to tag `v0.20.1`, argus had long-form docs these replaced:
`CLAUDE.md`, the Vocabulary section of `docs/bug-classes.md` (each
shared word with its readers) and design notes in `docs/design/`. Read
them there (`gh repo clone QuinnWilton/argus /tmp/argus_v0.20.1 --
--depth 1 --branch v0.20.1`) only when the current docs and the source
comments leave a question open, and check what they say against the
code.

## Code

| Path | What |
|---|---|
| `lib/argus/pipeline.ex`, `pipeline/` | `Argus.Pipeline.run/3` (extract to a facts dir) and `extract/2` (in memory); disassembly, normalization, the base emitter (`emit.ex`, `emit/`), the writer. |
| `lib/argus/extractor.ex`, `extractor/` | The `Argus.Extractor` behavior and the shared helpers (extractors.md). |
| `lib/argus/extractors/` | Argus's domain extractors. |
| `lib/argus/instr.ex`, `instr/`, `cfg.ex`, `cfg/`, `dataflow.ex` | Instruction semantics, CFGs, reaching definitions. |
| `lib/argus/instr_id.ex` | Function and instruction IDs. |
| `lib/argus/schema.ex`, `schema/` | Relation declarations; `mix argus.gen.dl` generates `priv/dl/{base,layer2,priors}.dl` from them. |
| `lib/argus/analysis.ex`, `analysis/` | The `Argus.Analysis` behavior; `run_rules/3`, `derive_stage0/2`, `derive_points_to/2`; `Catalog`, which discovers **only `:argus_beam`'s modules**. |
| `lib/argus/analyses/` | The built-in analyses, each paired with `priv/dl/analyses/<name>.dl`. |
| `lib/argus/findings.ex`, `findings/`, `located.ex`, `lines.ex` | Finding constructors and `build/2`; a finding placed in source; line tables. |
| `lib/argus/souffle.ex`, `souffle/` | Running Soufflé, which must be on `PATH`. |
| `lib/argus/graph/` | The incremental query graph the driver runs. Internal. |
| `lib/argus/driver.ex`, `cli.ex`, `cli/`, `report*`, `lib/mix/tasks/argus.ex` | The run every frontend makes, option parsing, rendering, `mix argus`. |
| `priv/dl/stage0.dl` | The call graph, derived once per run into the facts dir. |
| `priv/dl/points_to.dl`, `points_to_bounded.dl` | Process points-to (pids and ETS tables), staged. |
| `priv/dl/clientlib/` | The shared rule library; `imports.dl` is the entry point. |
| `priv/dl/analyses/` | One program per built-in analysis. |

## Which built-in to copy

- The three layers, small: `priv/dl/analyses/structure.dl` with
  `lib/argus/analyses/structure.ex`. Each detection rule is a few lines
  named for the bug, and each word below it says what it means.
- Value flow through terms and calls (per-function summaries chained in
  Datalog): `lib/argus/extractors/pid_flow.ex` with
  `priv/dl/clientlib/processes.dl`.
- Taint-style "argument made from parameter": `extractors/param_flow.ex`.
- Forward must-dataflow over CFG edges (tests that hold on every path):
  `extractors/param_flow/bounded.ex`.
- Data and control dependence summaries: `extractors/dependence.ex`.

An analysis outside argus: see outside-analysis.md.
