# Algebraic Attacks on Post-Quantum Cryptography

> A first-principles technical note on the history and current state of algebraic attacks. The emphasis is on **why lattice-based PQC schemes (ML-DSA, ML-KEM) structurally resist** the attack techniques that ended HFE, Rainbow, and SIDH.

## Contents

- [Part 1: Defining the Algebraic Attack](#part-1-defining-the-algebraic-attack)
- [Part 2: The Four Algebraic Attack Families](#part-2-the-four-algebraic-attack-families)
- [Part 3: Gröbner Basis — A Deep Dive](#part-3-gröbner-basis--a-deep-dive)
- [Part 4: Historical Case Studies and NIST PQC Standardization](#part-4-historical-case-studies-and-nist-pqc-standardization)
- [Part 5: Why Lattice Schemes Are Immune to Gröbner Basis](#part-5-why-lattice-schemes-are-immune-to-gröbner-basis)
- [References](#references)

---

## Part 1: Defining the Algebraic Attack

### One-line definition

**An algebraic attack rewrites a cryptosystem as a system of equations and applies algebraic equation-solving techniques to recover the secret directly.**

"Algebraic" here is broad. It covers:

- Linear algebra
- Polynomial rings and finite-field arithmetic
- Gröbner basis methods
- Lattice reduction

The underlying idea is plain:

1. Every cryptosystem is a sequence of operations.
2. Operations can be written as equations.
3. If those equations are easy enough, the secret falls out.

Decades of cryptographic design have been about making those equations hard enough that no known technique solves them in feasible time.

### Two simple examples

#### Caesar cipher

Caesar shifts each letter by k: C = (P + k) mod 26.

If an attacker has a single (P, C) pair:

```
k = (C − P) mod 26
```

One line. Secret recovered. The simplest possible algebraic attack.

#### Linear cipher

Consider a poorly designed scheme where the key k encrypts via:

```
C = k · P + b   (mod q)
```

Two (P, C) pairs give:

```
Equation 1: C₁ = k·P₁ + b
Equation 2: C₂ = k·P₂ + b
```

Two unknowns, two equations — Gaussian elimination resolves it.

This is the historical reason why **linear systems offer no cryptographic security**, and why every modern LWE-based design deliberately includes a noise term `e`.

---

## Part 2: The Four Algebraic Attack Families

| Family | Core technique | Typical targets | Threat to ML-DSA |
|---|---|---|---|
| Linear-algebra attacks | Gaussian elimination | Linear systems; noiseless LWE variants | Blocked structurally by the noise term `e` |
| Differential algebraic attacks | Statistical analysis of polynomial relations | Block ciphers (DES, AES); hash functions | Keccak's 24-round design has wide differential margin |
| Gröbner basis attacks | Multivariate polynomial system solving | Multivariate cryptosystems (HFE, Rainbow) | Noise breaks the exact-equation requirement; immune |
| Lattice attacks (lattice reduction) | LLL, BKZ, sieving | RSA, DSA, LWE, NTRU | **Primary threat**; sets parameter choices |

### Family 1: Linear-algebra attacks

**Tool**: Gaussian elimination. Complexity O(n³); a system with 1000 unknowns solves in well under a second.

**Targets**: Anything expressible as a linear system over a finite field or ring.

**Modern defenses**: Modern designs deliberately introduce nonlinearity. Designers consistently ask:

- Can this operation be linearly approximated?
- If so, what is its nonlinearity (in the cryptographic sense)?

Common sources of nonlinearity:

- **S-boxes**: AES and DES use lookup tables to inject nonlinearity.
- **Modular exponentiation**: RSA's `x^e mod N`.
- **Keccak's χ step**: the only nonlinear step in SHA-3 / SHAKE, which underlies the symmetric primitives in ML-DSA and ML-KEM.

### Family 2: Differential algebraic attacks

**Core idea**: track how input differences propagate to output differences.

```
Input pair:   (P, P + Δ)
Output pair:  (C, C')
Difference:   ΔC = C' ⊕ C
```

When particular ΔP → ΔC transitions occur with elevated probability, statistical analysis can recover internal state.

**Canonical case**: Biham and Shamir (1990) used differential cryptanalysis against DES:

- Full 16-round DES required 2⁴⁷ chosen plaintexts.
- Marginally better than brute force (2⁵⁶), but it demonstrated DES was not mathematically optimal.
- It directly shaped the AES design — AES's wide-trail strategy is explicitly an anti-differential measure.

**Relevance to ML-DSA**: in principle, the SHAKE underlying ML-DSA (i.e., Keccak) is exposed to differential analysis. In practice:

- Keccak was designed with substantial differential margin.
- The best published Keccak differential attack reaches only 6 rounds (out of 24).
- No realistic threat.

### Family 3: Gröbner basis attacks (see Part 3)

**Problem form**: solve a system of nonlinear multivariate polynomial equations:

```
f₁(x₁, x₂, ..., xₙ) = 0
f₂(x₁, x₂, ..., xₙ) = 0
        ...
fₘ(x₁, x₂, ..., xₙ) = 0
```

Gaussian elimination fails because the `fᵢ` contain `xᵢxⱼ`, `xᵢ²`, etc.

**Gröbner basis** (Buchberger, 1965) is the multivariate-polynomial analog of Gaussian elimination — a systematic technique that reduces the system to a form that can be solved variable-by-variable.

### Family 4: Lattice attacks (lattice reduction)

**The dominant threat to ML-DSA.** Composed of LLL (1982), BKZ (1987), and modern sieving algorithms (BDGL16 and successors). Treated in detail in [`pqc-lattice-reduction-notes`](https://github.com/codebat-research/pqc-lattice-reduction-notes).

**The headline cost formula**: classical attack cost on BKZ-β is approximately 2^(0.292β). For ML-DSA-65, breaking the scheme requires β ≈ 660, giving a cost of roughly 2^193 — and that figure is the source of the NIST Level 3 security claim.

---

## Part 3: Gröbner Basis — A Deep Dive

### Prerequisites: polynomial rings

#### Univariate vs multivariate

**Univariate**: f(x) = 3x² + 5x − 7. The set ℝ[x] is all polynomials in x with real coefficients.

**Multivariate**: f(x, y, z) = 2x²y − 3yz + 5xz² + 7. The set ℝ[x, y, z] is all polynomials in x, y, z with real coefficients.

**For cryptography**: 𝔽_q[x₁, ..., xₙ] — polynomials over a finite field 𝔽_q with n variables.

#### Why "ring"

The set of polynomials forms an algebraic ring:

- Addition is closed.
- Multiplication is closed.
- There is an additive identity (0) and a multiplicative identity (1).

But division is **not** closed — dividing two polynomials does not generally yield a polynomial. This absence of division is the central structural problem Gröbner basis solves: providing a "division-like" reduction process inside the ring.

### Monomial orderings

Gröbner basis computation depends on a key choice: an ordering on monomials.

A **monomial** is a power-product of variables — examples: `x²yz³`, `xy²`, `x³`. Coefficients are excluded.

#### Three common orderings

**Lexicographic (lex)** — like a dictionary: compare exponents of x first, then y, then z.

Example: `x²yz³` vs `x²y²z` — both have x² (tie); compare y exponents (1 vs 2); `x²y²z` is larger.

**Graded lex (grlex)** — compare total degree first, then break ties lexicographically.

Example: `x²yz³` (degree 6) vs `x²y²z` (degree 5) → `x²yz³` is larger.

**Graded reverse lex (grevlex)** — compare total degree first, then break ties from the smallest variable downward.

#### Why ordering matters

Different orderings produce **completely different Gröbner bases**:

- **lex** ordering produces the most "triangulated" basis, ideal for solving systems.
- **grevlex** ordering produces the fastest computation.
- Practical workflow: compute with grevlex, then convert to lex for solving.

### Ideals

The **ideal** I generated by polynomials {f₁, ..., fₛ} is the set of all linear combinations:

```
I = ⟨f₁, ..., fₛ⟩ = { h₁·f₁ + ... + hₛ·fₛ : hᵢ ∈ K[x] }
```

The **Ideal Membership Problem** — given f, decide whether f ∈ I — is the central problem Gröbner basis is designed to solve.

### The formal definition of a Gröbner basis

A **Gröbner basis** G = {g₁, ..., g_t} of an ideal I is a generating set with the following property:

> For every f ∈ I, the multivariate division of f by G yields a unique remainder r, regardless of the order in which divisors are applied.

This resolves the ambiguity inherent in multivariate polynomial division.

**Equivalent technical definition**: the leading-term ideal of G equals the leading-term ideal of I:

```
⟨LT(g₁), ..., LT(g_t)⟩ = ⟨LT(I)⟩
```

### Buchberger's algorithm (1965)

The original Gröbner basis algorithm is built around the **S-polynomial**:

```
S(f, g) = (lcm(LT(f), LT(g)) / LT(f)) · f − (lcm(LT(f), LT(g)) / LT(g)) · g
```

Intuition: the S-polynomial constructs a combination that cancels leading terms, exposing hidden relations within the ideal.

**Algorithm**:
1. Start with the original generators G = {f₁, ..., fₛ}.
2. Compute the multivariate-division remainder r of S(gᵢ, gⱼ) by G for all pairs.
3. If r ≠ 0, append r to G.
4. Repeat until all S-polynomial remainders reduce to 0.

**Drawback**: Buchberger's algorithm is impractically slow on real systems — it generates huge numbers of "useless" S-polynomials.

### F4 and F5 (Faugère, 1999 and 2002)

**F4 (1999)**: accelerates S-polynomial reduction by batching many of them into a single sparse linear-algebra problem. Roughly 10–1000× faster than Buchberger.

**F5 (2002)**: introduces a "signature criterion" that avoids computing S-polynomials destined to reduce to zero. Roughly 10–100× faster than F4.

**F5 is the algorithm that actually broke HFE and Rainbow.** Without F5, those attacks would not have been computationally feasible.

### Complexity

In the worst case, Gröbner basis computation is doubly exponential — 2^(2^n) — which is worse than brute force. For "structured" polynomial systems such as HFE and Rainbow, however, F5 runs in feasible time on real instances.

**Hard practical limit**: roughly n ≈ 60–80 variables. Beyond this, even F5 stalls. ML-DSA's effective variable count is on the order of 1000–2000 — far above the limit. This is not a "current technology" gap but the structural consequence of doubly-exponential complexity.

---

## Part 4: Historical Case Studies and NIST PQC Standardization

### Case 1: HFE collapses (2002)

**Background**: in 1996, Patarin proposed HFE (Hidden Field Equations), the most promising multivariate public-key scheme of the era. HFE Challenge 1 claimed 80-bit security.

**Attack**: in 2002, Faugère broke HFE Challenge 1 in 96 hours using F5.

**Impact**: ended HFE as a serious post-quantum candidate.

### Case 2: Rainbow collapses (2022)

**Background**: Rainbow is a successor to HFE based on the Oil-Vinegar structure. It advanced to Round 3 of NIST PQC standardization in 2017 and was considered a strong candidate for the signature category. Rainbow I claimed 158-bit security.

**Attack**: in 2022, Beullens broke Rainbow I in 53 hours on a laptop — actual security around 2^53. Rainbow III and V fell similarly.

**Impact**: in July 2022, Rainbow was eliminated from NIST PQC standardization.

### Case 3: SIKE/SIDH collapses (2022)

**Background**: SIKE (built on SIDH) was an isogeny-based post-quantum scheme that advanced to NIST Round 4 as an alternate candidate.

**Attack**: Castryck and Decru applied algebraic geometry (a torsion-point attack) and broke SIKE in hours. Subsequent optimizations by Maino–Martindale and Robert reduced this to seconds.

**Impact**: the entire SIDH-based signature line collapsed. Isogeny-based cryptography was marginalized in NIST signature standardization (variants like CSIDH still exist but are not adopted as standards).

### Case 4: ML-DSA's structural immunity to Gröbner basis

ML-DSA is based on the LWE structure:

```
A · s + e = t
```

Why Gröbner basis methods fail here:

1. **The equations are not exact** — the noise term `e` makes them approximate.
2. Gröbner basis methods only handle exact solutions; "approximate" has no representation in the framework.
3. Ignoring `e` and solving the exact system → with high probability, no solution exists.
4. Enumerating possible values of `e` → exponential blowup.

This is the structural reason **lattice cryptography is naturally immune to Gröbner basis**: noise breaks the algebraic equality.

In the 2010s, attempts using the Arora–Ge polynomialization (treating LWE as a high-degree polynomial system, then applying Gröbner basis) consistently produced complexity bounds tens of bits worse than BKZ. Gröbner basis methods are not competitive against LWE.

### NIST's selection logic

The history of algebraic attacks directly determined the outcome of NIST PQC standardization:

| Family | Representative scheme | Outcome | Reason |
|---|---|---|---|
| Multivariate | HFE, Rainbow | Eliminated | Killed by Gröbner basis |
| Isogeny (SIDH) | SIKE | Eliminated | Killed by algebraic-geometry attacks |
| Lattice | ML-DSA, ML-KEM | **Standardized** | Noise structure resists Gröbner basis |
| Hash-based | SLH-DSA | **Standardized (backup)** | Doesn't depend on algebraic structure at all |
| Code-based | Classic McEliece, HQC | Round 4, ongoing | Information-set decoding is hard |

**Lattice did not win because it is simpler. It won because, after 50 years of algebraic-attack research, it is the only remaining family that combines efficiency with resistance to algebraic attacks.**

---

## Part 5: Why Lattice Schemes Are Immune to Gröbner Basis

### Noise as structural defense

Express LWE as:

```
t = A · s + e   (mod q)
```

with `e` a bounded noise vector (e.g., `e ∈ {−η, ..., η}^n`). The attacker observes (A, t) and seeks to recover s.

Gröbner basis methods presume a system of *exact* equations f_i = 0. LWE does not provide one:

- `t − A·s = e` is not zero — it lies in some small range.
- Gröbner basis has no notion of "in some range" — only "equal to 0" or "not equal to 0."

### Why brute-force expansion fails

A natural attempt: treat `e` as a new variable and polynomialize the system. The problem:

- `e` has n components, each with 2η+1 possible values.
- For ML-DSA-65: n ≈ 256, η = 4 → 9^256 ≈ 2^812 possibilities.
- Any enumeration of `e` is blocked by exponential blowup.

### The Arora–Ge polynomialization attempt

In 2011, Arora and Ge proposed converting LWE to a polynomial system via the constraint that each noise component eᵢ lies in {−η, ..., η}, captured by:

```
∏_{j=−η}^{η} (eᵢ − j) = 0
```

Substituting `eᵢ = tᵢ − ⟨aᵢ, s⟩`:

```
∏_{j=−η}^{η} (tᵢ − ⟨aᵢ, s⟩ − j) = 0
```

This is a (2η+1)-degree polynomial in s. For ML-DSA-65, that's degree ≈ 9 in 256 variables.

Gröbner basis complexity for such a system:

```
2^(O(n · D))   where D is the regularity degree
```

For ML-DSA-65 parameters, this is 30+ bits worse than the BKZ attack — **not competitive**.

### Conclusion

Noise is not a side effect of LWE. It is the structural foundation of the scheme's defense against algebraic attacks. This is a deliberate design choice, not an engineering compromise.

> **Algebraic attacks and lattice attacks lie at opposite ends of a single trade-off. Multivariate cryptography has no noise: it resists lattice attacks but is destroyed by Gröbner basis methods. Lattice cryptography has noise: it is immune to Gröbner basis methods but exposes itself to BKZ. Post-quantum cryptography chose the lattice path — choosing BKZ as a controlled adversary.**

---

## Historical Timeline: Algebraic Attacks Driving Cryptographic Evolution

| Year | Breakthrough | Casualties |
|---|---|---|
| 1940s | Linear cryptanalysis | Early classical ciphers |
| 1990 | Biham–Shamir differential | Reduced-round DES, FEAL |
| 1993 | Matsui linear cryptanalysis | DES security margin reduced |
| 1999 | Faugère F4 | Computationally feasible Gröbner basis |
| 2002 | Faugère F5 / Gröbner basis | HFE family collapses |
| 2010s | BKZ improvements | Early LWE parameters obsoleted |
| 2020s | Quantum BKZ estimates | NIST PQC standardization driven |
| 2022 | Beullens, Rainbow attack | Rainbow eliminated |
| 2022 | Castryck–Decru | SIDH/SIKE collapses |

---

## References

1. **Cox, D., Little, J., O'Shea, D.** *Ideals, Varieties, and Algorithms* (4th ed.). Springer, 2015. — Standard textbook on polynomial rings and Gröbner basis.
2. **Buchberger, B.** *Ein Algorithmus zum Auffinden der Basiselemente des Restklassenringes nach einem nulldimensionalen Polynomideal*. Doctoral dissertation, University of Innsbruck, 1965.
3. **Faugère, J.-C.** *A new efficient algorithm for computing Gröbner bases (F4)*. Journal of Pure and Applied Algebra, 1999.
4. **Faugère, J.-C.** *A new efficient algorithm for computing Gröbner bases without reduction to zero (F5)*. ISSAC 2002.
5. **Patarin, J.** *Hidden Fields Equations (HFE) and Isomorphisms of Polynomials (IP)*. EUROCRYPT 1996.
6. **Beullens, W.** *Breaking Rainbow Takes a Weekend on a Laptop*. CRYPTO 2022.
7. **Castryck, W., Decru, T.** *An efficient key recovery attack on SIDH*. EUROCRYPT 2023.
8. **Arora, S., Ge, R.** *New Algorithms for Learning in Presence of Errors*. ICALP 2011.
9. **Albrecht, M., Player, R., Scott, S.** *On the concrete hardness of Learning with Errors*. Journal of Mathematical Cryptology, 2015.
10. **Biham, E., Shamir, A.** *Differential Cryptanalysis of DES-like Cryptosystems*. Journal of Cryptology, 1991.

---

*Maintained by Codebat Technologies as part of a series of PQC cryptanalysis study notes. Issues, corrections, and pull requests are welcome on GitHub.*
