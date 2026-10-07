# Performance iteration

Follow this when the job is to make working Python faster or leaner: an optimization run, a speedup target, or beating another implementation. Speed work is a search. Measure, change one thing, prove it is still right, race it against the baseline, keep or revert, and repeat until the gains converge.

The rules here come in three strengths; name in the report which applied.

- **Firm:** the benchmark contract (section 6 of SKILL.md) and the oracle. A change that breaks either one is not a result. Report it as a failed experiment.
- **Default:** the numbers below. Use them unless the user sets others.
- **Judgment:** which hypothesis to try next, and whether a gain is worth its code. Base the call on measurements and write down the reason.

## 1. Set up

Finish setup before changing production code.

1. Name the primary metric: wall latency, CPU time, throughput, goodput, requests/sec, allocations, allocated bytes, peak memory, RSS, startup/import time, serialization time, event-loop latency, or tail latency. Name the secondary metrics that must hold.
2. Name the oracle. For an optimization this is the old implementation kept importable (section 2 of SKILL.md), a reference implementation, or properties and invariants. When speed can trade against output quality (accuracy, loss, compression ratio), quality becomes a metric with its own tolerance. Compare what the semantics require, not only return values: exceptions, mutated arguments, ordering, emitted events, filesystem and database effects, and floating-point tolerance.
3. Write the workload matrix from real use: small, typical, large, and pathological inputs; cold and warm; sparse and dense where relevant; success and error paths where relevant; low and high concurrency where relevant. The matrix is part of the contract.
4. Commit the benchmarks and the matrix. That commit is the **baseline**, and the contract is frozen there.
5. Measure the baseline. Check out the baseline commit in its own worktree and run the matrix with the same command you will use on candidates, through `scripts/impeccable bench-guard <baseline> -- <command>`. Repeat until the median and spread are stable. Record the interpreter (implementation, exact version, free-threaded or not), platform, dependency versions, and inputs.

| Default | Value |
|---------|-------|
| Target | at least 1.2x on the primary metric, against the baseline |
| Regression | at most 3% on any workload in the matrix |
| Memory | at most 5% growth in peak memory |
| Quality | within the baseline's run-to-run variation |

The target is a floor, not a stopping point. When the contract and the oracle cannot both hold at the target, report the best result that keeps them.

## 2. Loop

Profile before you guess.

- **CPU time:** `py-spy record -o out.svg -- <command>` (works under `ptrace_scope=1` because the target is its child) or `py-spy record --pid <pid>` (needs `ptrace_scope=0`, root, or `CAP_SYS_PTRACE` on a host like a shared CI box). `--native` merges native frames; `--idle` includes idle threads; `--gil` samples only frames holding the GIL.
- **Allocations:** `memray run script.py`, then `memray flamegraph`, `memray stats`, or `memray tree`. `--native` tracks native frames, `--follow-fork` child processes, `--trace-python-allocators` the Python allocator domain. memray reports lifetime churn and peak live bytes as different numbers: do not present total allocated as memory usage.
- **Line-level CPU and memory:** `scalene run prog.py` then `scalene view` — it splits Python and native time per line. Heavier than py-spy; use it when you need per-line attribution rather than a cheap stack sample.
- **`cProfile`** is a last resort: deterministic tracing added about 2.5x overhead to a ~100us function in verification, so its per-function times exaggerate call-heavy code. Use it for call counts and structure, not for trusting the milliseconds.
- **Startup and import cost:** `python -X importtime -c "import pkg"`, and `python -m timeit` or `hyperfine` on the cold process for end-to-end startup.
- **`perf`** needs `perf_event_paranoid` under 1 and a CPU PMU; it is unusable on many shared VMs, so never make it the only profiler. CPython 3.12+ has `-X perf` (and 3.13+ `-X perf_jit`) trampoline frames that let perf name Python frames where perf can run.

Write each **hypothesis** down before trying it: the hot boundary, the profile evidence that it is hot, the mechanism, the expected size, the correctness risk, and the measurement that would refute it. Rank by expected value. Search across these lenses, in this order:

