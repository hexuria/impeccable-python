---
name: impeccable-python
description: >-
  Use when writing, reviewing, hardening, or designing Python for high-stakes or
  long-lived packages and services, including their C, C++, or Rust
  extensions, threads, asyncio, or free-threaded CPython; when rewriting or
  optimizing Python that already works; or when auditing or setting up how a
  Python project is verified (Hypothesis, mutmut, CrossHair, Atheris, strict
  typing, TLA+, Nagini, Dafny, Lean). Checklist for exhaustive
  testing, trustworthy benchmarks, misuse-resistant APIs a type checker
  enforces, decision records, API compatibility, locked and audited
  dependencies, and risk-driven formal verification with anti-drift rules.
---

# Impeccable Python

Faults may still happen outside the code (spec, hardware, ops). The
implementation should not be what is to blame. There is no single trick. Heavy
tools cost time and compute; spend a deliberate risk budget where failure
hurts.

## Operating rules

1. Prefer making incorrect use inexpressible (to the type checker, or at a validated boundary) over documenting "do not do that."
2. Treat "it works" as insufficient. Show it is not broken under chaos and edge cases, and name the oracle (reference implementation, property, or model) that would catch it being wrong.
3. Prefer automation that catches human misses (a strict type checker, Hypothesis, mutmut, Griffe, pip-audit).
4. When you accept a downside or skip a corner case, write it down.
5. Stagnation is a choice with rising cost. Surface it; do not silently defer forever.
6. Apply expensive verification where failure actually hurts.
7. Give every important failure mode an owner. Link any second model to production Python before treating its results as evidence about the Python.

## Tooling

`scripts/impeccable`, in this skill's folder, sets up and runs the command-line tools this skill names. Its toolbox is a Linux Docker image with pinned CPython 3.12, 3.13, 3.14, and free-threaded 3.14t, standalone tools, TLC, osv-scanner, Valgrind, and an Atheris build; it installs nothing on the host. The provers (Nagini, Dafny, Lean) are not in the toolbox.

- Run `scripts/impeccable doctor` before choosing checks. It reports which tools run on this host, which run only in the toolbox, which are missing or older than the pinned version, and which pinned dev dependencies the project environment lacks.
- Libraries and CLIs that import the project (pytest and its plugins, Hypothesis, CrossHair, mutmut, the type checker) belong in the project's dev dependency group, pinned by `uv.lock`: `uv add --dev "hypothesis==<pin>"`. Run them with `uv run --locked`. In CI use `uv sync --locked` (or `UV_LOCKED=1`): `--frozen` installs a stale lock without complaint, and a bare `uv run` rewrites `uv.lock`.
- Run anything the host lacks, or that needs Linux, in the toolbox: `scripts/impeccable run uv run --locked pytest`. The project gets its own Linux environment in a Docker volume, never the host `.venv`. `UV_PYTHON=3.14t scripts/impeccable run ...` uses free-threaded CPython in a separate environment.
- Run tests against C or C++ extensions rebuilt under a sanitizer with `scripts/impeccable sanitize <address|undefined|address,undefined> [pytest args]`, under Valgrind with `scripts/impeccable valgrind [pytest args]`, and an Atheris harness with `scripts/impeccable fuzz <harness.py> [libFuzzer args]` (60 seconds unless `-max_total_time` or `-runs` is given). Each moves to the toolbox when the host cannot run it, which on macOS is always.
- Install tools on the host with `scripts/impeccable setup host` only when the user asks for it.

## Checklist (run what applies)

Work through each section that fits the change. Skip sections that clearly do
not apply (for example, no threads or asyncio means skip section 5). Say what
you ran and what you deliberately skipped. A new verifier, a second semantic
model, or a concurrency, recovery, or persistence change also follows
Verification architecture.

### 1. Testing

