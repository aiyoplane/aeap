# Contributing to AEAP

AEAP is published as an open specification for review, comment, and third-party implementation. This document explains how to contribute.

## Current cycle

AEAP is at **v0.9.1-DRAFT** — a public Request-for-Comment draft. The next milestone is v1.0, which is gated on the Appendix A conformance test vector artifact being authored and passed identically by at least two independent reference implementations. We welcome:

- **Review and comment** on the specification prose
- **Independent implementations** of AEAP in any language or runtime
- **Test vector contributions** toward the Appendix A conformance suite
- **Edge-case reports** — serialization issues, policy composition surprises, verifier result code gaps, canonicalization divergences
- **Interop Profile contributions** — translation surfaces to additional external authorization systems beyond the HAP, XAA, and AOA profiles in §11-§13

## How to submit feedback

**GitHub Issues** on this repository are the preferred channel for most feedback — specification clarifications, errata, design questions, edge cases, missing requirements. Please tag issues:

- `clarification` — the specification language is ambiguous
- `errata` — a factual or cross-reference error
- `edge-case` — a case the current specification does not clearly resolve
- `interop-profile` — proposal for a new Interop Profile or an update to an existing one
- `test-vector` — contribution to or question about Appendix A test vectors
- `discussion` — strategic or architectural question for public discussion

**Email** at info@aiyoplane.com for private feedback, security concerns, or discussion that is not suitable for a public issue tracker. Response commitment: within 30 days of receipt.

## Guidelines for substantive change proposals

Change proposals that affect normative behavior SHOULD include:

- The specific sections affected (e.g., §4.5.1, §7.3)
- The current normative behavior and the proposed replacement
- A rationale explaining why the change improves interoperability, security, or specification precision
- Where applicable, a draft of the exact replacement language
- Any backward-compatibility implications

Change proposals affecting only editorial clarity, cross-references, or informative notes do not require this structure but should still identify the sections affected.

## What AEAP will NOT accept

To preserve architectural discipline (see §1.5 and the specification's "spec is source of truth" principle), AEAP will not accept:

- Proposals that make the base protocol prescriptive about Policy expression languages (e.g., mandating Rego, Cedar, or any specific DSL)
- Proposals that collapse the Evidence Signal three-component structure (value / provenance / attestation) into a single notion of "evidence"
- Proposals that make settlement, identity, or any single evidence category required at Core conformance in the general case
- Proposals that introduce execution obligation (ALLOW does not mean the Composition Boundary must execute)
- Proposals that elevate the HMAC v1 development profile to base-protocol conformance eligibility
- Proposals that make the specification depend on, endorse, or be governed by any single external authorization system

Proposals of this form can be accommodated as **AEAP Profiles** (see §1.5) — optional additive conformance declarations — rather than as changes to the base protocol.

## Independent implementations

AEAP is open to third-party implementations under the same conformance criteria that apply to the reference implementations. Implementers do NOT need to coordinate with Aiyoplane, Inc. except insofar as the Appendix A test vectors evolve across specification versions.

An implementation declares conformance by:

- Publishing a conformance report naming the test vectors it passes (per §A.2 pass criteria)
- Declaring its Conformance Level (Core, Extended, or Full) and any supported Interop Profiles

Third-party conformance reports are welcomed; Aiyoplane, Inc. does not gatekeep implementation announcements and does not require pre-authorization for implementers to use the AEAP name in describing their work.

## Code of conduct

Public discussion in issues and PRs is expected to be substantive, respectful, and focused on improving the protocol. Participants are expected to engage in good faith.

## License

Contributions to the specification prose are accepted under [CC BY 4.0](./LICENSE-SPECIFICATION). Contributions to the conformance test vectors are accepted under [Apache License 2.0](./test-vectors/LICENSE-TEST-VECTORS). By submitting a contribution, you confirm that you have the right to grant these licenses for your contribution.

---

**Contact:** info@aiyoplane.com · [aiyoplane.com](https://aiyoplane.com) · [aiyoplane.com/trust](https://aiyoplane.com/trust)
