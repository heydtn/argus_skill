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

Written against argus 0.20.1 (Elixir 1.19 or later) and Soufflé 2.5.
Argus's source is the authority where they disagree: its repo, or
`deps/argus_beam` in a project that depends on it.

## Read first

- Argus's `CLAUDE.md` (in its repo, where Claude Code loads it on its
  own): the layout, the design principles, the schema, the dev loop, how
  tests solve, the corpus.
- `reference/argus-layout.md`: where argus's code and docs are, and which
  built-in analysis to copy.
- `reference/extractors.md`: the extractor contract and the shared
  helpers. Use them; do not re-walk instructions or rebuild graphs.
- `reference/datalog.md`: the base facts, the call graph, the shared
  words, the rule style, and Soufflé's traps and performance levers.
- `reference/outside-analysis.md`: an analysis argus does not ship:
  runner, caching, placement, Mix task, CI.

## Workflow

1. **Read the docs.** In argus's repo they are in `docs/`. A project
   that depends on argus has only `lib/` and `priv/` (under
   `deps/argus_beam`), so clone the repo for them:
   `gh repo clone QuinnWilton/argus /tmp/argus_src -- --depth 1`. Look
   through `docs/`, `docs/design/` included. You **MUST** read
   `docs/design/rule-style.md` (short, binding) and the Vocabulary
   section of `docs/bug-classes.md` (every shared word, which way it
   errs, who reads it). Read `CLAUDE.md` there for the layout.
2. **Look for the word before writing it.** Before deriving any relation,
   search `priv/dl/base.dl`, `layer2.dl`, `stage0.dl` and `clientlib/`
   for it, and `lib/argus/extractor/` and `lib/argus/extractors/` for
   the Elixir side. Most "new" call-graph, reach, closure and module
   concepts already exist. Check what a word means (context-insensitive?
   atoms only? taint "made from" vs identity?) before reusing it; a word
   with the wrong meaning loses coverage or adds false findings.
3. **Write the extractor** on `Argus.Extractor.ValueFlow`, `Helpers.cfg/3`,
   `CallSites`, `Instr`, `Facts`, `Terms` (see extractors reference).
   Declare every relation in `relations/0`.
4. **Write the rules** in the three layers of rule-style.md: a report
   relation (`.output`, an ABI), a detection rule named for the bug
   (three to seven lines, domain words only), and the words below it.
5. **Write the analysis module.**
   - A built-in (in argus's repo): the module in `lib/argus/analyses/`, its program in
     `priv/dl/analyses/` (from `.include "../clientlib/imports.dl"`). A
     new extractor's relations are declared in the schema
     (`lib/argus/schema/`, then `mix argus.gen.dl`), and
     `Argus.SchemaVersionTest` holds the schema version to its shape
     (every bump gets a CHANGELOG entry). The bug class gets its entry
     in `docs/bug-classes.md`, the rule a corpus pair
     (`test/corpus/pairs.exs`), and `mix argus.pins` regenerates the
     pinned inputs (`test/argus/analysis_inputs.exs`). A word two
     analyses use goes in `priv/dl/clientlib/`.
   - A built-in extractor reads the schema only through `Argus.Schema`'s
     accessors. One that reads decoded facts (`module_data.typed`) is
     listed in `Argus.Pipeline.typed_readers/0`, and each relation it
     reads is in `typed_relations/0` (`TypedRelationsTest` and
     `ProducerClosureTest` fail until they are; `CLAUDE.md`,
     "Incrementality"). After changing the schema, the keys or what a
     producer reads, run `mix test --include identity_verify`; before
     touching the graph, `mix test --include parity`; after a rendering
     change, record the goldens again (`ARGUS_RECORD_GOLDENS=1`) and
     review the diff.
   - Outside argus: the runner and Mix task too (outside-analysis
     reference). Argus's driver runs only analyses in `:argus_beam`.
6. **Test.** In argus's repo, tests solve through `Argus.Test.Memo`
   and batch a module's fixture sets in `setup_all` (`Argus.Test.Batch`;
   run `ARGUS_VERIFY_BATCH=1` after adding a set or changing a rule the
   fixtures meet). Test modules are `async: true` unless they touch
   VM-wide state, and then say why; tests that drive VM-wide state run
   in a peer (`Argus.Test.Peer`). `CLAUDE.md` has the rules. Outside
   argus, compile fixtures into a temp dir
   (`Kernel.ParallelCompiler.compile_to_path/3`), solve them once in
   `setup_all`, and assert findings and their placed lines.
7. **Verify refactors by identity**: snapshot the extracted facts and
   the output (and any internal value relations) before a change and
   diff row for row after. Tests passing is not enough; a refactor
   changes no finding (rule-style.md).
8. **CI.** Argus's own CI installs Soufflé from the Ubuntu PPA, which
   is 2.4, so a built-in's rules must compile under 2.4. It rejects a
   bare `_` inside a destructured record or ADT (`arguments = [value, _]`
   is "Ungrounded"); name it (`[value, _rest]`). A package outside argus
   can run 2.5 in CI (outside-analysis reference). Either way, CI on x86
   catches unguarded divisions (below).

## Argus's own principles

These come from argus's `CLAUDE.md` and hold for any analysis built on
it.

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
  with a budget Soufflé enforces (`.limitsize`), not a timeout.
- **Err quiet by default.** A fact that cannot be sure says `"dynamic"`,
  and rules ask what is *not* handled. An analysis that errs loud
  somewhere says where, and why.
- **Built-ins are BEAM-specific.** Every analysis argus ships targets a
  BEAM bug class, and generic vocabulary goes in `priv/dl/clientlib/`. A
  library's bugs (Nx's, Ecto's) belong in a package outside argus.

## Rules that bit before

- **Keep coverage in mind.** A speedup that lowers a depth, drops cases
  or narrows a word changes what the analysis finds. Know what a change
  costs in findings and make the trade on purpose, whichever way it
  goes. Argus's own default is to err quiet (above). Its docs frame
  this as which way a word errs, quiet or loud
  (`docs/bug-classes.md`), with each bug class's assumptions, limits and
  precision there, and the Soundness sections of `docs/design/`.
  Measure before assuming what is slow.
- **Soufflé reorders a rule's conditions.** A guard does not protect a
  division: write `x / max(d, 1)` where the rule requires `d >= 1`. An
  integer division by 0 traps on x86 and silently gives 0 on ARM Macs,
  so local tests will not catch it.
- **A `.plan` can move a functor ahead of its guard.** `to_number(substr(...))`
  behind a `match(...)` crashes on `""` once a plan reorders the atoms.
  Leave such rules unplanned, or make the functor total.
- **Recursive rules must start from their delta.** The biggest speed
  lever: lead with the recursive atom whose new facts should fire the
  rule, and `.plan` the other versions from their own delta atom. This
  took one analysis from 12.8s to 4.3s with identical output.
- **Pass `stage0: :provided` to `run_rules/3` after `derive_stage0/2`.**
  Otherwise it compiles your whole program just to learn whether it reads
  process points-to (seconds).