- Raise on broken assumptions; silent wrongness is worse. `python -O` strips `assert`, so an invariant that must hold in production raises explicitly, and the verification suite never runs under `-O`.
- It works is not the same as it is not broken.
- Ruff formats and lints: pin it (`required-version`), start from `select = ["ALL"]` with a short, commented ignore list, and relax tests per file (`S101`, `PLR2004`). At minimum keep `B`, `S`, `ASYNC`, `PT`, `RUF`, `TRY`, `BLE`, `FBT`, and `PGH`. New rules arrive with each release; upgrade on a schedule. Ruff has no type inference, so type-dependent bugs belong to the type checker.
- Run one primary type checker in strict mode, pinned in the lock and run in the project environment (`uv run --locked mypy --strict src`, or Pyright or basedpyright with `typeCheckingMode = "strict"`). Outside that environment, Pyright and Pyrefly pick the first `python` on PATH and report bogus errors.
- A second checker is advisory; checkers disagree at the edges, so switching is not neutral. Pyrefly can be primary only with a pinned version, an explicit `--preset default` or `strict` (unconfigured it runs the lax `basic`), and `--python-interpreter-path`. ty is beta; keep it advisory.
- Ban blanket `# type: ignore` (Ruff `PGH003`) and report unused ignores (mypy strict does). ty does not honor a coded `# type: ignore[code]`; only a bare `# type: ignore` or `# ty: ignore[rule]` suppresses its diagnostic. `Any` from an untyped dependency switches checking off silently. `cast`, `TypeGuard`, and `TypeIs` are unchecked assertions by the author.
- Annotations are not enforced at runtime. Validate untrusted data at the boundary (section 8), and run the suite under typeguard with `pytest --typeguard-packages=pkg --typeguard-collection-check-strategy=ALL_ITEMS` (or the ini keys of the same names); by default typeguard checks only the first item of a collection, and beartype checks one random item.
- Configure pytest 9 with `strict = true` and `filterwarnings = ["error"]`, ignoring a third-party deprecation by module rather than globally. Run the suite with `PYTHONDEVMODE=1`: it shows `ResourceWarning` for leaked files, sockets, and processes, adds allocator debug hooks, enables faulthandler, and turns on asyncio debug, so leaks and unawaited coroutines fail under the warning filter. Interpreter-wide `-W error` also makes startup warnings fatal; use the pytest filter. Dev mode is not a sanitizer: it sees only executed paths and objects actually finalized.
- A test that passes only on retry has failed. Use pytest-rerunfailures only with `--fail-on-flaky` (exit 7); plain `--reruns` reports green. Randomize order with pytest-randomly, keep the printed seed in the log, and reproduce with `-p randomly --randomly-seed=<n>`. Stress a suspect test with `pytest --count 20` (pytest-repeat) and find the interleaving (section 5) instead of loosening the test. Bound hangs with pytest-timeout (`timeout = 300`); `faulthandler_timeout` alone only dumps tracebacks and lets the hung test run on.
- Coverage (`coverage run -m pytest` with `branch = true`, `coverage report --fail-under=N`, and `diff-cover coverage.xml --compare-branch=origin/main --fail-under=90` for the diff) shows what no test executes. It is a gap finder, not a claim: full line and branch coverage can leave most mutants alive.
- mutmut finds code that runs but is never checked. `mutmut run` exits 0 with survivors, so gate on `mutmut export-cicd-stats` (fail on `survived` above budget or `no_tests` above 0 for code the change touched). It has no diff mode; scope with a name glob (`mutmut run "pkg.mod.x_func*"`). Configure `source_paths` and `pytest_add_cli_args = ["-p", "no:randomly"]`.
- Mark a confirmed equivalent mutant with `# pragma: no mutate` and a reason. Count pragmas in CI, and never add one to turn a run green.
- When a git-diff filter is required, use cosmic-ray (`cr-filter-git`) on a clean worktree, and assert zero `INCOMPETENT` mutants: a broken test command reports every mutant incompetent and 0 survivors.
- Test error paths, not only the happy path. Assert the exact exception and message (`pytest.raises(ValueError, match=...)`); a bare `pytest.raises(ValueError)` lets message and argument mutants survive.
- Error litmus test: on a copy or clean worktree, replace one `raise` with `pass` at a time and rerun the suite. Neither mutmut nor cosmic-ray deletes a `raise`, so this is a separate step. A suite that stays green does not check that error path. Report it as "error paths checked by removal litmus."

### 2. Chaos

Add at least one chaos layer that fits:

| Kind | Tools |
|------|--------|
| Thread schedules | `frontrun` (DPOR), pytest-run-parallel on free-threaded 3.14t |
| asyncio order, virtual time, simulated network faults | `simloop` (seeded, replayable), trio's randomized scheduler with `MockClock` |
| Value | Hypothesis (`@given`, `RuleBasedStateMachine`) |
| Logic | `mutmut` |

For reimplementations (custom mapping, codec, parser), property-test against a trusted oracle (for example `dict` or `json`) and assert broad invariants such as "raises only the documented exceptions." For a stateful API, write a `RuleBasedStateMachine` (`Bundle`, `@initialize`, `@precondition`, `@invariant`) whose model is the oracle; a failure shrinks to a replayable step script.

For a rewrite, port, or optimization of code that already works, the old implementation is the oracle. Keep it importable (a `_reference` module or `tests/_oracle/`), run generated inputs through both, and compare return values, mutated arguments, exception types, and side effects. Delete it only after that differential suite is green. A change with no oracle and no invariant has a weak correctness claim; say so in the report.

- `hypothesis write --errors-equivalent old.f new.f` scaffolds the test; edit its strategies (untyped parameters become `st.nothing()`).
- `crosshair diffbehavior --exception_equivalence SAME_TYPE --per_condition_timeout=30 pkg.old.f pkg.new.f` searches symbolically for a differing input (exit 1 with the input). "No differences found" is not equivalence.

Write each property once, as a plain typed function whose body starts with `assert`, and wrap it per engine: `@given(...)` for pytest, the `crosshair` profile below for a solver over the same strategy, `test_x.hypothesis.fuzz_one_input` as the Atheris entry point, and the bare function under `crosshair check --analysis_kind=asserts`. One harness keeps the property test, fuzz target, and symbolic check from drifting into three different properties.

The engines miss different bugs: the solver finds a bug behind arithmetic that random examples and Atheris miss, and Atheris finds coverage-reachable bugs that defeat the solver. Run at least two engines before claiming the property. The strategy is shared, so a narrow strategy is a blind spot for all of them.

Register profiles in `conftest.py` and select with `HYPOTHESIS_PROFILE` or `--hypothesis-profile`. With `CI=true`, Hypothesis loads a derandomized `ci` profile that replays the same inputs every run, so add a randomized nightly profile. Registering `backend="crosshair"` raises at import when the extra is missing, so guard it:

