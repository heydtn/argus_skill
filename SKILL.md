---
name: argus-analysis
description: Build, extend, speed up or debug an Argus (hex package `argus_beam`) analysis — a Soufflé Datalog program over facts extracted from compiled BEAM modules, with an Elixir extractor and findings. Use for a built-in analysis in argus's repo, a new extractor or shared word, rule or performance work, finding placement, or an analysis package outside argus (like `argus_nx_tensor_analyses`).
---

# Building an Argus analysis

Argus disassembles `.beam` files, emits Datalog facts from the bytecode,
and solves Datalog programs over them. An analysis is three things:

1. **An extractor** (Elixir, `@behaviour Argus.Extractor`): per module,
   the facts the base emitter does not give you.
2. **A Datalog program**: the bug as rules over argus's facts, its
   shared words (`priv/dl/clientlib/`) and your extractor's facts.
3. **An analysis module** (`@behaviour Argus.Analysis`): declares the
   output relations and turns their rows into findings.

Written against argus 0.20.1 (Elixir 1.19 or later) and Soufflé 2.5. The
doc pointers follow argus's `main`, which reorganized its docs after
0.20.1. Argus's source is the authority where they disagree: its repo, or
`deps/argus_beam` in a project that depends on it.

## CRITICAL: make rules demand-driven

One of the most important rules here. Derive a relation only for the
rows a detection rule reads, never for the whole program and then
filtered: a walk over everything grows with the program (a closure with
its square), one over what the bug asks about with the few things that
matter.

- **Seed walks from what the bug names** (its sinks, handlers, sources,
  sites) through the seeded shared words: a reach component's `seed`,
  `RunsAfter`'s and `escape.dl`'s `asked`. Never read `call_reachable`,
  the full, quadratic closure (its comment in `imports.dl`); no built-in
  does.
- **A word that answers "for any f" takes a demand relation** its
  consumers seed and each of its rules starts from. The model is
  `calls.dl`'s `site_demand`: `blocking.dl` seeds it with
  `site_demand(h) :- handle_call_function(_, h).`, and it grows only
  along callees a demanded function enters (`literal_entry`). A stage
  writes only what the analyses ask about (`points_to.dl`'s
  `source_process`).
- **Join the demand before the recursion.** Filtering a fixpoint's
  output pays its whole cost.
- **Write the demand by hand**, named for what the rules ask, rather
  than with Soufflé's magic sets (`-m`).
