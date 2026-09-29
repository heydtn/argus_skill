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
  files only for its own schema relations. An outside runner must
  create empty files for the rest, or Soufflé fails on the missing
  input.
- Rows are lists of strings. Numbers are spelled (`"3"`), and so are
  positions: PidFlow keeps positions as symbols so rules join them
  without `to_number`.
- Name things with `Argus.InstrId`: `func_id(module, name, arity)` gives
  `"Mod:func/arity"`, and `mint(func_id, idx)` gives `"Mod:func/arity#idx"`
  (idx indexes the raw `beam_disasm` stream, labels and lines included).
- Accumulate with `Argus.Extractor.Facts.add_fact(facts, relation, row)`,
  then `Enum.uniq/1` and `Enum.sort/1` each relation. Argus requires the
  same modules to give `==` facts in any VM, and a map or set holding
  atoms iterates in atom-table order, which differs between VMs: sort
  what you build from one, and spell literals with `Terms.spell/1`,
  which sorts map keys.

`module_data` (see `@type module_data` in `lib/argus/extractor.ex`) has
`:module`, `:exports`, `:attributes` and `:functions` (a list of
`{:function, name, arity, entry_label, instructions}`). When the pipeline
calls you it can also carry `:call_sites`, `:cfg`, `:typed`, `:reaching`,
`:line_table`, `:beam`, `:imports`, `:origins_index`, `:debug_info`.
Always read these through the `Helpers` accessors below, which build
them when absent (unit tests on bare disassembly).

## The shared helpers (`lib/argus/extractor/`, `lib/argus/`)

### Value flow in one function: `Argus.Extractor.ValueFlow`

The engine PidFlow, ParamFlow and Dependence share: a value per register
write, solved to a fixpoint over reaching definitions. You say what an
instruction writes; it re-evaluates only instructions whose reads changed.

- `reads_by_function(Helpers.reaching(module_data))` gives
  `%{func_id => %{idx => %{"x0" => [{:param, k} | {:def, idx}]}}}`.
- `solve(idxs, reads, state, evaluate, opts)` returns `{outs, state}`.
  `evaluate.(idx, outs, state)` returns `{[{"x0", value}, ...], state,
  also_idxs}`. Values join as sets; keep `evaluate` monotone.
  `:max_evaluations` (default 64) guards bugs.
- `inputs(reads, outs, idx, &param_value/1)` gives `%{reg => MapSet}`:
  what each register an instruction reads may hold, with parameter `k`
  as `param_value.(k)`. Use it, not a hand-written join.

### Reading a module: `Argus.Extractor.Helpers`

- `reaching(module_data)`: the reaching definitions (with parameters),
  or `nil`.
- `cfg(module_data, name, arity)`: the function's `Argus.Cfg.Function`,
  or `nil`.
- `typed(module_data)`: the decoded Layer-1 facts, by relation.
- `each_remote_call/3`, `each_call/3`, `scan_functions/4`,
  `match_remote_call/1`, `match_local_call/1`, `find_function/3`,
  `instructions_from_label/2`, `get_behaviours/1`, `attribute_values/2`,
  `debug_info/1`, `copies/1` and `copy_read/2` (copy instructions,
  registers spelled as facts spell them), `register/1`.
- Call sites: `Argus.Extractor.CallSites.for_module(module_data)`
  indexes every call once, as
  `%{func_id:, instrs:, idx:, mfa: {mod, fun, arity}, remote?:}`, local
  and remote calls alike. Group by `func_id` rather than scanning
  instructions. `call_fun`/`apply` instructions are not in it.

### Control flow: `Argus.Cfg.Function` (from `Helpers.cfg/3`)

Fields: `entry`, `blocks` (id to `%Argus.Cfg.Block{range: {first, last},
label, preds, succs, terminator}`), `rpo`, `idom`, `dom_children`,
`ipdom`, `loop_headers`, `labels` (label to block id), `selects`.
Functions: `block_at/2`, `dominates?/3`, `postdominates?/3`,
`precedes?/3`, `forward_succs/2`, `control_deps/1`,
`completing_blocks/2`, `region/2`.

