# The New Mathematics of the Riemann–GRH Proof — reading guide

> Independent reading guide. Sections 1–6 and 10–11 map the proof's structure and the
> author's stated ideas onto the Lean library. Sections 7–9 are engineering extrapolations
> and experiments kept for transparency; treat them as a separate discussion.
> Corrections are marked, not silently edited. No Lean source is distributed here.


**Subject:** Levent Alpöge's 2026 proof of the Riemann Hypothesis (with Claude), as read from the formalized Lean library
**Companion materials:** `~/rh-verify/` — verified proof export, con-ron checker, full source clone `rh-source/`
**Compiled:** 2026-11-22

---

## 1. What is proven — the five headline theorems

All stated in `src/Challenge.lean` in pure Mathlib terms (this is what MIT and the Lean community compared the proof against):

| Theorem | Statement |
|---|---|
| `riemannHypothesis : RiemannHypothesis` | Mathlib's `RiemannHypothesis` — every non-trivial zero of ζ(s) has Re(s) = ½ |
| `riemannZeta_eq_zero_re : RiemannZetaZerosOnLine` | ∀ s, ζ(s) = 0, s not a negative even integer, s ≠ 1 → Re(s) = ½ |
| `LFunction_ne_zero_of_half_lt_re : DirichletLNonvanishingRightOfHalf` | L(χ, s) ≠ 0 for Re(s) > ½, for every modulus N and Dirichlet character χ (except trivial χ = 1 at s = 1) — **the master statement** |
| `LFunction_eq_zero_re : DirichletLZerosOnLine` | every zero of L(χ, s) in 0 < Re(s) < 1 sits on Re(s) = ½ |
| `LFunction_eq_zero_re_of_isPrimitive : PrimitiveDirichletLZerosOnLine` | same, restricted to primitive characters |

Note the shape: **RH for ζ is a corollary of the Generalized RH for Dirichlet L-functions**, which is proven first. The proof literally derives ζ from characters (`Claim.zeta allChi`).

### Local verification receipt (this machine, 2026-11-22)

- Inputs: `challenge.ndjson.zst` (47 MB), `solution.ndjson.zst` (7.7 GB) from `math.mit.edu/~drew/grh/`
- SHA-256 of decompressed exports match MIT's published hashes exactly (`bd484ac9…` / `5ea74b91…`), sizes exact (`61,570,595,788` bytes)
- Checker: `con-ron` (Rust port of con-leche, Lean FRO, built in this session from source, commit master)
- Verdict: **exit 0 — accepted.** All five theorems "justified by the solution"; 51,692 challenge declarations and 1,468,418 solution declarations accepted in `--verified` mode; only axioms `propext`, `Classical.choice`, `Quot.sound`.
- Wall time: ~25 min on an M1 Max (10 cores, 64 GB RAM).

---

## 2. The proof's logical spine (as encoded in `src/Main.lean`)

```
DirichletLNonvanishingRightOfHalf  =  UpperTheory.FinalTheorems.everyCharacter
        ↓                                            ↑
Claim.zeta / Claim.strip                    Galois ↔ automorphic bridge:
        ↓                                   UpperTheory.Gal.Int.galoisLocalGlobalStrong_recParameter
RiemannHypothesis,                          + UpperTheory.One.vwin_of_row
DirichletLZerosOnLine                       → LiteratureTheorems.every_character_of_literature_of_vWin
```

The full endgame chain of import/theorem names:

1. `UpperTheory.EveryCharacter` := ∀ N χ s, Re(s) > ½ → (χ ≠ 1 ∨ s ≠ 1) → L(χ, s) ≠ 0 (in `Development/AutomorphicArithmetic/CharacterBoxes/EveryCharacter.lean`)
2. `MainConclusion.lean`:
   - `everyCharacter_of_row(h)`: Galois local–global row (`GalStar.Row0b` + `Zero.LLC.IsRecT` — "RecT" = reciprocal/local-Langlands transfer) + `One.vwin_of_row` ⇒ `EveryCharacter`, via `LiteratureTheorems.every_character_of_literature_of_vWin`
   - `everyCharacter := everyCharacter_of_row Gal.Int.galoisLocalGlobalStrong_recParameter`
