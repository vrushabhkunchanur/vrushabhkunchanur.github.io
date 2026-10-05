# XenReality Engineering Standard

This is the tool-agnostic engineering standard for XenReality. It applies regardless of
which AI coding tool you're using — Claude Code, Codex, Cursor, Antigravity, or none at
all. It is read automatically, with no setup, by any tool that supports the `AGENTS.md`
convention (a neutral, multi-vendor standard, not owned by any one company).

If a rule here ever gets in the way of doing something right, push back. We'd rather
change the doc than have engineers ignore it.

We're a small team building computer vision systems. The work is hard, the deadlines are
real, and the cost of bad code compounds.

---

## Read this first

A few principles sit underneath everything else. If you internalize only these, you'll
write code we're happy to ship.

**1. AI is a tool, not an author.** We know deadlines are tight and that writing this
kind of software takes time. Use AI freely, to draft, explain, learn, debug. But take the
time to actually understand what it gives you.

**2. Strive for simplicity.** Keep the code simple.

**3. Fewer lines is better.** A PR that removes 200 lines and adds 100 is usually a great
PR. We measure progress by problems solved, not code shipped.

**4. Understand what you ship.** This is the most important rule we have. If you can't
explain a piece of code to a teammate, it doesn't belong in `main`. Ever. We can't
operate a system whose code we don't understand.

**5. Tests are not optional.** Every piece of code we ship is to be tested. We use
`pytest`, we measure coverage with `coverage`, and we never merge with a red CI.

The rest of this doc is the *how*.

---

## Working with AI

We use AI. It's a real productivity multiplier when used well, and a real liability when
used poorly.

**Do:**
- Use it to draft, explain, and explore.
- Use it to learn, ask it to walk you through unfamiliar code or libraries.
- Use it for boilerplate, tests, and tedious refactors.

**Don't:**
- Paste code without reading it.
- Ship something you can't explain.
- Use it to replace thinking.

If AI generated more than ~10 lines of code, spend at least 30 minutes understanding it
before merging. Read every function. Mentally trace what happens with a real input. Ask
it to explain anything that's unclear, then verify the explanation against the code.

If you can't explain it in code review, the PR isn't ready.

We say it again because it's the most important rule: **we cannot ship code our
engineers can't reason about.** Everything else flows from this.

---

## Writing code

### Simplicity is the goal

If a reviewer has to think hard to verify correctness, rewrite it. Clever code is a
liability, it's harder to read, harder to debug, and harder to hand off. Code that looks
unimpressive but is obvious on first read is what we want.

