# impeccable-python

An agent skill for writing, reviewing, verifying, and optimizing high-stakes
Python. It works with Claude Code, Codex, Cursor, and any agent that loads
`SKILL.md` skills.

The skill makes an agent treat "the tests pass" as the start of the job. The
agent checks the change against the failures that can actually happen, picks
the right verifier for each one, and reports what was checked, under which
bounds, and what was left out.

## Install

Copy `skill/` into your agent's skills folder under the name
`impeccable-python`:

```sh
git clone https://github.com/hexuria/impeccable-python
mkdir -p ~/.claude/skills
cp -r impeccable-python/skill ~/.claude/skills/impeccable-python
```

To get updates with `git pull`, link the folder instead of copying it:

```sh
ln -s "$PWD/impeccable-python/skill" ~/.claude/skills/impeccable-python
```

The folder name must be `impeccable-python`, matching the skill's `name`.
Strict loaders reject a folder named `skill`. For other agents, put the same
folder wherever they load skills from, for example `.claude/skills/` or
`.cursor/skills/` inside a project.

Rust extensions (PyO3, maturin): the skill hands their Rust side to the
impeccable-rust skill; install it too if you ship one.

## Use it

The agent loads the skill on its own when you work on serious Python. You
can also name it. Example prompts:

```text
Use impeccable-python to review the C extension in src/_speedups.c.
Harden this asyncio worker pool with impeccable-python.
I rewrote the tokenizer for speed. Verify it against the old one.
Make this parser 2x faster with impeccable-python, keeping output identical.
Audit how this project is verified and tell me what is missing.
Set up CI for this published package following impeccable-python.
```

Every change ends with a report like this:

```text
Evidence:     differentially tested tokenize() vs old_tokenize(): 500 examples
              (ci profile) plus the crosshair backend (200 examples, no
              deadline), no divergence
              type-checked with mypy 2.3.1 --strict; mutation-tested: mutmut,
              0 survivors in tokenize.py
Documented:   ADR for the token table; no __hash__ on Token on purpose
Deferred:     Atheris campaign on tokenize() moved to nightly CI
Compat/deps:  griffe check clean against v2.4.0; pip-audit clean
Verification: pure Python deterministic behavior; owner of each affected failure mode
```

## Toolbox

The skill ships `scripts/impeccable`, which sets up and runs the command-line
tools the skill names. The agent runs `impeccable doctor` when it starts
verification work, and runs anything the host lacks in a pinned Linux toolbox.

```sh
alias impeccable=~/.claude/skills/impeccable-python/scripts/impeccable

impeccable doctor                    # what runs here, in the toolbox, or nowhere
impeccable setup toolbox             # build the Linux image once (needs Docker)
impeccable run uv run --locked pytest              # any command, in Linux, on this project
UV_PYTHON=3.14t impeccable run uv run --locked pytest   # the same on free-threaded CPython
impeccable sanitize address          # pytest with C extensions rebuilt under ASan
impeccable valgrind                  # pytest under Valgrind memcheck
impeccable fuzz tests/fuzz_parse.py  # an Atheris harness, 60 seconds by default
impeccable bench-guard main -- uv run --locked pytest --benchmark-save=base
impeccable bench-guard main -- uv run --locked pytest --benchmark-compare=base --benchmark-compare-fail=mean:5%
impeccable setup host                # or install the same pinned tools on this machine
```

The toolbox is a Docker image with its versions pinned in
[`skill/toolbox/tools.txt`](skill/toolbox/tools.txt) and the
[`Dockerfile`](skill/toolbox/Dockerfile): uv, CPython 3.12, 3.13, 3.14, and
free-threaded 3.14t, Ruff, ast-grep, zizmor, actionlint, pip-audit, Griffe,
cyclonedx-py, pypi-attestations, osv-scanner, TLC, Valgrind, gcc for
sanitizer builds, and Atheris built from source with clang. The image is a few
GB, and the first build takes several minutes. It mounts your project at `/work` and keeps its
Linux environment and the uv cache in Docker volumes, so your `.venv` stays
untouched. Editing a pin rebuilds the image on the next run and removes the
superseded one.

Libraries that import your code (pytest and its plugins, Hypothesis,
CrossHair, mutmut, mypy, Pyright, Pyrefly, ty) belong in your project's dev dependencies
and `uv.lock`. `impeccable doctor` lists the versions the skill was written
against and shows which ones your environment lacks. The provers (Nagini,
Lean) are not included.

On macOS the toolbox runs what the host cannot: Valgrind, Atheris (its wheels
are Linux x86_64 only), and the sanitizer builds.

## What it covers

Verification by failure class — strict typing, error-path tests, property and
fuzz tests, known-answer vectors, mutation testing, sanitizers on C and C++
extensions, thread and asyncio checks, trustworthy benchmarks,
misuse-resistant APIs, API compatibility, public-API typing tests, tests on
the built wheel and sdist, downstream tests, and locked, audited
dependencies. Each failure class gets one verifier that owns it.

Optimization as a search — asked to make Python faster, the agent freezes a
baseline, profiles before guessing, tries one measured hypothesis at a time,
keeps only changes that beat the baseline without breaking the oracle, and
stops when gains converge. `impeccable bench-guard` fails the run when
benchmark files, build or tool configuration, interpreter, or
benchmark-affecting environment moved since the baseline; `perf-log.md`
records every attempt, including the ones reverted. See
[`skill/performance.md`](skill/performance.md).

Rewrites get a differential oracle — the old implementation stays until the
new one matches it on generated inputs.

Reports are honest — each claim uses a precise term such as property-tested,
type-checked, or symbolically checked, with its bounds, assumptions, and
blind spots. A bounded check is never called a proof.

Tools it knows: Ruff, ast-grep, mypy, Pyright, Pyrefly, ty, stubtest, pytest,
coverage.py, diff-cover, mutmut, Hypothesis, CrossHair, Atheris, OSS-Fuzz and
CIFuzz, typeguard, frontrun, pytest-run-parallel, free-threaded CPython,
ASan, UBSan, Valgrind, cibuildwheel, maturin-action, CodSpeed,
pytest-benchmark, pytest-memray, py-spy, memray, pyperf, hyperfine, Griffe,
uv, pip-audit, osv-scanner, PEP 740 attestations, Dependabot, zizmor,
actionlint, TLA+, Nagini, and Lean.

## License

MIT
