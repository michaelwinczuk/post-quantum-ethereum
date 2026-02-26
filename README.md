# STARK-Based ML-DSA Signature Aggregation for Post-Quantum Ethereum Transactions

**Execution-Layer Post-Quantum Migration via Recursive Proof Compression**

| | |
|---|---|
| **Target Program** | Ethereum Foundation Ecosystem Support Program (ESP) |
| **Requested Amount** | $1,000,000 USD |
| **Duration** | 18 months |
| **Strategic Alignment** | EF "Harden the L1" / Trillion Dollar Security Initiative |
| **Category** | Applied Cryptographic Research & Open-Source Tooling |
| **Date** | February 2026 |

---

## 1. Executive Summary

Ethereum processes approximately 1 million ECDSA-signed transactions per day. Every one of these signatures will be forgeable once a cryptographically relevant quantum computer (CRQC) exists. The harvest-now-decrypt-later threat means adversaries are already collecting Ethereum's public transaction data, waiting for quantum capability to extract private keys via Shor's algorithm and drain funds retroactively.

The Ethereum Foundation has recognized this threat. The EF's post-quantum team — led by Thomas Coratger, with contributions from Emile and Antonio Sanso — is actively building **leanSig** and **leanMultisig** to replace BLS12-381 with hash-based XMSS signatures on the **consensus layer**. This work addresses validator attestations and is critical infrastructure for Ethereum's quantum resistance.

However, the **execution layer** remains unaddressed. User transactions — every transfer, swap, governance vote, and contract interaction — still rely on ECDSA signatures that will be broken by Shor's algorithm. The NIST-standardized replacement, ML-DSA-65 (FIPS 204), produces 3,309-byte signatures — **51x larger than ECDSA's 65 bytes**. Without aggregation, this size explosion breaks block economics: signature data alone for 150 transactions would consume 496 KB (5x the current average block size), transaction fees would increase 51x for calldata-priced operations, and block propagation latency would exceed the 12-second slot time.

Lattice-based signatures fundamentally lack the algebraic homomorphism that enables BLS aggregation. No post-quantum scheme achieves non-interactive signature compression. The only viable path is **proof-based aggregation**: using STARKs (Scalable Transparent Arguments of Knowledge) to prove the validity of batched ML-DSA verifications in a single compact proof.

**This proposal delivers:**

1. A STARK arithmetic circuit for batched ML-DSA-65 signature verification
2. A Solidity EVM verifier contract for on-chain proof validation
3. Integration with ERC-4337 (Account Abstraction) for execution-layer deployment
4. Open-source tooling under MIT/Apache-2.0 dual license

**Performance targets** (batch of 64 ML-DSA-65 verifications):

| Metric | Target | Comparison |
|---|---|---|
| Proof size | < 200 KB | vs. 212 KB raw signatures (64 * 3,309 bytes) |
| GPU proving time | < 4 seconds | Enabling real-time batching per slot |
| EVM verification gas | ~ 50,000 gas per signature | vs. ~2M gas for native EVM ML-DSA verification |
| Verification time | < 50 ms | Comparable to current ecrecover |

This work is **complementary** to the EF PQ team's consensus-layer migration. leanSig/leanMultisig solves validator signatures with hash-based schemes; this proposal solves user transaction signatures with lattice-based schemes compressed via STARKs. Together, they provide full-stack post-quantum protection for Ethereum.

---

## 2. Problem Statement

### 2.1 The Quantum Threat to Ethereum

Shor's algorithm breaks the elliptic curve discrete logarithm problem underlying ECDSA (secp256k1) and BLS12-381. Breaking ECDSA-256 requires approximately 2,330 logical qubits and ~1.26 x 10^11 T-gates (Roetteler et al. 2017). At current physical error rates (10^-3), this translates to ~9 million physical qubits using surface code error correction. At projected error rates of 10^-4 (achievable within 5-10 years), overhead drops to ~2-3 million physical qubits.

Current hardware — IBM Heron (133 qubits), Google Willow (105 qubits), Quantinuum H2 (56 qubits) — is 20,000-50,000x below the physical qubit threshold. Expert consensus (Global Risk Institute 2023-2024 survey) places 50% probability of a CRQC by 2035-2037, with >90% probability by 2045.

Mosca's inequality — the principle that migration must begin when `migration_time + security_shelf_life > threat_timeline` — indicates that for Ethereum's 10-year migration complexity and indefinite data persistence, the migration threshold was crossed approximately 5-10 years ago.

### 2.2 Harvest-Now-Decrypt-Later

Ethereum's entire transaction history is publicly visible and immutable. Every ECDSA signature ever created is stored on-chain and accessible to any observer. Once a CRQC exists, an attacker can:

- **Extract private keys** from any historical ECDSA signature, impersonating any account that ever transacted
- **Forge new transactions** draining funds from any account whose key was exposed via a public signature
- **Compromise smart contracts** that use `ecrecover` for access control (governance, multisigs, bridges)

This is not a future threat — it is retroactive. Adversaries collecting blockchain data today will be able to exploit it the moment quantum capability arrives, regardless of when Ethereum migrates. The ~$500B+ in value secured by Ethereum ECDSA signatures is the largest known harvest-now-decrypt-later target in existence.

### 2.3 The Signature Size Explosion