3. `ThetaCorrespondence/ThetaPairLifts/ThirdPlaceSeries/EveryCharacterOfLiteratureOfVWin.lean` — the theta-correspondence layer supplying "vWin" characters
4. `ThirdPlaceSeries/TopPlug.lean` — the plug that converts the theta-lift tower into zero-freeness for all characters:
   ```lean
   theorem UpperTheory.Top.zero_free_all_of_green_v_win
     (hG : GlobalTheory.GLq.Green) (hV : UpperTheory.Top.VWin)
     (h : Nonempty (GlobalTheory.Fc.HypsAlt (fun 𝒞 => Closed 𝒞 fun Fm _ => LinkN Fm)
                             GlobalTheory.Fc.DefiniteSideCondition)) :
     Formalism.HC.ZeroFreeAll
   ```
5. `CharacterBoxes/HalfPlane.lean`: `Claim.Strip`, `Claim.Zeta` definitions + `Claim.factor_ne_zero` ("an extra Euler factor cannot vanish to the right of zero") — handles imprimitive characters by removing extraneous Euler factors
6. `Supplements/Kernels/MainTheorems.lean`: `rhOfAll : RiemannHypothesis := Claim.zeta allChi`

**Interpretation of the key new props** (names are unstated — the source has zero comments — so these are readings, not definitions):

