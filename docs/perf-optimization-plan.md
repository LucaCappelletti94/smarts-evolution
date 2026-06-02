# Performance optimization plan

This plan addresses the optimization opportunities found during the codebase review. Each change is implemented and measured in isolation so that every speed delta is attributable to one cause. Item 1 (the `QueryMol::smarts_len` allocation in `is_near_smarts_len_limit`) is tracked separately in `upstream-smarts-len-proposal.md` because it depends on an upstream release. This document covers items 2 through 5, plus item 1's downstream integration step once the upstream method lands.

## Optimization candidates

| Id | Change | File | Hypothesis |
| --- | --- | --- | --- |
| 1 | Replace `query.to_string().len()` with `query.smarts_len()` | `src/operators/mutation.rs` | Removes a per-mutation-step string allocation in the hottest operator. |
| 2 | Compute `operator_weight` once per step instead of twice | `src/operators/mutation.rs` | `sample_mutation_operator` evaluates every operator's eligibility and weight twice, so computing once halves that work per step. |
| 3 | Reuse screening scratch in `screening_proxy_of` | `src/fitness/evaluator.rs` | Guided mutation calls this once per candidate, allocating a fresh `Vec` and `TargetCorpusScratch` per fold each time. |
| 4 | Memoize mutation direction per parent in `offspring_pair_jobs` | `src/evolution/runner.rs` | Parents cycle, so `confusion_for_phenotype` (a full pass over all samples) is recomputed for repeated parents in the serial pre-parallel phase. |
| 5 | Cache the compiled query lazily in `SmartsGenome` | `src/genome/pattern.rs`, `src/fitness/evaluator.rs` | Every uncached evaluation runs `CompiledQuery::new(query.clone())` (around 484 ns), cloning the query graph and recompiling even for genomes evaluated more than once. |

## Measurement methodology

The goal is a defensible before-and-after number for every change, not a single end-of-project guess. The method below makes each delta reproducible and isolated.

### Environment control

- Build in the same profile every time. Benchmarks run under the criterion harness in the `bench` profile, which inherits the release settings (`opt-level = 3`, `lto = "thin"`, `codegen-units = 1`). Do not mix debug and release numbers.
- Run on a quiet machine with no other heavy load. Pin to a fixed set of cores and reduce scheduler noise, for example `taskset -c 0-15 nice -n -5 cargo bench --bench evolution`. Keep the same core set across baseline and candidate runs.
- Disable CPU frequency scaling or at least record the governor, because turbo and thermal throttling inflate variance. Note that several benches use rayon, so core count and pinning materially affect the parallel paths.
- Run each comparison back to back on the same boot to avoid cross-session drift.

### One change at a time

- Each optimization lives on its own commit (and its own branch off the shared baseline). No two optimizations are measured together.
- Establish a baseline first and save it with criterion: `cargo bench --bench evolution -- --save-baseline pre`. After a change, compare against the immediately preceding baseline: `cargo bench --bench evolution -- --baseline pre`. Criterion reports the relative change with a confidence interval and flags improvement or regression.
- Re-baseline after each accepted change so the next change is measured against the new state, never against the original. This keeps every reported delta local to its own change.
- Record results in the results table below before moving on.

### Correctness gate before timing

A change is only eligible for a timing decision if it does not alter behavior:

- `cargo test --all-features` stays green.
- `cargo clippy --all-targets --all-features -- -D warnings` stays clean.
- Evolution output is unchanged for fixed seeds. The `evolution` and `example_evolution` benches run with fixed seeds and fixed configs, and `example_evaluator_batch_modes` already asserts that the scalar, indexed, and indexed-batch evaluators agree. For changes that touch evaluation (item 5) or mutation sampling (items 2 and 3), add a temporary assertion or a small golden test that the best SMARTS and best MCC for a fixed seed match the baseline exactly. Performance that comes from a behavior change is rejected.

Items 2 and 3 must not change the random number stream consumed during mutation. Compute-once and scratch-reuse refactors are required to draw the same values in the same order so that seeded runs reproduce bit for bit. This is part of the correctness gate, not an afterthought.

### Acceptance rule

Accept a change when:

- The targeted microbenchmark improves by more than criterion's noise band (non-overlapping confidence intervals, treated as significant), and
- No guard metric regresses beyond noise, and
- The correctness gate passes.

If a change shows no significant improvement, or it regresses a guard metric, revert it and record why. Negative results are kept in the table.

### Harness gaps to close first

The current benches do not exercise two of the target code paths in isolation, so they cannot measure items 3 and 4 one at a time. Close these gaps before optimizing, as their own preparatory commits (these add benchmarks only, no behavior change):

- Guided mutation and screening proxy (needed for item 3). `operators/mutate` calls the non-guided `SmartsMutation::mutate`, which never touches the evaluator. Add a `screening_proxy_of` microbench over the example datasets and a guided-mutation microbench that drives `mutate_guided` with the screened selector, mirroring `build_offspring_pair`.
- Offspring assembly (needed for item 4). Add a `build_offspring` or `offspring_pair_jobs` microbench so the serial direction computation is visible without running a whole generation. Use the larger example datasets, since the cost of `confusion_for_phenotype` scales with sample count.

Adding these benches is itself a measured step: they must compile, pass the correctness gate, and produce stable numbers before being used as baselines.

## Per-item execution

For each item: state the hypothesis, make the smallest change that tests it, measure the primary metric, check the guard metrics, and decide.

### Item 2: compute operator weights once per step