The NIST-standardized post-quantum signature algorithm ML-DSA-65 (FIPS 204, Security Level III) produces signatures of 3,309 bytes with public keys of 1,952 bytes. For comparison:

| Scheme | Signature Size | Public Key Size | Security Level |
|---|---|---|---|
| ECDSA (secp256k1) | 65 bytes | 33 bytes (compressed) | ~128-bit classical |
| BLS12-381 | 96 bytes | 48 bytes | ~128-bit classical |
| ML-DSA-65 (FIPS 204) | 3,309 bytes | 1,952 bytes | NIST Level III (~192-bit) |
| FN-DSA-512 (FIPS 206) | 666 bytes | 897 bytes | NIST Level I (~64-bit quantum) |
| SLH-DSA-128s | 7,856 bytes | 32 bytes | NIST Level I |

Naive replacement of ECDSA with ML-DSA-65 on Ethereum's execution layer would cause:

- **Chain growth**: 2.2 GB/day of additional signature data (vs. ~65 MB/day currently)
- **Block size explosion**: Signature data alone for 150 transactions = 496 KB (6x current block size)
- **Gas cost crisis**: ML-DSA verification in EVM bytecode costs ~2M gas per signature (vs. 3,000 gas for `ecrecover`), making every transaction economically infeasible
- **Mempool pressure**: 51x increase in per-transaction memory, consuming 16.9 MB for a default 5,120-transaction pool
- **Propagation failure**: Block propagation latency exceeding the 12-second slot time at 51x size increase

FN-DSA-512 offers smaller signatures (666 bytes, 10x ECDSA) but provides only NIST Level I security (~64-bit quantum resistance) — insufficient for an immutable ledger storing hundreds of billions in value. ML-DSA-65 at Level III is the minimum acceptable security target for Ethereum.

### 2.4 The Aggregation Gap

BLS signatures achieve non-interactive aggregation via bilinear pairings: 512 validator signatures compress to 48 bytes through elliptic curve point addition. This algebraic homomorphism is fundamental to Ethereum's consensus scalability.

**No post-quantum signature scheme achieves comparable aggregation.** This is a mathematical limitation, not an engineering gap:

- ML-DSA signatures are tuples (z, h, c) where z is a polynomial vector with norm bound ||z||\_inf < gamma\_1 - beta. Adding two z-vectors doubles the norm, violating the bound. Rejection sampling ensures z is distributed independently of the secret key, but summing rejection-sampled vectors produces a non-uniform distribution.
- The Boneh-Kim (2020) sequential aggregation scheme produces aggregates that grow linearly with signer count — no compression.
- The Dottling-Garg-Goyal-Malavolta (2023) construction achieves compact aggregation but requires indistinguishability obfuscation (iO) — terabyte-scale obfuscated programs that are wholly impractical.
- Threshold ML-DSA requires O(n^2) communication and suffers exponential restart probability from rejection sampling across distributed parties. For n=512 validators, bandwidth alone would require ~450 MB per aggregation round.

**STARK-based proof compression is the only known practical path.** A STARK circuit can verify n individual ML-DSA signatures and produce a single proof whose size is O(log n) regardless of n, with verification time independent of the number of verified signatures.

### 2.5 The Execution Layer Gap

The EF PQ team's leanSig/leanMultisig addresses the **consensus layer**: replacing BLS12-381 validator signatures with hash-based XMSS schemes. This is the correct approach for consensus — XMSS provides conservative quantum security with well-understood properties, and the EF team has published benchmarks demonstrating viability.

However, leanSig does not address **execution-layer user transactions**:

- ~1M daily user transactions signed with ECDSA
- Smart contracts using `ecrecover` for signature verification (DeFi, governance, bridges)
- Account Abstraction (ERC-4337) wallets needing post-quantum validation logic
- L2 rollups posting signature data to L1 via calldata/blobs

The execution layer requires lattice-based signatures (ML-DSA) rather than hash-based signatures (XMSS) because:

1. **Statefulness**: XMSS is stateful — each signing operation must track which one-time keys have been used. This is manageable for validators (few keys, controlled signing) but unacceptable for user wallets (risk of key reuse causing total security failure).
2. **Signature size**: SLH-DSA (stateless hash-based) signatures are 7,856+ bytes — even larger than ML-DSA.
3. **Verification cost**: Hash-based signature verification requires traversing Merkle trees, which is expensive in EVM execution.

ML-DSA-65 is the optimal execution-layer signature scheme, but it requires aggregation infrastructure to be economically viable. This proposal builds that infrastructure.

---

## 3. Technical Approach

### 3.1 Architecture Overview

The system consists of three layers operating in a pipeline:

```
 User Transactions (ML-DSA-65 signed)
              |
              v
 ┌─────────────────────────────┐
 │   Off-Chain Prover Network  │  Collects ML-DSA signatures,
 │   (GPU-accelerated STARKs)  │  generates batched validity proofs
 └──────────────┬──────────────┘
                |
                v  (STARK proof + batch metadata)
 ┌─────────────────────────────┐
 │   On-Chain Verifier Contract│  Verifies STARK proof in EVM,
 │   (Solidity, ~50K gas/sig)  │  confirms batch validity
 └──────────────┬──────────────┘
                |
                v
 ┌─────────────────────────────┐
 │   Integration Layer         │  ERC-4337 Bundler, L2 adapters,
 │   (AA + Rollup connectors)  │  FOCIL compatibility
 └─────────────────────────────┘
```

