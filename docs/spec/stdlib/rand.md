# Spec — `std.rand` (random number generation with reified, named nondeterminism)

| Field | Value |
|---|---|
| **Status** | **Accepted** (2026-06-20, maintainer-ratified per DN-07 — guarantee matrix asserted in tests; open §7/§8 questions are design/scope calls, not contract violations; was *Implemented (Rust-first) — pending ratification* 2026-06-18, Draft/needs-design 2026-06-17) — the Rust-first code landed as `mycelium-std-rand` (M-531, #171, Batch P5-B; guarantee matrix asserted in tests). The Mycelium-lang migration (M-502-gated) remains. |
| **Module / Ring** | `std.rand` · Ring 2 (RFC-0016 §4.2) · Tier B (RFC-0016 §4.4) |
| **Tracks** | `M-531` (#171) — the Phase-5 task this spec delivers |
| **Scope** | Random number generation as two structurally distinct constructs: a **seeded / deterministic** generator (a reproducible value whose state is part of the value model) and an **entropy-backed** generator (which pulls real nondeterminism as a *declared effect*); plus a small honestly-tagged distribution family over them. |
| **Boundary** | Out of scope: the wall-clock as an entropy source — `std.time` (M-529) owns clock reads, but a wall-clock read **is** an entropy draw under the same RT3 rule, so the two modules share one declared-nondeterminism discipline (named, not duplicated, below). Statistical-quality / accuracy bounds on a distribution route through `std.numerics` (M-512) and honest empirical method — never fabricated here. The generator's value-semantic state is built on `std.core` (M-515). A *representation* change is `std.swap` (M-516). |
| **Depends on** | RFC-0016 §4.1 (the C1–C6 contract), §4.2 (ring layering), §4.4 (the `rand` row: nondeterminism reified/named, RT3); RFC-0014 (declared, bounded effects — entropy is a declared effect); RFC-0001 (the value model — `Value`/`Repr`/`Meta`, the guarantee lattice §4.3); RFC-0013 (the diagnostic record a refusal carries). |
| **Grounds on** | `std.core` (M-515) — the value model + `Option`/`Result`/error values the generator state and results are expressed in; the RT3 determinism rule (a deterministic-fragment program cannot draw entropy silently); the RFC-0014 effect-budget machinery for the entropy effect. KC-3: above the kernel — adds no trusted code. |

---

## 1. Summary

`std.rand` is the random-number surface, held to the §4.1 contract. Its **honesty crux** is C6 in its sharpest form: **nondeterminism is reified and named (RT3)** — a generator that consumes real entropy declares an `entropy` effect on its signature, so a deterministic-fragment program *cannot* pull randomness silently; the only way to get reproducible "randomness" is the structurally distinct **seeded** generator, whose state is an ordinary immutable value and whose stream is a pure function of `(seed, draws)`. Its second crux is C2/VR-5: a distribution's statistical guarantees are tagged honestly — `Declared` (asserted by construction) or `Empirical` (a measured test-suite verdict), and **never `Proven`** without a checked theorem (a uniformity/independence claim is not pre-claimed). Ring 2, Tier B; it adds no trusted code (KC-3), consuming `std.core` and the RFC-0014 effect surface.

## 2. Scope & module boundary

- **In scope:**
  - The **seeded generator** `Rng` — a value carrying its algorithm + state; `split` (derive an independent sub-stream), `next_*` (raw draws), and the bounded/distribution draws are *pure* functions of it, returning the value plus the advanced generator. No ambient global RNG.
  - The **entropy-backed generator** `EntropyRng` — a generator seeded from the platform entropy source; constructing one and drawing from it carry the declared `entropy` effect (RFC-0014). A `seed_from_entropy() -> Rng` bridge mints a reproducible seed *once* from entropy (the draw is declared) and hands back a pure seeded `Rng`.
  - A small **distribution family**: `uniform` over an explicit half-open integer/float range, `bool`/`bernoulli(p)`, `range(lo, hi)`, `choice`/`shuffle` over a value-semantic collection, and `normal`/`exponential` as the first continuous draws — each tagged honestly (see §4).
- **Out of scope (and who owns it):**
  - Wall-clock / monotonic / logical time reads — `std.time` (M-529). A wall-clock read is an entropy source and obeys the *same* RT3 declared-effect rule; `time` owns the clock surface, `rand` owns generators. The shared rule is named once (§5 seam), not duplicated.
  - Statistical-accuracy bounds (a tight bound on a sampler's bias, a CLT error term) — `std.numerics` (M-512) / honest empirical method. `rand` *carries* whatever tag the method establishes; it does not derive the bound.
  - The value-semantic state machinery (immutable values, content-addressed identity) — `std.core` (M-515).
- **Ring & layering:** Ring 2 (RFC-0016 §4.2). `rand` is **new library code written to the contract over Ring 0/1**: the seeded generator is a pure state machine over the value model; the entropy generator *wraps* the RFC-0014 declared-effect surface. It is a certificate/EXPLAIN **consumer**, not a producer of trusted bounds (KC-3).

## 3. Exported-op surface (design sketch)

A design sketch — enough to fix the surface and feed the guarantee matrix, not a committed grammar. Value-semantic, immutable-by-default: a seeded draw returns `(value, Rng')` (the advanced generator is a *new value*; the input is unchanged); an entropy draw declares its effect on the signature (C6).

```
// illustrative signatures (not a committed surface)

type Rng                                  // a seeded generator VALUE: { algo, state } — reproducible
type EntropyRng                           // an entropy-backed generator (effectful to construct/draw)

// --- seeded: PURE, reproducible. No ambient state; the advanced generator is returned. ---
fn seed(s: u64)            -> Rng                       // Exact, total — a value
fn split(g: Rng)           -> (Rng, Rng)               // Exact — two independent sub-streams (decl. discipline)
fn next_u64(g: Rng)        -> (u64, Rng)               // Exact (the stream is a pure function of the seed)
fn uniform_int(g: Rng, lo: i64, hi: i64) -> Result<(i64, Rng), RandErr>  // Err(EmptyRange) if hi <= lo
fn bernoulli(g: Rng, p: Real)            -> Result<(bool, Rng), RandErr>  // Err(BadProbability) if p∉[0,1]
fn choice<T>(g: Rng, xs: &Seq<T>)        -> Result<(T, Rng), RandErr>     // Err(EmptyDomain) if xs empty
fn shuffle<T>(g: Rng, xs: Seq<T>)        -> (Seq<T>, Rng)                 // Exact permutation (Fisher–Yates)
fn normal(g: Rng, mu: Real, sigma: Real) -> Result<(Approx<Real>, Rng), RandErr>  // sigma>0; carries its tag

// --- entropy: the draw is a DECLARED effect (RT3 / RFC-0014). A deterministic fragment CANNOT call these. ---
fn from_entropy()          -> EntropyRng           ! entropy            // declares the entropy effect
fn seed_from_entropy()     -> Result<Rng, RandErr> ! entropy            // mint ONE reproducible seed, then pure
fn next_entropy(g: EntropyRng) -> (u64, EntropyRng) ! entropy           // each draw is an entropy effect

enum RandErr { EmptyRange, BadProbability, EmptyDomain, BadParameter, EntropyUnavailable }
```

> **Note (the `!entropy` marker):** the `! entropy` on a signature is the RFC-0014 declared effect. It is what structurally prevents a deterministic-fragment program from drawing randomness: the effect does not type-check there (RT3). The *seeded* surface carries no such marker — it is pure — which is exactly why it is reproducible.

## 4. Guarantee matrix (the load-bearing deliverable — RFC-0016 §4.5)

Rows = exported ops. To be encoded as a checked table (the RFC-0003 §4 template) and asserted in tests once code lands — never prose only. **Tag legend:** an op with no accuracy semantics (a raw stream step, a permutation) is `Exact`; a *distribution* op's tag is the strength of its sampling-correctness claim, resolved by basis (`Declared` by construction, `Empirical` by a measured suite), never `Proven` without a checked theorem.

| Op | Guarantee tag | Fallibility (explicit error set) | Declared effects | EXPLAIN-able? |
|---|---|---|---|---|
| `seed` | `Exact` | total | none | n/a |
| `split` | `Exact` (independent sub-streams — the *independence quality* is `Declared`, see below) | total | none | yes (the split discipline) |
| `next_u64` / `next_*` | `Exact` (a pure function of the seed; the *uniformity* of the bit-stream is `Empirical`, see below) | total | none | n/a |
| `uniform_int` | `Declared` (unbiased rejection sampling — the bias bound is `Declared`/`Empirical`, not `Proven` here) | `Err(EmptyRange)` | none | yes (rejection record) |
| `bernoulli` | `Declared` | `Err(BadProbability)` | none | yes |
| `choice` | `Declared` (uniform over the domain) | `Err(EmptyDomain)` | none | yes |
| `shuffle` | `Exact` (a permutation; *uniformity over permutations* is `Declared`/`Empirical`) | total | none | yes |
| `normal` / `exponential` | `Empirical` (sampler-correctness measured) — carries its `Approx`/method tag | `Err(BadParameter)` | none | yes (method + measured profile) |
| `from_entropy` | `Declared` | `Err(EntropyUnavailable)` | **`entropy`** | yes |
| `seed_from_entropy` | `Declared` (the minted seed; thereafter pure) | `Err(EntropyUnavailable)` | **`entropy`** | yes |
| `next_entropy` | `Declared` | `Err(EntropyUnavailable)` | **`entropy`** | yes |

**Tag justification (VR-5 — downgrade rather than overclaim):**
- **`Exact` rows** carry no *accuracy* semantics: `seed`/`split` build values; `next_u64` is exactly determined by the seed; `shuffle` produces an exact permutation. Crucially, "`Exact`" here is about *reproducibility/determinism of the mechanism*, **not** a claim about statistical quality — that claim is separated out and tagged on its own footing below, so the table never lets a deterministic mechanism masquerade as a proven-uniform source.
- **Distribution rows** (`uniform_int`/`bernoulli`/`choice`/`shuffle`-uniformity/`normal`) carry a *sampling-correctness* claim. This spec tags them **`Declared`** (asserted by the construction — e.g. rejection sampling is unbiased *by the rejection argument*) or **`Empirical`** (a continuous sampler whose correctness is established by a measured test-suite, e.g. a goodness-of-fit profile). **No distribution row is `Proven`** absent a cited theorem with checked side-conditions (e.g. a proven bias bound on the integer reduction); the spec deliberately does not pre-claim it (VR-5). The exact bias/error magnitudes are **owned by `std.numerics`/the measured method**, not fabricated here (§7 Q3).
- **`entropy` rows** are `Declared` and effectful: the platform entropy source's quality is not something this library proves; what it *guarantees* is that the draw is **named** (the `entropy` effect) and fallible (`EntropyUnavailable`), never a silent fallback to a fixed value.

## 5. §4.1 contract conformance (C1–C6)

- **C1 — never-silent (G2).** Every domain restriction is an explicit `Result::Err` from `RandErr` — an empty range (`uniform_int` with `hi <= lo`) → `Err(EmptyRange)`, a probability outside `[0,1]` → `Err(BadProbability)`, a draw from an empty domain → `Err(EmptyDomain)`, an out-of-range distribution parameter (`sigma <= 0`) → `Err(BadParameter)`. An unavailable entropy source is `Err(EntropyUnavailable)` — **never a silent fallback to a fixed/zero seed** (which would make a "random" program silently deterministic, the worst silent failure here).
- **C2 — honest per-op tag (VR-5).** Mechanism-determinism (`Exact`) and statistical-quality (`Declared`/`Empirical`) are tagged on *separate* footings so neither inflates the other; no distribution claim reaches `Proven` without a checked theorem. The independence quality of `split` and the uniformity of the raw stream are `Declared`/`Empirical`, surfaced as such, never upgraded.
- **C3 — no black boxes / EXPLAIN (SC-3/G11).** A seeded generator's `(algo, state, seed-provenance)` is inspectable — a reproducible run can be replayed from its seed, and a rejection-sampling draw can report the rejection record. An entropy draw is reified as the named effect with its provenance. No opaque global RNG sets a user-visible outcome.
- **C4 — content-addressed, value-semantic (ADR-003 / RFC-0001).** A seeded `Rng` is an immutable value; a draw returns a *new* generator value rather than mutating ambient state, so identical `(seed, draw-sequence)` are the same value (reproducibility is value-equality). The carried method/profile metadata is `Meta`, **not** identity.
- **C5 — above the small kernel (KC-3).** `rand` adds no trusted code: the seeded generator is a pure state machine over the value model; the entropy generator consumes the *already-trusted* RFC-0014 effect surface and the platform entropy capability. Any platform-entropy `wild`/FFI floor is the capability's, inventoried there (LR-9), not minted at the `rand` layer — FLAGGED §7 Q4.
- **C6 — declared, bounded effects (RFC-0014).** This is the module's crux. The seeded surface is **pure** (`effects: none`) — that is what makes it reproducible. The entropy surface **declares** the `entropy` effect on every op that draws real nondeterminism, so a deterministic-fragment program cannot call it (RT3). Entropy is not silently ambient anywhere.

## 6. Grounding

- The reified-nondeterminism rule + the entropy-as-declared-effect discipline: **RT3** (nondeterminism is reified and named) and **RFC-0014** (declared, bounded effects) — the basis for the seeded-vs-entropy structural split and the `! entropy` marker.
- The honest-tags discipline (distribution claims `Declared`/`Empirical`, never `Proven` without a checked basis): **VR-5** (downgrade not upgrade), **RFC-0001** §4.3 (the `Exact ⊐ Proven ⊐ Empirical ⊐ Declared` lattice).
- The per-op contract C1–C6, the ring/tier placement, the guarantee-matrix obligation: **RFC-0016** §4.1 / §4.2 / §4.4 (the `rand` row) / §4.5.
- The value model + never-silent + EXPLAIN: **RFC-0001** (`Value`/`Repr`/`Meta`), **G2** (never-silent), **SC-3/G11** (EXPLAIN), **ADR-003** (metadata ≠ identity), **KC-3** (small kernel); the refusal record a fallible op carries: **RFC-0013** (the diagnostic record).

## 7. Open questions (FLAGGED — resolve before ratification)

- **(Q1) Scope ratification (M-501).** The exported distribution set (which continuous distributions, whether `shuffle`/`choice` ship in v0) is **ratified at M-501 with RFC-0016**, not here; the §3 sketch is illustrative. — *Disposition: FLAGGED; align with the ratified module set. Ties to RFC-0016 §8-Q1.*
- **(Q2) The seeded-generator algorithm + the `split` independence basis.** Which PRNG algorithm (and thus the concrete reproducibility/period/independence guarantees) and whether `split` independence is `Declared`-by-construction or `Empirical`-by-test is open; the choice fixes the `split`/stream tags. — *Disposition: FLAGGED; default to a well-studied splittable PRNG; tag `split`/stream quality at the *established* strength, never `Proven` without a checked basis.*
- **(Q3) Which distribution claims reach `Empirical` vs stay `Declared`, and the concrete bias/error magnitudes.** The matrix tags samplers `Declared`/`Empirical` at *basis-resolved* strength; the concrete bias bounds (e.g. the integer-reduction bias, a sampler's measured goodness-of-fit) are **owned by `std.numerics` (M-512) / the measured method**, not fabricated here. — *Disposition: FLAGGED; fill the strength column when the method lands; this spec fixes the discipline, not the numbers.*
- **(Q4) The platform-entropy `wild`/FFI floor.** The entropy source bottoms out in a platform facility (getrandom/`wild`, ADR-014); whether `rand` inherits a `wild` block to inventory (LR-9) or that floor lives in a `std-sys` phylum (the §8-Q6 split, shared with `fs`/`time`) is open. — *Disposition: FLAGGED; ties to RFC-0016 §8-Q6; keep pure `std.rand`'s seeded half leak-free regardless.*
- **(Q5) Ergonomics of the threaded generator (tension A).** Returning `(value, Rng')` everywhere (explicit, reproducible) vs an implicit-but-inspectable generator context is the RFC-0016 §8-Q3 ergonomics-vs-contract tension, instantiated for RNG. — *Disposition: FLAGGED; default to explicit threading until the per-ring ergonomics pass.*

## Meta — changelog

- **2026-06-17 — Draft (needs-design).** Stands up the `std.rand` (M-531, #171) module spec under RFC-0016 (Draft): Ring 2 / Tier B random number generation. Fixes the scope + boundary (shares the declared-nondeterminism rule with `time` M-529; defers statistical-accuracy bounds to `numerics` M-512; builds generator state on `core` M-515), the exported-op surface sketch (the structurally distinct **seeded** generator — pure, reproducible, returning the advanced generator as a value — and the **entropy-backed** generator — every real-nondeterminism draw declaring the `entropy` effect), and — the load-bearing deliverable — the per-op **guarantee matrix**: mechanism-determinism tags `Exact` while distribution sampling-correctness tags `Declared`/`Empirical` on a *separate footing*, **never `Proven`** without a checked theorem (VR-5), and an unavailable entropy source is an explicit `Err(EntropyUnavailable)`, **never a silent fixed-seed fallback** (C1/G2). The **honesty crux** is enforced: entropy is reified and named (RT3 / the `! entropy` effect, RFC-0014), so a deterministic-fragment program cannot draw randomness silently, and reproducibility is value-equality on the seeded generator. §4.1 conformance (C1–C6) stated concretely; grounding traces to RT3, RFC-0014, VR-5, RFC-0016 §4.1/§4.4/§4.5, RFC-0001, RFC-0013, ADR-003/KC-3. Five questions FLAGGED (scope ratification at M-501; the PRNG algorithm + `split` independence basis; which distributions reach `Empirical` + the concrete bias numbers, owned by M-512; the platform-entropy `wild`/std-sys floor §8-Q6; the threaded-generator ergonomics §8-Q3) — the numbers and the `wild` floor are not invented here. No code; no kernel change (KC-3). Append-only.

- **2026-06-20 — Accepted (maintainer ratification, DN-07).** The maintainer ratified this Rust-first spec: the §4.5 guarantee matrix is asserted in tests, never-silent fallibility and honest per-op tags hold, and the open §7/§8 questions are design/scope calls, not contract violations. No guarantee tag was upgraded without a checked basis (VR-5). Status moves *Implemented (Rust-first) — pending ratification → Accepted*. Append-only; no kernel change (KC-3).