```python
import importlib.util, os
from hypothesis import HealthCheck, settings

settings.register_profile("ci", max_examples=500, deadline=None, derandomize=True, print_blob=True, database=None)
settings.register_profile("nightly", parent=settings.get_profile("ci"), max_examples=20_000, derandomize=False)
if importlib.util.find_spec("hypothesis_crosshair_provider"):
    settings.register_profile("crosshair", backend="crosshair", max_examples=200, deadline=None,
                              database=None, suppress_health_check=list(HealthCheck))
settings.load_profile(os.getenv("HYPOTHESIS_PROFILE", "ci" if os.getenv("CI") else "default"))
```

Suppressing a health check or filtering heavily with `assume()` can starve a generator; check `--hypothesis-show-statistics` and say so if you do.

HypoFuzz needs a paid license in a commercial organization; do not make it a default gate.

### 3. Exhaustive and symbolic verification

- Python has no Loom. `frontrun` explores thread and asyncio interleavings with DPOR at bytecode granularity (`frontrun pytest`; plain `pytest` skips its tests). It does not track state inside C extensions, its schedules are not stable across Python versions, and it is young. State the granularity and the explored count with the claim. No Python tool models the memory model or C atomics; that code belongs to the native checks in section 4.
- CrossHair for symbolic inputs on small, pure, deterministic kernels whose named property must hold for every input: `crosshair check --report_all --analysis_kind=asserts,PEP316,icontract,deal --per_condition_timeout=20 pkg.mod.func`. The kinds are case-sensitive, and a target is `module.func` or `file.py:LINE`, not `file.py:func`.
- CrossHair exits 0 for both "Confirmed over all paths." and "Not confirmed.", so read `--report_all` output before claiming exhaustion. Timeouts are CPU seconds; report them as the bound. CrossHair executes the code, concretizes calls into C, treats nondeterminism as unexplored, and runs on CPython only (it re-enables the GIL on 3.14t).

Keep exhaustive tools on the smallest core that must be correct. frontrun owns small thread or task protocols; CrossHair owns symbolic inputs on a pure kernel. System interleavings, deadlock, liveness, and recovery architecture are assigned under Verification architecture.

### 4. Runtime and native extensions

Pure Python has no undefined behavior for a Miri to find. The risk sits in native code and at the interpreter boundary.

- A Rust extension (PyO3 or maturin): its `unsafe`, FFI boundary, `Send` and `Sync`, and panics across FFI follow impeccable-rust (Miri, cargo careful, sanitizers, Loom). A Python test that calls into Rust says nothing about the Rust's memory safety; link the two with a differential or property test against a pure-Python reference.
- A C or C++ extension: build it with `-fsanitize=address,undefined` and run the suite on an unmodified CPython with the ASan runtime preloaded, `PYTHONMALLOC=malloc` (pymalloc hides overflows inside its own blocks), and `ASAN_OPTIONS=detect_leaks=0` (CPython leaks at exit by design). `scripts/impeccable sanitize address,undefined` does all of this in a separate environment, with pytest's `--capture=sys` (fd-level capture discards the report when the sanitizer aborts).
- Confirm instrumentation with `nm -D ext.so | grep asan`. Free-threaded builds reject `PYTHONMALLOC=malloc`. LeakSanitizer does not run on macOS arm64, and the macOS preload route breaks under some launchers, so use the toolbox.
- Valgrind as a second pass on an uninstrumented build: `scripts/impeccable valgrind [pytest args]` (memcheck, `PYTHONMALLOC=malloc`, exit 9 on an error). It suppresses the uninitialised-value reports the clang-built uv interpreters raise in their own code, so an uninitialised value an extension hands back to the interpreter can go unseen; invalid reads, writes, and frees still fail. Never combine Valgrind with an ASan build.
- `PYTHONMALLOC=debug`, part of dev mode, reports a corrupted pad byte of a Python allocator block at free time, and only then.
- Fuzz a native parser with Atheris and the sanitizers: build the extension with `clang -fsanitize=address,fuzzer-no-link` and preload `asan_with_fuzzer.so` from `atheris.path()`. This works on Linux only.
- ThreadSanitizer needs a CPython built with `--disable-gil --with-thread-sanitizer` and extensions built with `-fsanitize=thread`. It is slow and sees only the schedules that ran; reserve it for native code under free-threading, in nightly CI.
- On free-threaded CPython, importing an extension that has not declared free-threading support silently re-enables the GIL with a `RuntimeWarning`. Fail the session if `sys._is_gil_enabled()` is true, or the free-threaded job proves nothing.
- Build and test wheels for every supported CPython, including `cp314t-*` if you claim free-threading, with cibuildwheel (`CIBW_TEST_COMMAND`). That matrix owns "imports and works on each interpreter"; it is not a sanitizer.

### 5. Concurrency and async

