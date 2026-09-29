# The Datalog side

## What a program starts from

Include `clientlib/imports.dl`: a built-in writes `.include
"../clientlib/imports.dl"`, and an outside package includes it from
`deps/argus_beam/priv/dl/` (see outside-analysis.md). It brings
`base.dl` + `layer2.dl` (every fact declaration), `priors.dl`, the
staged call graph, and the shared words (`reach.dl`, `closures.dl`,
`generated.dl`, ...). Relations a program never reads cost nothing at run
time, because Soufflé drops them before evaluating. They do cost compile
time: see Performance.

### Base facts (`priv/dl/base.dl`, always emitted)

IDs are strings: a function is `"Mod:func/arity"`, an instruction is
`"Mod:func/arity#idx"`.

```
function_def(func, mod, name, arity, exported)
function_entry(func, entry)
remote_call(id, caller, mod, func, arity)
local_call(id, caller, target, arity)
bif_call(id, caller, mod, func, arity, fail)
dynamic_call(id, caller, kind)            // call_fun, apply (and erlang:apply/2,3); no arity
resolved_apply(id, caller, target)        // an apply whose target the values name (context-insensitive)
closure_def(parent_func, closure_func)    // make_fun3 to a concrete MFA
fun_ref(caller, callee)                   // a literal fun handed to a call, not called
fun_handed(id, caller, callee, pos)       // the call a fun is handed to
tail_call(id)
conditional_call(id)                      // not on every completing path
spawn_call(id, caller, mod, func, arity, variant, api, source, param, args)
send_msg(id, caller)
recv_start(id, caller, blocking, fail)
try_start(id, caller, kind, handler)
literal_value(id, reg, val)               // a scalar a move writes
tuple_literal(id, reg, tag, size)         // a tuple with a literal atom head
type_test(id, test, src, fail)
line_info(id, line)                       // the source line in effect at an instruction
instruction(id, func, idx, op)  def(id, reg)  use(id, reg)  def_use(def_id, use_id)
next(from, to)  jump(id, target)  branch(id, fail, reserved)  select_branch(id, val, target)
label_at(label, id)  site_block(id, kind, block, idx)  block_flow(from, to)
```

`instruction`, `def`, `use` and `def_use` are positional and the bulk of
the facts: read them only when you need them.

### The call graph (stage 0, `priv/dl/stage0.dl`)

Derived once into the facts dir (`Argus.Analysis.derive_stage0/2`), and
read as inputs through `imports.dl`:

```
call_edge(caller, callee)                 // remote, local, BIF calls; closures built; funs handed on; resolved applies
call_site(id, caller, callee_mod, callee_func, arity)   // calls into project functions, and anchor APIs
unconditional_call_edge(caller, callee)
call_tag(id, caller, kind, tag)           // the message tag of a GenServer call/cast
fun_handed_to(caller, fun, callee)
```

`imports.dl` adds `call_instr(func, callee, id)` (the instruction behind
an edge into a project function), `in_module(func, mod)` (inline),
`call_reachable` (the full closure, quadratic, pruned unless read: do
not read it, seed a reach component instead), and `closure_count`.

A call of a protocol's function is a `call_site` to the protocol module.
Nothing shared resolves it to the program's implementations;
`priv/dl/analyses/unsafe_input.dl` detects `__impl__`/`__protocol__`
pairs locally.

### Shared words worth knowing (`priv/dl/clientlib/`)

The Vocabulary section of `docs/bug-classes.md` defines each one, says
which way it errs, and lists who reads it. The ones a non-process
analysis reaches for:

- **Reach components** (`reach.dl`): `.init r = CallReachSet`, seed with
  `r.seed(f) :- ...`, and read `r.reaches(f)`. Backward forms answer
  which functions reach a seed; forward ones, what a root reaches. The
  variants differ in their step: `CallReach`/`CallReachSet` (every
  `call_edge`), `SameProcessReach` (minus `runs_elsewhere`),
  `IntraModuleReach`, the `…Cut` forms, `HoldingReach`, `ParamReach`,
  the bounded forms, `ForwardUnguardedSet`/`BackwardUnguarded`. Seeds
  can depend on `reaches` (recursion through the instance is fine), so
  calls the graph does not see become extra seed rules.