- **Where demand cannot reach, say why.** A word a walk negates cannot
  take its demand from that walk (no negation within a recursive SCC;
  `calls.dl`'s `literal_first`): keep it non-recursive and cheap, and
  comment why.
- **Demand changes cost, never findings.** Seeding too little loses
  coverage ("Keep coverage in mind", below); check by identity
  (Workflow, step 7).

## IMPORTANT: write rules top down

Next to demand, the most important rule. Write the detection rule as
the bug's high-level concept (rule-style.md) and get more specific only
by drilling into shared predicates, each defined in a few more specific
words; only the bottom level reads extractor facts, keys and string
tests. From `races.dl`:

```prolog
// The bug, in its words.
missing_row_race(func, check, use, remove, row) :-
  checks_row_exists(check),
  decides(func, check, use, row),
  fails_if_row_missing(use, func),
  on_shared_table(use),
  removes_row(remove, row),
  runs_in_another_process(remove, func).

// One level down.
removes_row(remove, [table, ks, k]) :-
  removes_rows(remove, table),
  !keyed_apart(remove, ks, k).

// The bottom: extractor facts.
removes_rows(d, [tk, t]) :-
  ets_op(d, _, _, op, _),
  ets_removal_op(op),
  ets_table(d, tk, t).
```

- **Top first**, naming words not yet written; then each word; then
  theirs. Built bottom up, a rule ends as the facts joined in one body.
- **Reuse a word at every level** (`clientlib/`, the analysis's, a reach
  component) instead of re-deriving it (Workflow, step 2). One two
  analyses need moves to `clientlib/`.
- **One level per body.** Unrelated conditions, extractor details above
  the bottom level, a comment explaining a join, or a domain word beside
  a fact test mean a word is missing. A word is a concept of the bug,
  never a way to shorten a rule: every relation costs compile time
  ("Compile time", below).
- **Start each relation's comment with what it means**, then only the
  assumptions, unknowns or implementation reason a reader needs to use
  it, so a reader can stop at any level.
- **Demand flows down**: a costly word takes what the level above asks
  about as its demand (`races.dl`: `runs.seed(h) :- remover_in(_, h).`).

## Read first

- Argus's `AGENTS.md`: setup and checks, where to edit, and the
  correctness, cache and test constraints.
- `reference/argus-layout.md`: where argus's code and docs are, and which
  built-in analysis to copy.
- `reference/extractors.md`: the extractor contract and the shared
  helpers. Use them; do not re-walk instructions or rebuild graphs.
- `reference/datalog.md`: what a program starts from, the shared words,
  the rule style, and Soufflé's traps and performance levers.
- `reference/outside-analysis.md`: an analysis argus does not ship:
  runner, caching, placement, Mix task, CI.

## Workflow

1. **Read the docs** (argus-layout.md says where). You **MUST** read
   `docs/design/rule-style.md` and `docs/design/analysis-model.md` (the
   shared model, with the table of which reach component answers what),
   and the `docs/analyses/` guide of any analysis whose words you reuse.
   The comment beside each clientlib relation is the reference for what
   it means.
2. **Look for the word before writing it.** Before deriving any relation,
   search `priv/dl/base.dl`, `layer2.dl`, `stage0.dl` and `clientlib/`
   for it, and `lib/argus/extractor/` and `lib/argus/extractors/` for
   the Elixir side. Most "new" call-graph, reach, closure and module
   concepts already exist. Check what a word means (context-insensitive?
   atoms only? taint "made from" vs identity?) before reusing it; a word
   with the wrong meaning loses coverage or adds false findings.
3. **Write the extractor** on `Argus.Extractor.ValueFlow`, `Helpers.cfg/3`,
   `CallSites`, `Instr`, `Facts`, `Terms` (extractors reference).
   Declare every relation in `relations/0`.
4. **Write the rules**, demand-driven and top down (above), in
   rule-style.md's three layers: a report relation (`.output`, an
   interface), a detection relation stating the bug in its domain's
   words, and the supporting relations below it.
5. **Write the analysis module.**
   - A built-in (in argus's repo): the module in `lib/argus/analyses/`,
     its program in `priv/dl/analyses/` (from `.include
     "../clientlib/imports.dl"`), its guide in `docs/analyses/` (linked
     from `docs/bug-classes.md`), and a corpus pair
     (`test/corpus/pairs.exs`) for a new bug class. `AGENTS.md` says
     what else a change touches (schema, pins, typed readers) and which
     checks it runs.
   - Outside argus: the runner and Mix task too (outside-analysis
     reference). Argus's driver runs only analyses in `:argus_beam`, and
     its analyses are about processes, supervision, shared state and
     external input (`docs/bug-classes.md`); a library's own bugs (Nx's,
     say) belong in a package outside it.
6. **Test.** In argus's repo, follow `AGENTS.md`'s test conventions
   (`Argus.Test.Memo`, `Argus.Test.Batch` with `ARGUS_VERIFY_BATCH=1`,
   `Argus.Test.Peer`). Outside argus, compile fixtures into a temp dir
   (`Kernel.ParallelCompiler.compile_to_path/3`), solve them once in
   `setup_all`, and assert findings and their placed lines.
7. **Verify refactors by identity.** Before a change, snapshot the
   extracted facts, every output relation and the value relations that
   feed findings (`.output` them from a probe program that includes the
   rules, kept under `_build/`), sorted; after it, diff row for row.
   Passing tests are not enough: a refactor changes no finding
   (rule-style.md). In argus's repo, add the fixture, soundness and
   corpus comparisons `AGENTS.md` lists.