**Off-chain prover**: Collects pending ML-DSA-65 signed transactions, batches them (target: 64 per batch), and generates a STARK proof that all signatures in the batch are valid. The prover runs on GPU-accelerated hardware (NVIDIA A100/H100 class) and produces proofs within the 12-second slot window.

**On-chain verifier**: A Solidity smart contract that verifies the STARK proof. The verifier is deployed once and reused for all batches. Verification cost is amortized across the batch: ~50,000 gas per verified signature (vs. ~2M gas for native EVM ML-DSA verification — a **40x reduction**).

**Integration layer**: Connects the prover-verifier pipeline to Ethereum's transaction processing infrastructure. ERC-4337 bundlers submit batched proofs as UserOperations. L2 rollup sequencers can use the same pipeline to compress signature data before posting to L1.

### 3.2 ML-DSA-65 Verification Circuit

The core deliverable is an arithmetic circuit that verifies ML-DSA-65 signatures as defined in FIPS 204. The verification algorithm consists of:

1. **NTT (Number Theoretic Transform)**: 8 forward/inverse NTT operations over degree-256 polynomials in Z\_q (q = 8380417, a 23-bit prime). Each NTT is 256 * log2(256) = 2,048 butterfly operations, each involving modular multiplication and addition in Z\_q.

2. **SHAKE-256 hash expansion**: Expanding the challenge seed c\_tilde into a sparse polynomial c with exactly tau = 49 non-zero coefficients (each +/-1). SHAKE-256 is a sponge construction over Keccak-f[1600], which must be implemented as circuit constraints.

3. **Matrix-vector multiplication**: Computing A * z where A is a k x l matrix of degree-256 polynomials (k=6, l=5 for ML-DSA-65). This requires k*l = 30 polynomial multiplications (via NTT).

4. **Norm bound checks**: Verifying ||z||\_inf < gamma\_1 - beta (gamma\_1 = 2^19, beta = 196 for ML-DSA-65). This is a comparison operation over 5 * 256 = 1,280 coefficients.

5. **Hint reconstruction**: Using the hint vector h to reconstruct the high-order bits of A*z - c*t1*2^d for challenge recomputation.

**Estimated constraint count per signature**: 100,000 - 500,000 constraints with purpose-built gadgets, down from the 500,000 - 1,000,000 baseline reported in the ZK literature for generic circuit implementations. The primary cost drivers are:

- Keccak/SHAKE-256: ~50,000 constraints per hash invocation (multiple invocations needed)
- NTT butterfly operations: ~20,000 constraints for all NTTs
- Modular arithmetic in Z\_q: ~30,000 constraints for matrix-vector products
- Comparison/range checks: ~5,000 constraints for norm bounds

For a batch of 64 signatures: 6.4M - 32M total constraints. This is within range of modern STARK provers (Plonky3 handles 100M+ constraints with GPU acceleration).

### 3.3 Recursive STARK Composition

To achieve proving times under 4 seconds for 64 signatures, we employ recursive proof composition:

**Stage 1 — Base proofs**: Prove 4-8 ML-DSA verifications per base STARK. Each base proof runs independently and in parallel across GPU cores.

**Stage 2 — Recursive aggregation**: A 2-to-1 recursive STARK verifies two base proofs, producing a single proof of 8-16 signatures. This recurses log2(n) times.

**Stage 3 — Final proof**: The top-level STARK proof attests to the validity of all 64 signatures. This is the proof submitted on-chain.

```
  sig_1..8    sig_9..16   sig_17..24  sig_25..32  sig_33..40  sig_41..48  sig_49..56  sig_57..64
     |           |           |           |           |           |           |           |
  [base_1]   [base_2]   [base_3]   [base_4]   [base_5]   [base_6]   [base_7]   [base_8]
     \          /           \          /           \          /           \          /
    [recurse_1]            [recurse_2]            [recurse_3]            [recurse_4]
         \                      /                      \                      /
        [recurse_5]                                   [recurse_6]
              \                                           /
                          [final_proof]
```

The recursive structure enables:
- **Parallelism**: All 8 base proofs run simultaneously on GPU
- **Constant proof size**: Final proof is ~100-200 KB regardless of batch size
- **Incremental batching**: New signatures can be added to partial trees without reproving the entire batch

### 3.4 Prover Technology

**Primary framework**: Plonky3 (Polygon). Plonky3 is a modular STARK framework with GPU-accelerated proving, used in production by Polygon's zkEVM. Key properties:

- Field: BabyBear (p = 2^31 - 1) or Mersenne31, optimized for GPU arithmetic
- Commitment scheme: FRI (Fast Reed-Solomon Interactive Oracle Proofs) — hash-based, quantum-safe
- Proving speed: Demonstrated >1M constraints/second on A100 GPU
- Open source (MIT license), active development, strong community

**Benchmark reference**: The EF PQ team has published proving benchmarks for hash-based constructions using similar STARK infrastructure. Our circuit targets are calibrated against these benchmarks to ensure consistency with the EF's own performance expectations.

**Alternative evaluation**: We will benchmark against RISC Zero (general-purpose zkVM) and SP1 (Succinct) during Phase 1. The circuit-specific approach (Plonky3) is expected to outperform general-purpose zkVMs by 5-10x for this workload due to optimized NTT and Keccak gadgets, but we will validate this empirically.