- `closures.dl`: `enclosing_function(func, owner)`, `sole_closure`,
  `handed_closure(site, func, closure)`.
- `generated.dl`: `program_module(mod)`, `library_written(func)`.
- `calls.dl`: `resolved_arg(func, pos, value)` (a literal some caller
  passes, merged over callers) and the process dependency words.
- `test_code.dl` / `tooling.dl`: code only tests or developer tools run.
- `specs.dl`: `callee_returns(f, shape)`.
- Process points-to (`processes.dl`, staged by `points_to.dl`, read via
  `staged_processes.dl`): processes and ETS tables only,
  context-insensitive. A program that reads any of
  `Argus.Analysis.points_to_relations/0` gets the stage derived.

A clash check before including: Soufflé has no namespaces, so a
relation you declare with a name `imports.dl` also declares is an error
(relations inside `.comp` bodies are per instance and don't clash).

## Rule style (`docs/design/rule-style.md`)

1. **Report** (`.output`): joins the detection relation with what a
   finding shows. Its name and columns are an ABI; no detection logic.
2. **Detection rule**: named for the bug as a noun; three to seven lines,
   one idea each, in the order you'd say it; only domain words, no
   extractor facts, arithmetic or string comparisons of kinds; negations
   read as sentences; variables named for what they are; one comment
   above: the bug in a sentence.
3. **Words**: each atom a relation with a one-sentence comment, then its
   assumptions and limits. Extractor facts, key comparisons and precision
   filters live here. A word two analyses use moves to `clientlib/`.

Predicates are verb phrases, subject first (`fails_if_row_missing(use,
func)`). A new word may not reuse a name at another arity. A refactor
changes no finding, field for field. A precision change is its own commit,
with the findings it moves listed.

## Soufflé traps (2.5)

- **Soufflé 2.4 and 2.5 differ on `_`.** 2.4 rejects `_` inside a
  destructured record or ADT (`value = $Object(_, _, _)`, `arguments =
  [value, _]`) as "Ungrounded" (fixed in 2.5 by
  souffle-lang/souffle#2483). Argus's CI installs 2.4 from the Ubuntu
  PPA, so a built-in names those variables (`[value, _rest]`: a name
  starting with `_` draws no singleton warning). A package outside argus
  can require 2.5 instead.
- **Conditions are reordered.** A guard does not protect arithmetic in
  the same rule, nor in a recursive rule's head (the head tuple is part
  of the not-yet-derived test). Divide by `max(d, 1)` where the rule
  requires `d >= 1`. An integer division by 0 traps (SIGFPE) on x86 and
  gives 0 on ARM, so it only fails on CI. Audit the compiled program:
  `souffle --show=transformed-ram ... | grep -v DEBUG | grep -E '[/%](t[0-9]+\.[0-9]+|\()'`
  should print nothing.
- **Plans move atoms ahead of guards too.** A `.plan` that puts
  `path_steps(path, ...)` (which does `to_number(substr(path, ...))`)
  before the `match` that guards it crashes on `""`. Leave such a rule
  unplanned, or make the functor total.
- Reserved words cannot name variables or relations: `count`, `sum`,
  `min`, `max`, `mean`, `output`, `input`, `choice`.
- Inline relations (`.decl r(...) inline`) work only when used positively.
- No negation or aggregate over a relation in the same recursive SCC.
- An aggregate over ADT wildcard patterns crashes the interpreter
  ("variable not grounded").
- Provenance (`-t explain`) crashes on ADTs; debug by profile instead.
- OpenMP (`-j`) was slower than single-threaded at every thread count on
  a latency-bound fixpoint; measure before using it.

## Performance

**Demand comes first.** Derive a relation only for what the detection
rules ask about (SKILL.md, "CRITICAL: make rules demand-driven"): seed
walks from the sites the bug names, give a shared word a demand relation
its consumers seed (`calls.dl`'s `site_demand`), and stage only what the
analyses read. The levers below make a fixpoint cheaper; demand decides
how many rows it derives at all.

**Delta-first join order is the lever inside a recursion.** In semi-naive evaluation every
recursive rule runs once per iteration per version, one version per
recursive atom. A version that reads a large relation before its delta
rescans it every iteration. Lead each recursive rule with the atom whose
new facts should fire it, and `.plan` each other version from its own
delta atom: `.plan 1:(2,1,3), 2:(3,1,2)` (version numbers count
recursive atoms from 0; the tuple is the body order, atoms from 1). This
took a 900-iteration fixpoint from 12.8s to 4.3s with identical output.

Find the hot versions with the profiler:

```sh
souffle -p prof.log -F facts -D out program.dl
```

`prof.log` is JSON: `root.program.relation[name].iteration[i]
["recursive-rule"][rule][version]` has `runtime` (start/end, µs),
`num-tuples` and `atom-frequency`, whose key is the evaluated clause
with the delta atom spelled `@delta_...`. Rank versions by runtime and
fix those whose delta atom is not first. `souffle --show=scc-graph-text`
lists the SCCs.

**Disjunctions multiply.** Soufflé expands a rule with k alternatives
(`( a ; b ; c )`) into k rules, each with the whole body: compile time
and the join both grow k times. Test a condition on a few columns with a
small relation instead (argus's `check_then_act.dl` has `may_agree`).

**Measure cold runs with `ARGUS_NO_CACHE=1`.** In argus's repo, the suite
and the driver keep facts and solves in a blob store, so a second run
measures the cache. `ARGUS_NO_CACHE=1` gives every run a temporary
store.

**The fixed cost is Soufflé preparing the program**, not the data.
Solve over empty facts to measure it (same `.facts` files, all empty).
Two passes grow with relations × program size: `SemanticChecker` and
`SubsumptionQualifierTransformer` each scan the whole program once per
relation. So every declared relation costs compile time, including the
unused ones `imports.dl` brings in (hundreds). Compiled mode (`souffle
-o`) removes the preparation from each run, but builds one large C++
translation unit on one core, which takes a long time for a large
program. Its dominant file is the main recursive stratum, so `-G` plus a
parallel build helps only partly.

**`run_rules/3` defaults are costly for a custom program**: without
`stage0: :provided` it compiles the whole program once just to learn
whether it reads points-to. Derive stage 0 yourself
(`derive_stage0/2`), then pass `stage0: :provided`.

**Keep coverage in mind.** Lowering a depth, dropping cases or narrowing
a word buys speed with findings. That can be the right trade, but make
it knowingly, and say which way the change errs (quiet or loud, in
`docs/bug-classes.md`'s terms). Argus's own default is quiet: a fact
that cannot be sure says `"dynamic"`, and rules ask what is not handled.

**Bound work by the facts, not the clock.** A result must be a function
of the facts: cap a relation with `.limitsize` (Soufflé stops there
however fast it runs, as `points_to.dl` does), never with a timeout.

**A fixpoint several analyses need is a stage, not an include.** An
include recomputes it in every solve that reads it; a stage (like
`stage0.dl` and `points_to.dl`) derives it once into the facts, and
writes only what the analyses read. Measure first: removing a bounds walk
that looked costly saved 0.1s of 13s. The iteration count and the join
order were what cost.

## Checking a change changes nothing

Before a refactor, snapshot every output relation plus the value
relations that feed findings (`.output` them in a probe program that
includes the rules), sorted, and the extracted facts. After it, diff row
for row, and run the tests. Keep the probe out of the package (under
`_build/`).
