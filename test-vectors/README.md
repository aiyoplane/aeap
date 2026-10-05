# AEAP Conformance Test Vectors

This directory will hold the normative conformance test vectors referenced in **Appendix A** of the AEAP specification.

## Status

**Pending v1.0.** The test vector artifact is the gating item for AEAP v1.0.0 — the specification cannot be stamped from `v0.9.1-DRAFT` to `v1.0.0` without the vectors being authored, published, and passed identically by at least two independent reference implementations.

The specification's Appendix A §A.3 states the interoperability guarantee the vectors must prove:

> Two implementations conformant at the same Conformance Level, evaluating the same canonical Intent against a Policy identified by the same Policy digest, with the same canonical set of Evidence Signal values, MUST produce the same Verdict, the same reason code, and (for the signing-input portion) the same canonicalized bytes.

The vectors operationalize that guarantee into an executable test suite.

## Scope (per Appendix A §A.1)

The vector suite will cover:

- **Intent validation vectors** (§3) — required/optional field handling, atomicity violations, canonicalization round-trips through RFC 8785 JCS, supersession chains
- **Policy evaluation vectors** (§4) — ALLOW/ESCALATE/BLOCK decision paths, composition under the AEAP Default Composition Algebra, Policy silence, determinism, Policy version transitions, Policy digest reproducibility
- **Evidence Signal vectors** (§5) — value/provenance/attestation decomposition, all five signal categories, carried/collected/observed timings, freshness evaluation
- **Verdict output vectors** (§6) — all canonical reason codes, Core-vs-Extended ESCALATE behavior including `ESCALATE_UNSUPPORTED`, strict replay rejection
- **Receipt format vectors** (§7, §8) — v2 Ed25519 generation and verification; signature validation; `exp` enforcement; `shadow` claim semantics; supersession-chain reconstruction; evidence-envelope-ref dereferencing; verifier result code taxonomy (`VERIFIED`, `VERIFIED_EXPIRED`, `COMPROMISED_KEY`, `UNVERIFIABLE`, `INVALID`, `INTENT_MISMATCH`)
- **Interop Profile vectors** (§11, §12, §13) — translation surfaces for HAP, Okta XAA/ID-JAG, and Alibaba/Cisco AOA

**Important:** v1 (HMAC) development-profile vectors MAY be provided separately for reference-implementation and local-development testing, but MUST NOT contribute to any AEAP base-protocol conformance determination (per §8.1).

## Authoring guidelines

The specification's pre-v1.0 TODO document enumerates specific edge cases the vectors must cover, including:

- JSON serialization edge cases (Unicode normalization, duplicate keys, numeric boundaries, nested structures, key ordering)
- Numeric edge cases (IEEE 754 double-precision boundaries, very large integers, decimal precision)
- Monetary/precision cases (domain-convention representations)
- Policy digest determinism across whitespace-equivalent policies
- Receipt verification across all six result-code outcomes
- Interop Profile semantic-category extraction consistency

## Format (planned)

Each vector will pair:

- A canonical input (Intent, Policy, Evidence Signal set, or receipt)
- The expected canonicalized bytes
- For receipt vectors: the expected signing input, the expected signature (against a declared test key pair), and the expected Verifier result code
- For Verdict vectors: the expected Verdict and reason code

Implementations that produce different bytes, different signatures, or different result codes are non-conformant.

## License

Content in this directory is licensed under [Apache License 2.0](./LICENSE-TEST-VECTORS).

## Contact

Questions, contributions, or test-vector proposals: info@aiyoplane.com or GitHub issues on this repository.