> **Recommended reading:** [The Grug Brained Developer](https://grugbrain.dev) — short
> and captures everything about complexity.

For example,

```python

# Clever
status = "ok" if score >= 80 else "warn" if score >= 50 else "fail"

# Obvious
if score >= 80:
    status = "ok"
elif score >= 50:
    status = "warn"
else:
    status = "fail"

```

### Data structures over code

Bad programmers worry about code. Good programmers worry about data structures. Before
writing logic, design the shape of your data, if the data is right, the code falls out
naturally. If you're writing complex logic to compensate for a bad data shape, fix the
data instead.

> **Recommended reading:** [Discussion](https://softwareengineering.stackexchange.com/questions/163185/torvalds-quote-about-good-programmer)
> — There are many articles covering this, I have linked one of them.

### Eliminate special cases instead of handling them

If your function has 4 special-case `if/else` branches at the top, the abstraction is
wrong. Spend the time to find the unifying shape rather than adding a fifth branch.

### Avoid premature abstraction

Don't build a class hierarchy or plugin system for something used in one place. Wait
until you have at least 3 concrete use cases before extracting an abstraction.

If you find yourself adding an abstraction "because we'll need it later", we won't. Add
it when we need it.

### Functions do one thing

If you describe what your function does using "and," it should probably be two
functions. A function should fit on a screen.

### Limit indentation depth

If you need more than 3 levels of indentation, the function is doing too much. Use early
returns and guard clauses instead of deep nesting.

### Comments explain WHY, not WHAT

The code shows what it does. Comments should explain *why* it does it that way, the
constraint, the workaround, the non-obvious requirement.

### No dead code, no commented-out code

If it's not used, delete it.

### No magic numbers

```python
# Bad
if status == 7:
    retry()

# Good
class JobStatus(IntEnum):
    PENDING = 0
    RUNNING = 1
    NEEDS_RETRY = 7

if status == JobStatus.NEEDS_RETRY:
    retry()
```

### Prefer pure functions

A function that takes inputs and returns outputs, without touching global state or doing
I/O, is trivially testable and trivially correct. Push side effects (DB writes, API
calls, file I/O) to the edges of your system. Keep the core pure.

---

## Python conventions

### Type hints everywhere

Every function signature in shipped code gets type hints. We run `mypy` (or `pyright`)
in CI. Type hints catch bugs before review and serve as machine-checked documentation.

```python
# Bad
def detect(frames, threshold=0.5):
    pass

# Good
def detect(frames: list[np.ndarray], threshold: float = 0.5) -> list[Detection]:
    pass
```

### Use modern Python

Use Python 3.10+.

- `dataclasses` over raw dicts for structured data
- f-strings over `%` or `.format()`
- `enum` over magic constants
- `match` statements where they read more clearly than `if/elif` chains

### Mutable default arguments are forbidden

```python
# Bad
def add_item(item, items=[]):
    items.append(item)
    return items

# Good
def add_item(item, items: list | None = None) -> list:
    items = items if items is not None else []
    items.append(item)
    return items
```

### No `import *`

Explicit imports only. It prevents name collisions.

> **Recommended reading:** [The Zen of Python](https://peps.python.org/pep-0020/) —
> nineteen lines, two minutes to read. Sets the cultural baseline.

### No credentials, tokens, or API keys in source code — ever

Not even a "shared team token" someone decided was safe to commit. A hardcoded secret is
a live credential in git history the moment it lands, on a repo that only grows more
collaborators, granting access under an identity nobody can attribute or revoke
individually. Load it from an environment variable or a secrets manager, full stop.

```python
# Bad
API_TOKEN = "sk_live_abc123..."  # "shared token, safe to commit"

# Good
API_TOKEN = os.environ["API_TOKEN"]
```

If you find one already committed, say so immediately and ask whether to rotate it —
don't add a second one next to it.

---

## Error handling

Three things matter:

**1. Catch specific exceptions, not everything.**

**2. Catch at boundaries, not in the middle.** Errors should bubble up to a place where
someone can do something useful. In the middle of logic, let them propagate. Don't wrap
every line "just in case."

**3. When you catch, *do* something.** Acceptable: log with full context, transform into
a domain error, retry, return a sensible default *with a log*, re-raise after cleanup.
Unacceptable: `pass`, swallowing silently, `print("oops")` and continuing.

---

## Testing

Testing is the single biggest difference between a codebase you can change confidently
and one you can't. We take it seriously.

We use **three layers** of tests, because we work with ML models and the
unit-test-everything approach doesn't fit our reality.

### Layer 1 — Unit tests

For all the non-ML code, which is most of our codebase: video I/O, frame extraction, IoU
calculations, NMS, post-processing, API endpoints, data parsing, etc. These run on tiny
synthetic inputs. No model required.

```python
# tests/test_iou.py
def test_iou_identical_boxes():
    box = BoundingBox(0, 0, 10, 10)
    assert compute_iou(box, box) == 1.0

def test_iou_no_overlap():
    a = BoundingBox(0, 0, 10, 10)
    b = BoundingBox(20, 20, 30, 30)
    assert compute_iou(a, b) == 0.0
```

Use `pytest`. Use `coverage`. Aim for 80%+ on non-ML code, but remember: coverage is a
floor, not a goal. 100% coverage with bad assertions is worse than 70% with good ones.
Test behavior, not implementation.

### Layer 2 — Smoke tests

End-to-end runs on 2-3 tiny test clips. Doesn't check accuracy, checks that the pipeline
works: right output shape, right format, no crashes.

```python
# tests/smoke/test_pipeline.py
@pytest.mark.smoke
def test_detection_pipeline_runs(tmp_path):
    output_path = run_pipeline("test_data/tiny_clip.mp4", out_dir=tmp_path)
    assert output_path.exists()
    assert len(load_detections(output_path)) > 0
```

### Layer 3 — Model evaluation

Our "did the model find 4 people instead of 3" tests. Curated test set with ground truth
annotations. Tracks precision, recall, mAP. Fails if metrics drop more than X% from
baseline.

This layer runs **nightly** on a scheduled CI job, and **on any model change** (triggered
manually or via a PR tag). It doesn't run on every PR because it's slow.

```python
@pytest.mark.evaluation
def test_detection_recall_on_test_set():
    results = run_detection_on_dataset("test_videos/")
    metrics = compute_metrics(results, ground_truth="test_videos/labels.json")

    assert metrics.recall >= 0.85, f"Recall dropped to {metrics.recall:.3f}"
    assert metrics.precision >= 0.90, f"Precision dropped to {metrics.precision:.3f}"
```

### A few non-negotiables for our test set

- **Ground truth is versioned and immutable.**
- **Track metrics over time, not just pass/fail.** Log actual numbers somewhere: a CSV
  in-repo, a Google Sheet.
- **Tests should be fast.**

### Tests are documentation

A new engineer should be able to read your tests and understand how the code is meant to
be used. Write them with that reader in mind.

---

## Pull requests and process

### Small PRs

A PR that touches 50 lines gets a real review. A PR that touches 2,000 lines gets
rejected. If your change is large, split it into a stack of small reviewable commits.

If a PR genuinely has to be large, call that out in the description and walk reviewers
through it.

### Commit messages explain *why*

Format: imperative subject line under 72 chars, blank line, body explaining *why* this
change is needed. The diff already shows the *what*.

### One logical change per commit

### Code review

- Review for clarity, correctness, and tests.
- If the reviewer doesn't get it, neither will the next person.
- Disagreement is fine; ego isn't. The code goes where the argument is strongest.

---

## CI is mandatory

Every repo runs Github Actions on every PR:

- **Linter** — `ruff`
- **Type checker** — `pyright` or `mypy`
- **Unit + smoke tests** — `pytest`
- **Coverage report** — `coverage`
- **Package Manager** — `uv`

`main` is protected. Red CI blocks merge. No exceptions.

---

## Choosing libraries and tools

- Python 3.10+
- `pytest`, `ruff`, `mypy`, `uv`

Adopting something outside this list isn't forbidden, but it needs a justification.
Bring it to a team discussion before introducing it.

---

## Docker

Containerize everything since we have to ship the code to multiple devices.

---

## License

Make sure you check the licenses of the dataset used as well as the codebase. Acceptable
licenses are `MIT`, `Apache-2.0` and `BSD-3-Clause`. Non Acceptable include `GPL`,
`AGPL`, `CC BY-NC` or `CC BY-NC-SA`.

---

## Misc

- Dont use emojis in the code.
- AI assistants tend to add extra indentation like this, dont push it without changing it

```python
# Bad
var_aaaaaa = 1
var_a      = 1

# Good
var_aaaaaa = 1
var_a = 1
```
- Every program written, update the readme with benchmarks on various devices.
- When using a different python version, mention it in the docs.
- Be explicit on the library version numbers.

## This doc is not finished

This doc will change. If a rule isn't being followed, either we enforce it or we delete
it. If you hit something that should be in here, open a PR.

---

## Tool-specific layers on top of this file

This file is deliberately tool-agnostic — no assumptions about which AI coding tool
you're using. Some tools have their own additional layer on top of it:

- **Claude Code** — `CLAUDE.md` imports this file (`@AGENTS.md`) and adds Claude-specific
  protocol: a mandatory session-start summary, an Explain-Back Report on every
  substantial change, an explicit Hard Stops list, and five workflow skills
  (`/xen-review`, `/xen-explain`, `/xen-split`, `/xen-tests-first`, `/xen-delete-pass`).
  See `CLAUDE.md` and `plugins/xenreality-standard/`.
- **Codex, Cursor, Antigravity** — read this file directly, natively, with no extra setup.
