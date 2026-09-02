---
name: rtk
description: "Token-optimized CLI proxy: wrap noisy commands (test runs, builds, git, grep, gh, logs, directory walks) so a filtered summary reaches context instead of a raw dump. Use before running any command whose output is large or repetitive, and to decide when filtering would hide the line you actually need."
user-invocable: false
---

# rtk

`rtk <wrapper> <command>` runs the real command and filters its output before it reaches your context.
The captain's standing instruction (`~/AGENTS.md`, wrapper list in `~/RTK.md`) tells every agent on this box to prefer rtk for large or noisy output.
There is no automatic command hook in jcode, so reaching for the rtk form is a manual choice you make each time.

Every wrapper passes the underlying command's real exit status through, so branch on `$?`, not on the filtered text.

## When to reach for it

Wrap when the output is big, repetitive, and you only want the exceptions:

- Test suites, builds, type checks, linters: you want failures, not thousands of passing lines.
- Repository-wide search, directory walks, log files, long `git log`/`git diff`.
- `gh` list and view output, which is verbose JSON-ish prose by default.

Do NOT wrap:

- Interactive commands, anything that prompts or needs a TTY.
- Output you need verbatim: a file you are about to edit by exact bytes, a config you will copy, a hash or version string, a diff you will apply.
- Short commands. Filtering a 5-line output saves nothing and adds a process.
- The one failing case you are already debugging. Once you know a specific test or error matters, run it raw and read all of it; a filter that drops the one line you needed is worse than the raw dump.

`rtk proxy <cmd>` is the escape hatch: runs the command raw, still counts it in the savings stats.
`rtk run <cmd>` runs raw via `sh -c` with no filtering and no tracking.

## Workflows

Run a test suite and see only failures, then re-run the failing case raw:

```bash
rtk test cargo test          # only failures, exit status preserved
cargo test some_failing_test -- --nocapture   # raw, once you know what to look at
```

Compile or lint and keep only errors and warnings:

```bash
rtk err cargo check --message-format short
# [ok] Command completed successfully (no errors)   when clean
```

Orient in an unfamiliar repository without dumping the tree:

```bash
rtk find src -name "*.rs"    # compact grouped tree, not one path per line
rtk read Cargo.toml          # filtered read
rtk grep -r "fn main" src    # grouped by file, whitespace stripped
```

Review repository state before committing:

```bash
rtk git status               # branch plus "clean - nothing to commit"
rtk git log --oneline -5
rtk git diff HEAD~1 --stat   # condensed change summary
```

`rtk diff` is a wrapper around the `diff(1)` binary and expects two paths; for repository changes use `rtk git diff`.

Filter output you already have, without re-running the command:

```bash
git log --oneline -20 | rtk pipe
```

Check whether the wrapping is actually paying for itself:

```bash
rtk gain                     # totals plus per-command savings table
rtk gain --history           # recent command history
```

## Fleet conventions

- Heavy runs go through `fm-heavy-run.sh` and rtk goes INSIDE it, not around it: `fm-heavy-run.sh --task <id> -- rtk test <runner>`. The helper returns the command's real exit status; act on that.
- `~/RTK.md` is the canonical wrapper list for this box. This skill is the judgment layer on top of it; when the two disagree, `~/RTK.md` wins on which wrappers exist and this skill wins on when to use them.
- Wrapper coverage is not uniform. `rtk grep` needs `-r` for a directory (bare `rtk grep <pat> <dir>` hits the native `grep: is a directory` error), `rtk rg` needs `ripgrep` installed, and `rtk tree` needs `tree` installed. When a wrapper is missing its backing binary it says so and does nothing; fall back to the raw command rather than concluding there were no results.
- An empty filtered result is ambiguous by design. Before reporting "nothing found", re-run the underlying command raw once to distinguish "no matches" from "filtered away".
- Verify presence with `rtk --version` (installed at `/usr/local/bin/rtk`). Absent means run commands raw, not fail the task.
- This repository is a fork. Open pull requests against `yjuyjuy/rtk` with base `develop`, never against the `rtk-ai/rtk` upstream.

## Non-goals

- **rtk filtering is lossy by design.** It drops lines it judges uninteresting, and it can drop the line you needed. It is a context-budget tool, not a faithful transport. Any conclusion that depends on complete output must be drawn from a raw run.
- Not a replacement for reading a file you are about to edit. Use the normal read path for exact content.
- Not a test runner, build system, or search engine. It is a wrapper: the underlying tool must already be installed and correct on its own.
- Not a substitute for `--help`. This skill covers only the judgment calls; flags and usage come from `rtk <command> --help`.
- Not for output another program consumes. Filtered text is for human and agent eyes, never a pipeline that parses a fixed format.