- `GlobalTheory.GLq.Green` — the "go" hypothesis on the global side (perhaps the GRH "book" claim object)
- `UpperTheory.Top.VWin` — the central new automorphic statement produced by the theta-lift machinery ("V-window"; likely the window positivity that plays the role Stepanov's multiplicity plays over function fields)
- `GlobalTheory.Fc.DefiniteSideCondition` — the **definite-sign condition**, matching the remark's "quadric has definite sign from middle cohomology" (the qN = Mᵏ + R story)
- `Row0b / RowSrc / row` — the "row" of the argument: the Galois-side data feeding the window
- `RecTExists` — Local Langlands transfer existence ("reciprocal transfer")

---

## 3. The new mathematics proper

### 3.1 The central idea: multiplicity replaced by chained theta lifts

From the author's remark (page 1): Stepanov's method proves RH over *function fields* using a divisor **with multiplicity**. Over ℤ, multiplicity "corresponds to something like differentiating whole numbers" — it does not exist. The new move:

> examine **chaining theta lifts** … to have the same tricks recur but work in a **natively automorphic manner**. … for the von Mangoldt sum of length X, Claude's suggested **length of chain of theta lifts** … was **√X**. *(And yet, what else could it be?)*

Each theta lift contributes one unit of what Stepanov's divisor would supply via multiplicity; a chain of length ≈ √X supplies multiplicity √X, exactly the quality of cancellation needed for a von-Mangoldt sum of length X. This is the signature quantity √X appearing throughout `ThirdPlaceSeries`.

### 3.2 The "natively automorphic" determinant (definite sign)

Timeline (10/6–10/8/26): a **Stepanov determinant**; `W(1_S) ≥ 0 via a determinant` — positivity via a determinant rather than a divisor; bootstrap to multiplicity one, then more. The decisive milestone: **multiplicity 2 on O(2, n)** via the determinant, recognized as the function-field analogue of the definite sign of the left side of **qN = Mᵏ + R** (Weil–Deligne sign-only proof). In the library this is `GlobalTheory.Fc.DefiniteSideCondition`, with determinant machinery in `Formalism/SmithDeterminants/`, `Formalism/HermitianCurvature/`, and `Formalism/FormalDegreeBessel/` (`StatementBF`, `StatementDS`).

### 3.3 The Fourier-analytic toy: Hermitian Fock space and the chain recursion

The strongest candidates for the "simple, new, Fourier-analytic ideas in the toy setting":

- `Formalism/FockGram/StatementFock.lean` defines a Hermitian pairing on multivariable polynomials:
  ```lean
  def fock {R τ} [CommRing R] [StarRing R] (f g : MvPolynomial τ R) : R :=
    ∑ m ∈ f.support, f.coeff m * star (g.coeff m) * (Formalism.A.mfact m : R)
  ```
  — a Bargmann–Fock/Gram structure weighted by multinomial coefficients, with `StatementGram`, `StatementHerm`, `StatementLat` (lattice) companions.
- `Formalism/HermitianCurvature/Statement.lean` (`namespace Formalism.FourierDecayAlt`) states the chain-growth recursion:
  ```lean
  def RecursionSharp : Prop := ∃ C, ∀ r a ε e σ, … →
    σ (r+1) ≤ (1+ε)·√(1+e i)·σ i + ∑ t, a^(t−1)·σ (i−t)  ⟹
    σ (r+1)² ≤ (1 + C·(r+1)·(ε+a)) · ∏ (1 + e i) · σ 0²
  ```
  — a **multiplicative recursion bound**: the total cost of an r-step chain is controlled by one universal constant C times a product of per-step factors `∏(1+eᵢ)`. This is the bookkeeping that says each lift has a controlled "cost" (timeline item 14: "precise computation of the cost of each lead … would stop at 2/3 a priori … need for division").

### 3.4 Lemmata — the distilled keystones

`src/Lemmata/` (158 files, ~8.8k lines — 0.06% of the library) holds the elementary load-bearing blocks, e.g.:

- `FirstBlockLemmaA.goal1`: if a polynomial H ∈ MvPolynomial(σ, k) is homogeneous of degree d, then eval(c·v) H = c^d · eval(v) H — the scaling identity the Fock/Gram tower leans on (proved A…K = a "First Block" of such facts)
- `DetMonotoneStarMulSelf`, `DetNormLeProdRows` — determinant monotonicity/norm estimates (the positivity engine)
- `Cancellation`, `ClearingDenominators`, `Denominators`, `GeneralBFact`(+Alt, Bis) — the cancellation/divisibility bookkeeping ("winning more multiplicity", "cancellation of order-m terms")
- `CurryModusPonens` — a logical utility: chained implications as first-class objects (the chain-as-argument style)

### 3.5 The automorphic machinery that had to be built

Timline items 3–20 map into the tree as:

| Remark item | Library home | Files |
|---|---|---|
| theta lifts / chains | `Development/AutomorphicArithmetic/ThetaCorrespondence/ThetaPairLifts/` | ~3,000 (`ThetaPair*`) |
| the √X chain series | `…/ThetaPairLifts/ThirdPlaceSeries/` | 2,246 |
| iterated Parseval | `Foundations/GaloisRepresentations/ThetaParseval/` + `ReciprocityTheory/.../FibreParseval` | 791 |
| Bessel periods (seesaw, Rallis) | `Formalism/HermitianCurvature/BesselNonvanishingReduction`, `Formalism/FormalDegreeBessel`, Seesaw* files | 1,111 + 121 |
| Chowla–Selberg divisibility | `Development/AbelianVarietyTheory/Proofs/.../MeanLogPeriodConjugatesEqChowlaSelberg` | 8 |
| Siegel mass | `DecoratedLatticeProofs/CorankSizeHypothesisExplicitConstantSiegelMass`, `FlagMeasures/SiegelMassRatioUnimodularDefinite` | 11 |
| Iwasawa tower (pigeonhole on F^{q^n}/F^q analogue) | `GroundTheory/AutomorphicForms/*Iwasawa*`, `GroundTheory/IdeleRoots` | 6,162 |
| von Mangoldt sums | `Formalism/LandauZeroFreeBounds`, `Formalism/GL3MeanSquares/HookFormulaTorsion/WeightedVonMangoldtSumIdentity`, `CharacterTheory/AdditiveEqPolynomialVonMangoldtConvolutionOne` | 13 |
| relative trace formula, sixfold cover of GL₅ | folded into `AutomorphicArithmetic` (no literal "sixfold" name) | — |
| Weil representations / tower | `PositivityTheory/PublishedTheorems/{WeilRepresentations,TowerLemmas,UnitKilling}` | — |
| Galois↔automorphic ("Caraiani local–global dual") | `PublishedTheorems/GaloisGeometry₂/CaraianiLocalGlobalDual` | — |

**Section sizes** (current cleaned tree, 149,397 files / 15,168,600 lines):

| Section | Files | Lines | Reading |
|---|---|---|---|
| `Development/AbelianVarietyTheory` | 81,499 | ~8.1M | the bulk: theta lifts live on abelian schemes/CM theory |
| `Development/AutomorphicArithmetic` | 17,522 | ~1.8M | **the argument's home**: CharacterBoxes, ThetaCorrespondence, Foundations |
| `Development/AutomorphicSpectrum` | 13,644 | ~1.3M | spectral side |
| `Formalism` | 15,292 | 2.55M | the new formalism: FockGram, HermitianCurvature, determinants |
| `GroundTheory` | 6,667 | 845k | cohomology, ideles, p-adic Hodge, function fields |
| `ReciprocityTheory` | 3,581 | 524k | reciprocity, reductive groups, adeles |
| `CompanionTheory` | 1,946 | 193k | spectral geometry, p-adic cyclotomic fields |
| `PositivityTheory` | 355 | 44k | the endgame publications |
| `Lemmata` | 158 | 8.8k | keystones |
| `Supplements/Kernels` | 1 | 19 | the kernel exportable theorems |

### 3.6 Consequences flagged in the remark

- The method "easily extends much further": **GRH is proven; the Extended Riemann Hypothesis is expected** "with more time"
- **Timeline item 17**: "fight for a significant weakening of Vandiver … I believe vanishing of a number of K-groups of ℤ follows, and cyclicity up to 100ish maybe? I had Lichtenbaum" — an arithmetic payoff beyond primes
- The author expects "short arguments to follow fairly quickly" — the 15M-line version contains "strange suboptimal choices"
- "An astounding amount of mathematics is present … far more than I had imagined would be necessary"
- OpenAI's earlier results are credited as stepping stones (Ehrhart equality case → Monge–Ampère; the Siegel argument = "Stepanov without multiplicity")
- The qN = Mᵏ + R sign-only identity: averaging over a family via Gauss sums y(c·c′) vs y(c)y(c′) ("complete graph") simplifies to a path quadric; Gauss-sum cancellation replaced by **iterated Parseval**; "opposite signs meaning no cancellation of inverse roots, QED; vectorize to go up dimensions"; Bertini gives the definite sign from middle cohomology
- GRH "book" anecdote: Theorem 1 opens "because the reals are totally ordered, if σ ≠ ½ then σ > ½ or σ < ½"

---

## 4. What was verified about the artifact itself

- **Original submission:** commit `40e56bab` = archive `lean/` byte-for-byte; 175,187 .lean files, 1,030,531,854 bytes, 17,374,215 lines (15,531,759 non-blank). 818,755 theorem/lemma declarations; 242,067 def/structure/… declarations. **Zero comments in the entire submission.**
- **Static scan** (verification/static-scan.txt): 0 `axiom`, 0 `sorry`, 0 `admit`, 0 `native_decide`/`trustCompiler`, 0 `implemented_by`/`extern`, 0 `unsafe`, 0 `opaque`, 0 `skipKernelTC`. Only exotic touches: 5 `run_cmd`-class uses (BuildAid build helpers + one Formalism file), 37 `allowUnsafeReducibility`, 57,554 `maxHeartbeats` options.
- **The name space is the documentation**: 1,722 notation declarations formalize the authors' own mathematical dialect; the current cleaned tree has 608,839 top-level theorem/lemma names.
- Community maintenance (PRs #57/#60/#65/#66/#68) pruned unused files and shortened some proofs; the archive-compatible 4.36 port is tracked in `verification/PR66-MERGED-WORK.md`.
- Build cache: 175,187 .olean/.ilean pairs from Jack McCarthy's published archive (28.6 GB compressed / 99.3 GB extracted), hash-pinned, served via the TauCeti Cloudflare cache.

---

## 5. Open questions for further reading (honest unknowns)

1. What exactly is `VWin`? (Best guess: the "window" positivity on characters produced by the theta chain — the analogue of Stepanov multiplicity. The name never expands in the source.)
2. What is a `ThirdPlaceSeries`, and why "third place"? (Possibly the third series of the theta correspondence used; unverifiable from names alone.)
3. Where precisely does the √X chain length enter — presumably in `ThirdPlaceSeries` bounds and `Formalism/HermitianCurvature` cost analysis; a targeted import-graph trace from `MainConclusion.lean` downward would localize it.
4. How does `GlobalTheory.GLq.Green` get discharged? (Galois side: `Gal.Int.galoisLocalGlobalStrong_recParameter` — a Caraiani-local–global-style statement.)
5. The claimed Vandiver weakening and ℤ K-group vanishing — which modules? `ReciprocityTheory/Hypotheses/{Descent,GalCoh,GalCohDefs}.lean` are the candidates to read next.

## 6. File index (this investigation)

| Path | Role |
|---|---|
| `~/rh-verify/remark.pdf` | Alpöge's 4-page announcement (extracted text in §3) |
| `~/rh-verify/rh-source/` | full Lean source clone (leanprover-community/riemann-hypothesis) |
| `~/rh-verify/solution.ndjson`, `challenge.ndjson` | hash-verified proof export + independent statements |
| `~/rh-verify/con-ron/target/release/con-ron` | the checker built here; verdict: **accepted** |
| `src/Main.lean` | the five theorems + how they follow |
| `src/Challenge.lean` | Mathlib-only statements |
| `src/Development/AutomorphicArithmetic/CharacterBoxes/` | endgame: EveryCharacter, HalfPlane, ClaimStrip, MainConclusion |
| `src/Development/.../ThirdPlaceSeries/` | the theta-lift chain (2,246 files) |
| `src/Formalism/FockGram`, `src/Formalism/HermitianCurvature` | the toy Fourier framework + chain recursion |
| `src/Lemmata/` | First Block Lemmas A–K and friends |
| `verification/NOTES.md`, `verification/static-scan.txt` | community verification notes |

## 7. Transferable patterns for inference algorithms & LLM training

Readings of the formalization *as engineering patterns*, aimed at local-model stacks (Ollama, model-council, structured inference):

1. **Chain-cost certificate** (`Formalism/HermitianCurvature/RecursionSharp`). A chain whose every step carries slack `eᵢ ≥ 0` and admixture `a` has certified squared residual `σ(r+1)² ≤ (1+C·r·(ε+a))·∏(1+eᵢ)·σ₀²` — a *growth* certificate: safety requires small per-step slack, and the formula prices exactly how many steps a budget buys. *Use:* accept refinement steps while the certificate stays under target; stop at the first step whose slack would cross it. Prototype: `patterns/chain_bound_stop.py`.
2. **Multiplicity-by-chaining (√X).** A nonexistent primitive (multiplicity over ℤ) simulated by m ≈ √X chained cheap units. *Use:* cheap-verifier chains with a running `∏(1+eᵢ)` evidence bound instead of one expensive verifier.
3. **Fock/Gram structure** (`Formalism/FockGram`). Inner product `∑ f(m)·conj(g(m))·mfact(m)` = Hermite-basis similarity under Gaussian weighting. *Use:* ensemble health via Gram-matrix spectrum — effective rank (= real signal directions) and min/max eigenvalue ratio; effective rank 1 = mode collapse, high samples-per-direction = redundancy (`patterns/ensemble_collapse.py`, Hadamard pre-filter included).
4. **Hadamard bounds** (`Lemmata/DetNormLeProdRows`, `DetMonotoneStarMulSelf`). |det G| ≤ ∏ row-norms — an O(n) upper bound for log-det. *Use:* pruning filter before exact Cholesky in DPP sampling / diffusion ELBOs / GP uncertainty.
5. **Cross-implementation statement matching** (comparator/con-leche/con-ron/nanoda). Verify by having independent implementations *restate the claim in types/units/constraints* and comparing restatements — not by voting on answers. De-correlates systematic model errors; natural fit for multi-model councils.
6. **Static-scan data filters** (verification/static-scan.txt). Machine-enumerable forbidden-construct censuses (`sorry: 0, axiom: 0, unsafe: 0`) as an auditable first-pass filter over synthetic training data; LLM judging only on survivors. And: per-step kernel-verified verdicts (1.47M accepted declarations) are the highest-value process-supervision labels an artifact can carry — keep them whenever generation is formally checked or unit-tested.
7. **Names as documentation.** 818k semantic identifiers, zero comments — the artifact stayed navigable and verifiable because the naming was the documentation. Reward signal worth training for in LLM-authored code.

*Prototypes live in `patterns/` with synthetic-data tests.*

## 8. Application: speeding up LLM inference matrix computations

**Negative result (checked, not guessed):** the library contains *no* fast matrix-multiplication algorithm — zero Strassen/Karatsuba/Winograd-class content; all 17 `matMul` files are elementary. What it contains is **certificates for structured shortcuts**: theorems that let you replace an exact O(n²) matrix computation with a structured O(n log n) or low-rank one *while a nonvanishing/rank certificate holds*.

Grounded finds and their inference translations:

| Library content | Inference use |
|---|---|
| Hankel-determinant nonvanishing (`JacquetRallisHankelDetNeZero…`), log-Vandermonde bounds (`Formalism/LogVandermondeBounds`) | Toeplitz/Hankel-structured kernels (relative-position attention) admit O(n log n) FFT matvec; these det ≠ 0 / bound theorems are the *runtime guard* certifying the structured substitute is faithful |
| `Formalism.A.mfact` = product of factorials ⇒ the FockGram inner product is multinomial/Hermite-weighted Gaussian structure | Hermite-feature attention (Performer-class); the multinomial weight gives a tunable order-discount |
| `PsdBalancedFormVanishesLowRank` + `DetMonotoneStarMulSelf` | certificate-shaped pruning: Gram-spectrum test per KV/weight block; low effective rank ⇒ low-rank replacement is justified |
| `PhiPositiveDefiniteCholeskyFormula` | chunked exact attention / Cholesky KV-sampling with per-chunk positivity certificates |
| Gaussian Fourier self-duality (`GaussianFourierTransformFormula`) | backbone identity behind Hermite expansions — confirms the feature direction is load-bearing |

**Prototype** (`patterns/hermite_multinomial_attention.py`, tested): Hermite random features φ(q)_{i,m} = He_m(ωᵢ·q)·e^{ωᵢ·q−|q|²/2} / √(m·!), with extra multinomial discount (m!)^{−α} on order-m features. Synthetic n-token, d=32 benchmark:

- α = 0 (standard map): kernel rel-err 30%, softmax-normalized rel-err 21%
- α up to 0.75: raw kernel error grows (58%) but **softmax-normalized error is flat (20.6%)** — the renormalization cancels the systematic bias, so the multinomial discount changes almost nothing downstream while opening variance headroom. The Gram-structure of the proof (positivity under the definite-side condition) is the right frame for why.
- Speed: feature attention O(n·R·M·d) beats exact O(n²·d) past n ≈ 700 tokens (R=96, M=6): measured **2.8–3×** at n = 1024–4096 on CPU; the larger win (linear attention, streaming Φᵀ(ΦV)) applies when the value multiply uses the same features.
- Honest cost of the certificate framing: error stays high because R·M = 672 features undersample the kernel; the pattern's claim is the *guard-then-substitute structure*, not SOTA accuracy at toy R.

## 9. Phase 1/2 experiments on Qwen3.5-0.8B (2026-11-22)

Model: Qwen/Qwen3.5-0.8B — a **hybrid**: 18/24 layers linear attention (GatedDeltaNet, no KV cache), 6/24 full softmax attention (8 q-heads, 2 KV-heads, head_dim 256).

**Phase 1 — certificate-gated KV compression** (`patterns/kv_rank_gate.py`), greedy in-span gate `‖k_t − P_S k_t‖ > ε‖k_t‖`, 256-token prefix, ε=0.10: retention 54–64% per KV-head; dropped key-energy span-covered 30–43%; dropped-value reconstruction err 2.3–7.7%; attention TV 0.3–0.65, top-1 preserved 62–97% (best: layer 11 — 64% kept, 97% top-1). Honest caveat: a 256-token prompt is the *worst* case (low redundancy); the gate is correct but conservative there; deployment case is long-context.

**Phase 2 — certificate-gated low-rank weights** (`patterns/lowrank_weights.py`), energy-tail certificate τ=0.02 on weight-only SVD, all 186 Linear blocks: ranks 12–15 for GatedDeltaNet `in_proj` (genuinely near-low-rank structure — a real find), 285–390 for attention projections; **weight-factorized memory 110.5%** (U·Σ·Vᵀ costs more than dense unless r < dim/2); **perplexity 2.998 → 129,016** — weight-only energy is the wrong certificate. Lesson mirroring the proof's own language: the definite-side condition must be checked on the right object — energy on *weights* ≠ error on *activations*; deployment needs activation-aware scoring (ASVD/SVD-LLM-style), calibration data, and per-block acceptance, and even then only for matrices whose rank beats the factorization break-even.

**Phase 2 v2 — activation-aware + chain-budgeted** (`patterns/lowrank_weights_v2.py`): certificates computed on activation covariance (SVD of W·S; tail energy = exact relative activation error). Uncalibrated pass: 186/186 blocks pass 2% cert, FLOPs → 41.7%, yet PPL 3.0 → 171,373 — **per-block errors multiply through depth** (RecursionSharp: (1.02)^110 ≈ 9× minimum amplification, empirically far worse). Chain-budgeted acceptance (sort by cert, apply while ∏(1+eᵢ) ≤ 1.10): mechanically correct (prod 1.096, 5 blocks applied), but PPL still 13,071 — final lesson: **Frobenius (mean-error) certificates do not bound PPL, which is max-sensitive**; the dangling S⁻¹ whitening amplifies outlier tokens in low-variance directions. Deployment fix (future work): fold the whitening into the next layer (ASVD-style invariant propagation) instead of leaving it inside one matrix, plus worst-case/quantile certificates rather than mean ones. **CORRECTION (2026-11-26).** An earlier version of this document claimed GatedDeltaNet `in_proj` weights are 'naturally rank 12–15 of 1024'. That was wrong: `in_proj_a`/`in_proj_b` are 16x1024 matrices, so rank 12–15 is simply near-full rank of a skinny matrix, not low rank. The corrected experiments (below) show activation-weighted spectra are flat — dropping one singular direction of `in_proj_qkv` costs 3.4% worst-case activation error — so certified low-rank weight compression finds no acceptable blocks.

Debug history retained by design: searchsorted on a descending array gave rank 1 everywhere (caught by absurd 0.1% FLOPs), `state_dict()` entry swap meant nothing was applied (caught by ΔPPL = 0.0000), TV summed over probes instead of averaging (caught by TV > 1 being impossible for distributions), report-loop crash from a mixed-shape bucket (caught by dry-check before the run). Each bug was caught by an impossibility argument, not a guess.

## 11. Post-announcement developments (as of 2026-11-26)

- **Verification status: unchanged.** Lean community blog still carries only the 2026-11-21 certification; no retraction or correction. No arXiv version of the proof, and no analysis/simplification paper of the argument has appeared.
- **"Invalidity" preprint** (Jenny L. Nielsen, PhilPapers, 2026-11-21..24, rec NIEIOT-2). Caveats: the author is characterized as a rapid-preprint contrarian with a history of disputing major results (HN discussion); a commenter reports the Lean code being targeted is **unused scaffolding**, not load-bearing. PDF was Cloudflare-blocked, so the argument could not be read first-hand. No mathematical uptake visible.
- **The "junk" clarification (Alpöge, 2026-11-23, via HN summary).** The 250,000-page figure refers to an *auto-generated informal write-up* of the Lean proof (v1 ≈ 250k pages; a later attempt ≈ 60k), which he chose not to publish ("slop"). Combined with his remark's own admission of "strange suboptimal choices", "junk" = non-minimal scaffolding and a failed informalization — not a defect in the verified statement or proof term.
- **Corroborated locally:** our source exploration found the same picture — unused scaffolding modules, odd non-load-bearing blocks in `Lemmata/`, and community PRs (#57/#65/#66/#68) deleting unused files and shortening proofs, with one rollback recorded as a maintenance regression, not a change of conclusions.
- **Takeaway to keep:** *verified ≠ minimal.* The kernel certifies the theorem, not the economy of the route. For any future critique, the discriminator is the one used throughout this document: **is the attacked object load-bearing for the exported statement?** If not, it cannot move the five theorems.
- Watch: arXiv, Lean Zulip, and the repo's `original`-branch pruning diffs for the promised short proofs.


**Tier-B experiment v4 (sequential recalibrated, quantile-certified).** Per-layer recalibration of activations, rank chosen by binary search on quantile-95 activation error <= 5%, blocks accepted only when r*(out+in) < out*in: **0 blocks accepted** across the 18 GatedDeltaNet layers (PPL unchanged 2.2646 -> 2.2646, params saved 0.00%). Diagnostic: activation-whitened singular spectra decay too slowly for any memory-winning rank to sit under a 5% worst-case error bar. Conclusion: **no real weight compression from certified low-rank truncation on this family**; the only measured lossless-ish lever remains int4 group-wise quantization of MLP blocks (+0.0037 PPL). Certificate-gated compression is a correct *filter*, not a compressor.


**Generality check (2026-11-26).** Same protocol (sequential recalibration, quantile-95 <= 5%, memory-win condition) on three hybrid models:

| model | params | baseline PPL | blocks accepted | ranks | params saved | dPPL |
|---|---|---|---|---|---|---|
| Qwen3.5-0.8B | 752M | 2.2646 | 0 | - | 0.00% | +0.0000 |
| Qwen3.5-2B | 1882M | 2.1601 | 2 (in_proj_b) | 15 | ~0.00% | +0.0002 |
| Qwen3.5-4B | 4206M | 2.0358 | 10 (in_proj_a/b) | 30-31 | ~0.00% | +0.0001 |

Accepted blocks are only the skinny projections whose max rank is near-full; all large projections are rejected in every model. The negative result is consistent across scale.

*Compiled from: the announcement PDF, the Lean source tree, MIT's GRH verification page, the Lean/Mathlib joint statement, and a fresh independent con-ron run performed on this machine.*
