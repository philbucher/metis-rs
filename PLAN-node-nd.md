# Plan: add `METIS_NodeND` (nested-dissection ordering) to `metis`

**Status:** in-progress
**Owner:** philbucher
**Target:** PR against `LIHPC-Computational-Geometry/metis-rs:master`

## Goal

Expose METIS's fill-reducing nested-dissection ordering through an idiomatic
`Graph::node_nd` method. The crate already advertises itself as a "fill-reducing
matrix orderer" (package description + `ordering` keyword) but provides no
ordering entry point — only graph/mesh *partitioning*. This closes that gap.

Motivating downstream: arotau's in-tree sparse Cholesky needs a fill-reducing
reorder before symbolic factorization (`plans/improve-internal-cholesky.md`,
serial step 4).

## Non-goals

- `METIS_NodeNDP` (separator-sizes / parallel nested dissection) — possible
  follow-up, not needed for the fill-reducing-ordering use case.
- A dedicated ordering-only graph type. `node_nd` reuses `Graph`; the `ncon` /
  `nparts` passed to `Graph::new` are simply ignored (documented).
- New option types — every ordering-relevant option already exists in
  `src/option.rs` (`CType`, `RType`, `Seed`, `NSeps`, `Compress`, `CCOrder`,
  `PFactor`).

## Context

- **FFI already present.** `metis-sys` allowlists `METIS_.*`, so
  `metis_sys::METIS_NodeND` is generated. Signature:
  `int METIS_NodeND(idx_t *nvtxs, idx_t *xadj, idx_t *adjncy, idx_t *vwgt,
  idx_t *options, idx_t *perm, idx_t *iperm)`.
- **Everything to reuse exists** in `src/lib.rs`: the `Graph` struct (holds
  `xadj`/`adjncy`/`vwgt`/`options`), `slice_to_mut_ptr`, the `ErrorCode::wrap`
  result plumbing, and `option::Numbering` (`pub(crate)`).
- NodeND ignores `nparts`, `ncon`, `vsize`, `adjwgt`, `tpwgts`, `ubvec`.
- METIS requires the adjacency be that of a **symmetric** matrix with **no
  self-loops** (no `i` in `adjncy[xadj[i]..xadj[i+1]]`). `Graph::new` validates
  bounds but not self-loops — document this as a `node_nd` precondition.
- Output semantics (METIS manual): with `A` the original and `A'` the permuted
  matrix, `A'[i] = A[perm[i]]` and `A[i] = A'[iperm[i]]`.

## Design

Add one method on `Graph`, modeled almost line-for-line on `part_recursive`:

```rust
pub fn node_nd(mut self, perm: &mut [Idx], iperm: &mut [Idx]) -> Result<()>
```

- Force C numbering: `self.options[Numbering::INDEX] = Numbering::C.value()`.
- Assert `perm.len() == iperm.len() == nvtxs`.
- Early-return `Ok(())` for `nvtxs == 0` (METIS dislikes empty graphs).
- One `unsafe` call to `m::METIS_NodeND(...)` + `.wrap()?`, passing `vwgt` and
  `options` through, `null` where unset — same pattern as `part_*`.

Caller-provides-output (`&mut [Idx]`) mirrors `part_recursive` / `part_kway`,
keeping the API consistent rather than returning owned `Vec`s.

## Steps

- [ ] `Graph::node_nd` in `src/lib.rs` (after `part_kway`), with a doctest that
      asserts `perm` is a permutation of `0..n` and `iperm` is its inverse
      (structural assertions — exact orderings drift across METIS versions).
- [ ] `examples/ordering.rs` paralleling `examples/graph.rs`.
- [ ] `CHANGELOG.md` "Added" entry; short README usage snippet.
- [ ] Run the crate gauntlet across `vendored` + `use-system`:
      `cargo fmt -- --check`, `cargo clippy --no-default-features --features <f>`,
      `cargo test --no-default-features --features <f> --all`; MSRV **1.67.0**.
- [ ] Open PR against upstream `master`.

## Open questions

- API: caller-provided `perm`/`iperm` slices (chosen, mirrors `part_*`) vs.
  returning an owned `(Vec<Idx>, Vec<Idx>)`. Revisit if maintainers prefer the
  latter.
- The `ncon`/`nparts` wart for ordering-only graphs: document-and-ignore for v1,
  or propose a dedicated constructor / typestate? Raise with maintainers.
- Validate the no-self-loop precondition in `node_nd` (debug_assert) vs.
  document-only?

## Notes / log

- Built on fork `philbucher/metis-rs` off upstream 0.2.2 (edition 2021,
  MIT/Apache, MSRV 1.67, CI tests via doctests + `examples/`).