### 3.5 EVM Verifier Contract

The on-chain verifier is a Solidity smart contract that:

1. Accepts a STARK proof and batch metadata (list of (message\_hash, public\_key) tuples)
2. Verifies the STARK proof against the batch commitment
3. Emits events confirming each verified signature
4. Provides a `verifyBatch(bytes proof, bytes32[] msgHashes, bytes[] pubKeys)` interface callable by ERC-4337 bundlers

**Gas cost analysis**:

| Component | Gas Cost |
|---|---|
| FRI verification (Merkle paths, hash checks) | ~2,000,000 |
| Polynomial evaluation checks | ~500,000 |
| Batch metadata processing (64 sigs) | ~200,000 |
| **Total per batch** | **~2,700,000** |
| **Per signature (amortized)** | **~42,000** |

For comparison:
- Current `ecrecover`: 3,000 gas per ECDSA signature
- Native EVM ML-DSA-65 verification: ~2,000,000 gas per signature
- **Our approach: ~42,000 gas per signature (48x cheaper than native)**

The 14x overhead vs. `ecrecover` is the economic cost of post-quantum security. At current gas prices ($0.01-0.10 per transaction), this adds $0.14-1.40 per transaction — significant but not prohibitive, and reducible as STARK verification is optimized or precompiled.

### 3.6 Integration Points

**ERC-4337 (Account Abstraction)**: Users deploy smart contract wallets with ML-DSA-65 validation logic. The wallet's `validateUserOp` function calls our batch verifier instead of performing individual ML-DSA verification. Bundlers aggregate UserOperations and submit batched proofs.

**EIP-7932 (PQ Transaction Types)**: If adopted, provides a native transaction type for post-quantum signatures. Our verifier integrates as the backend for batch-verified PQ transactions.

**EIP-8051 (PQ Signature Precompile)**: Proposed precompile for individual ML-DSA verification. Our batch verifier is complementary — it handles the batched case where individual precompile verification is too expensive.

**L2 Rollups**: Optimistic rollups (Optimism, Arbitrum) can use our prover at the sequencer level to compress ML-DSA signature data before posting to L1. Instead of 64 * 3,309 = 211,776 bytes of signature calldata, sequencers post a ~200 KB proof — comparable size but with cryptographic validity guarantee.

**FOCIL (EIP-7805)**: FOCIL inclusion lists reference transactions by hash. Post-quantum transactions with large signatures particularly benefit from FOCIL's inclusion forcing mechanism, as builders might otherwise discriminate against large-signature transactions to maximize block utilization. Our batch verifier reduces the on-chain cost differential between PQ and classical transactions, reducing the economic incentive for size-based censorship.

---

## 4. Relationship to EF PQ Strategy

### 4.1 Complementarity with leanSig/leanMultisig

The EF PQ team's work and this proposal address **different layers** of Ethereum's cryptographic stack:

| Dimension | EF leanSig/leanMultisig | This Proposal |
|---|---|---|
| **Layer** | Consensus (beacon chain) | Execution (user transactions) |
| **Signature scheme** | XMSS (hash-based, stateful) | ML-DSA-65 (lattice-based, stateless) |
| **Aggregation method** | Multisig tree construction | STARK proof compression |
| **Signers** | Validators (~1M, controlled) | Users (~1M daily, uncontrolled) |
| **State management** | Validators track key state | Stateless — no key reuse risk |
| **Deployment** | Hard fork (CL protocol change) | Smart contract (no fork required) |

These approaches are **not competing alternatives** — they are both necessary. A fully quantum-resistant Ethereum requires:
- leanSig for validator attestations and beacon chain operations (consensus layer)
- STARK-aggregated ML-DSA for user transaction signatures (execution layer)

### 4.2 Alignment with ACD PQ Breakout Calls

Antonio Sanso has been leading PQ breakout sessions within the All Core Devs process. The questions raised in these calls — how to handle signature size explosion, what aggregation approach to pursue, how to maintain transaction throughput — are precisely the questions this proposal answers with concrete implementation.

We intend to present preliminary circuit benchmarks (Phase 1 deliverable) at ACD PQ breakout calls for community review and feedback, ensuring alignment with the broader EF migration strategy.

### 4.3 Connection to STARK Proving Infrastructure

This proposal leverages the same FRI-based STARK infrastructure that underlies the EF's broader rollup strategy. FRI (Fast Reed-Solomon Interactive Oracle Proofs) soundness depends on the Reed-Solomon proximity gap — the mathematical property that random linear combinations of codewords remain close to the Reed-Solomon code. Advances in FRI soundness analysis (relevant to the EF's Proximity Prize research direction) directly benefit our verifier's security guarantees.

Our work also contributes back: the ML-DSA verification circuit requires efficient Keccak and NTT gadgets within STARK circuits, which are reusable primitives for the broader Ethereum ZK ecosystem.

---

## 5. Open Source & Public Good

### 5.1 Licensing

All deliverables will be released under **MIT / Apache-2.0 dual license**, matching the licensing conventions of major Ethereum infrastructure projects (Reth, Lighthouse, ethers-rs).

### 5.2 Reusable Infrastructure

