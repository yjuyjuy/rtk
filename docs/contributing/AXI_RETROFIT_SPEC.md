# AXI retrofit spec for rtk

This document audits the installed `rtk` binary against the ten AXI (Agent eXperience Interface) principles and records the smallest set of changes that would bring it to the bar.
It is a specification only. No part of the retrofit is implemented here.

Audited binary: `/usr/local/bin/rtk`, version `rtk 0.44.1`.
Principles source: `/work/toolings/axi/principles.yaml` and the AXI skill at `/work/toolings/axi/.agents/skills/axi/SKILL.md`.
All ten principles are mandatory, and there is no tiering.
Every verdict below cites a command that was actually run and the output that came back.

## Scorecard

### 1. Token-efficient output - gap

`rtk gain` prints a human-formatted report with box-drawing rules and bar charts rather than a TOON document:

```
$ rtk gain
RTK Token Savings (Global Scope)
════════════════════════════════════════════════════════════

Total commands:    5764
Input tokens:      33.5M
Tokens saved:      31.3M (93.6%)
Efficiency meter: ██████████████████████░░ 93.6%
```

The same is true of `rtk init --show`, which prints `[ok]`/`[--]` status glyphs and a freeform usage block.
The rules and meter bars are pure decoration and cost tokens on every read.
This verdict is about rtk's own reporting surfaces, not about the filtered output of proxied commands, which necessarily mirrors the wrapped tool.

### 2. Minimal default schemas - gap

The default `rtk gain` table carries seven columns per row (`#`, `Command`, `Count`, `Saved`, `Avg%`, `Time`, `Impact`), where `Impact` is a redundant rendering of `Saved` as a bar:

```
  #  Command                   Count   Saved    Avg%    Time  Impact
 1.  rtk find                    153   25.0M    4.3%    1.9s  ██████████
 2.  rtk read                    480    5.9M   22.3%     4ms  ██░░░░░░░░
```

AXI asks for three to four fields per list item.

### 3. Content truncation - gap

The proxied filters do truncate with hints, which is the principle working correctly:

```
$ rtk grep -r "fn " src
  +9 more in src/cmds/cloud/wget_cmd.rs [see remaining: tail -n +17 ~/.local/share/rtk/tee/...]
+95 more files [see remaining: tail -n +1 ~/.local/share/rtk/tee/..._grep_skipped.log]
```

`rtk read` does not.
Its default filter level is `none`, and it returns the file byte-for-byte with no size hint and no note that a filtered form exists:

```
$ wc -c src/main.rs
131762 src/main.rs
$ rtk read src/main.rs | wc -c
131762
```

An agent reading a large file through `rtk read` gets the full 131762 bytes and no signal about the cost it just paid or the flag that would reduce it.

### 4. Pre-computed aggregates - gap

`rtk grep` does pre-compute the overflow counts an agent would otherwise have to discover (`+9 more in ...`, `+95 more files`), but it never states the total number of matches, so the agent cannot tell how much it is missing:

```
$ rtk grep -r "fn " src | wc -l
207
$ grep -r "fn " src | wc -l
4780
```

The 4780 total is known to the tool at filter time and is not surfaced.

### 5. Definitive empty states - gap

`rtk ls` on an empty directory is correct:

```
$ rtk ls /tmp/rtkaudit/emptydir
(empty)
exit=0
```

`rtk grep` on a pattern with no matches is not. It prints nothing at all and exits non-zero, which an agent cannot distinguish from a failed invocation:

```
$ rtk grep -r "zzz_no_such_symbol_zzz" src
exit=1
```

### 6. Structured errors and exit codes - gap

Errors are neither structured nor on stdout, and raw wrapped-tool output leaks through:

```
$ rtk grep --bogus-flag "Config" src   # exit 2
# stdout: empty
# stderr: /usr/bin/grep: unrecognized option '--bogus-flag'
#         Usage: grep [OPTION]... PATTERNS [FILE]...
#         Try 'grep --help' for more information.

$ rtk frobnicate                        # exit 127
# stdout: empty
# stderr: [rtk: No such file or directory (os error 2)]
```

The first response names the dependency (`/usr/bin/grep`) and points the agent at `grep --help` rather than `rtk grep --help`.
The second reports an unknown subcommand as an operating-system error with exit code 127 rather than a usage error with exit code 2.
Both put everything on stderr, which agents do not read, so the agent sees an empty stdout and a bare failure code.
No interactive prompt was encountered in any invocation, and exit codes are otherwise sane (`0` on success, `2` on the flag error).

### 7. Ambient context - gap

rtk has a real setup command, which is more than the inventory credited it with:

```
$ rtk init --show
[ok] Hook: rtk hook claude (native binary command)
[ok] settings.json: RTK hook configured
[--] OpenCode: plugin not found
[--] Cursor hook: not found
```

`rtk init` covers Claude Code, Codex, OpenCode, Cursor and several other agents, supports `--dry-run`, `--uninstall` and `--auto-patch`, and reports existing state idempotently.
What it installs is a `PreToolUse` command-rewrite hook (`hooks/claude/rtk-rewrite.sh` declares `"hookEventName": "PreToolUse"`), and a search of the source for `SessionStart` returns no matches.
So rtk registers itself into the tool-call path but never into session start, and an agent begins every session with no idea what rtk is saving or covering in this directory.
There is also no installable skill: the fleet-wide guidance lives in a hand-maintained `~/RTK.md` outside the tool, which is exactly the drift the principle's single-source-of-truth rule exists to prevent.

