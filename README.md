# newmath-rgh

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24569968.svg)](https://doi.org/10.5281/zenodo.24569968)

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
- **Compression note.** Certified low-rank truncation of activation-whitened weights accepts **zero** blocks at a 5% quantile-95 error bar in our experiments (spectra are flat); int4 group-wise quantization remains the only measured cheap lever (+0.0037 PPL on MLP blocks). Details in `docs/reading-guide.md` §8–9 (corrections marked).
- **Where it lives.** `Development/AutomorphicArithmetic/ThetaCorrespondence/ThetaPairLifts/ThirdPlaceSeries` (2,246 files) feeds `CharacterBoxes` (`UpperTheory.EveryCharacter`), which yields the five exported theorems. Toy Fourier layer: `Formalism/FockGram` (multinomial-weighted Hermite inner product, `mfact m = ∏ nᵢ!`) + `Formalism/HermitianCurvature` (chain-growth recursion). Keystones: `Lemmata/FirstBlockLemma{A..K}`.
- **v1.2.0:** §13 cites the rewrite's own GUIDE/INDEX directly for `Spanning` and `Determinant` (theorems A/B/C, `star_inequality`, `D.seesaw_identity`, `Exchange.exchange_identity`, with their literature), and flags our earlier √X claim as announcement-only.
- **v1.3.0:** §14 covers `UpperBound` (the composite lift) with direct citations: `D.seesaw_identity` and its unfolding, `Divisibility/LocalDivisibility` ("the chain regains the predicted power of p, up to about one lift's loss"), `ElementaryDivisors/RootLemma` (naming "Stepanov's count", citing Bombieri 1973), and `Seesaw/GluedLevel` (Nikulin).
- **New reading target (2026-11-25).** Luke Sernau (Anthropic) released a 300k-line "readable" rewrite (76k compact) of the same verified proof under Apache-2.0, verified by Sutherland on 2026-11-26; its `src/New` (306 files, ~122k lines) is the argument proper — see §12 of the guide.
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
