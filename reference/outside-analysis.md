# An analysis argus does not ship

Argus's catalog discovers analyses only among the `:argus_beam`
application's modules (`Argus.Analysis.Catalog`), so `Argus.Driver`,
`mix argus` and the `:argus` compiler never run yours, and `mix argus`'s
internals are private. An outside package implements the
`Argus.Analysis` behavior for its outputs and findings, and brings its
own runner, placement and Mix task. The template is
`github.com/heydtn/argus_nx_tensor_analyses` (read its
`lib/argus_nx_tensor_analyses/tensor_shapes.ex` and its Mix task):

```
lib/<package>.ex                       # analyses/0: the Argus.Analysis modules
lib/<package>/<analysis>.ex            # Argus.Analysis: outputs, findings, runner, placement
lib/<package>/<analysis>/<extractor>.ex
lib/mix/tasks/<package>.ex             # `mix argus` plus these analyses
priv/<analysis>.dl                     # the rules
test/<package>/<analysis>_test.exs     # fixtures compiled into a temp dir
.github/workflows/ci.yml               # Soufflé 2.5, on x86
```

This is the route argus itself points to. The query graph's API
(`Argus.analyze/3`, `Argus.Analysis.extract_facts/3`) runs only the
built-in extractors, and raises on `:extractors`: "for another
extractor's rows, extract with Argus.Pipeline.extract/2 and solve with
Argus.Analysis.run_rules/3" (`Argus.Run.check_options!/1`). The
`Argus.Extractor` moduledoc still suggests passing `:extractors` to
`extract_facts/3`, which raises.

## The analysis module

```elixir
defmodule MyPackage.MyAnalysis do
  @behaviour Argus.Analysis
  alias Argus.Findings

  @impl true
  def name, do: :my_analysis                        # what `mix argus my_analysis` selects
  @impl true
  def description, do: "One line for `mix argus --list`"
  @impl true
  def rules_file, do: Application.app_dir(:my_package, "priv/my_analysis.dl")
  @impl true
  def extractors, do: [MyPackage.MyAnalysis.MyFlow]

  @impl true
  def output_relations do
    [
      %{
        name: :my_bug,
        fields: [
          {:id, :instr_id, "the call"},             # types: :symbol | :number | :instr_id | :func_id | :label
          {:func, :func_id, "the function making it"},
          {:kind, :symbol, "which rule it breaks"},
          {:detail, :symbol, "what is wrong"}
        ],
        key: [:id, :kind],                          # rows agreeing on these are one finding
        doc: "One sentence."
      },
      %{
        name: :my_bug_via,                          # evidence: related frames of :my_bug's findings
        fields: [{:id, :instr_id, "the call"}, {:kind, :symbol, "the rule"},
                 {:at, :instr_id, "a call on the way"}],
        evidence: %{of: :my_bug, on: [:id, :kind], limit: 8},
        doc: "One sentence."
      }
    ]
  end

  @impl true
  def finding(:my_bug, [id, _func, kind, detail]) do
    Findings.new(:warning, "Title in the bug's words", "Detail: #{detail}.",
      at: Findings.at_instr(id),
      at_label: "what this line is",
      help: ["what to change, toward what"])
  end

  @impl true
  def evidence(:my_bug_via, [_id, _kind, at]),
    do: Findings.related("what this frame is", Findings.at_instr(at))
end
```

`lib/argus/analysis.ex` documents the rest of an output relation (a
per-kind `key`, `earliest:`, and `retier: :tooling`, which ties into
`Argus.Findings.Tooling` and `clientlib/tooling.dl`: read them first),
and `Argus.Findings` the rest of `new/4`'s options and the anchors.
Severities are `:error`, `:warning` and `:info`.

## The runner

```elixir
def solve(modules, program \\ rules_file(), options \\ []) do
  directory = Path.join(System.tmp_dir!(), "my_analysis_#{System.unique_integer([:positive])}")
  try do
    with {:ok, _} <- extract(modules, directory), do: solve_rules(directory, program)
  after
    File.rm_rf!(directory)
  end
end

defp extract(modules, directory) do
  with {:ok, directory} <- Argus.Pipeline.run(modules, directory, extractors: extractors()) do
    # Argus touches empty files only for its own schema relations.
    for extractor <- extractors(), relation <- extractor.relations(),
        path = Path.join(directory, "#{relation}.facts"), not File.exists?(path),
        do: File.write!(path, "")
    {:ok, directory}
  end
end

defp solve_rules(directory, program) do
  wrapper = directory <> ".dl"
  # Soufflé resolves an .include against the including file; argus's files
  # are wherever Mix put the dependency. Once the rules settle, include
  # less of argus (datalog.md, Performance).
  File.write!(wrapper, """
  .include "#{Application.app_dir(:argus_beam, "priv/dl/clientlib/imports.dl")}"
  .include "#{Path.expand(program)}"
  """)
  try do
    with :ok <- Argus.Analysis.derive_stage0(directory),
         do: Argus.Analysis.run_rules(directory, {:custom, wrapper}, stage0: :provided)
  after
    File.rm(wrapper)
  end
end
```