1. **Algorithm.** Accidental O(n^2) (`item not in list` inside a loop), repeated scans, repeated sorting, N+1 queries, redundant parsing, duplicated serialization, repeated regex compilation (`re.compile` once, not per call), repeated filesystem or network access, and computation that can safely be cached. A better algorithm outranks clever Python.
2. **Work moved out of Python.** Prefer C-backed primitives already inside the interpreter and the project's declared dependencies: builtins (`set`, `dict.fromkeys`, `sorted` with `key`, `bisect`, `heapq`), `str.join` over `+=` concatenation (smaller than folklore claims — CPython resizes a refcount-1 accumulator in place, so a local `out +=` loop is already amortized and join's win is often modest), comprehensions and `map`/`filter` where they read well, `re` internals, `array`, `collections`, `itertools`, `struct`/`memoryview`, and vectorized libraries such as NumPy only where the project already carries them. A new dependency — NumPy, pandas, polars, a native extension — is part of the optimization's cost; do not add one for a small loop.
3. **Allocation and churn.** Confirmed by memray first. Look for temporary lists, dicts, and tuples; intermediate strings; repeated encode/decode; object-heavy representations; dataclass or dict creation in hot paths; and materialized collections where streaming works. Remedies — generators, iterators, tuples, `slots=True`, arrays or `memoryview`, reuse and batching — are hypotheses, not laws. A generator is not always faster; it trades allocation for per-element overhead. Measure.
4. **Lookup and dispatch.** Only for code the profile proves extremely hot: repeated attribute or global lookup, dynamic dispatch, expensive descriptors, avoidable abstraction. Local-binding and attribute-hoisting tricks shave nanoseconds; do not trade readability for them without measured need.
5. **I/O.** Distinguish CPU work from waiting: batching, connection reuse, round trips, N+1 queries, synchronous I/O inside async code, bounded concurrency, backpressure, coalescing, caching, serialization cost. `asyncio` is not automatically a performance optimization — it moves waiting, not work.
6. **Concurrency.** asyncio, threads, processes, free-threaded CPython, and native extensions that release the GIL are different machines. "Python threads cannot run CPU work concurrently" is false on free-threaded builds, and "free-threaded means just add threads" is equally false: measure contention, synchronization, allocator behavior, native dependencies, and scaling, and require throughput or goodput, not thread count. `multiprocessing` pays serialization and IPC per task; on free-threaded CPython `sys._is_gil_enabled()` can also flip at import when an extension has not declared support, which silently changes the machine the benchmark runs on.
7. **Memory and GC.** Allocations, peak live bytes, RSS, native-allocator memory, leaks, and retained objects are different quantities; a `gc` threshold or `gc.freeze()` change is a real hypothesis, not a detail to omit. `tracemalloc` diffs attribute allocation by line inside a test.
8. **Startup and import.** Module-level work, unnecessary imports, dependency initialization, plugin discovery, and filesystem scans are their own workload. Lazy imports move cost rather than remove it; use them when the moved cost lands somewhere the user does not feel.

For each hypothesis, run the smallest experiment that tests it:

1. Confirm in the profile that the path is hot. If it is not hot, do not optimize it.
2. Make one change.
3. Run the oracle. A mismatch reverts the change. For a rewritten function, run the differential test suite — generated inputs through old and new, comparing results, exceptions, and effects — not only the example tests. A fast path guarded by a cheap predicate (`x.isalpha()`, a length check, a cache) must hold for every input the predicate admits: `str.isalpha()` and `str.islower()` admit Unicode letters and non-cased characters that a byte-level or ASCII-only reading of the slow path would not.
4. Race the change against the baseline through bench-guard. There is no off-the-shelf interleaved A/B harness for Python: use pytest-benchmark's `--benchmark-compare` with a saved baseline (or a literal saved file for cross-interpreter races), `pyperf compare_to` for process-level statistics, or `hyperfine` for whole-command comparisons, and alternate runs when the machine drifts. Benchmarks in one pytest session share a heap: making an earlier test faster changes the allocator and GC environment later benchmarks measure in, so an "improvement" on code the edit did not touch is contamination, not speed. Answer it by comparing per-workload runs (`-k`) of baseline and candidate code, then claim per workload.
5. Measure secondary metrics: peak memory with pytest-memray's `limit_memory`, churn with memray, RSS where it matters, quality where it applies.
6. **Keep** the change when its gain clears the measured noise, every workload stays within tolerance, secondary constraints hold, and the gain pays for its code. Commit it on its own with its numbers, so the gain stays attributable. Otherwise **revert** it.
7. Append the result to the **run log** (`perf-log.md`, unless the user names a place): hypothesis, profile evidence, change, baseline, candidate result, spread, memory impact, correctness result, KEEP or REVERT, and the reason. A reverted idea stays in the log so later rounds know it was tried.