Edges carry their kind: `:fallthrough`, `:jump`, `:branch_pass` (a
test's fall-through), `:branch_fail` (its fail label), `{:select_arm,
value}`, `:select_fail`, `:exception`. A test ends its block, so what
holds on a block's entry holds at every instruction in it.

**Pattern: a forward must-analysis over edges** (what holds on every
path, like the literal tests a value passed). Seed the entry block,
compute each edge's out-state from the block's last instruction and the
edge kind, meet (intersect) at the target, and requeue a target only
when its state shrinks. `Argus.Extractors.ParamFlow.Bounded` is the
model; `ArgusNxTensorAnalyses.TensorShapes.ShapeFlow.path_guards/2` is a
compact one.

`Argus.Cfg.Walk.explore(fun, instrs, starts, opts)` walks every path from
instruction indices (a `follow?` per edge, an `on_instr` per instruction)
and halts on a verdict.

### Instructions: `Argus.Instr`

Argus reads the instruction set one way, here: the emitter's facts,
`Argus.Cfg` and every register walk come from it. Keep no instruction
table of your own; where an extractor reads more than `Instr` says (the
term an instruction builds, which operand of a `call_fun` is the
callee), it says why in a comment.

`defs/1`, `uses/1`, `targets/1`, `falls_through?/1`, `exits?/1`,
`call?/1`, `tail_call?/1`, `defines?/2`, `clobbers?/2`, `known?/1`,
`register/1` (drops a type annotation).

`copy_source(instr, reg)` is the operand a `move`, `fmove`, `swap` or
`trim` copied into `reg`, else `nil`; the empty list comes back as
`{:literal, []}`. One clause over it replaces per-opcode copy handling.
`carry(instr, holding)` gives the registers holding a value after an
instruction, for forward register walks.

### What a register holds, statically: `Argus.Extractor.Resolve`

Backward through the writes that reach a read, within one function:
`resolve_register/3` (`{:ok, value}`), `resolve_atom/3` (inspected, or
`"dynamic"`), `value_at/3`, `module_target/3`, `arg_position/3` (is it
still a parameter), `map_field_of/3`, `call_result_origin/3`,
`fun_origin/3` and `fun_target/3` (which fun), `list_length/3`,
`keyword_value_register/4`, `writers/3`, `access_paths/4`, `trace/5`,
`timeout_ms/3`, `node_list/3`.

### Other helpers

- `Argus.Extractor.Terms`: `spell/1` (how a fact column spells a term;
  the empty list is `"[]"`), `proper_list?/1`, `list_elements/1`,
  `mentions?/2`, `value_contains?/2`. No helper turns an instruction
  operand (`{:integer, n}`, `{:atom, a}`, `{:literal, t}`, `nil`) into a
  term; extractors unwrap their own.
- `Argus.Extractor.Dispatch`: clause selection. `argument_tags/2` (tags
  paths establish on the first argument), `reached_with/4` and
  `reached_holding/3` (every path with a parameter fixed to an **atom**),
  `compared_atoms/2`, `total?/1`, `total_on?/2`, `labels/1`. It handles
  atoms and parameters only.
- `Argus.Extractor.Identity`: `key_identity/4` and
  `tuple_element_identity/5` (what names a value, for joining two sites).
- `Argus.Extractor.Shapes.return_shapes/1`: the tuples a function returns.
- `Argus.Extractor.Runtime.module?/1`: is the module ERTS, Kernel,
  STDLIB, Elixir or Logger (no summaries to follow).
- `Argus.Pipeline.Disassemble.disassemble_path/1`: the disassembly plus
  the Line-chunk table (`:line_table`) and the beam.
  `Disassemble.marker_line(marker, line_table)` is the line a `line` or
  `debug_line` marker sets, whichever form the disassembler gave it.
- `Argus.Dataflow.reaching_uses/2`, `def_use_edges/1`: reaching
  definitions over Layer-1 facts.
- `Argus.Lines`: `from_facts/1`, `from_facts_dir/1`, `resolve/2`
  (instruction, then its function's first line), `declaration_line/1`.

## What argus's own extractors already emit

Check these before writing a fact of your own; they run only when an
analysis names them in `extractors/0`. Mind what each word *means*: most
are context-insensitive, or answer a narrower question.

| Extractor | Emits | Mind |
|---|---|---|
| `PidFlow` | Per-function value summaries (`pid_arg`, `pid_return`, `pid_object`/`pid_field`/`pid_base`/`pid_sets`/`pid_load`, ...) chained by `clientlib/processes.dl` into points-to. | Only terms that hold a pid or ETS table; parameters merged over callers (Andersen). Not a general value flow. |
| `ParamFlow` | `call_arg_derived` (argument made from a parameter), sink bounds (`sink_arg_bounded`). `Propagators` lists library calls whose result carries an argument's data. | Taint, "made from": `Enum.map`'s result is made from its list. Wrong for identity of values. |
| `Dependence` | Data and control dependence of calls, returns and sites (`site_depends`, `call_arg_depends`, `returns_depends`, `field_decides`, ...). | Per function, for check-then-act races. |
| `CallArgs` | `call_arg` (literal argument), `call_arg_forward` (a parameter passed on), `call_arg_field`, `call_arg_element`, `call_arg_tuple`, `mfa_arg`. | Per (caller, callee, position), merged over sites. `clientlib/calls.dl`'s `resolved_arg` chains them. |
| `ClauseCall` | `clause_call(id, func, tag)`: the first-argument tag paths to a call establish. | Atoms (or tuples headed by one), first argument. |
| `ShutdownReason`, `StateGate` | Paths with a parameter fixed to `:shutdown`; handlers gated on a state field. | Atoms. |
| `Purity` | `impure_call`, `pure_contract`, `unknown_call`, `protocol_dispatch`, `dynamic_call` (`dot_dispatch`). | `protocol_dispatch` covers a fixed list of library protocols, not the program's own. |
| `Specs` | `callee_returns(f, shape)`. | A spec's claim, never verified. |
| `Generated`, `Tooling` | `macro_written`/`macro_generated`; developer-tool and test modules. | `library_written` hides code another package's macro wrote; `defn` bodies may count. |

The base emitter (no extractor needed) gives the call, closure, apply and
line facts: see `datalog.md`.
