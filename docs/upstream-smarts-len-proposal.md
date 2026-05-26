# Upstream proposal: `QueryMol::smarts_len`

## Summary

Add an allocation-free method `QueryMol::smarts_len(&self) -> usize` to `smarts-rs` that returns the byte length of the query's SMARTS rendering. The returned value must equal `self.to_string().len()` for every query, but it must compute that length without allocating the rendered `String`.

## Motivation

The downstream crate `smarts-evolution` gates several mutation operators on whether a candidate query is close to a maximum SMARTS length. Today that check is written as:

```rust
fn is_near_smarts_len_limit(query: &QueryMol) -> bool {
    query.to_string().len() * 100 >= MAX_SMARTS_LEN * NEAR_SMARTS_LEN_LIMIT_PERCENT
}
```

This runs on the innermost mutation loop. It is evaluated once per mutation step, for every attempt, for every candidate proposal, inside the genetic algorithm's offspring generation. Each call serializes the whole query graph into a freshly allocated `String` and then discards everything except `.len()`. In the `operators/mutate` benchmark the mutation operator is the dominant cost (around 85 microseconds per call on the reference machine), and a large share of that is throwaway serialization for length checks. The allocation is pure overhead: the rendered text is never used.

Only `smarts-rs` can remove this allocation cleanly, because only `smarts-rs` owns the serialization logic. A downstream reimplementation would duplicate the writer and drift from the canonical rendering. A method on `QueryMol` keeps the length definition tied to the one source of truth (the `Display` implementation) and stays correct across any future change to SMARTS rendering.

## Proposed API

```rust
impl QueryMol {
    /// Returns the byte length of this query's SMARTS rendering.
    ///
    /// The result is always equal to `self.to_string().len()`, but it is
    /// computed without allocating the rendered string.
    #[must_use]
    pub fn smarts_len(&self) -> usize;
}
```

Semantics and invariants:

- `q.smarts_len() == q.to_string().len()` for every `QueryMol q`, including canonical and non-canonical queries, recursive queries, and multi-component queries.
- The method does not allocate a `String` for the output and does not change observable rendering.
- The method is `no_std` compatible (it uses only `core::fmt`).

## Reference implementation

The existing `impl fmt::Display for QueryMol` already renders into a `core::fmt::Formatter`. We can drive that same rendering into a sink that only counts bytes, so the output text is never materialized:

```rust
impl QueryMol {
    #[must_use]
    pub fn smarts_len(&self) -> usize {
        use core::fmt::{self, Write};

        struct LenCounter(usize);

        impl Write for LenCounter {
            fn write_str(&mut self, s: &str) -> fmt::Result {
                self.0 += s.len();
                Ok(())
            }
        }

        let mut counter = LenCounter(0);
        // Display rendering never fails when writing into a counting sink.
        let _ = write!(counter, "{self}");
        counter.0
    }
}
```

This reuses the canonical `Display` path verbatim, so the value can never drift from `to_string()`. It removes the output `String` allocation that `to_string()` performs on every call.

Note on internal buffers: `write_query_mol_nonrecursive` builds internal traversal vectors (`query_writers`, `tasks`) inside `QueryDisplayWriter`. The counting sink above removes the output-string allocation but not those internal traversal allocations. That is acceptable for the immediate goal, since the output string is the allocation the downstream hot path pays for today. If a fully allocation-free length is wanted later, the writer can be refactored to reuse a caller-provided scratch buffer, and `smarts_len` can be layered on top of that. That refactor is out of scope for this proposal.

## Optional companion API

If `smarts-rs` wants a general primitive for callers that stream SMARTS into a reused buffer (for example to avoid repeated `String` allocations across many queries), expose:

```rust
impl QueryMol {
    /// Writes this query's SMARTS rendering into `out`.
    pub fn write_smarts<W: core::fmt::Write>(&self, out: &mut W) -> core::fmt::Result {
        write!(out, "{self}")
    }
}
```

`Display`, `smarts_len`, and `to_string` can all be expressed in terms of this primitive. This is a convenience and is not required for the length use case.

## Tests

Add a property-style test that compares `smarts_len` against `to_string().len()` over a representative corpus, covering canonical and non-canonical forms:

```rust
#[test]
fn smarts_len_matches_to_string_len() {
    for smarts in SAMPLE_QUERIES {
        let query = QueryMol::from_str(smarts).unwrap();
        assert_eq!(query.smarts_len(), query.to_string().len(), "{smarts}");

        let canonical = query.canonicalize();
        assert_eq!(
            canonical.smarts_len(),
            canonical.to_string().len(),
            "{smarts} (canonical)"
        );
    }
}
```

Reuse the existing rendering and canonicalization corpora so the test tracks the same queries those suites already exercise, including recursive and multi-component cases.

## Benchmark

Add a microbenchmark comparing the new method against the current `to_string().len()` on a range of query sizes, to document the saved allocation and the per-call cost:

```rust
group.bench_function("smarts_len", |b| {
    b.iter(|| black_box(query.smarts_len()));
});
group.bench_function("to_string_len", |b| {
    b.iter(|| black_box(query.to_string().len()));
});
```

Expected result: `smarts_len` is faster and allocation-free, with the gap widening for larger queries.

## Downstream migration

Once released, `smarts-evolution` replaces the body of `is_near_smarts_len_limit` (`src/operators/mutation.rs`) with:

```rust
fn is_near_smarts_len_limit(query: &QueryMol) -> bool {
    query.smarts_len() * 100 >= MAX_SMARTS_LEN * NEAR_SMARTS_LEN_LIMIT_PERCENT
}
```

This is a behavior-preserving change because `smarts_len` equals `to_string().len()` by construction. The downstream lockfiles (root, `apps/web`, `apps/web-worker`) are then bumped to the `smarts-rs` commit that contains the method.

## Rollout

1. Land `smarts_len` (and optionally `write_smarts`) in `smarts-rs` with the test and benchmark above.
2. Tag or note the commit on the `main` branch that `smarts-evolution` tracks.
3. Bump the three `smarts-evolution` lockfiles to that commit and apply the downstream migration above as part of optimization item 1 in the performance plan.
