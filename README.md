# Algebraic Attacks on Post-Quantum Cryptography

A study notes repository covering the historical evolution and current state of algebraic attacks, with emphasis on why lattice-based schemes (ML-DSA, ML-KEM) structurally resist the techniques that broke HFE, Rainbow, and SIDH.

## Scope

This repository contains technical notes on:

- **Linear algebra attacks**: Gaussian elimination on linear cryptosystems (the original "algebraic attack").
- **Differential algebraic attacks**: Biham–Shamir style differential cryptanalysis, with notes on Keccak/SHAKE resistance margins.
- **Gröbner basis attacks**: Buchberger's algorithm, Faugère's F4/F5, and the historical breakage of HFE (2002), Rainbow (2022), and related multivariate schemes.
- **Lattice reduction attacks**: LLL/BKZ as a member of the algebraic attack family, with cross-references to the companion repository on lattice reduction.
- **Why ML-DSA / ML-KEM are immune to Gröbner basis**: the role of the noise term in breaking the exact algebraic structure.

The notes are written at a level appropriate for engineers and security researchers with a background in undergraduate algebra and basic cryptography. No prior knowledge of multivariate cryptanalysis is assumed.

## Contents

| File                          | Description                                                  |
| ----------------------------- | ------------------------------------------------------------ |
| `algebraic-attacks-on-pqc.md` | Main technical note. Five parts covering definitions, four attack families, history, Gröbner basis deep dive, and case studies (HFE, Rainbow, ML-DSA immunity). |

## Why This Repository Exists

Discussions about post-quantum cryptography often emphasize Shor's algorithm and quantum resistance, while overlooking the **classical** cryptanalytic record that determined which families of schemes survived NIST's standardization process.

Of the four major PQC families NIST evaluated (lattice-based, code-based, hash-based, multivariate, isogeny-based), three were eliminated or downgraded primarily by **classical algebraic attacks**:

- **HFE family** — broken by F5/Gröbner basis (Faugère, 2002).
- **Rainbow** — broken in 53 hours on commodity hardware (Beullens, 2022).
- **SIKE/SIDH** — broken by torsion point attacks using algebraic geometry (Castryck–Decru, 2022).

Lattice-based schemes survived not by accident, but because the **noise term** in LWE-based constructions structurally breaks the exact algebraic equality that Gröbner basis methods require. Understanding this is essential for evaluating any PQC product.

## Reading Order

1. Start with `algebraic-attacks-on-pqc.md` Part 1–2 for the four-family taxonomy.
2. Part 3 (Gröbner basis deep dive) requires familiarity with polynomial rings; the prerequisites section is self-contained.
3. Part 4 (case studies) is independent and can be read first if the reader prefers historical motivation over theory.

## References

The notes are based primarily on the following sources:

- Cox, Little, O'Shea, *Ideals, Varieties, and Algorithms*, Springer (4th ed., 2015).
- Faugère, J.-C., *A new efficient algorithm for computing Gröbner bases (F4)*, JPAA 1999.
- Faugère, J.-C., *A new efficient algorithm for computing Gröbner bases without reduction to zero (F5)*, ISSAC 2002.
- Beullens, W., *Breaking Rainbow Takes a Weekend on a Laptop*, CRYPTO 2022.
- Castryck, W., Decru, T., *An efficient key recovery attack on SIDH*, EUROCRYPT 2023.
- Albrecht, M., Player, R., Scott, S., *On the concrete hardness of Learning with Errors*, JMC 2015.

Specific citations are inline.

## Companion Repositories

- [`pqc-lattice-reduction-notes`](https://github.com/codebat-research/pqc-lattice-reduction-notes) — LLL, BKZ, primal/dual attacks on LWE, Core-SVP methodology.
- [`pqc-sieving-cryptanalysis-notes`](https://github.com/codebat-research/pqc-sieving-cryptanalysis-notes) — LDSieve/BDGL16 LSH analysis, MATZOV BDD reduction.

## License

Content released under [Creative Commons Attribution 4.0 (CC BY 4.0)](LICENSE).

## Maintainer

Raymond Chang — Codebat Technologies (codebat.ai). Issues, corrections, and pull requests are welcome.
