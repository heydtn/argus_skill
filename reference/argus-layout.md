# Argus: where things are

Argus is published as the hex package `argus_beam` (renamed from
`panoptes` at 0.20; modules keep the `Argus` namespace). Its source and
docs are at `github.com/QuinnWilton/argus`, and paths below are relative
to that repo's root. A project that depends on it gets `lib/` and `priv/`
under `deps/argus_beam`, but not `docs/`; read those in a clone:

```sh
gh repo clone QuinnWilton/argus /tmp/argus_src -- --depth 1
```

## Docs

Look through all of `docs/`; these are the ones an analysis author needs
first.

| Path | What it is |
|---|---|
| `CLAUDE.md` | Layout and architecture: pipeline, schema, stages, query graph, frontends. Read first. |
| `docs/design/rule-style.md` | How a rule reads: report / detection / words layers, naming, "a refactor changes no finding". Short and binding. |
| `docs/bug-classes.md` | Every bug class argus reports (property, assumptions and limits, fixtures, precision) and the **Vocabulary**: every shared word, which way it errs (quiet/loud), and who reads it. The catalog a rule change is checked against. |
| `docs/design/{exclusions,monitor-leaks,races,restart-state,runs}.md` | Design notes for specific concerns. Their Soundness sections say what a narrowing must not excuse, pinned by `test/soundness/`. `exclusions.md` weighs each suppression against the findings it costs. How argus reasons about coverage against precision. |

## Code

| Path | What |
|---|---|
| `lib/argus/pipeline/` | Disassemble (via beam_spy), normalize, emit the generic bytecode facts. `emit.ex` (base facts), `emit/{applies,fun_refs,spawns}.ex` (resolved applies, funs handed to calls, spawns), `disassemble.ex` (`disassemble_path/1`: disassembly plus Line-chunk table), `writer.ex`. |
| `lib/argus/pipeline.ex` | `Argus.Pipeline.run/3` (extract to a facts dir), `extract/2` (in memory). Touches empty files only for argus's own schema relations. |
| `lib/argus/extractor.ex` | The `Argus.Extractor` behaviour. |
| `lib/argus/extractor/` | Shared extractor helpers: `ValueFlow`, `Helpers`, `CallSites`, `Resolve`, `Dispatch`, `Identity`, `Terms`, `Facts`, `Shapes`, `Runtime`, `GenStarts`. |
| `lib/argus/extractors/` | The domain extractors (about 30): `PidFlow`, `ParamFlow`, `Dependence`, `CallArgs`, `ClauseCall`, `ShutdownReason`, `StateGate`, `Purity`, `Specs`, `Generated`, `Tooling`, OTP/ETS/sockets/TLS/Phoenix/Ecto ones. |
| `lib/argus/cfg.ex`, `cfg/` | Basic-block CFGs with typed edges, dominators, post-dominators, loop headers, control dependence (`Cfg.Function`), all-paths walks (`Cfg.Walk`). |
| `lib/argus/dataflow.ex` | Reaching definitions (`reaching_uses/2`). |
| `lib/argus/instr.ex`, `instr/reaching.ex` | Instruction semantics (defs, uses, targets, copies) and per-instruction reaching sources. |
| `lib/argus/instr_id.ex` | IDs: `"Mod:func/arity"` (function), `"Mod:func/arity#idx"` (instruction). |
| `lib/argus/analysis.ex`, `analysis/` | The `Argus.Analysis` behaviour; `run_rules/3`, `derive_stage0/2`, `derive_points_to/2`; `Catalog` (discovery: **only `:argus_beam`'s modules**), `Extraction`. |
| `lib/argus/analyses/*.ex` | Built-in analyses (blocking, coupling, coverage, effects, ets, exposure, failure, mailbox, races, shutdown, startup, state_machine, structure, unsafe_input). Each pairs with `priv/dl/analyses/<name>.dl`. |
| `lib/argus/findings.ex`, `findings/` | Finding constructors (`new/4`, `related/3`, anchors), `build/2`, dedup, evidence. |
| `lib/argus/located.ex`, `lines.ex` | A finding placed in source; `Argus.Lines` (line tables from `line_info`, resolve, module declaration line). |
| `lib/argus/graph/` | The roux query graph the driver runs (placement: `graph/locate.ex`). Internal. |
| `lib/argus/driver.ex`, `cli.ex`, `cli/options.ex`, `report*` | The run every frontend makes; option parsing; rendering. |
| `lib/mix/tasks/argus.ex` | `mix argus` (compile, `Driver.run`, report, `--fail-above`). |
| `priv/dl/base.dl`, `layer2.dl` | Generated fact declarations (from `Argus.Schema`). |
| `priv/dl/stage0.dl` | The call graph, derived once per run into the facts dir. |
| `priv/dl/points_to.dl`, `points_to_bounded.dl` | Process points-to (pids and ETS tables), staged. |
| `priv/dl/clientlib/` | The shared rule library. `imports.dl` is the recommended entry point. |
| `priv/dl/analyses/` | One program per built-in analysis. |

## Which built-in to copy

- The three layers, small (187 lines): `priv/dl/analyses/structure.dl`
  with `lib/argus/analyses/structure.ex`. Each detection rule is two or
  three lines named for the bug, each word below it says what it means
  and why its joins are there.
- Value flow through terms and calls (per-function summaries chained in
  Datalog): `lib/argus/extractors/pid_flow.ex` with
  `priv/dl/clientlib/processes.dl`.
- Taint-style "argument made from parameter": `extractors/param_flow.ex`.
- Forward must-dataflow over CFG edges (tests that hold on every path):
  `extractors/param_flow/bounded.ex`.
- Data and control dependence summaries: `extractors/dependence.ex`.

## An analysis outside argus

Worked example: `github.com/heydtn/argus_nx_tensor_analyses`. Its
layout:

```
lib/<package>.ex                       # analyses/0: the Argus.Analysis modules
lib/<package>/<analysis>.ex            # Argus.Analysis: outputs, findings, runner, placement
lib/<package>/<analysis>/<extractor>.ex
lib/mix/tasks/<package>.ex             # `mix argus` plus these analyses
priv/<analysis>.dl                     # the rules
test/<package>/<analysis>_test.exs     # fixtures compiled into a temp dir
.github/workflows/ci.yml               # Soufflé 2.5, on x86
```

## Gaps an outside package works around

`Argus.Analysis.Catalog` discovers analyses only among `:argus_beam`'s
own modules, and `mix argus`'s internals are private, so an outside
package keeps its own runner, placement and Mix task (a copy of
`lib/mix/tasks/argus.ex`'s shape). A way to register analyses from
another application would let the driver run them, place their findings
and cache them like its own.
