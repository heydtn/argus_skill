# Extractors

## The contract

```elixir
defmodule MyAnalysis.MyFlow do
  @behaviour Argus.Extractor

  @impl true
  def relations, do: [:my_relation, :my_other_relation]

  @impl true
  def extract(module_data) do
    # %{relation_atom => [[string, ...], ...]}: every column a string
    %{my_relation: [["Mod:f/1#12", "0"]]}
  end
end
```

- `relations/0` lists every relation `extract/1` can emit. Argus writes
  a relation's `.facts` file only when it has rows, and touches empty
  files only for its own schema relations: an outside runner creates
  empty files for the rest, or Soufflé fails on the missing input.
- Rows are lists of strings. Numbers are spelled (`"3"`), and so are
  positions: PidFlow keeps positions as symbols so rules join them
  without `to_number`.
- Name things with `Argus.InstrId`: `func_id(module, name, arity)` gives
  `"Mod:func/arity"`, and `mint(func_id, idx)` gives `"Mod:func/arity#idx"`
  (idx indexes the raw `beam_disasm` stream, labels and lines included).
- Accumulate with `Argus.Extractor.Facts.add_fact/3`, then `Enum.uniq/1`
  and `Enum.sort/1` each relation, so facts stay deterministic (SKILL.md,
  principles).

`module_data` (`@type module_data` in `lib/argus/extractor.ex`) has
`:module`, `:exports`, `:attributes` and `:functions` (a list of
`{:function, name, arity, entry_label, instructions}`). When the pipeline
calls you it carries more (`:call_sites`, `:cfg`, `:typed`, `:reaching`,
`:line_table`, ...). Read those through the `Helpers` accessors, which
build them when absent (unit tests on bare disassembly).

## The shared helpers

Each has a moduledoc: read it before use, and use the helper rather than
re-walking instructions.

| Module | For |
|---|---|
| `Argus.Extractor.ValueFlow` | A value per register write, solved to a fixpoint over reaching definitions: the engine PidFlow, ParamFlow and Dependence share. `solve/5` takes what each instruction writes (keep it monotone); `inputs/4` is what each register an instruction reads may hold. |
| `Argus.Extractor.Helpers` | Reading a module: `reaching/1`, `cfg/3`, `typed/1`, call scans, attributes, copies. |
| `Argus.Extractor.CallSites` | `for_module/1` indexes every local and remote call once; group by `func_id` rather than scanning. `call_fun` and `apply` are not in it. |
| `Argus.Cfg.Function` (from `Helpers.cfg/3`) | Basic blocks with typed edges, dominators, post-dominators, loop headers, control dependence. `Argus.Cfg.Walk.explore/4` walks every path to a verdict. |
| `Argus.Instr` | What every instruction reads and writes and where control goes. `copy_source/2` replaces per-opcode copy handling; `carry/2` steps a forward register walk. |
| `Argus.Extractor.Resolve` | What a register holds, backward within one function: `resolve_register/3`, `resolve_atom/3` (inspected, or `"dynamic"`), which fun or module, `trace/5`, ... |
| `Argus.Extractor.Terms` | `spell/1`, how a fact column spells a term (the empty list is `"[]"`). No helper turns an instruction operand into a term; extractors unwrap their own. |
| `Argus.Extractor.Dispatch` | Clause selection: the tags paths establish on an argument, the paths with a parameter fixed to an atom. Atoms and parameters only. |
| `Argus.Extractor.Identity` | What names a value, for joining two sites. |
| `Argus.Extractor.Shapes` | The tuples a function returns. |
| `Argus.Extractor.Runtime` | `module?/1`: is the module ERTS, Kernel, STDLIB, Elixir or Logger (no summaries to follow). |
| `Argus.Pipeline.Disassemble` | `disassemble_path/1` (the disassembly, Line-chunk table and beam); `marker_line/2`, the line a `line` or `debug_line` marker sets. |
| `Argus.Lines` | Line tables from `line_info` facts, and `resolve/2`. |

A test ends its block, and edges carry their kind (`:branch_pass`,
`:branch_fail`, `{:select_arm, value}`, `:exception`, ...), so what holds
on a block's entry holds at every instruction in it.

**Pattern: a forward must-analysis over edges** (what holds on every
path, like the literal tests a value passed). Seed the entry block,
compute each edge's out-state from the block's last instruction and the
edge kind, meet (intersect) at the target, and requeue a target only
when its state shrinks. `Argus.Extractors.ParamFlow.Bounded` is the
model; `ArgusNxTensorAnalyses.TensorShapes.ShapeFlow.path_guards/2` is a
compact one.

## What argus's own extractors already emit

Check these before writing a fact of your own; they run only when an
analysis names them in `extractors/0`. Mind what each word *means*: most
are context-insensitive, or answer a narrower question.

- `PidFlow`: per-function value summaries (`pid_arg`, `pid_return`, ...)
  that `clientlib/processes.dl` chains into points-to. Only terms that
  hold a pid or ETS table, parameters merged over callers: not a general
  value flow.
- `ParamFlow`: `call_arg_derived` (an argument made from a parameter)
  and sink bounds (`sink_arg_bounded`). Taint, "made from":
  `Enum.map`'s result is made from its list, so it is wrong for value
  identity. `Propagators` lists the library calls whose result carries
  an argument's data.
- `Dependence`: data and control dependence of calls, returns and sites
  (`site_depends`, ...), per function, for check-then-act races.
- `CallArgs`: `call_arg` (a literal argument), `call_arg_forward` (a
  parameter passed on), `mfa_arg`, ..., per (caller, callee, position)
  and merged over sites; `clientlib/calls.dl`'s `resolved_arg` chains
  them.
- `ClauseCall`: `clause_call(id, func, tag)`, the first-argument tag
  paths to a call establish; atoms, or tuples headed by one.
- `ShutdownReason`, `StateGate`: paths with a parameter fixed to
  `:shutdown`; handlers gated on a state field. Atoms.
- `Purity`: `impure_call`, `pure_contract`, `unknown_call`,
  `protocol_dispatch` (a fixed list of library protocols, not the
  program's own).
- `Specs`: `callee_returns(f, shape)`, a spec's claim, never verified.
- `Generated`, `Tooling`: `macro_written`/`macro_generated`, and
  developer-tool and test modules. `library_written` hides code another
  package's macro wrote; `defn` bodies may count.

The base emitter (no extractor needed) gives the call, closure, apply and
line facts: see `datalog.md`.
