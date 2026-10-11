## Stage 1: Field

**1. Theory and Purpose**
Finite fields are the mathematical bedrock of all zero-knowledge proofs. Standard computer arithmetic (floating-point or unbounded integers) either loses precision or grows infinitely large. A prime field restricts all numbers to a strict range ($0$ to $p-1$) where operations wrap around a prime modulus. This ensures absolute precision, prevents overflow, and guarantees that division (multiplicative inverses) is always mathematically well-defined.

**2. Description**
Implement the Goldilocks prime field ($p = 2^{64} - 2^{32} + 1$). You will build a `Field` trait and implement modular addition, subtraction, multiplication, and negation. You will also implement a square-and-multiply algorithm for fast exponentiation, extended Euclidean algorithm (or Fermat's Little Theorem) for multiplicative inverses, and logic to generate roots of unity for subgroups of order $2^k$.

**3. Theory Resources to View Before Coding (Important)**

* *Vitalik Buterin's blog*: "[Quadratic Arithmetic Programs: from Zero to Hero](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649)" (specifically the primer on prime fields and modular math).
* *Stark 101*: Part 1 (Field element implementation concepts).

**4. Theory Resources (Optional)**

* *Handbook of Applied Cryptography* by Menezes et al.: Chapters covering modular reduction and Montgomery/Barrett multiplication (if you decide to optimize beyond naive `% p`).

**5. Reference Implementations**

* **Plonky3**: `p3-goldilocks` crate.
* **Winterfell**: `winter-math` crate (specifically their 64-bit prime field implementations).

---

## Stage 2: Poly

**1. Theory and Purpose**
Polynomials are used to encode data (like execution traces) into continuous mathematical objects. By representing data as polynomials, we can use the Schwartz-Zippel lemma, which states that two different low-degree polynomials will almost never intersect at the same points. To manipulate these massive polynomials efficiently, we use the Number Theoretic Transform (NTT), which is a Fast Fourier Transform (FFT) adapted for finite fields, bringing polynomial multiplication from $O(n^2)$ down to $O(n \log n)$.

**2. Description**
Build a univariate polynomial struct capable of Horner's method evaluation, naive multiplication, and division by the vanishing polynomial ($x^n - 1$). The core milestone is implementing the NTT and Inverse NTT (iNTT) to quickly convert polynomials between coefficient form and evaluation form over a domain of roots of unity.

**3. Theory Resources to View Before Coding (Important)**

* *Introduction to Algorithms (CLRS)*: The chapter on Polynomials and the FFT. This is mandatory for understanding how the Vandermonde matrix splits into even and odd halves.
* *Anatomy of a STARK* by Alan Szepieniec: Section on Polynomials and the NTT.

**4. Theory Resources (Optional)**

* *Vitalik Buterin's blog*: "STARKs, Part I: Proofs with Polynomials".

**5. Reference Implementations**

* **arkworks**: `ark-poly` crate (dense polynomial structs and evaluations).
* **Plonky3**: `p3-dft` (Discrete Fourier Transform implementations).

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