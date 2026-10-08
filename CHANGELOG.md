# Changelog

## 0.9.2-DRAFT (October 2026)

Additive update. Adds §15 **Security Model and Responsibility Boundaries**. §15 codifies the responsibility boundary between AEAP Core, the RDP Implementation, and the Protected Party / Executor; names the properties AEAP Core guarantees; and explicitly names the properties AEAP Core does NOT guarantee — exactly-once execution, pre-expiration revocation of individual receipts, policy correctness, Evidence Signal truthfulness, enforcement exhaustiveness at the Composition Boundary in a given deployment, and complete execution lineage without cooperative Executor participation.

No change to any previously-defined element of AEAP Core. No change to receipt format, signing, verification, or canonicalization rules. Conformant 0.9.1-DRAFT implementations remain conformant to 0.9.2-DRAFT.

## 0.9.1-DRAFT (October 2026)

Initial public draft of AEAP. Published for review and comment ahead of v1.0, which will be gated on Appendix A conformance test vectors passing identically against the reference implementations. Feedback welcomed per Appendix D of the specification.