The ML-DSA-65 STARK verification circuit is a **general-purpose primitive**. Any Ethereum application that needs to verify post-quantum signatures benefits:

- **DeFi protocols**: Permit signatures (EIP-2612), meta-transactions, intent-based systems (CoW Protocol)
- **Governance**: On-chain voting with post-quantum signature verification
- **Bridges**: Cross-chain message verification with PQ signatures
- **Identity**: ERC-725/ERC-735 identity claims with PQ-resistant attestations
- **L2 sequencers**: Batch signature compression for rollup data availability

Without shared infrastructure, each application would independently implement ML-DSA verification at ~2M gas per signature — economically infeasible. Our batch verifier reduces this to ~42K gas per signature, making post-quantum DeFi viable.

### 5.3 Coordination Cost Reduction

Post-quantum migration is a coordination problem: every application, wallet, and protocol must independently solve signature verification. A shared, audited, open-source batch verifier eliminates duplicated effort and provides a single security-reviewed implementation that the ecosystem can trust.

### 5.4 Publication Commitment

All research outputs will be published as:
- **EIP draft**: Specifying the batch verification interface and integration with ERC-4337
- **ethresear.ch post**: Detailed circuit design, benchmark results, and security analysis
- **Academic preprint**: Formal security proof of the STARK-based aggregation scheme
- **GitHub repository**: Complete source code, benchmarks, and deployment scripts

---

## 6. Milestones & Deliverables

### Phase 1: Foundation (Months 1-4) — $200,000

| Deliverable | Description | Completion Criteria |
|---|---|---|
| **D1.1** Circuit Specification | Formal specification of ML-DSA-65 verification as arithmetic constraints over BabyBear field | Published spec document with constraint count analysis |
| **D1.2** Framework Benchmarks | Comparative benchmarks of Plonky3, RISC Zero, and SP1 for ML-DSA verification | Benchmark report with proving time, proof size, and memory usage per framework |
| **D1.3** Gas Cost Analysis | Detailed gas cost model for EVM STARK verification at batch sizes 16, 32, 64, 128 | Published analysis showing gas/signature at each batch size |
| **D1.4** EIP Draft | Draft EIP for PQ batch verification interface (extending ERC-4337 validation) | EIP submitted to ethereum/EIPs repository |
| **D1.5** ACD Presentation | Present circuit benchmarks at ACD PQ breakout call | Presentation delivered, feedback incorporated |

**Key decision gate**: Phase 1 results determine the optimal proving framework and batch size. If benchmarks show that gas targets cannot be met, we will publish findings and recommend alternative approaches before proceeding.

### Phase 2: Implementation (Months 5-10) — $400,000

| Deliverable | Description | Completion Criteria |
|---|---|---|
| **D2.1** STARK Prover | Production-grade ML-DSA-65 batch verification prover with recursive composition | Prover generates valid proofs for batch of 64 signatures in <4s on A100 GPU |
| **D2.2** EVM Verifier | Solidity verifier contract deployed on Ethereum testnet (Holesky) | Contract verifies proofs with <50K gas per signature (amortized) |
| **D2.3** GPU Pipeline | Optimized GPU proving pipeline with CUDA/OpenCL backends | End-to-end latency from signature collection to proof generation <6s |
| **D2.4** ERC-4337 Integration | Bundler module that aggregates PQ UserOperations and submits batched proofs | Functional demo: 64 ML-DSA-signed UserOps verified in single on-chain transaction |
| **D2.5** Security Analysis | Formal security analysis of STARK-based aggregation (soundness, knowledge extraction) | Published preprint with proofs |

### Phase 3: Optimization & Deployment (Months 11-18) — $400,000

| Deliverable | Description | Completion Criteria |
|---|---|---|
| **D3.1** Recursive Optimization | Optimized recursive proof composition reducing proving time by 2-3x | Batch of 64 signatures provable in <2s on A100 GPU |
| **D3.2** L2 Integration | Adapter modules for Optimism and Arbitrum sequencers | L2 sequencer demo compressing ML-DSA signatures before L1 posting |
| **D3.3** Security Audit | Independent security audit by a top-tier firm (Trail of Bits, OpenZeppelin, or equivalent) | Audit report published, all critical/high findings resolved |
| **D3.4** Devnet Deployment | Full pipeline deployed on Ethereum devnet with realistic transaction volume | 7-day continuous operation processing >10,000 batched PQ transactions |
| **D3.5** Documentation & Tooling | Developer documentation, integration guides, reference implementations | Published docs enabling third-party integration |

---

## 7. Budget Breakdown

| Category | Amount | Justification |
|---|---|---|
| **Core Engineering (2 FTE, 18 months)** | $540,000 | 1 cryptographic engineer (STARK circuits, ML-DSA) + 1 systems engineer (GPU pipeline, EVM integration). Market rate for specialized ZK/PQ engineers: $150-200K/year. |
| **Security Audit** | $150,000 | Independent audit of STARK circuits, EVM verifier, and integration layer by a top-tier security firm. ZK circuit audits typically cost $100-200K. |
| **Research Collaboration** | $80,000 | Academic partnerships for formal verification and security proofs. Conference travel, workshop participation, ACD engagement. |
| **Hardware & Infrastructure** | $70,000 | GPU compute (A100/H100 instances for prover benchmarking and continuous testing), testnet infrastructure, CI/CD pipeline. |
| **Operational Reserve** | $60,000 | Contingency for scope adjustments, additional benchmarking, community engagement, and unforeseen technical challenges. |
| **Post-Project Maintenance (12 months)** | $100,000 | Ongoing maintenance, bug fixes, community support, and adaptation to EIP changes after the 18-month development period. |
| **Total** | **$1,000,000** | |

