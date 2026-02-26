# pq-discovery — Post-Quantum Cryptography Scanner

> **Relevant to**: [@CISA](https://github.com/CISA) | [@AWS](https://github.com/aws) | [@OpenAI](https://github.com/openai)
>
> Scans codebases for quantum-vulnerable cryptography and recommends NIST PQC replacements.
> Critical for federal CNSA 2.0 compliance, cloud infrastructure hardening, and AI system security.

## What It Does

An open-source Rust CLI that scans directory trees for:

| Quantum-Vulnerable | Recommended Replacement |
|---|---|
| RSA-2048, RSA-4096 | ML-KEM-768/1024 (key exchange), ML-DSA-65/87 (signatures) |
| ECDSA (secp256k1, P-256, P-384) | ML-DSA-65/87 |
| Ed25519, X25519 | ML-DSA-65 (signatures), ML-KEM-768 (key exchange) |
| ECDH (P-256, P-384) | ML-KEM-768/1024 |
| DSA | ML-DSA-65/87 |
| 3DES, Blowfish, RC4 | AES-256-GCM |

## Output

Structured JSON/SARIF 2.1.0 reports with:
- File path, line number, detected primitive
- Risk level (CRITICAL/HIGH/MEDIUM/LOW) stratified by context (blockchain vs TLS vs test code)
- Recommended NIST PQC replacement with migration guidance
- CI/CD integration (GitHub Actions, GitLab CI)

## Architecture

- **Two-phase scanning**: Fast regex pre-filter + language-native AST parsing
- **Language support**: Rust (syn), Go (go/parser), Python (ast), C/C++ (libclang), Java, JavaScript/TypeScript, Solidity
- **Config formats**: OpenSSL, GnuTLS, Java KeyStore, PEM/DER/PKCS12 certificates
- **Performance**: Sub-60s for 10K-file repos, incremental Git-based scanning (90-99% CI time reduction)
- **Parallelism**: Hybrid tokio + rayon for I/O and CPU saturation

## Status

**Research phase complete.** The architect brief in [`brief/pq-discovery_brief.json`](brief/pq-discovery_brief.json) contains the full implementation spec produced by the [Think Tank Swarm](https://github.com/michaelwinczuk/think-tank-swarm) — 68 SMEs across 14 knowledge clusters.

Ready for architect build.

## License

MIT / Apache-2.0 dual license
