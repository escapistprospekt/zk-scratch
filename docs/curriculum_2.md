## Stage 1: Finite field

**1. Theory and purpose.** All ZK arithmetic happens in a finite field: integers mod a prime p, with +, −, ×, ÷. Fields have no rounding or overflow surprises, so polynomials, hashes and proofs behave exactly. A prime like Goldilocks (2^64 − 2^32 + 1) is chosen so elements fit in a machine word and so p − 1 is divisible by a large power of two, which enables the NTT in stage 2.

**2. Description.** A `Field` trait, a `Goldilocks` type with add, sub, mul, neg, inverse, pow and roots of unity, and later a degree-2 extension field. Done when property tests confirm the field laws and 7 is verified as a generator.

**3. Read before coding.**
- Victor Shoup, *A Computational Introduction to Number Theory and Algebra* (free PDF): the chapters on congruences and finite fields.
- RareSkills ZK Book (free online): the opening chapters on finite fields and modular arithmetic.

**4. Optional.**
- *Handbook of Applied Cryptography* (free chapters): the efficient-implementation chapter, for Montgomery multiplication and modular reduction.
- Montgomery's 1985 paper "Modular Multiplication Without Trial Division".

**5. Production references.**
- Plonky3: `p3-goldilocks`, `p3-baby-bear`, `p3-mersenne-31`, `p3-field`.
- `arkworks` (`ark-ff`), `lambdaworks` (math crate), `winter-math`.

Compare how they do the Goldilocks reduction with your naive `u128 % p`. Do that after writing yours.

## Stage 2: Polynomials and NTT

**1. Theory and purpose.** A polynomial of degree d is determined by d+1 values, so you can represent it by coefficients or by evaluations and convert between them. Almost every proof system works by encoding data as polynomials and checking claims about them at random points. The NTT is the fast conversion (O(n log n)), and it makes multiplication, interpolation and low-degree extension cheap.

**2. Description.** A `Poly<F>` type with Horner evaluation, naive multiplication, NTT and inverse NTT, NTT-based multiplication, Lagrange interpolation, and division by x^n − 1. Done when the NTT product equals the naive product on random inputs, `intt(ntt(a)) == a`, and you've benchmarked the crossover size.

**3. Read before coding.**
- Vitalik Buterin, "Fast Fourier Transforms" (blog, 2019). The clearest NTT walkthrough I know of.
- CLRS, the chapter on polynomials and the FFT, for the even/odd split.

**4. Optional.**
- Berrut and Trefethen, "Barycentric Lagrange Interpolation" (SIAM Review, 2004).
- Cooley and Tukey's original 1965 paper.
- Bit-reversal and in-place radix-2 variants in the arkworks `ark-poly` docs.

**5. Production references.**
- Plonky3 `p3-dft`, `ark-poly` (radix-2 evaluation domains), the FFT module in `winter-math`.

Production versions are parallelized and cache-aware, and are worth reading once your naive version works.

## Stage 3: Hash, Merkle tree and transcript

**1. Theory and purpose.** A Merkle tree commits to a long list with one 32-byte root and lets you open any entry with a short path. Fiat-Shamir turns an interactive proof into a non-interactive one by replacing the verifier's random challenges with hashes of everything said so far. If the transcript leaves anything out, a malicious prover can cheat. This stage is small but it is where many real-world soundness bugs live.

**2. Description.** SHA-256 implemented from the spec, a Merkle tree with `open` and `verify`, and a `Transcript` with `absorb`, `challenge` (field element) and `challenge_index`. Done when SHA-256 passes the NIST test vectors and every kind of tampering with a leaf, index or path is rejected.

**3. Read before coding.**
- NIST FIPS 180-4: the SHA-256 sections (padding, message schedule, compression).
- Thaler, *Proofs, Arguments, and Zero-Knowledge*: the section on the Fiat-Shamir transformation.

**4. Optional.**
- Fiat and Shamir, "How to Prove Yourself" (1986).
- Bernhard, Pereira and Warinschi, "How Not to Prove Yourself: Pitfalls of the Fiat-Shamir Heuristic" (2012).
- Merkle's 1987 signature paper and the `merlin` crate's design notes.

**5. Production references.**
- RustCrypto `sha2`, Plonky3 `p3-merkle-tree` and `p3-challenger`.
- The `merlin` transcript crate, and `spongefish` (a Fiat-Shamir library from the arkworks organization).

Compare how production transcripts label and domain-separate messages. Mine will be simpler.

## Stage 4: Sumcheck