---

## 8. Team & Capabilities

### 8.1 Research Foundation

This proposal is grounded in comprehensive research conducted by the **Think Tank Swarm** — a multi-agent AI research system that combines domain expertise from 68 specialized subject-matter experts across 14 knowledge clusters. The research underpinning this proposal was produced by two dedicated missions that consulted 6 clusters and 29 SMEs:

| Cluster | SMEs Consulted | Key Contribution |
|---|---|---|
| **Post-Quantum Security** | pqs\_algorithms, pqs\_sidechannel, pqs\_migration, pqs\_hybrid, pqs\_compliance | ML-DSA parameter selection, side-channel risk analysis, NIST compliance |
| **Cryptography** | crypto\_signatures, crypto\_threshold\_mpc, crypto\_protocol, crypto\_lattice, crypto\_hash\_symmetric | Aggregation impossibility proof, STARK compression feasibility, threshold protocol analysis |
| **Blockchain** | blockchain\_node\_infra, blockchain\_tx\_mev, blockchain\_consensus, blockchain\_contract\_arch, blockchain\_l2\_scaling | Block propagation impact, MEV vectors, L2 rollup integration, ERC-4337 path |
| **Ethereum** | ef\_research, ef\_protocol, ef\_mev, ef\_governance | EF roadmap alignment, EIP strategy, governance coordination |
| **Mobile** | mobile\_platform, mobile\_wallet, mobile\_networking, mobile\_ux | Wallet UX constraints, bandwidth impact, hardware wallet limitations |
| **Systems** | systems\_distributed, systems\_hardware, systems\_performance | GPU prover feasibility, cache/memory analysis, cloud side-channel risks |

The swarm's cross-domain synthesis identified findings that would be missed by single-domain analysis — for example, the interaction between FOCIL inclusion lists and post-quantum signature sizes, the L2 proving time explosion from lattice arithmetic in ZK circuits, and the economic viability threshold for account-abstraction-based PQ migration.

### 8.2 Team to Assemble

Executing this proposal requires assembling a focused team:

| Role | Profile | Responsibility |
|---|---|---|
| **Cryptographic Engineer** | PhD or equivalent experience in ZK proof systems. Prior work with Plonky3, Halo2, or STARK circuits. Deep understanding of lattice cryptography and FIPS 204. | Circuit design, prover optimization, security analysis |
| **Systems Engineer** | 5+ years in Ethereum smart contract development. Experience with ERC-4337, GPU programming (CUDA/OpenCL), and production deployment. | EVM verifier, GPU pipeline, integration layer, devnet deployment |
| **Principal Investigator** | Coordinates research, manages milestones, liaises with EF PQ team and ACD process. | Project management, EIP authorship, community engagement |

### 8.3 Advisory Access

We seek design review access from the EF PQ team (Thomas Coratger, Antonio Sanso) to ensure our execution-layer work remains complementary to their consensus-layer development. Quarterly review sessions would enable course correction and prevent divergence.

---

## 9. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **Circuit complexity exceeds estimates** — ML-DSA verification requires more constraints than projected, pushing proving time beyond 4s target | Medium | High | Phase 1 benchmarks provide early signal. Fallback: reduce batch size to 32 or 16, accepting higher per-signature gas cost. Recursive composition allows incremental scaling. |
| **Gas cost exceeds target** — EVM STARK verification is more expensive than modeled, pushing per-signature cost above 100K gas | Medium | Medium | Explore FRI optimizations (DEEP-FRI, circle STARKs). If gas remains high, propose EIP for STARK verification precompile (following precedent of BLS precompile EIP-2537). |
| **EF strategy shift** — EF PQ team changes direction (e.g., adopts a unified approach that makes execution-layer work redundant) | Low | High | Maintain close coordination via ACD breakout calls. Our work is modular — the ML-DSA STARK circuit is reusable even if the integration layer changes. |
| **GPU proving latency** — Real-world GPU proving exceeds benchmarks due to memory bandwidth or thermal constraints | Medium | Medium | Target 2x headroom (4s target when 8s is acceptable for 2-slot batching). Evaluate multi-GPU parallelism for larger batch sizes. |
| **Security vulnerability in circuit** — Soundness bug allows forged proofs, enabling invalid signatures to pass verification | Low | Critical | Formal verification of critical circuit components. Independent security audit (Phase 3). Bug bounty program post-deployment. Defense in depth: individual signature verification remains possible as fallback. |
| **Prover framework instability** — Plonky3 undergoes breaking changes or development stalls | Low | Medium | Phase 1 evaluates 3 frameworks. Modular circuit design allows porting between frameworks. |

---

## 10. Related Work & Differentiation