- Threads: a read-modify-write race can lose updates even with the GIL. Run thread-sensitive tests under free-threaded 3.14t with `pytest --parallel-threads=8 --iterations=20` (pytest-run-parallel). It samples schedules with no control, replay, or shrinking; it runs tests that use mocks or capture fixtures on one thread only (listed as "not run in parallel"), so they add no race coverage; and a race with no assertion passes silently, so assert on the shared state. On GIL builds, `sys.setswitchinterval(1e-6)` raises the switch rate. Use frontrun (section 3) when the schedule must be searched.
- asyncio: prefer `asyncio.TaskGroup` to bare `create_task` (Ruff `RUF006`: the loop keeps only a weak reference to a task). Ruff `ASYNC` flags blocking calls in `async def` (`ASYNC210`, `ASYNC230`, `ASYNC251`) and an `asyncio.timeout` with no `await` (`ASYNC100`). Re-raise `CancelledError` after cleanup; no Ruff rule catches a bare `except CancelledError: pass`, so review for it and test cancellation. Pyright reports an unawaited coroutine (`reportUnusedCoroutine`), and dev mode adds where it was created.
- The asyncio slow-callback warning is a log line (callbacks over 100 ms), not a failure. Capture and assert on it if blocking matters.
- Control time and ordering in tests: `simloop` (seeded order, virtual time, simulated network faults, `--simloop-replay=SEED`; young), `trio.testing.MockClock` and `wait_all_tasks_blocked`, or `time-machine` for wall-clock time. Trio's scheduler is randomized, not systematic.
- Correctness that depends on processes, services, crashes, or retries across them belongs to TLA+ (Verification architecture), not to these tools.

### 6. Benchmarks

Cover the full performance profile:

- Pathological cases
- Micro and end-to-end
- Under, at, and over capacity
- All relevant interpreters and platforms, including 3.14t if you ship for it

Trustworthy measurements (CI should fail on regression):

- Prefer instruction or simulated-cycle metrics over wall time alone. CodSpeed's simulation mode (`pytest --codspeed`, Valgrind-based, cycles estimated from instructions and cache misses) is the Python analogue of gungraun; its gate and history live in CodSpeed's hosted service, and a local run is a smoke test.
- Turn the gate on explicitly. pytest-benchmark fails only with `--benchmark-compare=<id> --benchmark-compare-fail=mean:5%`. `pyperf compare_to` exits 0 even on a 3x regression, so pyperf measures and never gates.
- Compare old and new in the same job on the same host; on Linux, `pyperf system tune` reduces noise. Leave headroom (under 100% load).
- Gate memory too: pytest-memray `@pytest.mark.limit_memory("1 MB")` and `limit_leaks` (Linux and macOS; it measures the allocation high-water mark, not RSS).

Measure what matters, not only speed:

- Throughput and goodput (a flood of 500 responses can show high throughput with zero useful work)
- Memory (average and max)
- Latency distributions, not only the mean
- Outcomes: realistic inputs, measured outputs, compare to ground truth
- Prefer the real deployment target, not only a beefy CI box

Record how you load the system (open, closed, partly-open), which statistic you report (mean, median, histogram, CDF), and how you decide a regression. "y is greater than x" is not enough. Profile before optimizing (py-spy, memray, scalene); when a profile shows per-object overhead, prefer `slots=True` dataclasses and contiguous arrays over many small objects.

### 7. Documentation

Decisions:

- Alternatives discarded and why
- Downsides accepted and why
- Short ADRs, YADRs, or design notes

What is not there:

- Missing corner-case handling. Tell callers what the code cannot do. A silent `raise NotImplementedError`, `...`, or `pass` stub is a landmine; Ruff's `FIX` and `TD` rules find the TODOs.
- Known future optimizations
- Deliberate absence of behavior (for example, no `__hash__`, no implicit `str` conversion, no `__iter__` on purpose)

### 8. Misuse resistance

Make misuse inexpressible to the checker, and impossible at runtime where data crosses a trust boundary:

- `NewType`, not type aliases (`Meters = NewType("Meters", int)`). Checkers reject `Miles` for `Meters`; an alias is the same type to them.
- Two-phase data: decode untrusted input into a raw type that rejects unknown fields (`msgspec.Struct` with `forbid_unknown_fields=True`, or pydantic with `strict=True, extra="forbid"`), then validate into a frozen, resolved type. pydantic's default mode is lax and turns `"80"` into `80`.
- `Literal`, `Enum`, or one union type for linked arguments, so conflicting combinations cannot be constructed. Make boolean and optional parameters keyword-only (`*`; Ruff `FBT`); use `/` for parameters callers must not name.

State-machine ladder (stop at the first rung that rules out the illegal states):

- Keep `bool` for flags that are independent and not a lifecycle (`verbose` and `color`).
- Turn lifecycle bools and phase-only `Optional` fields into a union of frozen dataclasses (`Connecting | Open | Closed`), each carrying only its own fields, and `match` it with `assert_never` in the default arm. Enable exhaustiveness errors (Pyright `reportMatchNotExhaustive`, mypy `--enable-error-code exhaustive-match`).
- Nest when a flat union copies the same fields onto many variants: keep shared context on an orchestrator and delegate to a phase union (`Session(ctx, phase: AuthPhase | WorkPhase)`).
- Use typestate when the next call must be impossible until the transition runs: a phantom `Generic` parameter with `self: Rocket[Ground]` methods returning `Rocket[Air]`. The checker enforces it and the runtime does not, and Python cannot consume `self`, so the old handle stays usable; invalidate it at runtime if reuse is dangerous. Stay on a runtime union for a mixed collection or a phase read from outside data.