`run_rules/3` returns `{:ok, %{"relation" => [[string, ...]]}}`, and
`Findings.build(__MODULE__, rows)` turns the rows into findings
(dedup by `key`, evidence joined through `evidence/2`). Soufflé must be
on `PATH`; `Argus.Driver.Result.souffle_missing?/1` tells.

**Cache** the rows under a digest of what the solve reads, keyed as
argus keys its own solves:
- every extracted `.facts` file except `line_info.facts` (a comment moves
  lines, not code);
- the program and every file it includes, argus's too, without their
  comments (`Argus.Souffle.Program.uncommented/1`: a comment is not an
  edit), since argus's rules can change under one version number;
- argus's version;
- the solver, `Argus.Souffle.version(Argus.Souffle.executable())`, so an
  upgraded Soufflé solves again.

Then an edit that leaves the compiled code alone solves nothing. A hit
still extracts, which is cheap next to the solve.

## Placement (findings to source lines)

A finding names an instruction, not a line. Place it with `Argus.Lines`,
as argus places its own. Build the table from the facts you extracted,
before the runner removes them (`Argus.Lines.from_facts_dir(directory)`,
which reads `line_info.facts`). Then resolve: the instruction's line,
else its function's first (`Argus.Lines.resolve/2` does both), else the
module's declaration line (`Argus.Lines.declaration_line(beam_path)`),
else 1. Take the file from the beam's `compile_info` `:source` (argus's
own lookup is private).

Build `%Argus.Located{finding:, file:, line:, end_line: nil, related:
[%{file:, line:, end_line: nil}, ...]}` for each finding.

## The Mix task

Mirror `lib/mix/tasks/argus.ex` (`deps/argus_beam/lib/mix/tasks/argus.ex`
in the consuming project). Its internals are private, so this copies its
shape:

1. `Argus.CLI.Options.parse(args, :mix)`. For `:list` and help, run
   `Mix.Task.run("argus", args)`, then print your analyses.
2. For `:analyze`: select yours when `options.all` or named in
   `options.analyses`, and remove your names from `options.analyses` so
   argus does not reject them.
3. Compile as argus does:
   `Mix.Task.run("compile", ["--no-prune-code-paths", "--return-errors"])`,
   failing only on errors from compilers other than `"argus"`.
4. `config = Argus.CLI.override(Argus.Config.load(), options)`, then
   `result = Argus.Driver.run(config, force: options.force)`.
5. Run yours over the project's own beams
   (`Mix.Project.compile_path() |> Path.join("*.beam") |> Path.wildcard()`)
   with a cache under `Mix.Project.build_path()`, and
   `Map.put(result.located, name, {:ok, located})`.
6. `Argus.Report.Notice.from_result(result, config, cwd)`, then
   `Argus.CLI.report(result, notices, config, options, cwd, cwd)`, then
   honor `options.fail_above`.

The consuming project aliases it over `mix argus`:
`aliases: [argus: "my_package"]`, with the dependency declared `only:
[:dev, :test], runtime: false` beside `{:argus_beam, "~> 0.20.1"}` and
`compilers: Mix.compilers() ++ [:argus]`.

## Tests

- Fixture modules in one source file, compiled once in `setup_all` into
  a temp dir with `Kernel.ParallelCompiler.compile_to_path([source],
  dir, return_diagnostics: true)`, then solved once over every beam there.
- Per case, assert the finding (relation, kind, detail) or its absence.
- For placement, assert `Enum.at(source_lines, located.line - 1)` is the
  call's line, and the same for related frames.
- A module that is not `async: true` says why in a comment above its
  `use ExUnit.Case` (compiling fixtures loads them into the VM;
  capturing `:stderr` is shared).
- Test a Mix task in a separate VM against a small Mix project, as argus
  tests its own (its helpers are in its `test/support`, not the hex
  package; `CLAUDE.md` at `v0.20.1` lists the Mix traps).
- An oracle helps where one exists: the tensor analysis runs each fixture
  through Nx and checks that Nx raises exactly where the analysis says.

## CI

- **Soufflé 2.5**, not the PPA's 2.4. Install the release package and
  verify it:

  ```yaml
  - name: Install Souffle
    env:
      SOUFFLE_DEB: x86_64-ubuntu-2204-souffle-2.5-Linux.deb
      SOUFFLE_SHA512: <from the release's sha512sum.txt>
    run: |
      wget -q "https://github.com/souffle-lang/souffle/releases/download/2.5/$SOUFFLE_DEB"
      echo "$SOUFFLE_SHA512  $SOUFFLE_DEB" | sha512sum -c -
      sudo apt-get update
      sudo apt-get install -y "./$SOUFFLE_DEB"
      souffle --version
  ```

- The OTP versions you support, on x86: an integer division by 0 traps
  there and not on ARM Macs, so x86 is where an unguarded divisor shows.
- `mix format --check-formatted`, `mix compile --warnings-as-errors`,
  `mix test`, and argus's own gates: `mix credo --strict`, `mix dialyzer`.