### 8. Content first - gap

The bare invocation prints the full usage manual, on stderr, with a usage exit code:

```
$ rtk
exit=2
stdout bytes=0  stderr bytes=5929
```

The 5929 bytes are the complete list of all 80-odd subcommands.
The agent receives nothing on stdout and no live data, and must make a second call to learn anything about the current directory or the tool's state.

### 9. Contextual disclosure - gap

Successful outputs carry no next-step hints:

```
$ rtk git status
* fm/dev-56
clean — nothing to commit
exit=0

$ rtk ls src
analytics/
cmds/
core/
main.rs  128.7K
exit=0
```

The one hint-shaped affordance is the grep overflow line, and it points at a temporary tee log path rather than at an rtk command.

### 10. Consistent way to get help - partial gap

Per-subcommand help is genuinely good and is the principle's strongest showing in this tool:

```
$ rtk grep --help
Compact grep - strips whitespace, truncates, groups by file
Usage: rtk grep [OPTIONS] [EXTRA_ARGS]...
Options:
  -l, --max-len <MAX_LEN>      Max line length [default: 80]
  -m, --max <MAX>              Max results to show [default: 200]
```

It is scoped to the subcommand, lists defaults, and goes to stdout with exit code 0.
The version fast path is also fine: `rtk --version` and `rtk -V` both print `rtk 0.44.1` and exit 0, and the measured cost is 1.9 ms per call against a 0.5 ms `/bin/true` floor in the same harness, so there is no import-graph tax to remove.
Two things are missing.
The tool never identifies itself before live data, because there is no home view to identify itself in.
And `-v` is bound to verbosity rather than version, so the third spelling the principle requires fails outright:

```
$ rtk -v
exit=127
# stderr: [rtk: No such file or directory (os error 2)]
```

## Change list

1. Add a bare-invocation home view on stdout, exiting 0, that shows the executable path with `~` collapsed, a one-sentence description, and live savings state scoped to the current directory. (Principles 8 and 10.)
2. Emit rtk's own reporting surfaces - the home view, `rtk gain`, `rtk init --show` - as TOON, dropping the rules, glyphs and meter bars. (Principle 1.)
3. Reduce the default `rtk gain` per-command schema to command, count and saved, and move the remaining columns behind an explicit field selection. (Principle 2.)
4. Include the true total alongside every truncated filter result, so overflow lines state what fraction of the whole is shown. (Principle 4.)
5. Append a trailing size hint to `rtk read` stating bytes and lines returned and the flag that filters them, without changing the content returned. (Principle 3.)
6. Print an explicit zero-result line and exit 0 when a filter matches nothing. (Principle 5.)
7. Translate wrapped-tool failures into structured errors on stdout that name rtk's own commands, and report an unknown subcommand as a usage error with exit code 2. (Principle 6.)
8. Extend `rtk init` to install a `SessionStart` ambient-context hook, printing the home view, for the same agent targets the existing rewrite hook supports. (Principle 7.)
9. Ship an installable skill generated from the home view's own guidance, so the fleet's rtk instructions come from the tool rather than from a hand-maintained file. (Principle 7.)
10. Add `help` next-step hints to list-shaped and mutation-shaped outputs, phrased as complete rtk commands with placeholders for runtime values. (Principle 9.)

## Non-goals

Converting the filtered output of proxied commands to TOON is out of scope.
That output is a compressed rendering of another tool's result, its shape is the product's value, and reformatting it would break every agent and hook that reads it. Principle 1 is satisfied here on the surfaces rtk itself owns.

Changing what `rtk read` returns by default is out of scope.
Agents call `rtk read` when they need file content they can act on, and silently truncating it would turn a safe read into a lossy one, and change 5 adds the missing size hint without touching the content.

Rebinding `-v` to `--version` is waived.
`-v` is rtk's established verbosity flag, documented in the top-level options and accepted in the `-v`/`-vv`/`-vvv` form, and taking it over would silently change the meaning of existing invocations.
`-V` and `--version` already satisfy the fast-path requirement, so the residual gap is one spelling with a compatible substitute.

Version-path latency work is waived as unnecessary.
The measurement above shows 1.9 ms against a 0.5 ms process-spawn floor, so the principle's concern does not apply to a single static Rust binary.

Opening a pull request against the upstream `rtk-ai/rtk` repository is out of scope.
This spec lives in the `yjuyjuy/rtk` fork.

## Evidence

Bare invocation, captured in the fork worktree:

```
$ rtk
exit=2
stdout bytes=0 stderr bytes=5929
--- stderr (first 12 lines) ---
Rust Token Killer - Minimize LLM token consumption

Usage: rtk [OPTIONS] <COMMAND>

Commands:
  ls             List directory contents with token-optimized output (proxy to native ls)
  tree           Directory tree with token-optimized output (proxy to native tree)
  read           Read file with intelligent filtering
  smart          Generate 2-line technical summary (heuristic-based)
  git            Git commands with compact output
  gh             GitHub CLI (gh) commands with token-optimized output
  glab           GitLab CLI (glab) commands with token-optimized output
```

Hot-path invocation, using `rtk grep`, one of the wrappers `~/RTK.md` tells every agent on this box to prefer:

```
$ rtk grep -r "pub struct Config" src
src/cmds/system/local_llm.rs:pub struct Config {
src/core/config.rs:pub struct Config {
exit=0
```

The result is correct and compact, and it carries no identification, no total, and no next step.