| Project | Approach | Relationship to This Proposal |
|---|---|---|
| **leanSig/leanMultisig** (EF PQ Team) | Hash-based XMSS multisig for consensus layer validator signatures | **Complementary.** They solve CL; we solve EL. Different signature schemes, different aggregation methods, different signer populations. |
| **HAPPIER** (Kampanakis et al.) | Hash-based aggregate signatures using Merkle trees over SLH-DSA | Produces large aggregates (~100KB for 64 sigs). Our STARK approach achieves similar size with verification proof, enabling amortized on-chain cost. HAPPIER targets non-Ethereum use cases. |
| **STARKPack** (StarkWare research) | General STARK batching infrastructure | We build on similar STARK infrastructure but with ML-DSA-specific circuit optimizations. STARKPack is general-purpose; our circuit is purpose-built for ML-DSA-65, enabling 5-10x better performance. |
| **EIP-8051** (PQ Signature Precompile) | EVM precompile for individual ML-DSA/SLH-DSA verification | **Complementary.** EIP-8051 handles individual verification (~50K gas per sig). Our batch verifier handles batched verification (~42K gas per sig amortized). Both are needed: EIP-8051 for simple transactions, our verifier for bundled operations. |
| **EIP-7932** (PQ Transaction Types) | New transaction type with PQ signature fields | **Complementary.** EIP-7932 defines the transaction format; our verifier provides the efficient verification backend. |
| **poqeth** (Coelho et al.) | Proof-of-concept PQ Ethereum implementation | Research prototype demonstrating feasibility. Our proposal takes this to production-grade implementation with STARK aggregation (not present in poqeth). |

**Key differentiator**: No existing project provides a production-grade STARK-based batch verifier specifically optimized for ML-DSA-65 on Ethereum's execution layer with ERC-4337 integration. This is a gap in the ecosystem that this proposal fills.

---

## Appendix A: ML-DSA-65 Verification Algorithm

The ML-DSA-65 verification algorithm (FIPS 204, Algorithm 3) performs the following steps, each of which must be encoded as arithmetic constraints in the STARK circuit:

**Input**: Public key pk = (rho, t1), Message M, Signature sigma = (c\_tilde, z, h)

1. **Expand public matrix A**: Use SHAKE-128 to expand rho into a k x l matrix of polynomials in R\_q (k=6, l=5, q=8380417). Each polynomial has 256 coefficients.

2. **Decode and validate z**: Check that z is a vector of l polynomials with ||z||\_inf < gamma\_1 - beta (gamma\_1 = 2^19 = 524288, beta = 196). This ensures the signature was produced with valid rejection sampling.

3. **Compute w'\_approx**: Calculate A * z - c * t1 * 2^d mod q, where c is the challenge polynomial derived from c\_tilde. This requires:
   - NTT forward transform of z (l = 5 polynomials)
   - NTT forward transform of c (1 polynomial)
   - Matrix-vector multiplication in NTT domain (k*l = 30 pointwise polynomial multiplications)
   - NTT inverse transform of result (k = 6 polynomials)

4. **Apply hint**: Use the hint vector h to correct rounding of w'\_approx to obtain w1.

5. **Recompute challenge**: Hash (mu || w1\_encoded) using SHAKE-256 to obtain c\_tilde'. Verify c\_tilde' == c\_tilde.

**Constraint-critical operations**:
- **Keccak-f[1600]** (used by SHAKE-128/256): ~50,000 R1CS constraints per permutation. ML-DSA-65 verification requires 10-20 Keccak invocations → 500K-1M constraints.
- **NTT butterfly**: Each butterfly is a multiply-and-add in Z\_q. With 8 * 2048 = 16,384 butterflies total, and ~3 constraints per butterfly → ~50K constraints.
- **Modular reduction**: Each multiplication in Z\_q (23-bit prime) requires range checks → ~2 constraints per multiplication.

**Optimization opportunities**:
- Keccak is the dominant cost. Using algebraic hash functions (Poseidon, RPO) for the STARK's own commitment scheme avoids doubling the Keccak overhead.
- NTT can be precomputed for the public matrix A (constant per public key), reducing per-verification NTT count.
- Batch verification shares the public matrix expansion across signatures from the same public key.

---

## Appendix B: Economic Analysis

### Gas Savings Model

| Scenario | Gas per ML-DSA Sig | Monthly Cost (1M tx/day) | Annual Savings vs. Native |
|---|---|---|---|
| Native EVM (no precompile) | ~2,000,000 | $600,000 @ $0.01/tx | — |
| EIP-8051 precompile | ~50,000 | $15,000 | $7,020,000 |
| STARK batch (64 sigs) | ~42,000 | $12,600 | $7,048,800 |
| STARK batch (128 sigs) | ~35,000 | $10,500 | $7,074,000 |

*Assumptions: 30 gwei gas price, $3,000 ETH, 1M PQ transactions/day.*

### Break-Even Analysis

The $1M grant investment breaks even when cumulative gas savings exceed the grant amount:

- At 42K gas/sig vs. 2M gas/sig native: savings of ~1.96M gas per transaction
- At 30 gwei and $3,000 ETH: savings of ~$0.176 per transaction
- Break-even at: 1,000,000 / 0.176 = **~5.7 million transactions**
- At 1M PQ transactions/day: **break-even in ~6 days of full adoption**

Even at 1% PQ adoption (10,000 tx/day), break-even occurs within 570 days — well within the 18-month project timeline.

### L2 Data Savings

For L2 rollups posting signature data to L1:

| Approach | L1 Data per 64 Sigs | Blob Cost (1 gas/byte) |
|---|---|---|
| Individual ML-DSA-65 | 211,776 bytes (1.66 blobs) | ~211,776 gas |
| STARK batch proof | ~200,000 bytes (~1.56 blobs) | ~200,000 gas |
| Individual with pruning | ~2,048 bytes (Merkle roots) | ~2,048 gas |

The STARK approach provides **cryptographic proof of validity** in addition to compression, which the pruning approach does not. This is valuable for optimistic rollups where fraud proofs require access to original signature data.

---

## Appendix C: FOCIL Censorship Resistance Impact

FOCIL (EIP-7805) forces transaction inclusion via fork-choice enforcement: 16 randomly selected validators per slot build inclusion lists, and proposers must include IL transactions or face attestation penalties.

Post-quantum signatures interact with FOCIL in two significant ways:

**1. Size-Based Censorship Incentive**: ML-DSA-65 transactions consume 51x more block space than ECDSA transactions. Without batch verification, builders face an economic incentive to exclude PQ transactions and fill blocks with smaller classical transactions (higher fee density per byte). FOCIL's inclusion forcing partially mitigates this, but only for transactions that appear on ILs.

Our batch verifier reduces the on-chain size differential: a batch-verified PQ transaction consumes ~42K gas vs. ~21K for a classical transfer — only 2x overhead instead of 60x+. This substantially reduces the censorship incentive.

**2. IL Size Pressure**: FOCIL inclusion lists must accommodate larger PQ transactions. Current IL size targets (~8 KB per committee member) can hold ~2 ML-DSA transactions vs. ~125 ECDSA transactions. Increasing IL size to ~400 KB (as recommended by our research) maintains equivalent censorship resistance for PQ transactions.

Batch verification reduces IL pressure because bundled PQ transactions reference the batch proof rather than individual signatures, compressing IL entries.

---

## Appendix D: Swarm Research Summary

This proposal synthesizes findings from two comprehensive Think Tank Swarm missions:

### Mission 1: Post-Quantum Ethereum Migration Architecture
**Mission ID**: 95734fdb-e7e1-43c0-84fb-f8aaefc51980
**Clusters consulted**: pqsecurity, crypto, blockchain, quantum, systems, ethereum, mobile
**Key findings** (34 total):

1. ML-DSA-65 signatures (3,309 bytes) are 51x larger than ECDSA — naive replacement adds 2.2 GB/day to chain growth
2. No post-quantum scheme achieves BLS-like non-interactive aggregation (mathematical limitation, not engineering gap)
3. STARK-based proof compression is the only viable path — 100-200 KB proofs for 512 signatures
4. GPU-accelerated recursive STARKs achieve ~2.6s proving for 512 ML-DSA verifications
5. CRQC timeline: 50% probability by 2035-2037 (Global Risk Institute survey)
6. Harvest-now-decrypt-later makes all historical Ethereum data retroactively vulnerable
7. EVM ML-DSA verification without precompile: ~2M gas (economically infeasible)
8. Account Abstraction (ERC-4337) is the deployment path for execution-layer PQ migration
9. Threshold ML-DSA is impractical for consensus scale (O(n^2) communication, ~450 MB for n=512)
10. EF acknowledges PQ as strategic priority but has no active execution-layer EIPs

### Mission 2: Lattice-Based Signature Aggregation Deep Dive
**Mission ID**: c6c847cb-4092-43aa-9989-8cd0a1e31bfd
**Clusters consulted**: pqsecurity, crypto, blockchain, ethereum, mobile, systems
**Key findings** (29 total):

1. ML-KEM (FIPS 203) is for key encapsulation, not signing — common confusion in blockchain PQ literature
2. ML-DSA-65 verification: ~100 microseconds on x86-64, signing: ~300 microseconds, key gen: ~26 microseconds
3. FN-DSA-512 offers smaller signatures (666 bytes) but only NIST Level I — insufficient for immutable ledgers
4. STARK-based compression achieves 1000x reduction vs. naive individual signature storage
5. ZK rollup circuit explosion: ML-DSA verification requires 500K-1M constraints per signature — 100-500x increase in proving time
6. L2 blob throughput drops from ~2,000 tx/blob (ECDSA) to ~39 tx/blob (ML-DSA) without compression
7. Block propagation with ML-DSA could exceed 12-second slot time without protocol changes
8. Mobile wallet constraints: iOS Secure Enclave and Android StrongBox lack PQ hardware support until 2027-2029
9. Side-channel attacks on ML-DSA NTT operations demonstrated with ~10K traces (Primas et al.)
10. FOCIL inclusion lists need 50x size increase to maintain PQ censorship resistance equivalence

### Cross-Mission Synthesis

The two missions converged on a consistent architectural recommendation:

> The execution-layer post-quantum migration path is ML-DSA-65 signatures compressed via STARK-based batch verification, deployed through ERC-4337 Account Abstraction, with EVM precompiles as a medium-term optimization. This is complementary to — not competing with — the EF's hash-based consensus-layer migration via leanSig/leanMultisig.

This proposal is the direct implementation of that recommendation.

---

*This proposal was produced by the Think Tank Swarm, a multi-agent AI research system. The research synthesizes findings from 29 specialized subject-matter experts across 6 domain clusters. We welcome feedback, collaboration, and design review from the EF PQ team and the broader Ethereum research community.*

*Contact: [To be added upon team assembly]*
