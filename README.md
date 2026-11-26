# newmath-rgh

A structured reading guide to the **new mathematics in Levent Alpöge's Lean-verified proof of the Riemann Hypothesis** (announced 2026-11-20), plus an **independent verification receipt** produced on a 10-core Apple M1 Max.

The proof itself is 17.37 million lines of Lean with **zero comments** — names and the module tree are the only documentation. This repo maps the announcement's key ideas onto concrete locations in the library, reconstructs the proof's logical spine from `Main.lean` down, and records what a reader can and cannot responsibly claim. No Lean source is vendored; everything links to the primary sources.

## Contents

| Path | Contents |
|---|---|
| [`docs/reading-guide.md`](docs/reading-guide.md) | Full guide: headline theorems, proof spine, new mathematics, concept→file map, keystones, developments, open questions |
| [`verification/README.md`](verification/README.md) | Independent verification receipt: inputs, hashes, checker build, accepted verdict, timings, reproduction |

## The short version

- **What is proven.** RH for ζ and GRH for Dirichlet L-functions: `L(χ,s) ≠ 0` for `Re s > 1/2` for every character χ (trivial χ=1 at s=1 excepted). RH follows as a corollary (`Claim.zeta allChi`).
- **The new idea (author's remark).** Stepanov's function-field proof uses a divisor *with multiplicity*; over ℤ, multiplicity "corresponds to something like differentiating whole numbers". Replacement: **chains of theta lifts**, each mimicking one unit of multiplicity, chain length ≈ √X for a von Mangoldt sum of length X.
- **Where it lives.** `Development/AutomorphicArithmetic/ThetaCorrespondence/ThetaPairLifts/ThirdPlaceSeries` (2,246 files) feeds `CharacterBoxes` (`UpperTheory.EveryCharacter`), which yields the five exported theorems. Toy Fourier layer: `Formalism/FockGram` (multinomial-weighted Hermite inner product, `mfact m = ∏ nᵢ!`) + `Formalism/HermitianCurvature` (chain-growth recursion). Keystones: `Lemmata/FirstBlockLemma{A..K}`.
- **Verified independently.** Inputs byte-identical to MIT's (`sha256 5ea74b91…244e`); con-ron (Rust, Lean FRO) accepted all 1,468,418 declarations and justified all five theorems under `--verified`, standard three axioms only. See receipt.

## Honest caveats

- The Lean source has **no comments**; names are interpretive. Statements in this guide that go beyond literal identifiers are flagged as readings.
- This repo is by an independent reader, not the proof's author. The mathematics is Alpöge's (with Claude); the mapping is ours.
- The submission "belongs to its author and was published without a license" — we link, we do not redistribute.
- **Corrections policy:** where we published something and later found it wrong, the correction is marked, not quietly edited.

## Primary sources

- Announcement: <https://alpo.ge/remark.pdf> · Lean archive parts `lean.tar.gz.part01..14` at alpo.ge
- Community repo: <https://github.com/leanprover-community/riemann-hypothesis> (certification post: <https://leanprover-community.github.io/blog/posts/riemann-hypothesis/>)
- Verification files: <https://math.mit.edu/~drew/grh/> (Andrew Sutherland) · checkers: <https://github.com/leanprover/con-ron>, <https://github.com/leanprover/con-leche>

## License

MIT — see [LICENSE](LICENSE) and [NOTICE](NOTICE).
