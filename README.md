# argus-analysis

An agent skill for building analyses on [Argus](https://github.com/QuinnWilton/argus)
(the hex package `argus_beam`): Soufflé Datalog programs over facts
extracted from compiled BEAM modules, with an Elixir extractor and
findings.

It gives an agent what it would otherwise have to dig out of argus's
source: where the code and docs are, the extractor helpers and what each
of argus's own extractors already emits, the base facts, call graph and
shared rule library, argus's rule style and principles, Soufflé's traps
and performance levers, and a template for an analysis package that
lives outside argus (runner, caching, finding placement, Mix task, tests
and CI). It covers built-in analyses in argus's own repo too.

Written against argus_beam 0.20.1 and Soufflé 2.5.

## Install

For Claude Code, clone it into a skills directory, under the skill's
name:

```sh
# for every project
git clone https://github.com/heydtn/argus_skill.git ~/.claude/skills/argus-analysis

# for one project
git clone https://github.com/heydtn/argus_skill.git .claude/skills/argus-analysis
```

The agent loads it when a task matches its description (building,
extending, speeding up or debugging an Argus analysis), or when asked for
it by name.

## Contents

- `SKILL.md`: the demand-driven and top-down rules, the workflow,
  argus's principles, and the rules that bit before.
- `reference/argus-layout.md`: where argus's code and docs are, and which
  built-in analysis to copy.
- `reference/extractors.md`: the extractor contract and the shared
  helpers.
- `reference/datalog.md`: the facts, the call graph, the shared words,
  the rule style, Soufflé's traps and performance.
- `reference/outside-analysis.md`: an analysis argus does not ship.

A worked example of an outside analysis is
[argus_nx_tensor_analyses](https://github.com/heydtn/argus_nx_tensor_analyses).

## License

MIT. See `LICENSE`.