- Change: in `sample_mutation_operator`, evaluate each operator's `operator_weight` a single time, store the per-operator weights in a fixed-size array indexed by `MutationOperator::ALL`, sum them, then roll against the stored weights. Today the weights are computed once for the sum and again in the selection loop, and `operator_weight` calls `is_eligible`, which is O(atoms) for several operators.
- Primary metric: `operators/mutate`.
- Guard metrics: `evolution/*` and `example_evolution/*` (end to end), to confirm the win survives in context.
- Correctness: the random roll must consume the same value and select the same operator as before for a given RNG state. Verify with the seeded golden check.
- Risk: low. Pure compute deduplication.

### Item 3: reuse screening scratch in `screening_proxy_of`

- Change: thread a reusable `candidates` buffer and `TargetCorpusScratch` through `screening_proxy_of` instead of allocating per fold on every call. Provide an internal entry point that accepts caller-owned scratch so the guided-mutation selector can reuse one allocation across all candidates in a proposal set. Clear, do not reallocate, between folds and candidates.
- Primary metric: the new `screening_proxy_of` and guided-mutation microbenches.
- Guard metrics: `example_evolution/*` (guided offspring run end to end), and `evaluator/objective_of` to confirm the non-guided evaluation path is untouched.
- Correctness: candidate ordering and counts per fold must be identical, so the same candidate is selected for a fixed seed.
- Risk: low to medium. Scratch lifetime and clearing need care to avoid stale state across candidates.

### Item 4: memoize mutation direction per parent

- Change: in `offspring_pair_jobs`, compute `mutation_direction_for_parent` once per distinct parent index and reuse it for the repeated cycles, rather than recomputing `confusion_for_phenotype` (a full pass over every sample) each time a parent is reused. A small map from parent index to direction, or precomputing directions for the selected parents up front, both work.
- Primary metric: the new `build_offspring` microbench on the largest example dataset, where the per-sample cost is highest.
- Guard metrics: `example_evolution/*` end to end. The relative win grows with sample count and shrinks with population size, so report across dataset sizes.
- Correctness: directions per parent are unchanged, so offspring are identical for a fixed seed.
- Risk: low.

### Item 5: cache the compiled query in the genome

- Change: store the compiled query lazily on `SmartsGenome`, for example an `Arc<OnceLock<CompiledQuery>>` populated on first evaluation, and have the evaluator reuse it instead of `CompiledQuery::new(query.clone())`. This removes both the recompilation and the query-graph clone for any genome evaluated more than once (elites that fell out of the fitness cache, and repeated microbench evaluations) and for the screening path where applicable.
- Primary metric: `evaluator/objective_of` and `example_evaluator/objective_of`, which re-evaluate the same genome repeatedly and so directly expose the saved compile and clone.
- Guard metrics: `operators/build_genome`, `operators/crossover_pair`, and `operators/mutate` must not regress from the added field and its clone cost, and peak memory should be checked because every live genome now may hold a compiled query. Also run `evolution/*` and `example_evolution/*`.
- Correctness: identical match results and identical evolution output for fixed seeds. The compiled query is a pure function of the canonical query already stored, so caching it changes nothing observable.
- Risk: medium. It changes `SmartsGenome` size and clone semantics, and genomes are cloned heavily (into offspring jobs and parent pairs). The `Arc<OnceLock<...>>` keeps the clone cheap (a pointer bump) and shares the compiled result across clones, but the guard metrics above must confirm there is no net regression in the operator path.

## Sequencing

1. Establish the shared baseline. Save it as `pre`. Confirm the correctness gate is green at this point.
2. Add the harness benchmarks for the guided-mutation, screening-proxy, and offspring-assembly paths (preparatory, behavior-neutral). Re-baseline.
3. Item 2 (low risk, isolated to mutation sampling). Measure, decide, re-baseline.
4. Item 3 (depends on the screening and guided benches from step 2). Measure, decide, re-baseline.
5. Item 4 (depends on the offspring bench from step 2). Measure, decide, re-baseline.
6. Item 5 (most invasive, kept last so its guard-metric checks run against an otherwise stable tree). Measure, decide, re-baseline.
7. Item 1, once `QueryMol::smarts_len` is released upstream: bump the three lockfiles, apply the one-line downstream change, and measure `operators/mutate` plus end to end. This is the integration step for the upstream proposal.

Each numbered step is one commit (or a small preparatory commit plus one change commit), measured against the immediately preceding baseline.

## Results table

Fill one row per measured step. Keep rejected changes in the table with the reason.

| Step | Change | Primary metric | Baseline | New | Delta (CI) | Guard metrics ok | Correctness gate | Decision |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | Baseline `pre` | n/a | n/a | n/a | n/a | n/a | green | baseline |
| 2a | Add guided/screening/offspring benches | n/a | n/a | n/a | n/a | n/a | green | prep |
| 2 | Item 2 | operators/mutate | | | | | | |
| 3 | Item 3 | screening_proxy_of | | | | | | |
| 4 | Item 4 | build_offspring | | | | | | |
| 5 | Item 5 | evaluator/objective_of | | | | | | |
| 1 | Item 1 (upstream landed) | operators/mutate | | | | | | |

## Reporting

For each accepted change, record the criterion comparison output (relative change and confidence interval) for the primary metric and each guard metric, and confirm the correctness gate. The final summary is the sum of accepted per-item deltas, plus an independent end-to-end measurement on `example_evolution/*` comparing the original baseline against the final tree, to confirm the isolated wins compose as expected and that nothing regressed in aggregate.