Idioms a type checker enforces (the strict checker runs in CI, `Any` stays out, suppressions are audited):

- `Final` and `@final` for names and classes that must not change; `@override` (3.12) on every override, required with mypy `explicit-override`.
- `@dataclass(frozen=True, slots=True, kw_only=True)` or attrs `@frozen` for values. Frozen is shallow; store tuples and frozensets, not lists.
- Return `Sequence`, `Mapping`, or a tuple rather than your internal `list` or `dict`, so callers cannot mutate your state through the result.
- `ReadOnly` (3.13) and closed `TypedDict`s (PEP 728, through `typing_extensions`) for dict-shaped data; `Protocol` for structural interfaces (`@runtime_checkable` checks method names only); `Self` for builders.
- Prefer `TypeIs` (3.13) to `TypeGuard` for narrowing; both are unchecked promises, so property-test the predicate.
- Mark removals with `@deprecated` (PEP 702, 3.13; `typing_extensions` before) so checkers flag call sites before the major release.
- Return `X | None` when absence is the whole story; raise a specific exception when the caller must branch on why. Chain with `raise ... from err` (Ruff `B904`); never catch blind (`BLE001`) or swallow (`S110`).
- Ruff finds the rest of the everyday traps: mutable default arguments (`B006`), `subprocess.run` without `check` (`PLW1510`), `shell=True` (`S602`), `print` in a library (`T20`).

### 9. Compatibility

Public surface area is a liability. Prefer:

- An explicit `__all__`, underscore-private internals, and a `py.typed` marker shipped in the wheel (`unzip -l dist/*.whl | grep py.typed`)
- Keyword-only parameters, so adding one does not break positional callers
- Protocols or your own types in signatures, not a dependency's (pydantic models, httpx types) unless the coupling is intended
- A stable, small core; document the versioning and deprecation policy for callers

Automate, and state what each run cannot see:

- `griffe check pkg -s src --against <last release tag>` gates removed objects and parameter shape changes (moved, removed, newly required, changed kind). It misses changed return, parameter, and attribute annotations, `async def` to `def`, generator to list, and behavior. A clean run is not compatibility.
- `pyright --verifytypes pkg --ignoreexternal` scores type completeness of the public surface of an installed (not editable) package; run it with the environment activated. An explicit `Any` counts as known.
- Review a stub or public-API snapshot diff before each release, and keep behavior under the differential and property tests.
- Test every Python version in `requires-python`, and pin the interpreter with `.python-version`.

### 10. Dependencies

1. Track the complete dependency closure across every deployment that matters. Lock with `uv.lock`, install with `uv sync --locked`, and for pip consumers export with hashes (`uv export --locked --no-emit-project --format requirements-txt`, then `pip install --require-hashes`). The lock pins packages, not the interpreter, OS, or system libraries, so scan what ships too: `pip-audit` inside the deployed environment, or `osv-scanner scan image <img>`, Trivy, or Grype on the container. Generate an SBOM for what ships (`uv export --format cyclonedx1.5` or `cyclonedx-py`).
2. Join against known issues: `pip-audit -r <exported requirements> --disable-pip --strict` (bare `pip-audit` audits the current environment, not the lock), `uv audit` (preview) on `uv.lock`, or `osv-scanner scan source --lockfile uv.lock`. Tools count aliases differently; compare advisories, not totals.
3. Vet for unknown issues. PyPI has no cargo-vet: no shared audit records exist. PEP 740 attestations say which repository and workflow built a file (`pypi-attestations verify pypi --repository <url> pypi:<file>`); pip and uv do not check them at install, and many packages do not publish them. A cooldown (`[tool.uv] exclude-newer = "7 days"`, Renovate `minimumReleaseAge`, Dependabot `cooldown`) delays fresh releases; `[tool.uv.audit] malware-check = true` checks packages against OSV's known-malicious advisories on sync and add. Review each new dependency by its exact name, and run deptry for declared-but-unused and transitive imports.

Publish with Trusted Publishing. `pypa/gh-action-pypi-publish` generates and uploads attestations by default; `uv publish` only uploads attestations already in `dist/`, so sign first (`pypi-attestations sign dist/*`). Be able to answer operational questions such as which deployed units still run a vulnerable transitive package.

### 11. Stagnation as a choice

