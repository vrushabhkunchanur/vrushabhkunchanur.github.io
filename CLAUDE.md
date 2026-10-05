@AGENTS.md

# Claude Code layer on top of the standard above

Everything above this line is `AGENTS.md` — the tool-agnostic XenReality engineering
standard, imported automatically. Everything below is specific to Claude Code: how
Claude should actually operate, on top of those rules.

This section is also shipped as the `xenreality-standard` Claude Code plugin (see
`plugins/xenreality-standard/`), which injects it automatically at the start of every
session instead of requiring this file to be copied into each repo by hand. Either path
gets you the same content.

---

## 0. The one rule everything else serves

**We cannot ship code our engineers cannot reason about.**

So Claude does not optimise for "task completed". Claude optimises for
"the human who owns this PR can defend every line of it in review".

This has teeth. See the Explain-Back Report below — it is mandatory output, not a
nicety.

---

## 1. Session start

At the beginning of a session, before touching anything, Claude states in **three
lines**:

```
Repo: <name> | Python: <version from pyproject> | Stack: <key libs>
Task as I understand it: <one sentence>
Plan: <2-5 numbered steps, or "need to explore first">
```

Then Claude waits for a "go" if the task will touch more than one file or more than
~50 lines. Small, obvious edits do not need a check-in.

If the repo has no `pyproject.toml`, no tests directory, or no CI config, say so in that
first message. Do not silently work around missing infrastructure.

---

## 2. The loop for every task

```
UNDERSTAND -> PLAN -> IMPLEMENT (small) -> VERIFY (run it) -> EXPLAIN-BACK
```

**UNDERSTAND.** Read the existing code before writing new code. Match the surrounding
style.

**PLAN.** Say what you are going to change and where, in files and function names.

**IMPLEMENT.** Smallest change that fully solves the stated problem. No opportunistic
refactors, no "while I was in here".

**VERIFY.** You are not done when the code is written. You are done when it ran. If you
could not run something, say exactly that — never imply a test passed that you did not
execute.

**EXPLAIN-BACK.** Below. Always.

---

## 3. Explain-Back Report (mandatory)

Whenever Claude writes or changes more than ~10 lines of code, the final message ends
with this block. No exceptions, no shortened version.

```
EXPLAIN-BACK
Files touched: <paths, with the one-line reason each was touched>
What this does: <plain-English walkthrough, 3-6 lines, no jargon smuggling>
Why this shape: <the design decision, and what the obvious alternative was>
Trickiest part: <the single line/block a reviewer is most likely to misread, and why it is correct>
Assumptions I made: <or "none">
Not covered: <edge cases, perf, error paths I deliberately left out>

Check yourself before you push. You should be able to answer:
  1. <question about the core logic>
  2. <question about an edge case>
  3. <question about why an alternative was rejected>
```

The three questions are real questions about *this* diff, not generic ones. If a change
is genuinely too clever to explain in six lines, that is a signal the change is wrong —
simplify it instead of writing a longer explanation.

---

## 4. Claude's known formatting habits — corrected

- **No emojis.** Not in code, not in comments, not in log strings, not in commit
  messages, not in PR descriptions.
- **No column-aligned assignments** — padding with spaces to line up `=` creates noisy
  diffs.

```python
# No
var_aaaaaa = 1
var_a      = 1

# Yes
var_aaaaaa = 1
var_a = 1
```

- No decorative banner comments (`# ====== SECTION ======`).
- No docstrings that just restate the signature.
- Do not reformat lines you did not otherwise change.

---

## 5. Verification commands

Claude runs these before claiming a task is done, and pastes the real output. Prefix is
whatever section 10 declares for this repo — `uv run <cmd>` on a `uv` repo, plain
`<cmd>` inside an activated `.venv` on a pip repo, `poetry run <cmd>` on poetry:

```bash
<prefix> ruff check .
<prefix> ruff format --check .
<prefix> mypy .            # or: <prefix> pyright
<prefix> pytest -q
<prefix> pytest -q --cov --cov-report=term-missing   # when coverage matters
```

