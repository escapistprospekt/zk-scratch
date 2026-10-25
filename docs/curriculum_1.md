## Stage 1: Field

**1. Theory and Purpose**
Finite fields are the mathematical bedrock of all zero-knowledge proofs. Standard computer arithmetic (floating-point or unbounded integers) either loses precision or grows infinitely large. A prime field restricts all numbers to a strict range ($0$ to $p-1$) where operations wrap around a prime modulus. This ensures absolute precision, prevents overflow, and guarantees that division (multiplicative inverses) is always mathematically well-defined.

**2. Description**
Implement the Goldilocks prime field ($p = 2^{64} - 2^{32} + 1$). You will build a `Field` trait and implement modular addition, subtraction, multiplication, and negation. You will also implement a square-and-multiply algorithm for fast exponentiation, extended Euclidean algorithm (or Fermat's Little Theorem) for multiplicative inverses, and logic to generate roots of unity for subgroups of order $2^k$.

**3. Theory Resources to View Before Coding (Important)**

* *Vitalik Buterin's blog*: "[Quadratic Arithmetic Programs: from Zero to Hero](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649)" (specifically the primer on prime fields and modular math).
* *[Stark 101](https://github.com/lambdaclass/STARK101-rs/blob/main/stark101/src/field.rs)*: Part 1 (Field element implementation concepts).

**4. Theory Resources (Optional)**

* *Handbook of Applied Cryptography* by Menezes et al.: Chapters covering modular reduction and Montgomery/Barrett multiplication (if you decide to optimize beyond naive `% p`).

**5. Reference Implementations**

* **Plonky3**: `p3-goldilocks` crate.
* **Winterfell**: `winter-math` crate (specifically their 64-bit prime field implementations).

---

# Stage 2: Polynomials & the NTT

## 1. Theory and Purpose

Polynomials are the encoding layer of a proof system. An execution trace — a table of field elements — becomes a set of univariate polynomials by interpolating each column over a domain of roots of unity. Once data is polynomial, you get a degree bound, and a degree bound is a commitment you can test cheaply: two distinct polynomials of degree ≤ d agree on at most d points, so over a field of size q a single random evaluation catches a difference with probability at least 1 − d/q. This is the quantitative core of the Schwartz–Zippel argument, and the ratio d/q is what later forces sampling from an extension field when the base field is small.

Constraints are natural in **evaluation form** (check a relation pointwise across the trace). Degree arguments and division are natural in **coefficient form**. The Number Theoretic Transform — the FFT specialized to a finite field, using an element ω of multiplicative order 2^k in place of e^(2πi/n) — is the bridge between the two, converting in O(n log n) instead of O(n²). Nearly all of a prover's polynomial choreography is a decision about which side of that bridge to stand on.

The second structural fact this stage depends on: for a multiplicative subgroup H of order n, the vanishing polynomial is exactly Z_H(x) = xⁿ − 1. On a **coset** gH it takes the constant value gⁿ − 1 at every point. Dividing by Z_H therefore costs one field inversion plus n multiplications — provided you do it on a coset rather than on H, where Z_H is zero everywhere and the division is undefined.

## 2. Description

Build a univariate dense polynomial struct over the Stage 1 field, supporting:

- **Evaluation** by Horner's method; degree tracking with trailing-zero normalization.
- **Arithmetic**: addition, scalar multiplication, naive O(n²) multiplication (retained permanently as a test oracle).
- **Division by a linear factor** (x − a) via synthetic division, O(n). Required for boundary constraints and DEEP quotients downstream.
- **Division by the vanishing polynomial** Z_H(x) = xⁿ − 1, performed pointwise on a coset.
- **Lagrange interpolation** through an arbitrary small point set, and **barycentric evaluation** of a polynomial from its values on H without materializing coefficients.

Then the core milestone:

- **Domain construction.** Derive a two-adic generator ω of order 2^k from the field's multiplicative generator by raising it to (p − 1)/2^k. Build H = {1, ω, ω², …, ωⁿ⁻¹}. Decide and document the padding policy for trace lengths that are not powers of two.
- **NTT and iNTT.** Recursive first, verified against naive evaluation; then iterative in-place. The inverse is the same transform with ω⁻¹ and a final scaling by n⁻¹.
- **Ordering discipline.** Commit explicitly to whether the forward transform consumes or produces bit-reversed order. Start with an explicit bit-reversal pass; the production form pairs decimation-in-frequency forward with decimation-in-time inverse so the two permutations cancel and never execute. Silent ordering mismatches are the most common unexplainable-garbage bug in this stage.
- **Coset NTT and low-degree extension.** Scale coefficients by gⁱ before the transform to evaluate on gH. Interpolate a trace column on H of size n, then evaluate on a coset of size 2n–8n. This is the single most load-bearing operation in a STARK prover.

## 3. Theory Resources to View Before Coding (Important)

- **RareSkills, Number Theoretic Transform series.** Primary text. Works in finite fields from the first page rather than translating from ℂ afterward, and includes dedicated chapters on the inverse of a Vandermonde matrix, the inverse NTT, and FFT-friendly fields used in practice.
- **Vitalik Buterin, "Fast Fourier Transforms."** Short, finite-field-native, runnable Python, and it covers cosets — which almost nothing else at this level does. Read immediately after the above.
- **Anatomy of a STARK (Alan Szepieniec).** *Sequencing note:* the basic-tools part deliberately uses slow O(n²) polynomial arithmetic for readability. The NTT, fast coset evaluation, and fast division appear in the later "Speeding Things Up" part. Check the table of contents — the material is split across two non-adjacent parts.
- **cp-algorithms, "Fast Fourier transform."** Reference for the iterative in-place implementation, bit-reversal, and the NTT section. Consult when writing the iterative version, not before.

## 4. Theory Resources (Optional)

- **CLRS, "Polynomials and the FFT."** Rigorous and well-written, but works over ℂ, omits cosets entirely, and treats the modular case only in passing. Useful for the complexity proofs and the halving lemma; not on the critical path.
- **Vitalik Buterin, "STARKs, Part I: Proofs with Polynomials."** Motivation for why any of this matters. Does not help you write the code.

## 5. Reference Implementations

Read in this order:

- **Winterfell, `math::fft`.** Flattest and most readable: concrete over one field, butterfly loops followable line by line. Start here.
- **arkworks, `ark-poly`.** Dense/sparse polynomial structs, `EvaluationDomain`, coset handling. Good model for API shape.
- **Plonky3, `p3-dft`.** Production structure — `TwoAdicSubgroupDft`, generic over field and packing, with `coset_lde_batch` as exactly the operation built in this stage. Heavily trait-abstracted; read it after Winterfell to see how the concrete version gets generalized, not as a first encounter.
- **lambdaworks** as an optional third perspective.

## 6. Acceptance Tests

The stage is complete when all six pass over random inputs at sizes up to 2¹²:

1. `intt(ntt(p)) == p` for random p, every supported size.
2. NTT output matches naive O(n²) evaluation at every point of H.
3. NTT-based multiplication matches schoolbook multiplication.
4. Synthetic division: for random p and random a, `(p(x) − p(a)) / (x − a)` has no remainder and degree exactly deg(p) − 1.
5. Coset LDE: interpolate random values on H of size n, evaluate on gH of size 8n, interpolate back, confirm the top 7n coefficients are zero.
6. **Round-trip through the vanishing polynomial.** Take random p of degree < n, compute q = p · Z_H, then recover p by dividing q by Z_H on a coset. Must be exact.

Test 6 is the stage's real exit criterion — it cannot pass without correct coset handling, and correct coset handling is what the rest of the system is built on.

---

## Stage 3: Transcript

**1. Theory and Purpose**
Interactive zero-knowledge protocols require a verifier to send unpredictable, random challenges to the prover to prevent cheating. The Fiat-Shamir heuristic removes the need for a live verifier by replacing them with a cryptographic hash function. By hashing every single message the prover generates into a "transcript", the prover can generate secure, deterministic, pseudo-random challenges themselves, making the proof non-interactive.

**2. Description**
Implement three components in a single crate: A SHA-256 hash function (passing NIST test vectors), a standard Merkle Tree for committing to large arrays of data, and a Fiat-Shamir `Transcript` state machine. The transcript must securely absorb prover data and public inputs, and squeeze out field elements or random indices to act as challenges.

**3. Theory Resources to View Before Coding (Important)**

* *Anatomy of a STARK*: The chapter on the Fiat-Shamir heuristic and Transcripts.

**4. Theory Resources (Optional)**

* *FIPS 180-4*: The official NIST specification for the SHA-256 algorithm.

**5. Reference Implementations**

* **Plonky3**: `p3-challenger` (sponge-based transcript) and `p3-commit` (Merkle trees).
* **arkworks**: `ark-sponge` (cryptographic sponge functions used for Fiat-Shamir).

---

## Stage 4: Sumcheck

**1. Theory and Purpose**
The Sumcheck protocol allows a prover to convince a verifier of the sum of a massive multilinear polynomial evaluated over a boolean hypercube (e.g., all combinations of $\{0,1\}^n$). It systematically reduces a claim about a gigantic sum down to a claim about a single, random evaluation point, drastically minimizing the verifier's computation overhead.

**2. Description**
Build a `MultilinearPoly` struct that operates over $n$ variables with $2^n$ evaluations. Implement partial evaluation (fixing the first variable to a constant). Construct the interactive sumcheck protocol: the prover calculates and sends univariate polynomials round-by-round, and the verifier checks them against the running claim, generating transcript challenges to bind variables.

**3. Theory Resources to View Before Coding (Important)**

* *Proofs, Arguments, and Zero-Knowledge* by Justin Thaler: Read the foundational chapters on multilinear extensions and the Sumcheck protocol.

**4. Theory Resources (Optional)**

* *Algebraic methods for interactive proof systems* by Lund, Fortnow, Karloff and Nisan (1992): The original paper establishing the sumcheck protocol.

**5. Reference Implementations**

* **arkworks-ext/Spartan**: Reference implementations of multilinear extensions and sumcheck in the broader Arkworks ecosystem.

---

## Stage 5: FRI

**1. Theory and Purpose**
Fast Reed-Solomon Interactive Oracle Proofs of Proximity (FRI) proves that a massive, committed dataset actually represents evaluations of a low-degree polynomial. It does this by repeatedly splitting the polynomial into even and odd coefficients, taking a random linear combination of them (folding), and halving the domain. This recursive halving shrinks the data down to a single constant, allowing the verifier to check it with high cryptographic soundness.

**2. Description**
Implement the FRI prover and verifier. The prover will execute the commit phase, mathematically folding the domain and building a Merkle tree for each halved layer. The verifier will execute the query phase, utilizing the transcript to request random evaluation indices, verifying Merkle paths, and ensuring the mathematical relationship holds between folded layers.

**3. Theory Resources to View Before Coding (Important)**

* *Anatomy of a STARK*: The FRI chapter (this provides cleaner indexing theory than almost any other source).
* *Stark 101*: Part 3 (FRI Commit) and Part 4 (FRI Query).

**4. Theory Resources (Optional)**

* *Fast Reed-Solomon Interactive Oracle Proofs of Proximity* by Ben-Sasson et al. (The original FRI paper).

**5. Reference Implementations**

* **Winterfell**: `winter-fri` crate (a robust, heavily optimized FRI implementation).
* **Plonky3**: `p3-fri` crate.

---

## Stage 6: STARK

**1. Theory and Purpose**
The culmination of the system. A STARK uses polynomials and FRI to prove that a specific computation (like a Fibonacci sequence or a virtual machine execution) was performed correctly without revealing the underlying data. It converts state transitions into mathematical rules (polynomial constraints), ensuring that if the constraints hold true, the computation was valid.

**2. Description**
Build a complete Fibonacci proof system. You will generate an execution trace of a Fibonacci sequence, use the NTT to interpolate it, and define transition and boundary constraints. You will divide these constraints by the zerofier to construct the massive composition polynomial, mix it with random weights, and pass it directly into your FRI module to generate the final zero-knowledge proof.

**3. Theory Resources to View Before Coding (Important)**

* *Stark 101*: Parts 1 and 2 (Trace and Constraints).
* *Anatomy of a STARK*: Chapters on Arithmetization and the complete STARK protocol.

**4. Theory Resources (Optional)**

* *Vitalik Buterin's blog*: "STARKs, Part II and III".
* *Scalable, transparent, and post-quantum secure computational integrity* by Ben-Sasson et al. (The original STARK whitepaper).

**5. Reference Implementations**

* **Winterfell**: `winter-prover` and `winter-verifier` crates.
* **Plonky3**: The main `plonky3` crate integrating all prior primitives into a STARK.