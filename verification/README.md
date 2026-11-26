# Independent verification receipt

**Verdict: accepted.** The Riemann–GRH proof export was checked on a stock Apple M1 Max
(10-core CPU, 64 GB RAM, macOS 26.6.2, 2026-11-22) with `con-ron`, the Rust port of
con-leche (Lean FRO), built from source at commit master.

## Inputs (from <https://math.mit.edu/~drew/grh/>)

| file | size | sha256 (compressed) |
|---|---|---|
| `challenge.ndjson.zst` | 46,881,299 B | `c24ef99dab3556a52c7fb886e03cf22a242f3b96f82ae4cc40539e5d2f678ac6` |
| `solution.ndjson.zst` | 7,699,933,588 B | `61a8938ad5f5e57397c1c0e3b954916af546d9004ade01c3555744a681b05381` |

Decompressed: `challenge.ndjson` sha256 `bd484ac9886a96f26b1d9a7a59ebd45b7a70d102c015d4f31f9302bbdd6ab2b9`
(351 MB); `solution.ndjson` sha256 `5ea74b912147c5c93cc2b6d70cc727cf908df2f62ac6a20480a3fa02f6be244e`
(61,570,595,788 B — byte-identical to MIT's published export). Both match the published
SHA256SUMS exactly.

## Run

```
con-ron --compare --jobs=8 challenge.ndjson solution.ndjson
```

Wall time ≈ 25 minutes (parse phase is single-threaded by design; check phase uses 8 workers).

## Verdict (verbatim)

```
con-ron: accepted 51692 declarations, 5 axioms justified by the solution
         solution.ndjson (1468418 declarations accepted) (--verified)
con-ron: theorem riemannHypothesis justified by the solution
con-ron: theorem riemannZeta_eq_zero_re justified by the solution
con-ron: theorem LFunction_ne_zero_of_half_lt_re justified by the solution
con-ron: theorem LFunction_eq_zero_re justified by the solution
con-ron: theorem LFunction_eq_zero_re_of_isPrimitive justified by the solution
exit code: 0
```

Axioms used: `propext`, `Classical.choice`, `Quot.sound` only.

## Reproduction

```bash
# inputs
curl -O https://math.mit.edu/~drew/grh/{challenge,solution}.ndjson.zst
curl -O https://math.mit.edu/~drew/grh/SHA256SUMS && shasum -a 256 -c SHA256SUMS --ignore-missing
zstd -d challenge.ndjson.zst && zstd -d solution.ndjson.zst

# checker (needs Xcode CLT; Rust via rustup)
git clone --depth 1 https://github.com/leanprover/con-ron && cd con-ron
cargo build --release
./target/release/con-ron --compare --jobs=8 ../challenge.ndjson ../solution.ndjson
```

Note: the decompressed export is 61.6 GB; ~70 GB free disk is required. The exports were
deleted from our machine after successful verification and can be re-downloaded any time.

## Environment notes

- The checker is CPU-only; it does not use a GPU on any platform.
- A first `cargo build` requires accepting the Xcode license (`sudo xcodebuild -license accept`).
- MIT's page also covers independent checks (nanoda, con-leche, lean4lean, leanchecker) and
  reports the same accept verdicts; this receipt is a separate reproduction, not a substitute
  for theirs.