If section 10 doesn't say which prefix this repo uses, say so at session start instead
of guessing, and ask. If a command can't run here (no GPU, no dataset), say which one and
why, and state what the human needs to run manually.

---

## 6. Git discipline

- **Never** `git push`, open a PR, force-push, rewrite history, or touch `main` unless
  explicitly asked in that session. Committing locally when asked is fine.
- Branch off `main`. Never commit directly to `main`.
- One logical change per commit. Commit message: imperative subject under 72 chars,
  blank line, body explaining *why*. No emojis, no attribution footers unless asked.

---

## 7. Hard stops — Claude asks before doing any of these

- Adding a new dependency, or bumping a major version.
- Deleting or rewriting a test, lowering a threshold, adding `xfail`/`skip`.
- Editing ground truth, labels, or anything under a test-data directory.
- Changing CI config, Dockerfile base image, or CUDA/driver versions.
- Anything touching credentials, `.env`, keys, or deploy scripts — including writing a
  new token or key literal into any file, "shared team credential" or not.
- A change that will exceed ~200 lines.
- Introducing a new abstraction, service, or directory structure.
- Rewriting code you were only asked to fix a bug in.
- **Anything that pushes to a remote, opens/merges a PR, creates or deletes a repo or
  branch, or otherwise changes state on GitHub or another company org/service** — ask
  first, every time, regardless of what was approved in a previous session.

---

## 8. Definition of Done

Claude may only say "done" when every line is true:

```
[ ] Ran ruff, mypy/pyright, pytest — output pasted, all green (or blockers named)
[ ] Tests written at the right layer, and they fail without the change
[ ] Type hints on every new signature
[ ] No dead code, no commented-out code, no emojis, no aligned assignments
[ ] Diff is minimal: nothing changed that did not need to change
[ ] README/Dockerfile/benchmarks updated if affected
[ ] Explain-Back Report included
```

If something on that list is not true, say which item and why, instead of saying done.

---

## 9. How we use Claude (for the human)

Claude is a tool, not an author. The PR has your name on it.

**The 30-minute rule.** If Claude generated more than ~10 lines, spend real time
understanding it before merging. Use the Explain-Back questions as a self-check.

**Skills** (type `/` to see them, from the `xenreality-standard` plugin):

| Skill | Use it when |
|---|---|
| `/xen-review` | Reviewing a diff or PR against this standard, violations only. |
| `/xen-explain` | Defending a piece of code in review tomorrow, self-check before merging. |
| `/xen-split` | A diff or plan is going to exceed ~200 lines. |
| `/xen-tests-first` | Starting a new module — write the failing tests before the implementation. |
| `/xen-delete-pass` | Cleanup pass — dead code, unused imports, premature abstractions. |

---

## 10. Repo-specific info — not covered by this file

Sections 0-9 above (plus everything imported from `AGENTS.md`) are company-wide. This
file has no way to know *this* repo's purpose, entry points, package manager, or test
command — that has to come from somewhere local. If this repo has its own
`CLAUDE.md`-specific section already (many mature repos do — some run to a few hundred
lines of real architecture and gotchas), read the whole thing; it's doing its job. If
none exists, fill in:

```
Purpose:          <what this repo does, one sentence>
Entry points:     <main scripts / services and how to run them>
Python:           <version>
Package manager:  <uv sync | pip install -e ".[dev]" into .venv | poetry install>
Run tests:        <exact command — this is the <prefix> from section 5>
Test data:        <where the tiny clips and ground truth live>
Models:           <which weights, where they come from, license>
Secrets/tokens:   <which env vars are required, where they're documented — never a
                   literal value here or anywhere else in the repo>
Target devices:   <e.g. Jetson Orin, x86 + RTX, shared GPU host, etc.>
Known traps:      <the things that bite newcomers>
Do not touch:     <generated files, vendored code, calibration data>
```

If no such section exists, say so at session start instead of guessing at the package
manager or test command, and ask the human for the missing pieces.

---

*If a rule here gets in the way of doing something right, push back and open a PR on
this repo. We would rather change the doc than have engineers ignore it.*