A kept change that adds a dependency, a native extension, threads, or processes also passes the check the failure class owns (section 10 of SKILL.md for dependencies, section 4 for native, section 5 for concurrency) before it counts.

Helper agents can widen the hypothesis search. They read code and propose hypotheses. Only the main loop edits the tree and runs benchmarks, because benchmarks run one at a time on the machine.

## 3. Converge

Two rounds in a row with low-single-digit cumulative gain (under 3% by default) mean the incremental loop has **converged**. On noisy wall-clock benchmarks, read the gain against the measured spread, not the raw mean: a 2% median shift inside a 15% spread is not a gain.

Then run one **breakthrough** pass, which changes the shape of the computation instead of its details: a better or bespoke algorithm for the real workload, removing whole passes, fusing operations, batching, moving computation into a database or native primitive already in the project, a different data representation, incremental computation, caching at a better boundary, vectorization where the project justifies it, parallel decomposition, or dropping generality the contract does not need. A native rewrite — PyO3/maturin, a C extension, Cython, Numba — is one breakthrough option among many, not the default one. It adds build, packaging, portability, debugging, and supply-chain cost; choose it only when the profile proves Python execution itself is the bottleneck, and hand the Rust portion to impeccable-rust when PyO3 is used. A breakthrough is farther from the baseline, so it gets the heavier oracle: the differential suite under a bigger generated corpus or a fuzzer, plus `crosshair diffbehavior` on pure functions, not only the short random sample.

Stop when the breakthrough pass also converges and no untried hypothesis of a different kind remains. An optional cleanup pass can shrink the code afterward, with every workload held within tolerance.

## Refused

The run refuses these, and any "win" built on one is reported as a failed experiment:

- **Benchmark cheating.** Different workloads, inputs, or rounds per side; warmed caches or reused fixtures only on the candidate; mocks that remove real work on one side; benchmark edits shipped with the candidate without re-baselining; reporting the favorable workloads and hiding the regressing ones; comparing different interpreters or environments; lowering output fidelity, validation, or error handling to gain time.
- **Cargo-cult Python.** `__slots__` everywhere, generators everywhere, comprehensions swapped for loops or the reverse, local-binding tricks, regex tricks, manual caching of everything, asyncio for CPU work, threads for every wait, multiprocessing without IPC cost, NumPy for small collections, a native extension for an insignificant hot spot — applied without profile evidence, they are churn, not optimization.
- **False complexity claims.** Concurrency does not change O(n) work into O(1) work. Keep computational complexity, parallel execution, latency, and throughput as separate quantities in every claim.
- **Unbounded concurrency.** `await asyncio.gather(*(work(x) for x in everything))` with no concurrency limit, no connection or rate budget, no backpressure, no cancellation or failure plan is an outage, not an optimization.
- **Benchmark-only design.** Complexity bought for a tiny measured gain is still complexity; its maintenance cost goes in the report next to the gain. Performance is an engineering tradeoff, not a leaderboard.

## 4. Report

Re-run the whole matrix on the final commit against the baseline through bench-guard. Then report:

- The metric, the baseline, the final result, the speedup, and the spread, for each workload
- Memory (peak and churn), quality, and interpreter details, compared with the baseline
- Each kept change and why it is faster
- The reverted experiments, from the run log
- The bench-guard output, including any waiver and its reason
- The evidence records SKILL.md asks for, and the blind spots

Claim the speedup only for the workloads and bounds you measured, on the interpreter and platform you measured. A speedup on 3.13 with the GIL does not transfer to 3.14t, and a gain measured without I/O says nothing about a service that waits.