8. **CI.** Argus's CI installs Soufflé 2.4 from the Ubuntu PPA, so a
   built-in's rules must compile under 2.4; a package outside argus can
   run 2.5 (outside-analysis reference). On both, name every `_` inside
   a record or ADT pattern (`[value, _rest]`): 2.4 rejects a bare one as
   "Ungrounded", and 2.5's inliner silently drops rows over one
   (datalog.md, Soufflé traps). Run CI on x86, where an unguarded
   division traps (below).

## Argus's own principles

From argus's `AGENTS.md` and `docs/design/`; they hold for any analysis
built on it.

- **Extraction is deterministic.** The same modules give `==` facts in
  any VM. A map or set holding atoms iterates in atom-table order, so
  spell a literal column with `Argus.Extractor.Terms.spell/1` (it sorts
  map keys) and sort rows built from such a set.
- **One reading of the instruction set.** `Argus.Instr` says what every
  instruction reads and writes and where control goes. A new walk keeps
  no instruction table of its own: backward through `Argus.Instr.Reaching`
  or `Argus.Extractor.Resolve`, forward with `Argus.Instr.carry/2`. Read
  more than it says only where you know more, and say why in a comment.
- **An absent relation file is never an empty relation.** Every writer
  leaves a file per relation, empty when it has no rows. A reader that
  cannot open one raises `Argus.MissingRelationError`; it never turns
  `{:error, _}` into `[]`.
- **A result is a function of the facts, never of time.** Bound work
  with a budget Soufflé enforces (`.limitsize`, as `points_to.dl` does),
  not a timeout.
- **Keep uncertainty explicit.** An extractor spells what it cannot
  resolve `"dynamic"`. An unread option, owner or caller is unknown,
  never a default: it is not evidence of safety or exclusivity, nor of
  the defect. A suppression needs evidence about the operation it excuses,
  not a similar one elsewhere in the module, and keeps the nearest defect
  it must still report as a counterexample (rule-style.md,
  "Suppressions"). A missing prior is not negative evidence.

## Rules that bit before

- **Keep coverage in mind.** A speedup that lowers a depth, drops cases
  or narrows a word changes what the analysis finds. Know what a change
  costs in findings and make the trade on purpose, whichever way it
  goes. Each analysis's limits are in its `docs/analyses/` guide, and
  `test/soundness/` and `test/exclusions/` keep the defects a narrowing
  must still report. Measure before assuming what is slow.
- **Soufflé reorders a rule's conditions.** A guard does not protect a
  division: write `x / max(d, 1)` where the rule requires `d >= 1`. An
  integer division by 0 traps on x86 and silently gives 0 on ARM Macs,
  so local tests will not catch it.
- **A `.plan` can move a functor ahead of its guard.** `to_number(substr(...))`
  behind a `match(...)` crashes on `""` once a plan reorders the atoms.
  Leave such rules unplanned, or make the functor total.
- **`to_float` and `to_number` on an out-of-range spelling abort the
  whole solve.** Floats are 32-bit, so an Elixir literal like `1.0e-40`
  does it. Guard with a regex that admits only the safe range, and call
  the functor in a non-inline relation's head (datalog.md, Soufflé traps).
- **Recursive rules must start from their delta.** The biggest speed
  lever after demand: lead with the recursive atom whose new facts should
  fire the rule, and `.plan` the other versions from their own delta
  atom. This took one analysis from 12.8s to 4.3s with identical output.
- **Compile time grows with relations × program size.** Every declared
  relation costs, used or not; `inline`, components and `-j` don't help.
  Share predicates and merge relations instead, and never disable the
  two passes behind that cost, `SubsumptionQualifierTransformer` and
  `SemanticChecker`: argus relies on the first for subsumptive (`<=`)
  clauses, and without the second a broken program segfaults or solves
  silently wrong (datalog.md, Performance).
- **Near-identical clauses in one relation blow up compile time** (41s,
  from inlining). Materialize the shared part over a demand relation,
  and write a family of similar rules as a fact table plus one rule
  (datalog.md, Performance).
- **Pass `stage0: :provided` to `run_rules/3` after `derive_stage0/2`.**
  Otherwise it compiles your whole program just to learn whether it reads
  process points-to (seconds). It then derives no points-to either: a
  program that reads it calls `derive_points_to/2` first.