Loud reminders when you are behind (Dependabot, Renovate) or a dependency is dead (PEP 792 archived, deprecated, or quarantined status, which `uv audit` reports; Renovate's `abandonmentThreshold` as a heuristic). Reduce friction:

- Auto-merge dependency bump PRs that pass tests
- Budgeted maintenance time
- Prefer upstreaming over long-lived forks
- Wrap unstable dependencies behind a stable internal facade
- Treat CPython lag the same as package lag: track end-of-life dates and run the next release candidate in a non-blocking job
- Upgrade Ruff and the type checker on a schedule; each release adds rules or diagnostics

Cost rises with every skipped upgrade cycle.

## Verification architecture

> Every important failure mode has an owner, every verifier has a reason to exist, claims match their limits, production Python is checked against the intended semantics, and independent models cannot silently drift.

More tools are not a stronger architecture. Inspect the project, assign an owner to each real failure, and add a model only when the Python checks cannot own that failure.

### Probe before recommending

- Read the project before naming a tool: `pyproject.toml` (`[project]`, `[dependency-groups]`, and `[tool.*]` sections for ruff, mypy, pyright, pytest, coverage, mutmut, importlinter, uv), `uv.lock`, `.python-version`, `src/`, `tests/`, every `conftest.py`, `noxfile.py` or `tox.ini`, `Makefile` or `justfile`, `.github/workflows/`, `py.typed`, `.pyi` stubs, native extension sources (`*.c`, `*.cpp`, `*.pyx`, `setup.py` `ext_modules`, `meson.build`, a PyO3 `Cargo.toml`), `fuzz/`, `benchmarks/`, `AGENTS.md`, `CONTRIBUTING.md`, `docs/`, and any `formal/`, `spec/`, or `proof/` tree.
- Search for state machines, reducers, events, commands, effects, workers, schedulers, queues, retry, cancel, timeouts, recovery, journals, replay, transactions, persistence, `threading`, `multiprocessing`, `asyncio`, `create_task`, locks, `ctypes`, `cffi`, C API calls, protocols, parsers, serialization, `pickle`, `eval`, and `subprocess`.
- Record verifiers already present: Hypothesis (and its profiles), CrossHair, mutmut, cosmic-ray, Atheris, frontrun, pytest-run-parallel, simloop, typeguard, beartype, deal, icontract, sanitizer or Valgrind jobs, Nagini, Dafny, TLA+, TLC, Apalache, Lean, and any written specification or model check, plus the gates already enforced: Ruff, the type checker, pip-audit or `uv audit`, Griffe, zizmor, import-linter, deptry.
- Count a tool as an owner only where it runs, in CI or another enforced gate, against that failure class. Recommend another only when a failure class below has no owner.

### Risk to owner

Assign every failure class the project actually has. This is a decision table, not a stack to install.

| Failure class | Preferred owner |
|---------------|-----------------|
| Deterministic logic | Unit, property, or differential tests |
| Rewrite, port, or optimization of working code | Differential tests against the retained old implementation or a reference; `crosshair diffbehavior` |
| Type misuse (wrong kind, `None`, unit mix-ups) | Strict type checker in CI; runtime validation where untyped data enters |
| Untrusted input | Hypothesis, then Atheris fuzzing in the toolbox |
| C or C++ extension memory safety | ASan and UBSan (`impeccable sanitize`), Valgrind, Atheris with sanitizers |
| Rust extension | impeccable-rust for the crate; a differential test for the Python surface |
| Small thread or task protocol | frontrun; pytest-run-parallel on 3.14t to expose races |
| System interleavings, deadlock, liveness, or recovery architecture | TLA+ for the design; fault and simulation tests for the Python that implements it |
| Crash persistence | Crash and fault tests; add TLA+ when recovery architecture is the risk |
| Mathematical kernel | CrossHair for a bounded symbolic check; Nagini or a Dafny-compiled kernel for contracts; Lean for a theorem, with a conformance link |
| Layering and import architecture | import-linter contracts |
| Several DSLs or frontends | Differential or conformance tests |
| Public API break | Griffe, `pyright --verifytypes`, a public-API snapshot review |
| Vulnerable or unvetted dependency (any project with dependencies) | `uv sync --locked`, pip-audit or `uv audit`, a cooldown, review of new names |
| Compromised CI workflow (any repo with GitHub Actions) | `zizmor` |
| Performance regression | Benchmark gate on non-noisy metrics |

Deterministic logic stays on tests. It becomes a mathematical kernel only when a named property must hold for every input and tests cannot close it. Add a formal tool only for a row whose preferred owner is that tool.

### TLA+

- Recommend TLA+ when correctness depends on how multiple actors interleave: workers, schedulers, queues, ownership handoff, retry, timeout, cancellation, crashes, recovery, distributed state, deadlock freedom, or liveness.
- TLA+ owns that system-level design. A retry or timeout inside one coroutine, or a trivial deterministic function, stays on Python tests and simulation.
- When a model exists, document its state variables, actions, invariants, liveness properties, fairness assumptions, bounds, abstractions, and the production Python each piece maps to.
- TLC (stable 1.7.4, in the toolbox as `tlc`) exhaustively enumerates the finite instance its configuration sets, exits 12 on an invariant violation, and checks deadlock unless `-deadlock` is passed; `-simulate` only samples. Apalache checks to a depth bound. State the bounds. None of these runs is an unrestricted proof; a checked TLAPS proof is.
- Python has no Stateright: nothing runs one actor definition as both the model and the implementation, and no Python trace-validation library was found. Link the spec to the code by replay: Apalache (`--length`) or Quint writes ITF traces, `itf-py` (young) loads them, and a pytest driver replays each trace against the implementation and asserts the expected states.

### Provers and contract verifiers

Route by the claim:

- A property over every input, explored to a CPU-time bound: CrossHair (section 3).
- Contracts checked on executed paths: deal (`@deal.pre`, `@deal.post`, `@deal.raises`, `@deal.has`; `python -m deal lint` statically, `deal.cases(f)` generates tests) or icontract. `crosshair check --analysis_kind=deal` or `icontract` searches them symbolically. `-O` and `deal.disable()` strip them, so production is not contract-checked. `deal prove` rests on an abandoned solver; do not count it.
- Contracts proved on the Python itself: Nagini 1.3.1 (Viper, Z3) checks `Requires`, `Ensures`, and `Invariant`, access permissions, termination (`Decreases`), and deadlock freedom through lock levels, for a Python subset. "Verification successful" means verified against the written spec under Nagini's encoding, not that the program is correct.
- Nagini needs Java 11+ and its own environment (it pins an old mypy), and takes tens of seconds per small function. For agents, install its MCP server with `uv tool install "nagini[mcp]==1.3.1" --with "mcp<2"` (`mcp` 2.x breaks it) and set `JAVA_HOME` in the server's `env`.
- A verified kernel shipped as Python: Dafny 4.11 (`dafny verify`, then `dafny build --target:py`) proves the Dafny source against its `requires` and `ensures`. The emitted Python is not re-verified and the glue is untested by the proof, so add differential tests between the compiled module and its callers' expectations.
- A theorem: Lean, only for a small kernel where the theorem is the point: replay algebra, effect identity or uniqueness, normalization, monotonicity, ordering, ranking, a scheduler algorithm, capability or policy composition, or another deterministic transformation that tests do not close.
- Every artifact answers one question: what theorem does this establish that Python tests do not? A weak answer means the prover is not justified.
- No tool derives Lean from Python the way Aeneas does from Rust. A Lean model of Python is a hand-written (or LLM-drafted) second model: it needs the conformance link below, and the report says so. Skip a Lean enum or step function whose only job is to mirror a Python enum or reducer.

### Semantic duplication

- Flag a Python reducer, a TLA+ transition relation, a Lean step function, a Dafny kernel, and a fixture interpreter that encode the same steps, and a property test, fuzz target, and CrossHair check that each restate one property. Four green suites can still be four different semantics.
- Classify each extra model as a necessary abstraction, a formal specification, a useful differential implementation, accidental duplication, or verification theater.
- A necessary abstraction drops detail so a different failure class can be checked, and a conformance link says what was dropped. Theater is a green run with no owner, no stated bounds, and no link to production Python.

### Conformance

- When more than one model is necessary, pin them with a canonical transition corpus, model-generated traces (ITF), trace replay, property-based equivalence, differential execution, shared fixtures, canonical serialization, or runtime assertions.
- Prefer records of `initial_state`, `event`, `expected_next_state`, and `expected_effects`, or the sequence form `initial_state`, `events[]`, `expected_states[]`, `expected_effects[]`.
- Production Python should execute those traces when that is practical. Require that executed link before treating similar sources, or two green suites, as the same semantics.

### DSL and workflow

- When the project has a workflow language or DSL, verify its meaning at the compile step into a shared intermediate representation owned by Python. The frontend is an optional producer of that IR.
- Python owns I/O, effects, networking, persistence, scheduling, workers, resource management, and runtime recovery.
- A verified frontend does not become the runtime architecture.

### Anti-drift

Follow this block on every change. In implementation mode, also copy the whole block into `AGENTS.md` or `CONTRIBUTING.md`:

> Any change to observable semantics names the verification boundary it affects.
>
> - Concurrency, interleaving, scheduling, retry, cancellation, recovery, ownership handoff, or liveness updates the system model, or the change states why that model is unaffected.
> - Executable Python behavior updates the Python verification layer. A theorem- or contract-owned kernel updates its proof. Workflow or DSL semantics update conformance or differential tests.
> - Do not clone one state machine across Python, TLA+, Lean, and a DSL for symmetry. Passing independent suites does not establish equivalence.

### Verification impact

Put this declaration in the PR description of a change to observable semantics. Adapt it to the repository and omit boxes the project cannot affect.

```text
Verification impact

[ ] Pure Python deterministic behavior
[ ] Concurrency / interleaving behavior (threads, asyncio, free-threading)
[ ] Distributed / system state model
[ ] Crash / recovery / replay behavior
[ ] Persistence semantics
[ ] TLA+ model
[ ] Proof or contract kernel
[ ] Workflow / DSL semantics
[ ] Native extension / memory behavior
[ ] Property-test / fuzz surface
[ ] Public API or types
[ ] No verification architecture impact

Reason:
Affected invariants:
Tests or proofs updated:
```

### Modes

Audit mode is the default for a verification review, and for any verifier or formal model you would add, remove, or replace without being asked. It is read-only: leave the tree unchanged on that first pass. Report:

- Architecture, risk, and current-verifier maps
- Duplication, drift, gaps, and the owner of each important failure
- Each recommendation marked REQUIRED, USEFUL, OPTIONAL, NOT JUSTIFIED, or REMOVE
- A conformance and anti-drift plan, a CI split, and a migration order

A requested code change is not an audit. Make it with the Python checks that Risk to owner assigns, recommend any new formal model instead of writing it, and fill in Verification impact.

Implementation mode starts when the user accepts that report or asks for the verification change, in this order:

1. Add missing conformance for each model the audit keeps.
2. Add the high-value Python checks the risk table already names.
3. Write down semantic ownership.
4. Strengthen a formal model only where it owns a real failure.
5. Remove accidental duplication.
6. Simplify CI.

Never remove a verifier until the guarantee that replaces it is named, in place, and passing.

### Terminology

- Name the claim with the narrowest of these that fits: type-checked, runtime-validated, unit-tested, integration-tested, property-tested, fuzz-tested, mutation-tested, dev-mode-checked, sanitizer-checked, Valgrind-checked, stress-tested, interleaving-explored, symbolically checked, contract-verified, model-checked, bounded model-checked, exhaustively enumerated under stated bounds, theorem-proven, differentially tested, conformance-tested, crash-tested, fault-tested, or simulation-tested.
- State the bounds and the assumptions next to the claim: the checker, mode, and version; examples and profile; executions or seconds and corpus; mutation score and tool; threads and iterations; tool, granularity, and schedules explored; for CrossHair, the per-condition and per-path CPU timeouts and whether it reported "Confirmed over all paths".
- If a formal model has no refinement or conformance relationship to production Python, say so in the report.
- Reserve "proof" for a theorem with stated assumptions. A test suite, a fuzz run, a CrossHair run (even "Confirmed over all paths," which holds only under its model of Python), and a bounded model check are not proofs.

## CI shape

Adapt to the project. Minimum credible set:

1. `uv sync --locked`, `ruff format --check`, `ruff check`, one strict type checker, and pytest with `strict = true`, `filterwarnings = ["error"]`, `PYTHONDEVMODE=1`, random order, and branch coverage
2. Hypothesis properties under the `ci` profile
3. At least one of: mutmut on touched code, or Atheris on parsers and codecs
4. Sanitizer and Valgrind runs on C or C++ extensions; impeccable-rust on Rust extensions; a wheel test matrix for every supported CPython
5. frontrun or free-threaded pytest-run-parallel runs gated to the modules that need them
6. Benchmark regression gate with non-noisy metrics
7. pip-audit or `uv audit` against the lock
8. Griffe and `pyright --verifytypes` on published packages
9. `zizmor` on `.github/workflows` when the repo uses GitHub Actions (template injection, excessive permissions, unpinned actions, persisted credentials); its default persona hides lower-confidence findings, so an audit runs `zizmor --persona=auditor`. It exits 11 to 14 by the highest severity found (14 = high); gate on any nonzero exit, or set `--min-severity`. `--format=sarif` and zizmor-action's default Advanced Security upload do not use those exit codes; gate on the uploaded alerts or run a plain-format step. Pin actions by commit SHA with an update path.

Item 1 runs on every project. Skip any other tool the risk table does not justify, and split the rest by cost:

- Pull request: format, lint, the type checker, tests with warnings as errors, a logged random seed, and flaky results failing the run, diff-cover, property tests that cover the diff, mutmut scoped to touched functions, CrossHair on touched pure kernels, the sanitizer run when native code changed, the audit, Griffe on published packages, `zizmor` when the diff touches a workflow, and the benchmark gate, plus small frontrun runs, small model checks, and conformance tests when those owners exist.
- Nightly: the randomized `nightly` and `crosshair` Hypothesis profiles, longer Atheris campaigns, full mutmut, a second type checker as advisory, `--count` stress and free-threaded `--parallel-threads` runs, Valgrind, large TLC state spaces, fault injection, and simulation.
- Release: the full interpreter and platform matrix when a failure class in the risk table justifies the cost, an SBOM, and Trusted Publishing with attestations.

## Review report

When finishing work under this skill, report:

- Evidence: each check run, or type-level misuse made impossible, named with a Terminology term and its bounds
- Documented: decisions and intentional gaps written down
- Deferred: what was skipped and why (follow-up if high stakes)
- Compat / deps: any new public surface or dependency hazard
- Verification: the Verification impact declaration, the owner of each failure mode this change can break, and any conformance or model update

Give each check its own evidence record, so a reader can separate what was checked from what was assumed:

```text
Claim (Terminology term):
Failure class and code boundary:
Verifier and command:
Property or invariant:
Inputs, state space, and bounds:
Assumptions:
Result, artifacts, and on failure the counterexample or minimal reproducer:
Blind spots:
```

Record only checks that ran. A clean type check is not a check of behavior.

## Anti-patterns

- Happy-path-only tests, and `pytest.raises(Exception)` with no `match`
- Wall-clock microbenchmarks on a shared machine as the sole perf signal, or a pyperf comparison treated as a gate
- Boolean soup, type aliases for distinct units, and public `list` or `dict` attributes callers can mutate
- Blanket `# type: ignore`, `Any` leaking from untyped dependencies, and pydantic's lax mode at a trust boundary
- Leaking a dependency's types (pydantic, httpx, SQLAlchemy) into a stable public API without intent
- Silent TODO debt, and forever-pinned dependency versions with no reminder
- `uv sync --frozen` or a bare `uv run` in CI; reruns without `--fail-on-flaky`; running the suite under `-O`
- Treating `mutmut run` exit 0, or a cosmic-ray run full of incompetent mutants, as a pass
- A free-threaded CI job in which an extension silently re-enabled the GIL
- Claiming this quality bar without the checks that apply (a strict type checker, property tests, sanitizers on native code), misuse-resistant types, or decision docs
- A second copy of the same state machine in TLA+, Lean, Dafny, or test fixtures with no conformance link to production Python
- Calling a CrossHair run, a bounded model check, a fuzz campaign, or a green unit suite a proof
- Adding a prover or a system model to mirror logic that unit and property tests already own
- Deleting the old implementation before the rewrite has run differentially against it
- Rerunning a failed test until it passes and reporting green