**1. Theory and purpose.** Sumcheck lets a prover convince a verifier that `Σ g(x)` over all 2^n points of the Boolean hypercube equals a claimed value H. The verifier does only O(n) rounds of low-degree checks plus one evaluation of g, instead of 2^n work. It is the engine under GKR, which proves layered computation, and under many modern proof systems. It's also the closest stage to the `sumcheck` and `gkr` crates in `subnet-2`'s dependency tree.

**2. Description.** A `MultilinearPoly` (evaluations over {0,1}^n) with multilinear-extension evaluation and `fix_first_variable`, then a prover and verifier for plain sumcheck, then products of two multilinear polynomials. Done when honest proofs pass, a wrong H is rejected, and flipping any single round-polynomial coefficient is rejected.

**3. Read before coding.**
- Thaler, *Proofs, Arguments, and Zero-Knowledge*: the sections on multilinear extensions and the sum-check protocol, in that order.

**4. Optional.**
- Lund, Fortnow, Karloff and Nisan, "Algebraic Methods for Interactive Proof Systems" (1992).
- Goldwasser, Kalai and Rothblum, "Delegating Computation: Interactive Proofs for Muggles", for GKR.
- Xie et al., "Libra" (CRYPTO 2019), and Setty, "Spartan" (2020), for how sumcheck is used in practice.

**5. Production references.**
- `ark-linear-sumcheck` in the arkworks organization, PolyhedraZK's `Expander` (the GKR prover that JSTProve builds on), and a16z's `jolt`.
- Microsoft's `Spartan` repository.

Expander is the one to read if you want to understand `subnet-2`'s proving side.

## Stage 5: FRI

**1. Theory and purpose.** FRI proves that a committed function is close to a low-degree polynomial, without revealing it. Each round folds the function in half using a random challenge, shrinking the degree and domain together, until a constant remains. It replaces the pairing-based trusted setup of earlier SNARKs with just hashes, which is why STARKs are called "transparent."

**2. Description.** Parameters (blowup factor, query count, final degree), a commit phase that Merkle-commits each folded layer, and a query phase that spot-checks consistency between layers. Done when a true low-degree polynomial verifies and a polynomial with degree one too high is rejected with overwhelming probability across many seeds.

**3. Read before coding.**
- Alan Szepieniec, *Anatomy of a STARK*: the "Basic Tools" and FRI parts.
- Vitalik Buterin, "STARKs, Part II: Thank Goodness It's FRI-day".

**4. Optional.**
- Ben-Sasson, Bentov, Horesh and Riabzev, the FRI paper (ICALP 2018).
- Ben-Sasson, Goldberg, Kopparty and Saraf, "DEEP-FRI".
- Ben-Sasson, Carmon, Ishai, Kopparty and Saraf, "Proximity Gaps for Reed-Solomon Codes", for soundness analysis.
- Haböck, Levit and Papini, "Circle STARKs".

**5. Production references.**
- Plonky3 `p3-fri`, `winter-fri`, StarkWare's `stwo` (Circle STARKs over Mersenne-31), Plonky2's FRI module.

## Stage 6: Mini-STARK

**1. Theory and purpose.** A STARK proves that a computation was run correctly. You write the computation as a trace table, express the rules as polynomial constraints (an AIR), combine all constraints into one composition polynomial, and use FRI to show it is low degree. If the claim is false, the composition polynomial isn't low degree and FRI catches it. This stage ties every earlier stage together.

**2. Description.** A Fibonacci proof: trace of length 2^k, a transition constraint and boundary constraints, quotients by the zerofiers, a random linear combination, low-degree extension on a coset, Merkle commitment, FRI, and query openings that check the constraint relation. Done when honest proofs verify and a trace with one corrupted value, or a wrong claimed output, is rejected.

**3. Read before coding.**
- StarkWare's Stark 101 (all parts, with code).
- *Anatomy of a STARK*, the STARK part, after the FRI part.
- Vitalik Buterin, "STARKs, Part I".

**4. Optional.**
- Ben-Sasson et al., "Scalable, Transparent, and Post-Quantum Secure Computational Integrity" (ePrint 2018/046).
- StarkWare's ethSTARK documentation.
- RISC Zero's "STARK by Hand" tutorial.
- Szepieniec's follow-up parts on speeding things up and Rescue-Prime.

**5. Production references.**
- `winterfell` (has a Fibonacci example), Plonky3's `uni-stark` crate (with a Fibonacci AIR test), `stwo`.
- Full zkVMs built on the same ideas: Miden VM, RISC Zero, SP1.

Compare your hand-written constraint handling with how Winterfell and Plonky3 define an AIR. Production code uses a trait-based constraint API instead of polynomials written out by hand.

## Order of use

For each stage, read part 3, write the code, then read part 5 to see what you'd do differently, and only then go into part 4. Reading production code first is the common way to end up copying instead of learning.

If you'd like this as a document you can keep and tick off, I can turn it into one.