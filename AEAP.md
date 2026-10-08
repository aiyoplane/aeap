# Aiyo Execution Authorization Protocol (AEAP)

**Version:** 0.9.2-DRAFT
**Status:** Public Draft — Request for Comment
**Published:** [DRAFT — publish date pending]
**Author:** Aiyoplane, Inc. — https://aiyoplane.com
**Document subtitle:** Aiyo Kernel Specification
**License:** This specification is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). Conformance test vectors are licensed under [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).

---

## Abstract

This document specifies the **Aiyo Execution Authorization Protocol (AEAP)**, an open protocol for authorization decisions at the boundary between autonomous intent and consequential execution. AEAP defines a Runtime Decision Point that evaluates a proposed action against a policy authored by the party being protected, returns a ternary verdict (ALLOW, ESCALATE, or BLOCK), and emits a cryptographically-signed, portable Execution Receipt that any downstream system can verify offline. The protocol is agnostic to the identity of the acting party, the rail on which value moves (if any), the wallet or signer involved (if any), and the specific category of consequential action being authorized.

AEAP is designed for use at any boundary where an autonomous system — a human-initiated workflow, an AI agent, or a machine-to-machine process — proposes to perform an action with real-world consequences. The protocol is substrate-neutral: payment, DevOps deployment, procurement, data write, inter-agent delegation, and other consequential-action classes are all first-class use cases.

Reference implementations of AEAP are published under Apache 2.0 across two independent language ecosystems. The composition pattern at the Model Context Protocol boundary is implemented in Node at [`github.com/aiyoplane/mcp-authz`](https://github.com/aiyoplane/mcp-authz) and in Python at [`github.com/aiyoplane/mcp-authz-python`](https://github.com/aiyoplane/mcp-authz-python). A reference verifier for AEAP Execution Receipts is published in Node at [`npmjs.com/package/@aiyoplane/verify`](https://www.npmjs.com/package/@aiyoplane/verify) and in Python at [`pypi.org/project/aiyoplane-verify`](https://pypi.org/project/aiyoplane-verify/). The two language implementations in each pair share no vendored code and derive their behavior independently from this specification; interoperability between them is established by the conformance test vectors in Appendix A.

---

## Table of Contents

1. [Terminology and Conformance](#1-terminology-and-conformance)
   - 1.1 [Document conventions](#11-document-conventions)
   - 1.2 [Defined terms](#12-defined-terms)
   - 1.3 [Conformance targets](#13-conformance-targets)
   - 1.4 [Conformance levels](#14-conformance-levels)
   - 1.5 [Optional AEAP Profiles](#15-optional-aeap-profiles)
2. [The Runtime Decision Point](#2-the-runtime-decision-point)
   - 2.1 [Purpose and architectural position](#21-purpose-and-architectural-position)
   - 2.2 [Architectural invariants](#22-architectural-invariants)
   - 2.3 [The evaluation lifecycle](#23-the-evaluation-lifecycle)
   - 2.4 [Composition boundaries](#24-composition-boundaries)
   - 2.5 [Non-goals](#25-non-goals)
   - 2.6 [Adjacent architectural layers (informative)](#26-adjacent-architectural-layers-informative)
3. [Intent Representation](#3-intent-representation)
   - 3.1 [Intent as an atomic authorization subject](#31-intent-as-an-atomic-authorization-subject)
   - 3.2 [Required Intent fields](#32-required-intent-fields)
   - 3.3 [Optional Intent fields](#33-optional-intent-fields)
   - 3.4 [Canonical Intent representation](#34-canonical-intent-representation)
   - 3.5 [Intent uniqueness and replay](#35-intent-uniqueness-and-replay)
4. [Policy Evaluation Model](#4-policy-evaluation-model)
   - 4.1 [Policy as Protected-Party-authored declarative rules](#41-policy-as-protected-party-authored-declarative-rules)
   - 4.2 [Policy versioning and binding](#42-policy-versioning-and-binding)
   - 4.3 [Policy evaluation determinism](#43-policy-evaluation-determinism)
   - 4.4 [Policy silence and the fail-closed boundary](#44-policy-silence-and-the-fail-closed-boundary)
   - 4.5 [Policy composition](#45-policy-composition)
5. [Evidence Signals](#5-evidence-signals)
   - 5.1 [Evidence Signal as an open-set input](#51-evidence-signal-as-an-open-set-input)
   - 5.2 [Signal categories](#52-signal-categories)
   - 5.3 [Required-for-Core signals](#53-required-for-core-signals)
   - 5.4 [Signal provenance and freshness](#54-signal-provenance-and-freshness)
   - 5.5 [Signal collection timing](#55-signal-collection-timing)
6. [Ternary Verdict Semantics](#6-ternary-verdict-semantics)
   - 6.1 [The three Verdicts](#61-the-three-verdicts)
   - 6.2 [ALLOW semantics](#62-allow-semantics)
   - 6.3 [ESCALATE semantics](#63-escalate-semantics)
   - 6.4 [BLOCK semantics](#64-block-semantics)
   - 6.5 [Verdict output and reason codes](#65-verdict-output-and-reason-codes)
7. [Execution Receipt Format](#7-execution-receipt-format)
   - 7.1 [Receipt as portable attestation](#71-receipt-as-portable-attestation)
   - 7.2 [Receipts across the ternary verdict](#72-receipts-across-the-ternary-verdict)
   - 7.3 [Required and optional claims](#73-required-and-optional-claims)
   - 7.4 [Receipt structure and canonicalization](#74-receipt-structure-and-canonicalization)
   - 7.5 [Receipt lineage model](#75-receipt-lineage-model)
   - 7.6 [Receipt expiration and acceptance window](#76-receipt-expiration-and-acceptance-window)
8. [JWKS and Key Rotation](#8-jwks-and-key-rotation)
   - 8.1 [Receipt Signature Profiles](#81-receipt-signature-profiles)
   - 8.2 [Published JWKS endpoint](#82-published-jwks-endpoint)
   - 8.3 [Key rotation discipline](#83-key-rotation-discipline)
   - 8.4 [Verifier behavior on unknown or rotated keys](#84-verifier-behavior-on-unknown-or-rotated-keys)
9. [Composition Boundary Invariants](#9-composition-boundary-invariants)
   - 9.1 [What a Composition Boundary MUST do](#91-what-a-composition-boundary-must-do)
   - 9.2 [What a Composition Boundary MUST NOT do](#92-what-a-composition-boundary-must-not-do)
   - 9.3 [Fail-closed opt-out at the boundary](#93-fail-closed-opt-out-at-the-boundary)
10. [Rollout Discipline](#10-rollout-discipline)
    - 10.1 [Shadow Mode](#101-shadow-mode)
    - 10.2 [Enforce Mode](#102-enforce-mode)
    - 10.3 [Transition between modes](#103-transition-between-modes)
11. [Interop Profile: Human Agency Protocol (HAP)](#11-interop-profile-human-agency-protocol-hap)
12. [Interop Profile: Okta Cross App Access and ID-JAG](#12-interop-profile-okta-cross-app-access-and-id-jag)
13. [Interop Profile: Alibaba / Cisco Agent Operation Authorization (AOA)](#13-interop-profile-alibaba--cisco-agent-operation-authorization-aoa)
14. [Versioning and Backward Compatibility](#14-versioning-and-backward-compatibility)
    - 14.1 [Specification versioning](#141-specification-versioning)
    - 14.2 [Receipt Signature Profile versioning](#142-receipt-signature-profile-versioning)
    - 14.3 [Policy versioning in cross-RDP contexts](#143-policy-versioning-in-cross-rdp-contexts)
15. [Security Model and Responsibility Boundaries](#15-security-model-and-responsibility-boundaries)
    - 15.1 [Responsibility boundaries](#151-responsibility-boundaries)
    - 15.2 [Properties AEAP Core guarantees](#152-properties-aeap-core-guarantees)
    - 15.3 [Properties AEAP Core explicitly does not guarantee](#153-properties-aeap-core-explicitly-does-not-guarantee)
    - 15.4 [Receipt validity over time](#154-receipt-validity-over-time)
    - 15.5 [Signature verification versus trust](#155-signature-verification-versus-trust)

Appendix A: [Conformance Test Vectors](#appendix-a-conformance-test-vectors)
Appendix B: [Change Log](#appendix-b-change-log)
Appendix C: [Glossary](#appendix-c-glossary)
Appendix D: [Author Contacts](#appendix-d-author-contacts)

---

## 1. Terminology and Conformance

### 1.1 Document conventions

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14) ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals.

Non-normative examples, informative notes, and rationale are set apart in prose and clearly labeled; they do not carry normative weight.

Code samples, JSON structures, and command-line examples appearing in this document are illustrative. Where they differ from the normative prose, the prose is authoritative.

**Specification is source of truth.** This specification is the normative definition of AEAP. Reference implementations — including those authored by Aiyoplane, Inc. — are conforming artifacts, not normative sources. Where an implementation's behavior differs from the specification, the implementation is non-conformant, not the specification. Interoperability between implementations is established by the Conformance Test Vectors in Appendix A; a candidate implementation is AEAP-conformant when it passes the test vectors applicable to its declared Conformance Target (§1.3) and Conformance Level (§1.4), not when it reproduces the behavior of any specific reference implementation.

**Authorship distinct from authority.** Aiyoplane, Inc. authored and maintains this specification. The authority of AEAP as a protocol is established by adoption, interoperability demonstrated by Appendix A conformance, and the collective practice of the implementing ecosystem — not by the identity of its author. Independent implementations by third parties are welcomed under the same conformance criteria that apply to the reference implementations, and the protocol's technical integrity does not depend on any specific organization — including Aiyoplane, Inc. — continuing to operate a commercial RDP.

### 1.2 Defined terms

The following terms are used throughout this specification. Each term is introduced here and may be referenced in subsequent sections without re-definition. All defined terms appear in **bold** on first use within each section.

**Consequential Action.** An action proposed by an autonomous or semi-autonomous system whose execution may create material effects, commitments, state changes, or transfers that are not guaranteed to be fully and immediately reversible by the Actor. Examples include but are not limited to: a payment release, a code deployment, a procurement commitment, a data write to a system of record, a delete or publish operation whose effect persists in third-party systems, an issued credential, an inter-agent delegation of authority, or the invocation of a tool whose effect persists after the invocation returns. The reversibility test is not literal — many actions are technically reversible by some party — but centers on whether the Actor can unilaterally and immediately undo the material consequences without external coordination.

**Protected Party.** The entity on whose behalf authorization is being decided. The Protected Party is the author of the Policy under which Consequential Actions are evaluated. In the common commercial case, the Protected Party is a merchant, enterprise, or individual whose exposure the Policy governs; in a public-infrastructure case, the Protected Party may be a protocol community or standards body. The Protected Party is distinct from the Actor that proposes the action and from the RDP Implementation that evaluates it.

**Actor.** The entity that proposes the Consequential Action. The Actor MAY be a human, an autonomous agent, a workflow, or a machine process. AEAP imposes no restrictions on the Actor's identity, nature, or representation. Where identity attestation is required by Policy, it is modeled as an **Evidence Signal** (see §5).

**Intent.** A structured representation of a proposed Consequential Action, submitted by the Actor to the Runtime Decision Point for evaluation. An Intent **MUST** include the action's type, target, actor attribution (where applicable), and any parameters the Policy requires in order to evaluate it. An Intent is not itself a request to execute; it is a request for an authorization decision regarding a potential execution. See §3.

**Policy.** A declarative set of rules authored by the Protected Party that specifies the conditions under which a Consequential Action is to be authorized, escalated, or blocked. The Policy is the sole **Protected-Party-authored** normative input to the authorization decision; the Runtime Decision Point does not inject rules of its own beyond the normative requirements of this specification itself. See §4.

**Evidence Signal.** A piece of information evaluated by the Runtime Decision Point in the course of applying the Policy to the Intent. Evidence Signals form an open set that includes but is not limited to: identity attestations, authority chain assertions, budget state, approval records, settlement verification on a payment rail, out-of-band webhook responses from merchant systems, and policy-version metadata. See §5.

**Runtime Decision Point (RDP).** The architectural primitive specified by this protocol. The RDP is the component that receives an Intent, applies the Protected Party's Policy against available Evidence Signals, and produces a Verdict and (on ALLOW or ESCALATE) an Execution Receipt. The RDP is defined in §2.

**Verdict.** The ternary output of RDP evaluation. The three permitted values are **ALLOW**, **ESCALATE**, and **BLOCK**. See §6.

**Execution Receipt.** A portable, cryptographically-signed attestation of the authorization decision made by the RDP for a specific Intent. The Execution Receipt binds the Verdict to the specific Intent, Policy version, and Evidence Signals evaluated. The Execution Receipt **MUST** be independently verifiable offline against a published key set by any party who receives it, without coordination with the issuing RDP. See §7.

**Composition Boundary.** The specific location in a system where the RDP is invoked. A Composition Boundary MUST satisfy the invariants specified in §9; implementations are otherwise free to compose the RDP at any boundary that satisfies those invariants.

**Execution Surface.** The downstream system that performs the Consequential Action if and only if the Verdict permits. The Execution Surface MAY be a payment rail, a deployment system, a messaging bus, a storage layer, an invocation of a downstream tool, or any other system in which the Consequential Action takes effect. AEAP does not specify or constrain the Execution Surface.

**Issuer.** The RDP implementation that signs an Execution Receipt. The Issuer is identified by a key identifier (`kid`) that resolves against a published JSON Web Key Set (JWKS). See §8.

**Verifier.** Any party that receives an Execution Receipt and validates its signature, claims, and expiration. Verifiers MUST NOT require coordination with the Issuer to perform verification; the published JWKS is sufficient. See §7 and §8.

### 1.3 Conformance targets

This specification defines conformance requirements for three distinct roles. An implementation MAY conform to one or more roles independently.

**RDP Implementation.** A software component that accepts Intents, evaluates them against a Policy, and emits Verdicts and Execution Receipts. An RDP Implementation **MUST** satisfy all normative requirements in §2, §3, §4, §5, §6, §7, §8, and §9 that apply to its chosen Conformance Level (see §1.4).

**Receipt Verifier.** A software component or process that consumes Execution Receipts issued by any AEAP-conformant RDP Implementation and verifies them. A Receipt Verifier **MUST** satisfy all normative requirements in §7 and §8 applicable to receipt consumption. A Receipt Verifier is not required to implement §2–§6; it operates on receipts that other systems produce.

**Policy Author.** The human or organizational role responsible for authoring the Policy under which an RDP evaluates Intents. The Policy Author is not a software component but is referenced in normative requirements regarding Policy construction (§4) and Policy versioning (§14). AEAP does not specify the authoring interface used by Policy Authors; it only specifies the structural requirements the resulting Policy must satisfy.

### 1.4 Conformance levels

AEAP defines three cumulative Conformance Levels for RDP Implementations. Each level is strictly cumulative: a Core-conformant implementation satisfies a subset of the requirements an Extended-conformant implementation satisfies, and so on. All three levels concern the **base protocol only** and are independent of the optional AEAP Profiles defined in §1.5.

**Core.** A Core-conformant RDP Implementation **MUST**:

- Accept Intents conforming to §3
- Apply a Policy conforming to §4
- Evaluate at minimum the Evidence Signals categorized as *required for Core* in §5
- Return Verdicts conforming to §6 (minimum ALLOW and BLOCK; ESCALATE is OPTIONAL at Core level)
- On ALLOW, issue an Execution Receipt conforming to §7, signed using the AEAP Receipt Signature Profile specified in §8
- Fail closed on any ambiguity, error, or missing required Evidence Signal (see §9)

A Core-conformant implementation is sufficient for use cases where the authorization decision is binary and no human-escalation path is required.

**Extended.** An Extended-conformant RDP Implementation **MUST** satisfy all Core requirements and additionally **MUST**:

- Return the full ternary verdict ALLOW / ESCALATE / BLOCK (ESCALATE is REQUIRED at Extended level)
- Implement the Shadow Mode and Enforce Mode deployment postures specified in §10
- Emit Execution Receipts that include optional claims where the Policy references them, including context identifier and signing-key identifier
- Support receipt verification against the current published key set with graceful rotation handling as specified in §8

Extended conformance is the recommended target for production RDP Implementations serving commercial Protected Parties.

**Full.** A Full-conformant RDP Implementation **MUST** satisfy all Core and Extended requirements and additionally **MUST**:

- Implement the complete receipt lineage model specified in §7, including the extensibility hooks for non-ALLOW receipt classes
- Publish a conformance report naming which test vectors from Appendix A the implementation passes
- Observe the versioning and backward-compatibility discipline specified in §14, including public commitment to the deprecation window for any supported Receipt Signature Profile

Full conformance is the target for Reference RDP Implementations and for implementations used in standards-track work. **Note:** Full conformance concerns only the AEAP base protocol. Support for external authorization systems (HAP, XAA / ID-JAG, AOA) is modeled as separate optional Profiles — see §1.5 — not as part of base-protocol conformance.

*Informative note:* Two independent reference implementations of the RDP composition pattern demonstrate Core conformance: [`@aiyoplane/mcp-authz`](https://github.com/aiyoplane/mcp-authz) (Node) and [`aiyoplane-mcp-authz`](https://github.com/aiyoplane/mcp-authz-python) (Python). The two implementations share no vendored code and are maintained as separate codebases precisely so that passing the Appendix A test vectors from both demonstrates AEAP's interoperable implementability. Aiyo's hosted RDP at `api.aiyoplane.com` implements Extended conformance for its production path and progresses toward Full conformance as the receipt lineage model in §7 matures.

*As stated in §1.1, independent implementations by third parties are welcomed under the same conformance criteria and are not required to coordinate with Aiyoplane, Inc. except insofar as the Appendix A test vectors evolve across specification versions. This specification restates that principle intentionally: AEAP is open to third-party implementations at every Conformance Level, and the specification's authority rests on interoperability demonstrated through Appendix A, not on the identity or participation of any specific implementer.*

### 1.5 Optional AEAP Profiles

AEAP defines a mechanism for OPTIONAL additional conformance declarations an implementation **MAY** make, each covering a specific interop surface with an external authorization system or protocol. Profiles are additive to base-protocol conformance; an implementation's AEAP Conformance Level (§1.4) is independent of the set of Profiles the implementation supports.

Profiles defined by this specification include:

- **AEAP-HAP Profile.** Translation of Human Agency Protocol (HAP) mandate tickets into AEAP Intents and identity Evidence Signals. See §11.
- **AEAP-XAA Profile.** Translation of Okta Cross App Access / ID-JAG grants into AEAP identity and authority Evidence Signals. See §12.
- **AEAP-AOA Profile.** Translation of Alibaba / Cisco Agent Operation Authorization tokens into AEAP authority Evidence Signals. See §13.

Additional Profiles for other external authorization systems MAY be defined in subsequent versions of this specification or in companion specifications published by this document's maintainers or by third parties.

An implementation declares the Profiles it supports by publishing a Profile Declaration alongside its Conformance Level. Example (non-normative):

```
AEAP Conformance:
  Base Protocol: Extended
  Profiles Supported: AEAP-HAP, AEAP-XAA
  Profiles Not Supported: AEAP-AOA
```

An implementation that is Core-, Extended-, or Full-conformant on the base protocol is fully conformant regardless of which Profiles it supports or omits. The decoupling exists specifically so that AEAP's base-protocol definition of completeness is not hostage to the stability or evolution of external authorization systems.

**AEAP Profile disclaimer.** Inclusion of an external authorization system as an AEAP Profile does not imply that AEAP depends upon, endorses, or is governed by that system's specification. Profiles are translation surfaces that let AEAP consume external attestations as Evidence Signals (§5); AEAP's base-protocol identity, versioning discipline, and governance remain independent of any specific Profile's upstream specification. The addition, modification, deprecation, or retirement of any individual Profile does not affect base-protocol conformance for implementations that do not declare that Profile.

---

## 2. The Runtime Decision Point

### 2.1 Purpose and architectural position

The **Runtime Decision Point** is the architectural primitive AEAP specifies. It sits at the point in a system's execution flow where a Consequential Action has been proposed by an Actor but has not yet been performed by the Execution Surface. At that specific moment — after the Intent is fully formed and before the action takes effect — the RDP evaluates the Intent against the Protected Party's Policy and returns a Verdict that determines whether the Execution Surface MAY proceed.

The RDP is **not**:

- An identity provider or authentication system. Identity attestations are consumed by the RDP as Evidence Signals; they are not produced by it.
- A payment processor, wallet, or signer. Where a Consequential Action involves the movement of value, the rail, wallet, and signer are external systems whose outputs the RDP may consume as Evidence Signals but which the RDP does not implement.
- An orchestration or workflow runtime. The RDP evaluates individual Intents; it does not sequence them or manage their state across time.
- A policy engine in the general-purpose sense. The RDP applies a specific Policy to a specific Intent at a specific moment; it is not a general rules engine, and it does not store or manage Policy state across evaluations.
- A tool-invocation protocol. The RDP operates at the Composition Boundary immediately before a tool is invoked, not as the invocation mechanism itself.

The RDP's architectural position is deliberately narrow. It performs one role: *at the moment of execution, apply the Protected Party's Policy to the proposed Consequential Action and produce an attested Verdict.* Everything else is explicitly out of scope and is expected to be provided by other systems in the implementer's architecture.

### 2.2 Architectural invariants

Every RDP Implementation **MUST** satisfy the following invariants. These are the invariants that define what it means to be an RDP; a component that violates any of them is not an RDP as defined by this specification, even if it performs related functions.

**Invariant 1: One authorization decision per Consequential Action.** The RDP evaluates one Intent and produces one Verdict for one Consequential Action. The RDP **MUST NOT** aggregate multiple Consequential Actions into a single evaluation. Session-level authorizations, batch-level approvals, and workflow-wide grants are not AEAP evaluations; where such authorizations are needed, they belong in systems that compose with — but are separate from — the RDP.

*Rationale:* Consequential Actions are evaluated individually because the Policy may permit the first N actions in a sequence while blocking the (N+1)th, based on cumulative state (e.g., a budget) that only becomes observable after each action commits. One evaluation per action is the only pattern that preserves this granularity.

**Invariant 2: The decision occurs strictly between Intent and Execution.** The RDP evaluation **MUST** occur after the Intent is fully formed (meaning the concrete parameters of the proposed Consequential Action are observable) and **MUST** occur before the Consequential Action takes effect on the Execution Surface. Evaluations that occur earlier in the lifecycle (e.g., at session connect, at tool registration, at policy load time) are not AEAP evaluations. Evaluations that occur after the action has taken effect (e.g., post-hoc audits, continuous compliance scans) are monitoring, not authorization, and are explicitly out of scope for AEAP.

*Rationale:* The RDP's value is that it decides whether the Consequential Action should proceed. A decision that happens before the Intent is observable cannot be about the specific action; a decision that happens after the action has taken effect cannot prevent it.

**Invariant 3: Policy is authored by the Protected Party.** The RDP **MUST NOT** inject rules, defaults, or opinions beyond those the Protected Party has authored in their Policy. The RDP applies the Policy; it does not supplement it. If the Policy is silent on a condition that would be required to make a decision, the RDP **MUST** fail closed (see Invariant 5) rather than imposing a decision of its own.

*Rationale:* The Protected Party is the party exposed to the consequences of the Consequential Action. Authorship of the Policy belongs to the party bearing the exposure. An RDP that inserts its own rules dilutes the Protected Party's authorship authority and may introduce liability the Protected Party did not consent to.

**Invariant 4: Authorization decisions that may be relied upon by other systems MUST be independently verifiable.** On every Verdict for which the Policy or the Composition Boundary requires downstream reliance — minimally including every ALLOW; MAY also include ESCALATE and BLOCK where the Policy requires auditable non-ALLOW lineage — the RDP **MUST** emit an Execution Receipt conforming to §7. The Execution Receipt **MUST** be verifiable offline by any party in possession of the receipt and the published key material (see §8), without coordination with the Issuer at verification time.

*Rationale:* The RDP's authorization decision has value to downstream systems only if those systems can independently verify that the decision was made under the Protected Party's Policy. A decision that cannot be independently verified is operationally equivalent to the RDP's unilateral word, which is not a sufficient trust basis for cross-system authorization lineage. The receipt mechanism is architected as a general portable-attestation primitive, not strictly a "proof of successful authorization" artifact; §7 specifies the extensibility hooks that allow non-ALLOW receipt classes (denial receipts, escalation receipts, supersession records) to be issued where the Policy or downstream auditing requires them.

**Invariant 5: The RDP fails closed by default.** In every ambiguous, erroneous, or indeterminate condition — missing required Evidence Signal, unreachable Policy source, malformed Intent, internal processing failure, Policy evaluation that does not produce a definitive outcome — the RDP **MUST** fail closed. Failing closed means returning a BLOCK Verdict (or raising a typed error that the Composition Boundary catches as BLOCK). Fail-open behavior is NOT permitted at the RDP level. If a Protected Party wishes to permit fail-open behavior under specific conditions, that behavior **MUST** be implemented in the Composition Boundary wrapping the RDP, with an audit trail that captures the operator's explicit decision to do so.

*Rationale:* An RDP that fails open in response to its own errors would permit a class of unauthorized executions that the Protected Party has not opted into. Preserving the Protected Party's authorship of what is authorized (Invariant 3) requires that the RDP's own error states not expand the authorized surface. The RDP therefore **MUST** fail closed; any opt-out that a Protected Party wishes to permit **MUST** be explicit, implemented at the Composition Boundary, and attributable to a specific operator decision with an audit trail.

### 2.3 The evaluation lifecycle

For each Consequential Action proposed by an Actor, an AEAP-conformant RDP Implementation follows the following lifecycle:

1. **Intent submission.** The Actor (or a component acting on the Actor's behalf) constructs an Intent conforming to §3 and submits it to the RDP. The Intent carries all parameters the Policy will require in order to evaluate it.
2. **Policy resolution.** The RDP resolves the Policy applicable to this Intent. For most implementations, this means retrieving the Protected Party's current Policy from a Policy store, and recording the Policy's version identifier for inclusion in the eventual Execution Receipt.
3. **Evidence Signal collection.** The RDP collects the Evidence Signals the Policy references. Evidence Signals MAY be included in the Intent, MAY be fetched from upstream systems at evaluation time, or MAY be observed from the Composition Boundary's execution context. The RDP MUST NOT proceed to Verdict without all Evidence Signals the Policy requires; absence of a required Evidence Signal triggers Invariant 5 (fail closed).
4. **Policy evaluation.** The RDP applies the Policy to the Intent and the collected Evidence Signals, producing a candidate Verdict. The evaluation is deterministic: given the same Intent, Policy version, and Evidence Signals, the RDP **MUST** produce the same Verdict.
5. **Verdict issuance.** The RDP returns the Verdict (ALLOW, ESCALATE, or BLOCK) to the Composition Boundary. On ALLOW or ESCALATE, the RDP additionally issues an Execution Receipt (§7).
6. **Composition Boundary enforcement.** The Composition Boundary consumes the Verdict and (where applicable) the Execution Receipt. On ALLOW, the Composition Boundary permits the Execution Surface to perform the Consequential Action and attaches the receipt to any downstream artifact where lineage is needed. On ESCALATE, the Composition Boundary routes the Intent to the human-in-the-loop surface the Protected Party has designated. On BLOCK, the Composition Boundary prevents the Execution Surface from performing the action and surfaces the typed denial to the Actor.
7. **Lineage closure.** After the Consequential Action has either executed (on ALLOW) or been held / rejected (on ESCALATE / BLOCK), the Composition Boundary MAY notify the RDP of the outcome. This closure is informative, not normative; it enables Policy learning and audit reporting but is not required for AEAP conformance.

The lifecycle described above is sequential within a single Intent. Multiple Intents MAY be processed concurrently by the same RDP Implementation; the invariants in §2.2 apply per-Intent and do not impose a global serialization requirement.

### 2.4 Composition boundaries

The RDP is a logical component. It is not a particular wire protocol, a particular network endpoint, or a particular code library. An RDP Implementation MAY be invoked by any Composition Boundary that satisfies the invariants in §2.2 and §9.

The reference AEAP composition points include but are not limited to:

- **Tool-invocation boundary** — The RDP is invoked at the moment a tool-invocation protocol (e.g., Model Context Protocol, OpenAI Function Calling, LangChain Tools) is about to dispatch a tool call. The reference implementation at [`@aiyoplane/mcp-authz`](https://github.com/aiyoplane/mcp-authz) demonstrates this pattern at the MCP boundary.
- **Workflow-step boundary** — The RDP is invoked between two steps of an orchestration or workflow (e.g., Temporal activity invocation, LangGraph node transition, Microsoft Agent Framework step dispatch).
- **Payment-rail boundary** — The RDP is invoked immediately before a payment is submitted to a rail (Open Payments outgoing-payment creation, x402 commit, Machine Payments Protocol dispatch, Stripe PaymentIntent confirmation).
- **Fulfillment boundary** — The RDP is invoked between a payment-settled event and the fulfillment action it should trigger (e.g., Shopify `orders/paid` webhook before the merchant's fulfillment system acts on it).
- **Deploy / release boundary** — The RDP is invoked at the moment a code deploy, infrastructure change, or configuration push is about to take effect (CI/CD release gates, IaC apply dispatch).
- **Data-write boundary** — The RDP is invoked immediately before a write to a system of record (CRM record update, ERP transaction post, data pipeline sink commit).

These examples are informative; the normative requirements for a Composition Boundary are in §9. Any Composition Boundary that satisfies §9 is a valid AEAP integration point.

**Deployment-model agnosticism.** The Composition Boundary examples above and the RDP's role at those boundaries are agnostic to the deployment model of the RDP itself. An AEAP-conformant RDP MAY be operated in-process by the Protected Party (self-hosted RDP), as a managed service by Aiyoplane, Inc. or a third party (hosted RDP), or in a hybrid configuration where some Composition Boundaries invoke a local RDP and others invoke a remote one. The conformance requirements in §2.2 and the Composition Boundary invariants in §9 apply identically across deployment models. As stated in §1.1, AEAP's authority rests on interoperability, not on the identity of the operator; a self-hosted enterprise RDP, a commercial managed RDP, and a hybrid composition of both are all equally conformant implementations when their behavior satisfies the specification.

### 2.5 Non-goals

This specification does not define:

- **The authoring interface for Policies.** How a Protected Party constructs, versions, publishes, or distributes a Policy is implementation-defined. AEAP specifies the structural requirements the Policy must satisfy (§4) and the versioning discipline it must observe (§14), but not the user interface or storage layer by which Policies are managed.
- **The transport protocol between Composition Boundary and RDP.** An RDP Implementation MAY be in-process, invoked over HTTP, invoked over gRPC, invoked over a message bus, or embedded in any other transport the implementer chooses. AEAP specifies the semantics of the RDP's inputs (Intent) and outputs (Verdict, Execution Receipt); it does not specify the wire format of the invocation.
- **The identity of the Actor.** AEAP accepts Actor identity as an Evidence Signal input (§5) but does not specify how that identity is attested. Identity providers, authentication protocols, and attestation formats (OIDC claims, verifiable credentials, SPIFFE identities, HAP mandates, ID-JAG grants, AOA tokens, etc.) are external to AEAP. See the AEAP Profiles in §1.5 for optional translation surfaces.
- **The semantics of the Consequential Action itself.** AEAP does not inspect, validate, or constrain what the Consequential Action does once it is permitted to execute. The Execution Surface is responsible for the action's own semantics; AEAP's role ends with the Verdict.
- **Monetary or regulatory considerations.** AEAP does not define currencies, tax treatment, regulatory compliance regimes, or financial reporting formats. Where the Protected Party's Policy references such considerations (e.g., "block transactions over a jurisdictional reporting threshold"), the Policy encodes the Protected Party's interpretation; AEAP applies the Policy.
- **Human-in-the-loop interface design.** AEAP's ESCALATE Verdict is a signal that a decision requires human review; the specific interface, routing, notification, and approval ceremony are implementation-defined.
- **Guaranteed execution of an authorized action.** AEAP authorizes execution; it does not guarantee it. A Verdict of ALLOW indicates that the RDP has evaluated the Policy and attested that the Consequential Action is permitted to proceed; it does not attest that the action subsequently committed, executed, succeeded, settled, or produced any particular outcome. These post-authorization states are architecturally distinct from authorization and are explicitly out of scope for AEAP — see §2.6 for an informative mapping of adjacent layers.

### 2.6 Adjacent architectural layers (informative)

AEAP is deliberately narrow. It defines **authorization** at the boundary immediately before a Consequential Action takes effect. Several adjacent layers participate in the full lifecycle of a Consequential Action; AEAP interacts with them but does not define them. Implementers composing AEAP into a larger system should preserve the distinction between these layers in their own architectures.

| Layer | Question answered | AEAP's relationship |
|---|---|---|
| **Authorization** | *May this specific Consequential Action proceed, under the Protected Party's Policy, with the Evidence Signals available right now?* | **Defined by AEAP.** The RDP's Verdict and the Execution Receipt attest to the authorization decision. |
| **Commitment** | *Has the system irrevocably committed to carry out the action? Has a two-phase commit entered the prepared state? Has the deploy been accepted into the release pipeline?* | **Not defined by AEAP.** Commitment is a property of the orchestration or execution infrastructure; AEAP's receipt may be an input to the commitment decision but is distinct from it. |
| **Execution** | *Did the action actually run? Did the API call return? Did the transaction submit to the rail?* | **Not defined by AEAP.** Execution is a property of the Execution Surface. The ALLOW receipt attests to authorization, not to execution. |
| **Outcome** | *What was the result of executing the action? Did the deploy succeed? Did the API call return the expected data? Did the write complete?* | **Not defined by AEAP.** Outcome is a property of the Execution Surface's own result reporting. Outcome may be reported back as an input to future RDP evaluations (via Lineage closure, see §2.3 step 7) but is not itself an AEAP output. |
| **Settlement** | *Where the Consequential Action involved the movement of value, did the economic transfer actually settle on the underlying rail?* | **Not defined by AEAP.** Settlement is a property of the payment rail. Settlement MAY appear in AEAP as an Evidence Signal input to a subsequent authorization decision (settlement of transaction A as evidence for authorization of fulfillment action B), but the settlement itself is not an AEAP concern. |

A reader familiar with the architectural history of Aiyo (the implementing company that authored this specification) will recognize that these distinctions mirror concepts the implementer has developed in parallel — "Execution Commitment," "Execution Circuit Breaker," "settlement-verified authorization." Those concepts are implementation-level architectural work that composes with AEAP; they are not AEAP primitives. Preserving the layer separation keeps the AEAP primitive narrow and composable while permitting rich execution-governance architectures to be built around it.

**Key corollary:** An ALLOW Verdict is not a success claim. It is an authorization claim. A system that treats an ALLOW receipt as evidence that the action also executed, succeeded, or settled is making an inference that AEAP does not warrant. Where downstream reliance on execution-, outcome-, or settlement-state is required, that reliance **MUST** be grounded in separate attestations from the appropriate layer — not in the AEAP Execution Receipt alone.

---

## 3. Intent Representation

### 3.1 Intent as an atomic authorization subject

An **Intent** is the structured representation of exactly one proposed Consequential Action, submitted to the Runtime Decision Point for a single authorization decision. The atomicity requirement is foundational to the protocol and derives directly from Invariant 1 (§2.2): one authorization decision per Consequential Action.

The authorization subject hierarchy is explicit:

> **Intent** identifies exactly one **authorization subject**, which describes exactly one **Consequential Action**, which MAY have a structured **action target** containing multiple referenced entities (see §3.1.1).

The distinction between an *authorization subject* and a *Consequential Action* matters: one Consequential Action may act upon multiple referenced entities (a payment transfer touches both source and destination; a procurement commitment touches both buyer and vendor) without becoming multiple authorization subjects. An Intent is scoped to one authorization subject; the authorization subject's internal structure may legitimately reference several entities when the Policy declares them as a composite target.

An AEAP-conformant Intent **MUST** identify exactly one authorization subject. Specifically, an Intent **MUST**:

- Describe exactly one action type (the class of Consequential Action, e.g., `payment.release`, `deploy.production`, `data.write`, `tool.invoke`)
- Describe exactly one action target (the specific entity against which the action is to be performed, e.g., a payment recipient, a deployment environment, a database record)
- Carry exactly one set of action parameters, where each parameter further specifies the single described action (e.g., amount, deployment artifact SHA, record identifier)
- Request exactly one Verdict from the RDP

An AEAP-conformant Intent **MUST NOT**:

- Bundle multiple distinct action targets into a single Intent, except where the Policy explicitly defines a composite action target (see §3.1.1 below)
- Bundle multiple distinct action types into a single Intent under any circumstance
- Request authorization for a time window during which multiple separate Consequential Actions are expected to occur ("permit me to spend $10,000 over the next hour") — such a construction is a session-level authorization, which Invariant 1 explicitly excludes
- Represent itself as a batch, a transaction group, or a workflow spanning multiple Consequential Actions

An RDP Implementation **MUST** reject any Intent that violates these constraints with a typed error that the Composition Boundary catches as a BLOCK Verdict with reason code `INTENT_NON_ATOMIC` (see §6.5). The RDP **MUST NOT** attempt to split a bundled Intent into component Intents on the Actor's behalf; splitting is a Composition Boundary responsibility, not an RDP responsibility.

#### 3.1.1 Composite action targets

A Policy MAY define a specific composite action target that is treated as one authorization subject for RDP purposes even though it internally references multiple underlying entities. The canonical example is a payment transfer that references both a source account and a destination account: the action type is `payment.transfer` and the action target is the `(source, destination)` tuple, with the amount as a parameter. This is one authorization subject, not two.

Composite action targets **MUST** be declared in the Policy with explicit structure; an RDP **MUST NOT** infer composite-target semantics from an ambiguously-structured Intent. If the Policy does not declare the target structure, the Intent is treated as attempting to bundle multiple targets and is rejected under §3.1 above.

*Rationale:* The atomic authorization subject constraint prevents a predictable class of authorization evasion in which an Actor constructs an Intent broad enough to carry subsequent Consequential Actions "under the umbrella" of one approval. Composite action targets exist for the small set of legitimate cases where one authorization decision applies to a tuple of entities (payment transfer being the paradigm); they are explicitly declared in the Policy rather than inferred, so the Protected Party retains authority over what counts as atomic in their domain.

### 3.2 Required Intent fields

Every AEAP-conformant Intent **MUST** include the following fields. Field names are canonical; the wire format MAY use different serialization (JSON, Protobuf, etc.) but **MUST** carry semantically equivalent fields.

| Field | Type | Description |
|---|---|---|
| `intent_id` | string | A unique identifier for this Intent. **MUST** be globally unique within the scope of the Issuer's receipt-replay-detection window (see §3.5). Format is implementation-defined; UUIDv4 or UUIDv7 are **RECOMMENDED**. |
| `action_type` | string | The class of Consequential Action. The Policy defines which action types it recognizes; an action type the Policy does not recognize triggers fail-closed per Invariant 5. |
| `action_target` | string or object | The specific target of the action. For a simple target, a string identifier is sufficient. For a composite target (§3.1.1), a structured object whose shape is declared in the Policy. |
| `actor` | object | The entity proposing the action. **MUST** include at minimum an `actor_type` field with the value `human`, `agent`, or `machine`, and an `actor_id` string that identifies the specific Actor within the Protected Party's domain. Additional attribution fields are permitted; see §3.3. |
| `parameters` | object | Action-specific parameters the Policy will evaluate. Shape is defined by the Policy per `action_type`. **MUST** include every parameter the Policy requires in order to evaluate the Intent; absence of a required parameter triggers fail-closed per Invariant 5. |
| `submitted_at` | number (Unix seconds) | The timestamp at which the Intent was submitted to the RDP. **MUST** be within the Issuer's accepted-skew window (see §3.5). |

Policy selection is an RDP responsibility, not an Actor assertion. The authoritative Policy version used for evaluation is recorded by the RDP in the Verdict (§6.5) and the Execution Receipt (§7), not asserted by the Actor in the Intent. Actors MAY express an expected version through the OPTIONAL `requested_policy_version` field (§3.3); the RDP is not bound by it.

*Rationale:* An authoritative Policy version supplied by the Actor would permit a race in which the Actor retrieves Policy v42, constructs an Intent, and submits during a window where the Protected Party has published v43; the RDP would then have to decide whether to honor v42 or reject the Intent. Separating the Actor's expectation from the RDP's authoritative evaluation closes that race. The RDP **MUST NOT** evaluate against any Policy version that is not currently authorized for use by the Protected Party, regardless of what the Actor requests.

### 3.3 Optional Intent fields

The following fields are OPTIONAL and MAY be included by the Composition Boundary or the Actor as appropriate. Where included, the RDP **MUST** honor them as described.

| Field | Type | Description |
|---|---|---|
| `requested_policy_version` | string | A Policy version constraint declared by the Actor. When present, the Actor is declaring: *"I require evaluation under this specific Policy version."* The RDP **MUST** either (a) evaluate under the exact requested version, or (b) decline evaluation and return a typed denial with reason code `POLICY_VERSION_UNAVAILABLE` (per §6.5). The RDP **MUST NOT** silently substitute a different version. An Actor that does not require a specific version SHOULD omit the field entirely, in which case the RDP selects the Policy version it chooses to serve. The authoritative evaluated version is always recorded in the Verdict and Execution Receipt. |
| `supersedes_intent_id` | string | The `intent_id` of a previously-submitted Intent that this Intent is intended to supersede. Used to construct retry lineage after a prior Intent received BLOCK with `EVIDENCE_MISSING`, `EVIDENCE_STALE`, or `POLICY_VERSION_UNAVAILABLE`, where the Actor has refreshed the condition and is submitting a new attempt. The RDP MAY record this reference in the Execution Receipt lineage model (§7). This field does NOT permit re-evaluation of the referenced `intent_id` itself; it establishes a successor relationship between two distinct Intents (see §3.5). |
| `context_id` | string | A correlation identifier that groups this Intent with others belonging to the same broader workflow or user session. Does NOT create a composite authorization subject; each Intent sharing a `context_id` is still evaluated atomically. The `context_id` flows into the Execution Receipt (§7) as the `cid` claim. |
| `workflow_id` | string | For agent- or machine-initiated actions, the identifier of the workflow in which the Actor is operating. Combined with `actor_id`, provides execution context attribution — a workflow is not itself an Actor, but it is a meaningful second axis of attribution alongside the primary Actor identity. |
| `evidence` | object | Evidence Signals carried with the Intent at submission time. See §5.5 for the semantics of carried-vs-collected Evidence Signals. |
| `requested_verdict` | string | An optional Actor-suggested Verdict (one of `allow`, `escalate`, `block`). The RDP **MUST NOT** treat this field as authoritative; it is informational only, usable in audit logs to detect Actors whose expectations systematically diverge from the Policy. |
| `trace_id` | string | A distributed-trace identifier (e.g., W3C Trace Context) for operational observability. **MUST NOT** be used by the RDP in authorization logic. |

An implementation MAY define additional fields prefixed with `x-` for implementation-specific use. Fields not prefixed with `x-` and not defined in this specification or a successor specification **MUST NOT** be accepted by a Core-conformant RDP.

### 3.4 Canonical Intent representation

For the purposes of signing and verification (see §7), AEAP uses the **JSON Canonicalization Scheme (JCS)** as defined in [RFC 8785](https://www.rfc-editor.org/rfc/rfc8785). The canonical form is used wherever an Intent is cryptographically bound into a receipt or into any other attestation.

AEAP does not define its own canonicalization algorithm. The authoritative algorithm is RFC 8785. In the event of any inconsistency between the text of this specification and RFC 8785 on a canonicalization question, RFC 8785 is authoritative. This specification imposes only the following additional constraints on top of RFC 8785:

- The Intent object **MUST NOT** contain duplicate keys. RFC 8785 does not define behavior for duplicate keys; AEAP forbids them.
- The Intent object **MUST NOT** contain `null` values for defined fields. Omit the field instead. (Fields the specification defines as optional are omitted when not present; the receipt interprets absence as "field not supplied," not as "field present with null value.")
- All timestamps **MUST** be represented as JSON numbers in Unix seconds (integer or integer-serialized), not as ISO-8601 strings. This ensures RFC 8785's number serialization rules apply uniformly to all temporal fields.

Non-normative example of an RFC 8785-canonicalized Intent:

```json
{"action_target":"env/production","action_type":"deploy.production","actor":{"actor_id":"agent_claude_17","actor_type":"agent"},"intent_id":"int_7f8b9c2a4d1e","parameters":{"artifact_sha":"a3f21e...","release_notes":"Fixes #123"},"submitted_at":1727735200,"workflow_id":"deploy_wf_q4_release"}
```

Implementations MAY accept non-canonical Intents at submission time (whitespace tolerated, key order tolerated) but **MUST** canonicalize them per RFC 8785 before signing or verification. Receipts (§7) **MUST** reference the canonical form, not the as-submitted form.

#### 3.4.1 Intent digest

The **Intent digest** is a cryptographic commitment to the complete canonicalized Intent. It is the primitive by which the Execution Receipt (§7) cryptographically binds to the authorization subject the Verdict was produced for.

The Intent digest is computed as:

```
intent_digest = sha256(jcs(intent))
```

where `jcs(intent)` is the RFC 8785 canonicalization of the Intent object as defined in this section, and `sha256` is the SHA-256 hash function per [FIPS 180-4](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.180-4.pdf). The resulting digest is a 256-bit (32-byte) value, encoded for inclusion in receipts and verification artifacts as a lowercase hexadecimal string prefixed with `sha256:` (e.g., `sha256:3a6ef8c7...`).

The digest algorithm is fixed at SHA-256 for AEAP v1 Receipt Signature Profiles. Future Receipt Signature Profiles MAY declare alternative digest algorithms; the digest algorithm in use is identified by the Receipt Signature Profile version carried in the receipt (§7.4.1), so Verifiers always know which algorithm to apply.

The Intent digest is used by:

- The Execution Receipt as its REQUIRED cryptographic commitment to the Intent (§7.3)
- Downstream Verifiers reconstructing the authorization claim from a presented (Intent, receipt) pair: the Verifier recomputes `sha256(jcs(presented_intent))` and confirms it matches the receipt's `intent_digest` claim before relying on any other receipt content

*Informative note on library availability:* RFC 8785 implementations are widely available at the time of this specification's publication, including (non-exhaustively): the `canonicalize` npm package (Node), the `jcs` PyPI package (Python), `go-jcs` (Go), the `canonical-json` crate (Rust), and `java-json-canonicalization` (Java). AEAP implementers are RECOMMENDED to use a well-maintained RFC 8785 library rather than reimplementing the algorithm.

*Rationale:* AEAP's receipt verification will exist in many languages, maintained by independent implementers. Canonicalization is exactly the kind of security-critical primitive where independent reimplementation introduces subtle bugs that can diverge between implementations and silently invalidate cross-implementation receipt verification. Using an existing internet standard (RFC 8785) rather than inventing an AEAP-specific canonical JSON scheme substantially reduces the risk of cross-implementation canonicalization divergence while making AEAP read as RFC-native rather than Aiyo-proprietary. Pairing canonicalization with a declared digest algorithm (SHA-256) and using the resulting Intent digest as the receipt's cryptographic commitment to the Intent closes the final link in the authorization-attestation chain: a Verifier presented with any (Intent, receipt) pair can independently confirm, without consulting the Issuer at verification time, whether the receipt attests to that exact Intent or to a different one.

*Informative note on numeric representation:* RFC 8785 JCS specifies deterministic serialization of JSON numbers using I-JSON's IEEE 754 double-precision representation. This is sufficient for most protocol metadata (timestamps as Unix seconds, identifiers, counts) but is **not** sufficient for arbitrary-precision decimals or integers larger than 2^53. Policies dealing with monetary amounts, high-precision decimals, or large integers SHOULD declare an appropriate domain convention for representing those values in Intent action parameters — for example: integer minor units (`"amount_minor_units": 100000` for USD $1,000.00), decimal strings (`"amount": "1000.00"`), or structured amount objects. A future AEAP economic-action profile MAY specify a canonical monetary representation. Implementations that rely on bare JSON numbers for financial amounts without such a convention risk precision loss at canonicalization time, with Verdict and digest consequences.

### 3.5 Intent uniqueness, replay, and supersession

AEAP maintains the following invariant without exception:

> **One `intent_id` corresponds to exactly one RDP evaluation and at most one Execution Receipt.**

This invariant is absolute: no condition — including recoverable error conditions such as `EVIDENCE_MISSING`, `EVIDENCE_STALE`, or `POLICY_VERSION_UNAVAILABLE` — permits the RDP to re-evaluate a previously-seen `intent_id`. Retry is accomplished by submitting a new Intent with a new `intent_id` that references the prior Intent via the OPTIONAL `supersedes_intent_id` field (§3.3), not by re-submitting the original `intent_id`.

#### 3.5.1 Replay rejection

Every Intent **MUST** have an `intent_id` that is **unique within the Issuer's authorization namespace and replay window**. The requirement is not global-forever uniqueness; it is uniqueness within the specific identifier namespace in which the Issuer evaluates Intents, for at least as long as the Issuer retains replay-detection state. UUIDv7 is RECOMMENDED as it provides both strong uniqueness and lexicographic time-orderability.

**The durable cross-RDP evaluation identity is the tuple `(iss, intent_id)`.** The `intent_id` alone is NOT globally unique across RDPs; two Issuers may legitimately assign the same `intent_id` to independent Intents in their respective authorization namespaces. The combination of Issuer identifier (as recorded in the receipt's `iss` claim) and `intent_id` uniquely identifies an AEAP authorization evaluation across the full ecosystem. Systems that index, correlate, or deduplicate AEAP evaluations across multiple RDPs **MUST** use the `(iss, intent_id)` tuple as the identity key.

The RDP **MUST** reject any Intent whose `intent_id` has been previously evaluated within the Issuer's **replay window** — a configurable but implementation-declared duration during which the RDP maintains memory of previously-seen `intent_id` values.

The replay window **MUST** be at least as long as the Execution Receipt's maximum expiration window (see §7). Shorter replay windows would permit an Actor to re-submit an identical Intent after the first receipt expires but before the Composition Boundary has reasonably forgotten about it.

A Core-conformant RDP Implementation's replay-window default **MUST** be at least 24 hours. Extended and Full implementations SHOULD document their replay window and its interaction with the receipt expiration they emit.

Rejection due to a replayed `intent_id` **MUST** return reason code `INTENT_REPLAY` (see §6.5). The RDP **MUST NOT** silently re-evaluate the same Intent and emit a new receipt; replay is treated as evasion-pattern behavior and surfaced as a typed denial.

#### 3.5.2 Supersession as the retry mechanism

Where an Actor's prior Intent received a BLOCK Verdict with a reason code indicating a recoverable condition — most commonly `EVIDENCE_MISSING`, `EVIDENCE_STALE`, or `POLICY_VERSION_UNAVAILABLE` — and the Actor has refreshed the condition and wishes to retry, the Actor **MUST** submit a new Intent with:

- A **new, previously-unused `intent_id`**
- The **`supersedes_intent_id`** optional field set to the prior Intent's `intent_id`
- Updated Intent content as appropriate (fresh Evidence Signals, updated `requested_policy_version`, etc.)

The RDP treats the new Intent as a wholly new authorization subject for evaluation purposes; it carries no inherited approval or denial from the superseded Intent. The RDP MAY, at Extended and Full conformance, record the supersession chain in the Execution Receipt's lineage model (§7) for auditability. The RDP MAY also apply Policy-declared rate limits or anti-shopping heuristics to supersession chains (e.g., "if an Intent has been superseded more than N times in M minutes, treat the N+1th supersession as a signal for ESCALATE") — but any such heuristic is a Policy decision, not an AEAP requirement.

*Rationale:* Receipt-based cross-system lineage requires that each Intent map to exactly one Execution Receipt. Permitting re-evaluation of the same `intent_id` under any error condition — however benign the retry — would weaken this invariant and permit an Actor to "shop" for a favorable Verdict by submitting under conditions that gradually improve. The supersession model preserves retry capability while making every retry a cryptographically distinct and auditable event. The resulting lineage ("Intent A → BLOCK (`EVIDENCE_STALE`) → superseded by Intent B → ALLOW") is both operationally useful and tamper-evident.

---

## 4. Policy Evaluation Model

### 4.1 Policy as Protected-Party-authored declarative rules

The **Policy** is the declarative artifact authored by the Protected Party that specifies the conditions under which the RDP produces each Verdict. AEAP is deliberately non-prescriptive about Policy *expression* — the Policy MAY be authored in any policy language, DSL, decision table format, or structured JSON schema the Protected Party and the implementation agree on. OPA/Rego, Cedar, Casbin, custom-DSL, and plain JSON-with-code are all permitted.

AEAP is prescriptive about Policy *structure*. Every Policy **MUST** satisfy the following structural requirements, regardless of expression:

- **Addressability per action type.** For each `action_type` the Policy covers, the Policy **MUST** provide a definite evaluation path that terminates in exactly one of ALLOW, ESCALATE, or BLOCK given a well-formed Intent and all required Evidence Signals.
- **Declared Evidence Signal dependencies.** For each `action_type`, the Policy **MUST** declare the set of Evidence Signals it requires in order to evaluate (see §5.3). The RDP uses this declaration to fail closed when a required signal is missing.
- **Explicit composite-target declaration.** Where the Policy accepts composite action targets (§3.1.1), the target structure **MUST** be declared explicitly in the Policy, not inferred from Intents.
- **Decidable in bounded time and bounded resources.** The RDP **MUST** be able to produce a Verdict within an implementation-declared evaluation budget. Policies that cannot be evaluated within budget trigger fail-closed per Invariant 5; implementations SHOULD surface such Policies to the Protected Party as misconfigurations.

The Policy MAY define arbitrary additional structure the implementation supports, including conditional logic, numeric thresholds, allowlists and denylists, rate limiting, approval chains, and references to external data sources (which become Evidence Signals upon evaluation).

### 4.2 Policy versioning, digest, and binding

AEAP binds Verdicts to Policies through two complementary identifiers: a human-readable **Policy version** and a cryptographic **Policy digest**. The two together give AEAP both the operational convenience of a mnemonic version string and the cryptographic certainty of a content hash.

#### 4.2.1 Policy version

Every Policy **MUST** carry a Policy version identifier. The Policy version is a **monotonically-advancing** value according to an implementation-defined ordering: semantic versioning (`2.3.0` → `2.3.1` → `2.4.0`), deployment sequence (`policy-000042` → `policy-000043`), or timestamp-based identifiers (`2026-10-01T10:00:00Z` → `2026-10-01T14:30:00Z`) are all acceptable. The implementation MUST define its ordering explicitly and MUST ensure that each successive Policy deployment carries a strictly greater version identifier under that ordering.

Content-addressed hashes are NOT acceptable as Policy version identifiers because they do not advance monotonically — a hash is a function of content, not of time or deployment sequence. Cryptographic content identity is captured separately by the Policy digest (§4.2.2); the two identifiers are complementary, not interchangeable.

The Policy version is a *human and system identifier*. It is suitable for display in audit logs, for use by Actors in constructing `requested_policy_version` hints (§3.3), and for cross-referencing Policy history. It is NOT by itself a cryptographic binding — two deployments with the same version string that differ in any evaluated byte would produce a different Policy digest, and the digest is the authoritative cryptographic identity.

#### 4.2.2 Policy digest

Every Policy **MUST** be computable into a **Policy digest**: a deterministic cryptographic hash of the Policy's normative content, computed by canonicalizing the Policy's authoring artifact and hashing the canonical form with a declared digest algorithm.

The specific canonicalization method depends on the Policy's expression language and is declared by the Policy itself. (A Policy expressed in Rego will canonicalize differently from one expressed in Cedar, which will canonicalize differently from one expressed as structured JSON.) AEAP does not mandate a single canonicalization algorithm across all Policy expression languages, but the Policy **MUST** declare its chosen algorithm explicitly, and the chosen algorithm **MUST** be deterministic (two implementations following the same algorithm on the same source bytes produce the same digest).

**Conformance level requirements for Policy digest:**

- At **Core** conformance, the Policy digest is OPTIONAL. If present, it SHOULD be recorded in the Execution Receipt.
- At **Extended** and **Full** conformance, the Policy digest is **REQUIRED** in both the RDP's internal evaluation record and in every Execution Receipt the RDP issues.

The RECOMMENDED digest algorithm is SHA-256.

#### 4.2.3 Receipt binding

The RDP **MUST** record the exact Policy version (and, where applicable per §4.2.2, the Policy digest) against which it evaluated each Intent, and **MUST** include that identifier (identifiers) in the Execution Receipt (see §7). This creates a durable binding between the Verdict and the specific Policy it was produced under.

An implementation **MUST NOT** retroactively re-evaluate past Intents against a newer Policy version in a way that invalidates previously-issued Execution Receipts. Previously-issued receipts remain valid attestations of the authorization decision that was made at the time it was made. If the Protected Party changes the Policy and wishes to re-evaluate past actions under the new Policy, that re-evaluation is a new authorization process producing new Intents and new receipts; it does not retroactively modify the old ones.

*Rationale:* The receipt-as-portable-attestation property (Invariant 4) requires that a receipt's validity be independent of future state changes. A receipt that could silently become invalid when the Policy changes would not meet the independent-verifiability bar, because the verifier would need to know the Policy's current state to interpret the historical receipt. The Policy digest further strengthens this binding: a Verifier that holds (a) the receipt, (b) the Policy source from which the historical decision was made, and (c) the declared canonicalization algorithm can independently reproduce the Policy digest and confirm that the receipt's Policy claim is cryptographically bound to the exact bytes of the Policy at the time of evaluation. The Policy version alone can be forged or re-used; the digest cannot.

*Informative note on Policy identity vs semantic equivalence:* The Policy digest attests to specific Policy content — the exact bytes of the Policy as canonicalized by the Policy's declared canonicalization method. It does **not** attest to semantic equivalence between differently-represented Policies. Two Policies that produce identical Verdicts on every possible Intent (semantically equivalent) but that are structurally or syntactically different will produce different digests. Conversely, a digest match proves content identity but does not by itself prove semantic correctness. Equality of digests is strictly a content-identity test; evaluation of semantic equivalence is outside AEAP's scope and belongs to Policy-language-specific tooling.

### 4.3 Policy evaluation determinism

The RDP's Policy evaluation **MUST** be deterministic. Given the same canonical Intent (per §3.4), the same Policy digest (per §4.2.2), the same canonical set of Evidence Signal values, and the same evaluation semantics, an AEAP-conformant RDP **MUST** produce the same Verdict on every invocation. The determinism guarantee is stated in terms of Policy digest rather than Policy version because the digest cryptographically identifies the exact Policy content; two deployments at the same version string that differ in any evaluated byte would produce a different digest and are therefore different Policies for determinism purposes.

At Core conformance where Policy digest is OPTIONAL (per §4.2.2), the determinism guarantee holds only when the Policy content is independently held constant across evaluations; implementations that cannot record the digest cannot cryptographically prove determinism after the fact and SHOULD rely on external Policy-content coherence mechanisms (version-control immutability, deployment pipeline guarantees).

Determinism requires that the Policy itself contain no non-deterministic constructs (random values, wall-clock reads that are not already part of the Evidence Signal set, machine-local state) in its evaluation path. Where the Policy requires a time-based decision (e.g., "block requests outside merchant business hours"), the relevant time **MUST** be sourced from a declared Evidence Signal — not from the RDP's local clock — so that evaluation can be reproduced on any machine with the same inputs.

Determinism is a testable property. Conformance test vectors (Appendix A) include deterministic-replay cases: implementations are tested by replaying previously-evaluated Intents against the Policy version they were originally evaluated under and verifying that the Verdict is identical to the original.

*Rationale:* Deterministic evaluation is what makes AEAP conformance verifiable by any party. An implementation whose evaluation is non-deterministic cannot be audited; its behavior is only its own, not the Policy's.

### 4.4 Policy silence and the fail-closed boundary

Invariant 5 requires that the RDP fail closed in all ambiguous, erroneous, or indeterminate conditions. Policy silence is one such condition and deserves explicit treatment.

If a well-formed Intent references an `action_type` the Policy does not cover, the RDP **MUST** fail closed with reason code `POLICY_SILENT_ON_ACTION_TYPE`. The RDP **MUST NOT** infer a default Verdict, apply a global allowlist or denylist in the absence of the Policy's explicit position, or route to a general-purpose approval surface. Silence is denial.

If the Policy covers the `action_type` but does not produce a definite Verdict for a specific combination of parameters and Evidence Signals (e.g., the Policy's conditional logic falls off the end of its decision tree), the RDP **MUST** fail closed with reason code `POLICY_INDETERMINATE`. The RDP **MUST NOT** interpolate a Verdict; the Policy's indecision is the Verdict.

Protected Parties who want to explicitly permit broad action types SHOULD author a catchall rule in their Policy that produces an explicit ALLOW (or ESCALATE) for the unenumerated case, rather than relying on the RDP's silence behavior. This keeps the authorization decision in the hands of the Protected Party per Invariant 3.

### 4.5 Policy composition

A Protected Party MAY compose a Policy from multiple sub-Policies (modules, imports, included files). Where composition is used, the top-level Policy **MUST** declare its composition structure. The RDP evaluates all declared sub-Policies and combines their individual Verdicts into one composite Verdict using the mandatory **AEAP composition algebra** defined below.

#### 4.5.1 The AEAP Default Composition Algebra

Every AEAP-conformant RDP evaluating a composed Policy **MUST** apply the following verdict combination rule — the **AEAP Default Composition Algebra** — in exactly this precedence:

> If **any** sub-Policy produces **BLOCK**, the composite Verdict is **BLOCK**.
> Else if **any** sub-Policy produces **ESCALATE**, the composite Verdict is **ESCALATE**.
> Else if **at least one** sub-Policy produces **ALLOW** and **no** sub-Policy produces BLOCK or ESCALATE, the composite Verdict is **ALLOW**.
> Else (no sub-Policy produces a definite Verdict), the composite Verdict is **BLOCK** with reason code `POLICY_INDETERMINATE` (per §4.4).

In pseudocode:

```
if any(v == BLOCK for v in sub_verdicts): return BLOCK
elif any(v == ESCALATE for v in sub_verdicts): return ESCALATE
elif any(v == ALLOW for v in sub_verdicts): return ALLOW
else: return BLOCK (POLICY_INDETERMINATE)
```

The composite reason code is inherited from the sub-Policy whose Verdict was selected as the composite outcome. Where multiple sub-Policies produce the selected Verdict (e.g., two sub-Policies both BLOCK), the composite reason code is inherited from the first such sub-Policy in the Policy's declared composition order. Implementations SHOULD surface the full set of contributing sub-Verdicts in audit output for diagnosability, but only one reason code is recorded as the Verdict's canonical reason.

#### 4.5.2 Why the Default Algebra is mandated at base-protocol level

The AEAP Default Composition Algebra defined in §4.5.1 is **mandatory at Core, Extended, and Full base-protocol conformance**. An implementation that applies a different combination rule (e.g., "first match wins," "any ALLOW wins," priority-weighted scoring) is NOT a conformant RDP Implementation under this specification unless it does so under an explicitly declared Profile per §4.5.3.

AEAP does not claim that BLOCK > ESCALATE > ALLOW is the mathematical truth of all policy composition in all authorization systems — other defensible algebras exist for other threat models. AEAP claims only that for interoperable AEAP base-protocol conformance, implementations converge on this single predictable default. The precedence BLOCK > ESCALATE > ALLOW is chosen for two reasons:

1. **Interoperability.** Two conformant RDPs evaluating the same composed Policy against the same Intent and Evidence Signals **MUST** produce the same composite Verdict. A permissive composition semantics (where each implementation chose its own algebra) would silently break this guarantee and make cross-implementation audit infeasible.

2. **Fail-closed consistency with Invariant 5.** The default precedence collapses a mixed set of sub-Verdicts toward the most restrictive outcome. A sub-Policy that would deny the Intent is not silenced by a sibling sub-Policy that would allow it; a sub-Policy that would escalate is not silenced by a sibling that would allow. This matches AEAP's fail-closed-by-default posture at the composition level.

#### 4.5.3 Alternative composition algebras via Profiles

Implementations MAY declare support for alternative composition algebras via AEAP Profiles (§1.5). A Profile that defines a non-default composition rule **MUST** specify the rule mathematically, **MUST** declare which sub-Policy structures the rule applies to, and **MUST NOT** weaken the fail-closed default of the base protocol in composed contexts where the Profile is not in effect.

Profile-defined composition rules are the sole supported mechanism for deviating from the AEAP Default Composition Algebra defined in §4.5.1. Implementations **MUST NOT** silently apply implementation-specific composition rules without a declared Profile.

#### 4.5.4 Structural requirements under composition

Composition does not change the fundamental requirements of §4.1–§4.4: the composed Policy as a whole **MUST** be addressable per action type, must declare its Evidence Signal dependencies in aggregate, must evaluate deterministically, and must have an explicit silence posture.

**Sub-Policy references within a composition set MUST be unique.** A composition structure that references the same sub-Policy more than once within the same composition scope is a Policy-structure error. If an RDP encounters such a structure, it **MUST** fail closed with reason code `POLICY_INDETERMINATE` (per §4.4). Implementations that need to evaluate the same sub-Policy under different contexts MUST distinguish those contexts through separately-identified sub-Policy references, not through duplicate references to a single sub-Policy identifier.

AEAP does not specify a composition language or a policy-registry format. Those are implementation-defined. The specification constrains only the resulting evaluation behavior and the verdict combination algebra, not the mechanism by which sub-Policies are assembled.

---

## 5. Evidence Signals

### 5.1 Evidence Signal as an open-set input

An **Evidence Signal** is a piece of information evaluated by the RDP in the course of applying the Policy to the Intent. The set of possible Evidence Signals is deliberately open — AEAP does not enumerate every possible signal; it defines signal *categories* (§5.2), requires certain signals at Core conformance (§5.3), and constrains how any Evidence Signal is sourced, structured, and interpreted (§5.4, §5.5).

Any input the Policy declares as required for evaluation is an Evidence Signal. This includes:

- Information carried in the Intent itself (e.g., the `actor_id` field)
- Information fetched by the RDP at evaluation time (e.g., current budget state from an external ledger)
- Information observed from the Composition Boundary's execution context (e.g., the inbound HTTP request's TLS metadata)
- Information produced by external attestation systems and surfaced to the RDP (e.g., an identity attestation from an identity provider, a settlement proof from a payment rail)

The AEAP protocol does not require any specific external system to produce a specific Evidence Signal. It requires the RDP to evaluate the Evidence Signals the Policy declares, obtain them from the sources the Policy declares, and fail closed when they are not obtainable.

#### 5.1.1 The three structural components of an Evidence Signal

Every Evidence Signal evaluated by the RDP decomposes into three structural components. The decomposition is not merely conceptual: implementations **MUST** be able to distinguish, record, and expose each component separately, because the components carry different trust properties and may be sourced from different systems.

**Value.** The underlying datum the Policy evaluates. Examples: the string `"agent_claude_17"`, the integer `9500`, the boolean `true`, the timestamp `1727735200`. The value alone has no inherent trust property; its trustworthiness is established by its provenance and attestation.

**Provenance.** The claim about where the value came from. Examples: `"source": "merchant_ledger_api.example.com/v1/budget"`, `"source": "intent.evidence.actor_credential"`, `"source": "composition_boundary.tls_peer_certificate"`. Provenance is itself a value — a label the RDP records about the value's origin — but it is not self-attesting. A provenance claim of `"source": "Okta"` does not by itself mean the value came from Okta; it means something upstream of the RDP has asserted that it did.

**Attestation.** The cryptographic or protocol-level evidence that supports the provenance claim. Examples: a JWS signature over the value by a verifiable key, a TLS-authenticated transport session from the declared source endpoint, a verifiable credential presentation with a resolvable issuer DID. Attestation is what elevates a provenance claim from assertion to verifiable attestation. An Evidence Signal MAY be presented with no attestation (in which case the Policy decides whether to accept it based on provenance and operational context alone), with a strong attestation (whose verification is a precondition for using the value), or with a chained attestation (where multiple attestations compose to vouch for the same value).

#### 5.1.2 Why the decomposition is normative

The three-component structure is normative because it closes a specific class of trust confusion. Consider the following malformed example:

```json
{
  "evidence": {
    "actor_id": "anthropic_agent_123",
    "source": "Anthropic"
  }
}
```

This payload asserts a value (`anthropic_agent_123`) and claims a source (`Anthropic`). But it carries no attestation that Anthropic in fact issued that identity to that actor. Without the decomposition, an implementation might treat the `source` label as evidence of origin and accept the value as "attested by Anthropic." It is not. It is an unattested value with a self-asserted provenance claim.

The decomposition requires implementations to distinguish:

| What the Intent carries | What the Policy sees |
|---|---|
| Value only | A value from an unknown source; use at the Policy's discretion |
| Value + provenance claim | An assertion of origin; trustworthy only to the extent the operational context supports the claim |
| Value + provenance claim + verifiable attestation | A cryptographically substantiated attestation; the RDP MUST verify the attestation before accepting the value |

Implementations **MUST NOT** collapse these three cases into a single notion of "the Policy received evidence that said X." The receipt (§7) records which of the three structural components were present for each Evidence Signal evaluated, so that any downstream Verifier can reconstruct the trust context under which the Verdict was produced.

### 5.2 Signal categories

AEAP groups Evidence Signals into five informative categories. The categorization is non-normative but is used throughout the specification and conformance tests. A Policy MAY reference signals from any combination of categories and MAY define signals that span categories.

**Identity signals.** Information attesting to the identity of the Actor. Examples include OIDC id_token claims, verifiable credential presentations, SPIFFE identity documents, HAP mandate tickets (surfaced via the AEAP-HAP Profile, §11), ID-JAG grants (via AEAP-XAA Profile, §12), and signed assertions from any attestation framework the Protected Party trusts.

**Authority signals.** Information attesting to the authority chain under which the Actor is operating. Examples include delegation chains, role assignments, scope grants, and signed authority tokens (including AOA tokens via AEAP-AOA Profile, §13).

**State signals.** Information about current state the Policy evaluates against. Examples include budget remaining, cumulative spend within a window, approval records from upstream human-in-the-loop systems, policy-version identifiers, feature flags, and change-freeze indicators.

**Settlement signals.** Information verifying that an economic event on a payment rail has actually occurred, where applicable. Examples include signed webhooks from payment processors (Stripe PaymentIntent succeeded), on-chain settlement proofs (USDC settlement on Base, pathUSD settlement on Tempo via Machine Payments Protocol), and Open Payments incoming-payment resource state. See §2.6 for the architectural distinction between settlement (consumed by AEAP as an Evidence Signal) and authorization (produced by AEAP as a Verdict).

**External signals.** Information from arbitrary upstream systems the Policy declares as decision inputs. Examples include webhook responses from merchant-operated systems of record, fraud-score outputs, device-posture attestations, geolocation lookups, and any other input the Protected Party has declared relevant.

The categorization is intended to help Policy Authors and Composition Boundary implementers reason about which signals they need to collect and how. It is not a constraint on which signals a Policy may reference.

*Informative note on protocol metadata vs external evidence:* Some inputs the RDP evaluates are not external evidence in the strict sense but intrinsic protocol metadata derived directly from AEAP structures: the Intent's `submitted_at`, the evaluated Policy version and digest, the Intent's `action_type`. These are treated as Evidence Signals for the purposes of §5.3 required-signal evaluation because the Policy references them, but they do not require provenance or attestation decomposition (§5.1.1) in the same way that externally-sourced evidence does — their "source" is the Intent itself under AEAP structural guarantees. The three-component decomposition still applies conceptually, but the provenance claim is trivially "the Intent" and the attestation is the RDP's syntactic validation of the Intent per §3.

### 5.3 Required-for-Core signals and protocol metadata

A Core-conformant RDP Implementation **MUST** evaluate the following minimum set of protocol-metadata signals and MUST additionally evaluate every Policy-declared external Evidence Signal. The signals below are the minimum protocol-structural inputs that a Core implementation must be capable of evaluating; they are NOT a sufficient trust basis for authorization in domains where the Policy requires authenticated identity or externally-attested evidence.

| Signal | Category | Normative requirement |
|---|---|---|
| **Actor attribution** | Identity (protocol metadata) | Core RDPs **MUST** evaluate the `actor` object from the Intent (§3.2) as the Actor's self-asserted attribution. This is *not* authentication of the Actor's identity. See §5.3.1 below. |
| **Evaluated Policy version** | State (protocol metadata) | Core RDPs **MUST** determine the authoritative Policy version to evaluate against (§4.2) and record it. The Actor's OPTIONAL `requested_policy_version` is a constraint when present (the RDP honors it or returns `POLICY_VERSION_UNAVAILABLE`); when absent, the RDP selects the Policy version it is authorized to serve (§3.3). |
| **Submission time** | State (protocol metadata) | Core RDPs **MUST** evaluate `submitted_at` from the Intent against the Issuer's accepted-skew window (§3.5). |
| **Action-type coverage** | State (protocol metadata) | Core RDPs **MUST** evaluate whether the Policy covers the Intent's `action_type` (per §4.4); silence is denial. |

Core implementations are NOT required to natively evaluate authority signals, settlement signals, or external signals in the general case. However, if the Policy *does* reference such signals as required for a given evaluation, the Core RDP **MUST** either evaluate them (per §5.1.1's three-component decomposition) or fail closed. Core conformance does not grant an implementation permission to selectively ignore Policy-declared signal requirements.

Extended and Full implementations SHOULD support a broader range of signal types natively (webhook consumption, settlement-proof verification, delegation-chain traversal, verifiable credential presentation verification) to reduce the operational burden on Composition Boundary implementers.

#### 5.3.1 Actor attribution is not Actor authentication

The `actor` object in the Intent is **self-asserted attribution**, not **authenticated identity**. The Actor claims an identity; the Intent carries that claim; the RDP records it. Nothing in Core conformance requires that the claim be verified. A Core RDP evaluating only protocol metadata will record the asserted `actor_id` without having verified that the entity submitting the Intent is in fact that Actor.

Authenticated Actor identity — the cryptographically verified binding between a specific Actor and their asserted identity — is only required where the Policy declares it as an Evidence Signal with an attestation requirement under §5.1.1. If the Policy declares, for example, that the `actor.actor_id` must be bound to a presented OIDC id_token signed by a trusted issuer, then that attestation becomes a required Evidence Signal and the RDP (Core or above) **MUST** verify it before producing ALLOW.

Implementations and documentation **MUST** preserve this distinction. Describing Core conformance as "providing Actor identity" without the attribution-vs-authentication qualification is misleading and will lead downstream systems to over-trust the attribution field.

*Rationale:* The alternative — mandating Actor authentication at Core — would couple AEAP's base protocol to specific identity-attestation formats (OIDC, verifiable credentials, SPIFFE, etc.) and would prevent the use of AEAP in domains where identity is established out-of-band or where the Policy's trust model does not require per-Intent identity verification. By keeping Core at attribution only and promoting authentication to a Policy-declared Evidence Signal requirement, AEAP accommodates both high-security (identity-verified) and low-friction (attribution-only, downstream-trust) deployment models.

### 5.4 Signal provenance and freshness

Every Evidence Signal evaluated by the RDP **MUST** carry attributable provenance. The RDP **MUST** be able to answer, for each signal that contributed to a Verdict, two questions:

1. **Source.** Where did this signal value come from? (The Intent itself, a specific external system identified by endpoint, a Composition Boundary context field, etc.)
2. **Freshness.** When was this signal value obtained? If the signal was fetched from an external system, the fetch time. If the signal was observed at request time, the observation time.

Provenance and freshness information **MUST** be included in the data referenced by the Execution Receipt (§7). The receipt itself need not inline every signal's full provenance; a receipt-referenced "signals envelope" is permitted as long as the envelope is itself independently verifiable and durably stored.

Freshness semantics are a Policy concern because different signals have radically different tolerance profiles: a settlement proof may be meaningful only within milliseconds of settlement; an employee credential may remain valid for months; a quarterly compliance certification may be valid for a quarter; a sanctions-screening result may be acceptable within hours. AEAP therefore mandates that **the Policy MUST declare a freshness requirement for any Evidence Signal whose validity is time-sensitive**.

If an Evidence Signal's freshness, as declared by its provenance (§5.1.1), exceeds the Policy's declared staleness tolerance for that signal, the RDP **MUST** treat the signal as `EVIDENCE_STALE` and fail closed per Invariant 5.

For Evidence Signals the Policy does not declare as time-sensitive, the RDP applies the signal's own validity semantics as carried in its attestation (e.g., the `nbf` and `exp` claims of a JWT-presented attestation, the `validUntil` field of a verifiable credential, the on-chain finality depth of a settlement proof). AEAP does not impose a universal default freshness window; a universal default would be arbitrary across signal classes and would mask Policy under-specification rather than surface it.

Implementations SHOULD surface Policies that reference time-sensitive signals without declaring a staleness tolerance as misconfigurations, since silence on freshness is Policy ambiguity rather than Policy decision.

*Rationale:* An authorization decision made against stale Evidence Signals is operationally equivalent to a decision made against unavailable signals. A hard-coded universal default (such as "60 seconds absent explicit tolerance") would silently normalize Policies to that threshold, which is correct for essentially no realistic signal class. Explicit Policy-declared freshness, backed by the attestation's own validity semantics as fallback, forces the Protected Party to engage with freshness as a per-signal design decision — which is where freshness decisions belong.

### 5.5 Signal collection timing

AEAP permits three distinct timings for Evidence Signal collection:

**Carried.** The signal value is included in the Intent at submission time, typically in the `evidence` field (§3.3). The RDP **MUST** verify the signal's provenance (the Intent's claim about the signal's origin) before trusting the value; signals carried in the Intent are not self-attesting and MAY be cryptographically signed by the originating system to be acceptable.

**Collected.** The signal value is fetched by the RDP at evaluation time from an external system the Policy declares as the signal's source. The RDP **MUST** apply the Policy's declared staleness tolerance to the fetched value (§5.4).

**Observed.** The signal value is read from the Composition Boundary's execution context at evaluation time (e.g., the inbound request's authenticated client identity, the TLS peer certificate's subject). Observed signals **MUST** be surfaced to the RDP by the Composition Boundary; the RDP does not reach back into transport-layer context on its own.

A single Evidence Signal MAY be sourced via multiple timings in different implementations; the Policy declares the acceptable sources, and the RDP uses whichever is available. When multiple sources are available for the same signal, **the Policy MAY declare a source-preference or trust requirement** specifying which source the RDP should prefer, or specifying that a specific source is REQUIRED and others MAY be used only as corroboration.

AEAP does NOT impose a universal source-strength hierarchy across all signal classes. A fresh value fetched from a trusted authoritative system may be stronger evidence than a cryptographically-signed value supplied by an untrusted Actor; the reverse is also sometimes true. Which source is strongest is a function of the specific signal, the specific trust context, and the Policy's threat model — not a universal property of signal-collection timing.

Absent a Policy-declared preference, the RDP MAY use any acceptable source and **MUST** record in the Execution Receipt (§7) which source actually supplied the value for the Verdict, so that downstream Verifiers can reconstruct the trust context under which the Verdict was produced.

---

## 6. Ternary Verdict Semantics

### 6.1 The three Verdicts

The RDP produces exactly one **Verdict** per Intent. The three permitted Verdicts are **ALLOW**, **ESCALATE**, and **BLOCK**. No other values are permitted. Verdicts are atomic — the RDP MUST NOT produce composite verdicts, probabilistic verdicts, conditional verdicts, or verdicts that defer to the Composition Boundary.

The semantics of each Verdict are defined below. These semantics are normative; implementations **MUST NOT** redefine them.

### 6.2 ALLOW semantics

An **ALLOW** Verdict indicates that the RDP has evaluated the Policy against the Intent and the available Evidence Signals, and the Policy's evaluation produces a definite outcome that the Consequential Action is permitted to proceed.

ALLOW claims:
- The Policy, at the specific version recorded in the receipt, permits the described Consequential Action
- The Evidence Signals the Policy required were present and within tolerance
- The RDP has issued an Execution Receipt (§7) binding this Verdict to the Intent

ALLOW does NOT claim:
- That the Consequential Action subsequently committed, executed, succeeded, settled, or produced any particular outcome (see §2.6)
- That the Composition Boundary is required to proceed with execution — the Composition Boundary MAY still decide not to execute for reasons outside AEAP's scope (e.g., downstream system unavailability)
- That any future Consequential Action with similar parameters will also be ALLOWed — each Intent is evaluated independently per Invariant 1

**ALLOW establishes authorization, not an obligation to execute.** Upon receiving an ALLOW Verdict and a valid Execution Receipt, the Composition Boundary **MAY** permit the Execution Surface to perform the Consequential Action. The Composition Boundary is not required by AEAP to execute. A Composition Boundary MAY decline to execute an ALLOWed action for any reason outside AEAP's scope — downstream unavailability, higher-order operational guardrails, resource constraints, circuit-breaker policies, business-logic cancellation, timing, or any other non-AEAP operational condition. Such refusal is NOT an AEAP conformance violation. AEAP prohibits unauthorized execution; AEAP does not require authorized execution.

### 6.3 ESCALATE semantics

An **ESCALATE** Verdict indicates that the RDP has evaluated the Policy and determined that an authorization decision requires input from a party beyond the automated evaluation path. ESCALATE is a first-class Verdict, not a deferred or partial decision.

ESCALATE claims:
- The Policy, at the specific version recorded in the receipt, explicitly routes this Consequential Action to a human-in-the-loop (or other out-of-band) decision process
- The RDP has issued an Execution Receipt (§7) binding this Verdict to the Intent, with the escalation routing information included
- A stable `escalation_id` (§6.3.1) has been assigned and is carried in the Verdict and receipt
- The Composition Boundary MUST hold the Consequential Action pending resolution of the escalation; execution during an unresolved escalation is a conformance violation

ESCALATE does NOT claim:
- A direction of eventual decision (an ESCALATE is not a soft-ALLOW or soft-BLOCK)
- A specific escalation surface (the routing information is Policy-declared and MAY vary per Intent)
- A specific resolution deadline (unless the Policy declares one)
- That the escalation target is necessarily a human (an automated secondary authorization authority is a permitted escalation target)

#### 6.3.1 Escalation identifier

Every ESCALATE Verdict **MUST** include a stable **`escalation_id`** — a string that uniquely identifies the pending escalation and is suitable for use as a durable correlation key across the escalation workflow. The `escalation_id` is distinct from the `intent_id`: a single `intent_id` produces exactly one ESCALATE Verdict, which produces exactly one `escalation_id`.

The `escalation_id` is used by:

- The escalation resolution workflow (the human review surface, the secondary authorization authority, etc.) to record its decision against a specific escalation
- Subsequent Intents that are submitted as follow-ups to a resolved escalation, which carry the `escalation_id` as an Evidence Signal demonstrating the resolution
- Receipt lineage models (§7) that chain the escalation receipt to the eventual follow-up Intent's receipt

The format is implementation-defined; UUIDv4 or UUIDv7 are **RECOMMENDED**. The RDP **MUST** ensure that each `escalation_id` is globally unique within the Issuer's identifier namespace.

The escalation resolution process itself is outside AEAP's scope. The resolver's eventual decision MAY be fed back to the RDP as a new Intent (with the `escalation_id` carried as an Evidence Signal demonstrating that the human-in-the-loop step was completed), or MAY be fed to the Composition Boundary directly, depending on the Policy's declared escalation workflow.

#### 6.3.2 Core conformance behavior

ESCALATE is OPTIONAL at Core conformance (§1.4) and REQUIRED at Extended. A Core-only implementation that encounters a Policy condition that would ESCALATE at Extended **MUST** fail closed with BLOCK and reason code `ESCALATE_UNSUPPORTED` (see §6.5) rather than silently execute; it **MUST NOT** convert ESCALATE semantics into ALLOW, and it **MUST NOT** disguise a would-be-ESCALATE as `POLICY_DENY`. The `ESCALATE_UNSUPPORTED` reason code exists specifically to preserve the audit distinction between "the Policy denied this action" and "the Policy would have escalated this action but the implementation does not support ESCALATE."

### 6.4 BLOCK semantics

A **BLOCK** Verdict indicates that the Consequential Action is not permitted to proceed. The Composition Boundary **MUST NOT** execute the Consequential Action in response to a BLOCK Verdict.

BLOCK is produced in several distinct cases, each distinguishable by reason code (§6.5):

- The Policy explicitly denies the Intent under its evaluated conditions (`POLICY_DENY`)
- The Policy is silent on the Intent's action type or evaluates indeterminately (`POLICY_SILENT_ON_ACTION_TYPE`, `POLICY_INDETERMINATE`)
- A required Evidence Signal is missing, stale, or unverifiable (`EVIDENCE_MISSING`, `EVIDENCE_STALE`, `EVIDENCE_INVALID`)
- The Intent is malformed, non-atomic, or replayed (`INTENT_MALFORMED`, `INTENT_NON_ATOMIC`, `INTENT_REPLAY`)
- The Intent's `requested_policy_version` constraint cannot be served by this RDP (`POLICY_VERSION_UNAVAILABLE`)
- An internal RDP error occurred during evaluation (`RDP_INTERNAL_ERROR`)

In all BLOCK cases, the Composition Boundary **MUST** surface the typed reason code to the Actor rather than failing silently or returning a generic error. The Actor needs to know whether the denial is a Policy decision (which may be correctable by proposing a different Intent) or a system condition (which is not).

A BLOCK Verdict MAY be accompanied by an Execution Receipt in Extended and Full implementations where the Policy requires auditable denial lineage (see Invariant 4 as updated in §2.2). Core implementations are not required to issue receipts for BLOCK Verdicts.

### 6.5 Verdict output and reason codes

The RDP returns a Verdict object with the following required fields:

| Field | Type | Description |
|---|---|---|
| `verdict` | string | One of `allow`, `escalate`, `block`. |
| `intent_id` | string | The `intent_id` of the Intent this Verdict was produced for (per §3.2). |
| `evaluated_at` | number (Unix seconds) | The timestamp at which the Verdict was produced. |
| `evaluated_policy_version` | string | The exact Policy version against which the Intent was evaluated. REQUIRED. Where applicable per §4.2.2, the Verdict additionally carries `evaluated_policy_digest`. |
| `evaluated_policy_digest` | string | The cryptographic digest of the Policy against which the Intent was evaluated. REQUIRED at Extended and Full conformance; OPTIONAL at Core. See §4.2.2. |
| `reason_code` | string | A categorical reason code (see table below). REQUIRED for `escalate` and `block`; OPTIONAL but RECOMMENDED for `allow`. |
| `reason_detail` | string | A human-readable explanation. OPTIONAL. |
| `receipt` | string | The Execution Receipt, present where §7 requires it. |
| `escalation_id` | string | The stable escalation identifier (§6.3.1). REQUIRED when `verdict` is `escalate`. |
| `escalation` | object | Escalation routing information. REQUIRED when `verdict` is `escalate`. Shape is Policy-declared. |

**Canonical reason codes.** All AEAP-conformant implementations **MUST** use the following reason codes where applicable. Implementations MAY define additional implementation-specific reason codes prefixed with `x-`.

| Reason code | Applicable verdicts | Meaning |
|---|---|---|
| `POLICY_ALLOW` | `allow` | The Policy's conditions for authorization were met. |
| `POLICY_ESCALATE` | `escalate` | The Policy routes this Intent to out-of-band review. |
| `POLICY_DENY` | `block` | The Policy explicitly denies this Intent under its evaluated conditions. |
| `ESCALATE_UNSUPPORTED` | `block` | The Policy would have produced ESCALATE at Extended or Full conformance, but the current RDP implementation is Core-only and does not support ESCALATE. The audit record preserves the distinction from `POLICY_DENY`: this was a would-be-escalation forced to BLOCK by conformance level, not a Policy-driven denial. See §6.3.2. |
| `POLICY_SILENT_ON_ACTION_TYPE` | `block` | The Policy does not cover the Intent's `action_type` (per §4.4). |
| `POLICY_INDETERMINATE` | `block` | The Policy evaluates but does not produce a definite Verdict (per §4.4). |
| `POLICY_VERSION_UNAVAILABLE` | `block` | The Intent's `requested_policy_version` constraint cannot be served by this RDP — the requested Policy version is not currently authorized for evaluation. The Actor may submit a supersession Intent with a different `requested_policy_version`, with the field omitted (allowing the RDP to select), or route to a different RDP that serves the required version. |
| `EVIDENCE_MISSING` | `block` | A required Evidence Signal was not available (per §5.4). |
| `EVIDENCE_STALE` | `block` | A required Evidence Signal was present but exceeded the Policy's staleness tolerance (per §5.4). |
| `EVIDENCE_INVALID` | `block` | A required Evidence Signal was present but failed verification (bad signature, malformed, failed attestation per §5.1.1). |
| `INTENT_MALFORMED` | `block` | The Intent did not meet the structural requirements of §3.2. |
| `INTENT_NON_ATOMIC` | `block` | The Intent attempted to bundle multiple authorization subjects (per §3.1). |
| `INTENT_REPLAY` | `block` | An Intent with this `intent_id` was previously evaluated within the replay window (per §3.5). |
| `RDP_INTERNAL_ERROR` | `block` | An internal RDP processing failure occurred. |

**Retry guidance per reason code.** The Composition Boundary MUST surface the reason code to the Actor on BLOCK or ESCALATE Verdicts. The Actor's expected handling varies by reason code and is governed by the strict one-intent-id-one-evaluation invariant (§3.5): **retry is accomplished by submitting a new Intent via supersession (§3.5.2), never by re-submitting the same `intent_id`.**

- `POLICY_DENY`, `POLICY_SILENT_ON_ACTION_TYPE`, `INTENT_NON_ATOMIC`, `INTENT_MALFORMED` → the Actor's Intent construction is the issue. The Actor should NOT retry without first changing the Intent's content substantively. Mechanical retry (new `intent_id`, same content) will produce the same outcome.
- `POLICY_INDETERMINATE`, `RDP_INTERNAL_ERROR` → the Intent itself may be well-formed; the Protected Party's Policy or the RDP implementation is at fault, and the Actor's recourse is through the Protected Party, not through retry.
- `EVIDENCE_MISSING`, `EVIDENCE_STALE`, `EVIDENCE_INVALID` → the Composition Boundary MAY refresh the signal and the Actor MAY submit a supersession Intent (new `intent_id`, `supersedes_intent_id` set to the prior Intent's `intent_id`) with refreshed evidence. See §3.5.2.
- `POLICY_VERSION_UNAVAILABLE` → the Actor's version constraint cannot be served by this RDP. The Actor may submit a supersession Intent with a different `requested_policy_version`, with the field omitted (allowing the RDP to select from its currently-authorized versions), or route to a different RDP that serves the required version.
- `ESCALATE_UNSUPPORTED` → the Actor's recourse depends on the Composition Boundary's capabilities. Submitting the same action to an Extended or Full RDP would produce ESCALATE and progress through the human-in-the-loop surface; mechanically retrying against the same Core RDP will not.
- `INTENT_REPLAY` → the Actor **MUST NOT** retry with the same `intent_id` under any condition. Submit a new Intent with a new `intent_id` (and `supersedes_intent_id` if the retry is intended as a supersession of a prior attempt).

Non-canonical reason codes carrying implementation-specific semantics **MUST** be prefixed with `x-` to prevent namespace collision with future canonical codes defined by this specification.

---

---

## 7. Execution Receipt Format

### 7.1 Receipt as portable attestation

An **Execution Receipt** is a cryptographically-signed, portable attestation of an authorization decision produced by an RDP for a specific Intent. The receipt is the primary artifact AEAP uses to satisfy Invariant 4 (independent verifiability).

A receipt is portable in two senses: it travels with the Consequential Action to downstream systems (ledgers, orchestration state, audit stores, fulfillment pipelines) without requiring coordination with the issuing RDP, and it is verifiable offline by any party in possession of (a) the receipt itself and (b) the Issuer's published JWKS (§8), without any live callback to the Issuer at verification time.

The receipt is NOT:

- An authorization grant that other systems may extend or re-use for subsequent Intents. Each receipt binds to exactly one Intent (per §3.5's one-intent-id-one-evaluation invariant); a receipt for Intent A is not evidence for Intent B, even if A and B are structurally similar.
- A guarantee of execution, commitment, or outcome. See §2.6 for the layer separation; the receipt attests to the authorization decision only.
- A record of success. An ALLOW receipt attests that authorization was granted; it does not attest that the Consequential Action subsequently executed or completed.

### 7.2 Receipts across the ternary verdict

AEAP specifies which Verdicts produce receipts at each Conformance Level, resolving the architectural question of whether non-ALLOW outcomes carry receipts.

| Verdict | Core conformance | Extended conformance | Full conformance |
|---|---|---|---|
| `allow` | **REQUIRED** | **REQUIRED** | **REQUIRED** |
| `escalate` | N/A (ESCALATE is unsupported at Core) | **REQUIRED** | **REQUIRED** |
| `block` | OPTIONAL | OPTIONAL by default; **REQUIRED** where the Policy declares auditable denial lineage | **REQUIRED** for all BLOCK outcomes where `reason_code` indicates a Policy-driven denial (`POLICY_DENY`, `ESCALATE_UNSUPPORTED`, `POLICY_SILENT_ON_ACTION_TYPE`, `POLICY_INDETERMINATE`, `POLICY_VERSION_UNAVAILABLE`); OPTIONAL for operational-failure BLOCKs (`RDP_INTERNAL_ERROR`, `EVIDENCE_MISSING`, `EVIDENCE_STALE`, `EVIDENCE_INVALID`, `INTENT_MALFORMED`, `INTENT_NON_ATOMIC`, `INTENT_REPLAY`) |

**Rationale for issuing receipts on non-ALLOW outcomes at Extended and Full:**

Policy-driven denial carries the same forensic weight as Policy-driven allowance: both are decisions by the Protected Party's Policy that other parties may need to independently verify after the fact. An auditable denial receipt lets a Verifier confirm that a BLOCK was in fact issued by the authoritative RDP under the Protected Party's Policy, rather than reconstructed, forged, or omitted from an audit trail. ESCALATE receipts similarly let downstream workflow systems confirm that an escalation was formally initiated with a specific `escalation_id` under a specific Policy version.

Operational-failure BLOCKs (missing evidence, replay rejection, internal errors) do NOT require receipt issuance by default. These are typically transient conditions where receipt issuance would create audit noise without adding attestation value. Implementations MAY issue receipts for operational-failure BLOCKs where their Protected Parties require it; AEAP does not forbid this but does not mandate it either.

The receipt's `verdict` claim (§7.3) records which Verdict the receipt attests to. A Verifier that receives a receipt **MUST** check the `verdict` claim in addition to validating the signature; a cryptographically-authentic receipt with `verdict: block` is not evidence of authorization.

### 7.3 Required and optional claims

Every AEAP Execution Receipt carries a set of claims. The following claims are REQUIRED on every receipt regardless of Verdict:

| Claim | Type | Description |
|---|---|---|
| `v` | string | The AEAP Receipt Signature Profile version this receipt conforms to (see §8.1). |
| `iss` | string | The identifier of the Issuer (the RDP that produced the receipt). |
| `kid` | string | The key identifier the Issuer used to sign this receipt. Resolves against the Issuer's published JWKS (§8.2). |
| `iid` | string | The `intent_id` of the Intent this receipt corresponds to (per §3.2). |
| `intent_digest` | string | The cryptographic digest of the complete canonicalized Intent, computed per §3.4.1 as `sha256(jcs(intent))` and encoded as a lowercase hex string prefixed with `sha256:`. The `intent_digest` is the receipt's authoritative cryptographic commitment to the authorization subject. A Verifier presented with any (Intent, receipt) pair **MUST** recompute the digest from the presented Intent and confirm it matches this claim before relying on any other claim in the receipt. REQUIRED on all receipts regardless of Verdict. |
| `verdict` | string | The Verdict this receipt attests to: `allow`, `escalate`, or `block`. |
| `reason_code` | string | The reason code from the Verdict (per §6.5). REQUIRED on all receipts (including ALLOW, where `POLICY_ALLOW` is the typical value). |
| `evaluated_policy_version` | string | The authoritative Policy version against which the Intent was evaluated (per §4.2.1). |
| `iat` | number (Unix seconds) | The timestamp at which the receipt was issued. |
| `exp` | number (Unix seconds) | The timestamp at which the receipt expires (see §7.6). |

The following claims are OPTIONAL and included based on Verdict type, Conformance Level, and Policy requirements:

| Claim | Conditions | Description |
|---|---|---|
| `evaluated_policy_digest` | REQUIRED at Extended and Full; OPTIONAL at Core | The cryptographic digest of the Policy against which the Intent was evaluated (per §4.2.2). |
| `cid` | OPTIONAL | The `context_id` from the Intent (per §3.3), if present. |
| `mid` | REQUIRED where the Protected Party is identified by a merchant identifier | The merchant identifier of the Protected Party. |
| `actor` | REQUIRED on ALLOW and ESCALATE | The Actor attribution object from the Intent (per §3.2), carried for downstream attribution. |
| `action` | REQUIRED on ALLOW and ESCALATE | A structured object carrying the `action_type` and `action_target` from the Intent for downstream matching. |
| `escalation_id` | REQUIRED on ESCALATE receipts | The stable escalation identifier (per §6.3.1). |
| `escalation` | REQUIRED on ESCALATE receipts | Policy-declared escalation routing information (per §6.3). |
| `supersedes_iid` | OPTIONAL | The `intent_id` of a prior Intent that this receipt's Intent supersedes (per §3.3), if present. |
| `evidence_envelope_ref` | REQUIRED when any externally-sourced Evidence Signal materially contributed to the Verdict; OPTIONAL otherwise | A reference to a separately-stored envelope containing the full value, provenance, and attestation records for each Evidence Signal the Verdict depended on. See §7.4.2 for the normative applicability rule. Protocol-metadata signals derived from the Intent itself (per §5.2's informative note) MAY remain represented inline in the receipt without triggering this requirement. |

Implementations MAY define additional claims prefixed with `x_` for implementation-specific use. Claims not prefixed with `x_` and not defined in this specification or a successor specification **MUST NOT** be accepted by a conformant Verifier.

### 7.4 Receipt structure and canonicalization

#### 7.4.1 Compact serialization

The default AEAP receipt serialization is a compact three-part string of the form:

```
<version>.<base64url-payload>.<base64url-signature>
```

Where:

- `<version>` is the Receipt Signature Profile identifier (e.g., `v2` for the Ed25519 profile; see §8.1).
- `<base64url-payload>` is the RFC 8785-canonicalized JSON claims object (§7.3) encoded as base64url without padding.
- `<base64url-signature>` is the signature over the canonical payload, computed per the Receipt Signature Profile, encoded as base64url without padding.

This compact format is suitable for inclusion in HTTP headers, log fields, URL fragments, database columns, and similar size-constrained contexts.

Non-normative example of a compact v2 receipt:

```
v2.eyJhY3Rpb24iOnsiYWN0aW9uX3RhcmdldCI6ImVudi9wcm9kdWN0aW9uIiwi...
   ...Qiwic3VwZXJzZWRlc19paWQiOiJpbnRfYWJjIn0.3n5tKjWqz-fWqRQAb...8pQ
```

Implementations MAY offer alternate serializations (JSON Web Signature JWS Compact Serialization, JWS JSON Serialization) as encoding profiles declared alongside the Receipt Signature Profile. The three-part compact serialization above is the default and **MUST** be supported by all conformant Issuers and Verifiers.

#### 7.4.2 Evidence envelope reference

The receipt payload does not inline the full value, provenance, and attestation records (§5.1.1) for every Evidence Signal evaluated. Doing so would push receipts past practical size limits for compact distribution.

Instead, the receipt carries an `evidence_envelope_ref` claim: a reference to a separately-stored structure (the **evidence envelope**) that records the full value, provenance, and attestation for each Evidence Signal the Verdict depended on.

**Applicability of the `evidence_envelope_ref` requirement.** If any externally-sourced Evidence Signal (per §5.2 — Identity, Authority, State, Settlement, or External categories, where sourcing is Carried, Collected, or Observed per §5.5) materially contributed to the Verdict, the receipt **MUST** either (a) contain the signal's value, provenance, and attestation inline, or (b) contain an `evidence_envelope_ref` pointing to a durably-stored envelope containing that information. Protocol-metadata signals derived from the Intent itself (Actor attribution, submission time, evaluated Policy identity, action-type coverage — the signals enumerated in §5.3) are already represented in the receipt's required claims and do NOT by themselves trigger the envelope requirement.

The envelope **MUST** be:

- Content-addressable (identified by a cryptographic digest, so the receipt's reference cryptographically binds to the envelope's contents)
- Independently verifiable once retrieved (a Verifier that fetches the envelope can confirm the envelope's digest matches the receipt's reference)
- Durably stored by the Issuer or a delegated storage system that the Verifier can access

**Two distinct verification questions.** AEAP's offline-verifiability guarantee applies to receipt signature verification, not to evidence retrieval. The specification distinguishes two verification layers:

- **Receipt signature verification** confirms that the receipt was signed by the identified Issuer and has not been tampered with. This verification is **strictly offline**: a Verifier in possession of (a) the receipt and (b) the Issuer's published JWKS (§8.2, which MAY be cached) requires no live callback to the Issuer. The `intent_digest` claim (§7.3) similarly permits strictly offline binding between the receipt and the presented Intent.
- **Evidence envelope verification** confirms that the Evidence Signals cited by the receipt match the actual evidence the RDP evaluated. This verification requires retrieving the evidence envelope from its storage location. Once retrieved, the envelope is independently verifiable (its content hash matches the receipt's reference, and its internal provenance and attestation records can be re-checked). Retrieval is not offline, but the trust basis for the envelope's contents, once held, is independent of the Issuer at verification time.

A Verifier that needs only to confirm "this receipt was issued by this Issuer and attests to this Intent and Verdict" performs receipt signature verification alone and can do so offline. A Verifier that additionally needs to confirm the evidentiary basis of the Verdict performs both layers and MUST retrieve the envelope. The specification does not require every Verifier to perform both layers; the Protected Party's audit requirements, not AEAP, determine which layers a specific downstream system needs.

The specific format of the evidence envelope is implementation-defined within these constraints. An envelope format definition is a conformance artifact that MAY be published as a companion specification or as an AEAP Profile.

#### 7.4.3 Signing and Intent binding

The receipt's claims object includes `intent_digest` (§7.3), which cryptographically commits the receipt to the complete canonicalized Intent independently of the signature. The signature then commits the Issuer's attestation to the complete claims object, which transitively commits to the Intent through the digest.

The signature is computed over the exact base64url-encoded canonical payload (not over the pre-encoded JSON, and not over the full three-part string). The signing input is therefore:

```
intent_digest = sha256(jcs(intent))
claims        = { v, iss, kid, iid, intent_digest, verdict, reason_code,
                  evaluated_policy_version, iat, exp, ... }
signing_input = base64url(jcs(claims))
signature     = sign(signing_input, private_key, algorithm)
```

Where `jcs(...)` is the RFC 8785 canonicalization (§3.4) and `sign` is the Receipt Signature Profile's declared signing algorithm (§8.1).

Verification inverts this process in two layers that MUST both succeed before downstream reliance:

1. **Signature verification.** The Verifier parses the three parts, verifies the signature over the payload bytes using the public key identified by `kid`, and decodes the payload as JSON. A failed signature yields verification result `INVALID` and the receipt MUST NOT be relied upon.

2. **Intent binding verification.** The Verifier, having access to the Intent the receipt purportedly authorizes, recomputes `sha256(jcs(presented_intent))` and compares the result to the receipt's `intent_digest` claim. A mismatch indicates that the receipt was issued for a different Intent and MUST NOT be relied upon for the presented one.

Both layers are strictly offline operations that require only the receipt, the presented Intent, and the Issuer's published JWKS (which MAY be cached per §8.4).

### 7.5 Receipt lineage model

#### 7.5.1 Supersession chains

Where an Intent carries `supersedes_intent_id` (§3.3), the receipt issued for that Intent carries `supersedes_iid` in its claims. This creates a durable, cryptographically-verifiable chain between a retried Intent and the Intent it superseded.

A Verifier reconstructing a supersession chain traverses the chain by following `supersedes_iid` references across receipts:

```
Receipt A: iid=int_001, verdict=block,   reason_code=EVIDENCE_STALE
Receipt B: iid=int_002, verdict=block,   reason_code=EVIDENCE_INVALID, supersedes_iid=int_001
Receipt C: iid=int_003, verdict=allow,   reason_code=POLICY_ALLOW,     supersedes_iid=int_002
```

Each receipt in the chain is independently valid and verifiable. The chain itself is an emergent property of the receipts, not a separately-stored structure. A system that holds all three receipts can reconstruct and audit the full retry lineage; a system that holds only Receipt C has an authoritative ALLOW attestation and the pointer to Receipt B, which it MAY retrieve if full lineage is needed.

#### 7.5.2 Escalation chains

Where an ESCALATE Verdict is subsequently resolved by an out-of-band decision process (§6.3), the resolution MAY be fed back as a new Intent whose Evidence Signals include the `escalation_id` of the resolved escalation. The new Intent is evaluated by the RDP under the Policy, and the resulting receipt (ALLOW, ESCALATE again, or BLOCK) completes the escalation lineage.

A Verifier reconstructing an escalation chain holds:

```
Receipt E: iid=int_010, verdict=escalate, escalation_id=esc_abc
Receipt F: iid=int_011, verdict=allow,    supersedes_iid=int_010
           (evidence_envelope_ref points to envelope containing the resolution attestation keyed to esc_abc)
```

The specific protocol for feeding escalation resolution back to the RDP is Policy-declared and is not normatively specified by AEAP. The lineage model above describes only how the resulting receipts chain together.

### 7.6 Receipt expiration and acceptance window

Every receipt carries an `exp` claim indicating when the receipt expires. The expiration semantics are specific to AEAP:

- **A receipt's `exp` is the window during which downstream systems may rely on the authorization decision for the purpose of initiating the authorized action.** A Composition Boundary that holds an ALLOW receipt whose `exp` has elapsed **MUST NOT** execute the Consequential Action on the basis of that receipt; a new authorization decision is required.
- **A receipt's `exp` does NOT bound the receipt's verifiability as an audit artifact.** A Verifier that receives an expired receipt in an audit context MAY still verify the signature, confirm the receipt was authentic at issuance time, and record the historical authorization decision. Expiration bounds reliance for new execution, not historical attestation.

Issuers **MUST** set `exp` based on the sensitivity of the Consequential Action class and the staleness tolerances declared by the Policy for the Evidence Signals the Verdict depended on. A receipt whose supporting Evidence Signals have aggregate staleness tolerance of 10 minutes **MUST NOT** carry an `exp` more than 10 minutes beyond `iat`.

Core-conformant RDPs' default `exp` **MUST** be no greater than the Issuer's declared minimum staleness tolerance across all Policy-referenced Evidence Signals, floored at a RECOMMENDED minimum of 60 seconds and capped at a RECOMMENDED maximum of 24 hours. Extended and Full implementations SHOULD document their expiration policy as part of their conformance report.

---

## 8. JWKS and Key Rotation

### 8.1 Receipt Signature Profiles

AEAP specifies two **Receipt Signature Profiles**, each identified by a version string that appears as the first component of the compact receipt serialization (§7.4.1).

**Profile v1 (HMAC-SHA256) — Local Development Profile.** A symmetric-key profile intended exclusively for local development, reference-implementation demonstrations, and single-process compositions where the Issuer and Verifier share a secret through an out-of-band channel.

- Signing algorithm: HMAC-SHA256 per RFC 2104
- Key format: a 256-bit (32-byte) shared secret, out-of-band distribution
- `kid` semantics: an opaque identifier for the shared secret

**Profile v1 does NOT satisfy AEAP base-protocol conformance at any Conformance Level.** The HMAC construction requires the Verifier to possess the shared secret, which cannot be published in a public JWKS and therefore cannot satisfy the independent-verifiability requirement of Invariant 4 (§2.2) for receipts that cross trust boundaries. Implementations MAY support v1 for in-process reference-implementation and test-harness use; implementations MUST NOT claim Core, Extended, or Full base-protocol conformance on the basis of v1 alone. Base-protocol conformance at every Conformance Level requires at least one asymmetric Receipt Signature Profile (currently v2 Ed25519, or any future asymmetric profile declared by this specification).

**Profile v2 (Ed25519) — Production Profile.** An asymmetric-key profile intended for production use. The v2 profile is the base-protocol conformance signing profile and is the default for all Core-, Extended-, and Full-conformant Issuers.

- Signing algorithm: Ed25519 per RFC 8032
- Key format: Ed25519 public key published via JWKS (§8.2); private key held by Issuer
- `kid` semantics: an opaque identifier matching a `kid` entry in the Issuer's published JWKS

Additional Receipt Signature Profiles MAY be defined in future versions of this specification or in companion specifications. Each profile **MUST** declare its algorithm, key format, and `kid` semantics explicitly, and **MUST** specify how it integrates with the compact serialization in §7.4.1.

### 8.2 Published JWKS endpoint

Every Issuer using v2 (Ed25519) or any future asymmetric-key profile **MUST** publish a JSON Web Key Set at a stable, HTTPS-accessible URL. The URL is included in the Issuer's conformance declaration and in any receipts the Issuer produces (via the `iss` claim resolving to a well-known location).

The JWKS **MUST**:

- Be served over HTTPS with a valid TLS certificate issued by a publicly-trusted CA.
- Include a `kid` for every key in the set.
- Omit private key material. The JWKS carries only public keys and metadata.
- Include at minimum the current signing key(s). The JWKS MAY include additional keys that are published for verification purposes but not currently used for signing (see §8.3).
- Be cacheable with explicit `Cache-Control` headers. The Issuer SHOULD support HTTP conditional requests (ETag or Last-Modified) to permit efficient revalidation.

Non-normative example JWKS entry:

```json
{
  "keys": [
    {
      "kty": "OKP",
      "crv": "Ed25519",
      "kid": "aeap-prod-2026-10",
      "x": "11qYAYKxCrfVS_7TyWQHOg7hcvPapiMlrwIaaPcHURo",
      "alg": "EdDSA",
      "use": "sig",
      "aeap_status": "current"
    }
  ]
}
```

#### 8.2.1 Key lifecycle states

Every key in the JWKS **MUST** carry an `aeap_status` field with exactly one of the following four values. The `aeap_status` field is a namespaced AEAP-specific JWK extension (RFC 7517 §4.3 permits additional members); the `aeap_` prefix is used to prevent collision with any future JWK registered member names. The field is normative within AEAP: it determines the Verifier's acceptance behavior per §8.4.

| State | Signs new receipts? | Verifies historical signatures? | Issuer trust for reliance |
|---|---|---|---|
| **`current`** | ✅ | ✅ | ✅ |
| **`rotating-out`** | Generally no (grace period only; see §8.3) | ✅ | ✅ |
| **`retired`** | ❌ | ✅ | ✅ for receipts issued while the key was active |
| **`compromised`** | ❌ | Mathematically yes; trust-wise no | ⚠️ Requires out-of-band evidence (see §8.2.2) |

A key SHOULD transition through the lifecycle: `current` → `rotating-out` → `retired`. The `compromised` state is reached only on evidence of private-key compromise and bypasses the normal rotation sequence.

#### 8.2.2 `compromised` state semantics

The `compromised` state exists because deleting a compromised key from the JWKS entirely would prevent historical verifiers from mathematically validating receipts that were legitimately issued before the compromise. AEAP preserves the ability to perform the cryptographic check while making the trust implication explicit:

- **A signature made with a `compromised` key MAY remain cryptographically verifiable, but the verification result MUST NOT by itself establish issuer trust.** Downstream reliance on the receipt requires additional evidence establishing that the signature was created before the key compromise or was otherwise authorized by the legitimate issuer.
- Keys in `compromised` state **MUST** include the `aeap_compromised_at` field: an ISO 8601 timestamp (UTC) indicating when the compromise was declared. Receipts whose `iat` precedes `aeap_compromised_at` have at least a temporal basis for pre-compromise origin claims; this is not proof but is diagnostic information the Verifier can use.
- Issuers MAY additionally publish an `aeap_compromise_notice` field pointing to a signed revocation notice or append-only transparency log entry documenting the compromise event.

Non-normative example of a compromised key entry:

```json
{
  "kty": "OKP",
  "crv": "Ed25519",
  "kid": "aeap-prod-2026-01",
  "x": "11qYAYKxCrfVS_7TyWQHOg7hcvPapiMlrwIaaPcHURo",
  "alg": "EdDSA",
  "use": "sig",
  "aeap_status": "compromised",
  "aeap_compromised_at": "2026-10-02T14:30:00Z",
  "aeap_compromise_notice": "https://api.aiyoplane.com/.well-known/key-compromise/aeap-prod-2026-01"
}
```

The AEAP key lifecycle gives Verifiers a three-part evaluation model:

1. **Cryptographic status.** Does the signature mathematically verify against the key's public material?
2. **Key status.** Was the signing key trusted at the relevant time (per the key's lifecycle state at verification time, and for `compromised` keys, per the receipt's `iat` relative to `aeap_compromised_at`)?
3. **Reliance decision.** Given the above, is the receipt acceptable for the specific downstream purpose for which it is being evaluated?

AEAP mandates how the first two layers are evaluated. The third layer (reliance decision) is the Protected Party's or the Verifier's own responsibility; AEAP provides the diagnostic information needed to make that decision but does not make the decision for the Verifier.

*Informative rationale:* This distinction between cryptographic verification and trust/revocation status is analogous to established public-key infrastructure practice, where signature verification and certificate revocation checking are separate operations.

### 8.3 Key rotation discipline

Issuers **MUST** rotate signing keys on a documented schedule. The RECOMMENDED maximum lifetime for a v2 (Ed25519) signing key is 90 days of active signing use, followed by a 90-day verification-only period during which receipts signed with the key remain verifiable but no new receipts are signed with it, followed by retirement.

**During rotation,** the Issuer **MUST**:

1. Generate the new key pair and publish the new public key in the JWKS alongside the current key, with an explicit `kid` for each.
2. Honor a brief grace period during which the Issuer MAY sign with either key. The grace period SHOULD be no longer than 24 hours and exists only to accommodate caching and operational constraints.
3. Begin signing exclusively with the new key at the end of the grace period.
4. Continue to publish the previous key in the JWKS throughout the verification-only period, so that Verifiers evaluating receipts signed before the rotation can still resolve the `kid`.
5. Remove the previous key from the JWKS only after the verification-only period AND after all receipts signed with the previous key have passed their `exp` (whichever is longer).

**Emergency key compromise** triggers an accelerated rotation to the `compromised` lifecycle state (§8.2.2) rather than removal from the JWKS. On evidence of private-key compromise, the Issuer **MUST**:

- Immediately transition the compromised key to `aeap_status: compromised` in the JWKS, with `aeap_compromised_at` set to the Issuer's best estimate of compromise effective time (if unknown, the time of discovery)
- Publish a signed revocation notice via a mechanism documented in the Issuer's conformance declaration (e.g., a well-known endpoint, an append-only transparency log entry), and link it from the key entry's `aeap_compromise_notice` field
- Begin signing new receipts with a replacement key per the standard rotation procedure above
- Treat the trust status of receipts signed by the compromised key as determined by the Verifier's reliance decision (§8.2.2's three-part model), not by the Issuer's unilateral invalidation. The Issuer SHOULD NOT re-issue valid receipts for Intents whose original receipts were signed by the compromised key; where downstream reliance on a pre-compromise receipt is required, Protected Parties and downstream Verifiers reconstruct trust through alternative channels (out-of-band evidence that the receipt predates the compromise, additional cross-system attestations, operator attestations).

Keys MAY be removed from the JWKS entirely only after the Issuer has determined that no downstream Verifier is reasonably expected to require even the mathematical verification of signatures made with the key. For routine `retired` keys, this is typically after all receipts signed with the key have passed their `exp`. For `compromised` keys, the removal timeline is Issuer policy; AEAP RECOMMENDS retaining `compromised` keys indefinitely as part of the public forensic record.

### 8.4 Verifier behavior on unknown, rotated, or compromised keys

#### 8.4.1 Verification result codes

A conformant Verifier **MUST** report one of the following result codes for every receipt it processes. The result codes carry distinct semantics that downstream reliance logic MUST honor; collapsing them into a single `verified: true/false` boolean is non-conformant because it discards diagnostic information the specification requires the Verifier to preserve.

| Result code | Signature verifies? | Key status | Meaning for reliance |
|---|---|---|---|
| **`VERIFIED`** | ✅ | Key is `current`, `rotating-out`, or `retired`; receipt is within `exp` | Receipt may be relied upon per its claims. |
| **`VERIFIED_EXPIRED`** | ✅ | Key is `current`, `rotating-out`, or `retired`; receipt's `exp` has elapsed | Receipt is a valid historical artifact (§7.6) but MUST NOT be relied upon to initiate new execution. |
| **`COMPROMISED_KEY`** | ✅ | Key is `compromised` | The signature is mathematically verifiable, but the verification result MUST NOT by itself establish issuer trust. Downstream reliance requires additional out-of-band evidence (§8.2.2). The Verifier SHOULD surface the key's `aeap_compromised_at` timestamp and the receipt's `iat` to assist the reliance decision. |
| **`UNVERIFIABLE`** | — | `kid` not resolved in the current JWKS | The receipt cannot be verified. This result includes the case where the JWKS was unreachable AND no non-expired cached copy exists. |
| **`INVALID`** | ❌ | Any | The signature does not verify against the key identified by `kid`. The receipt MUST NOT be relied upon under any circumstance. |
| **`INTENT_MISMATCH`** | ✅ | Any (except `compromised`) | The signature verifies, but the receipt's `intent_digest` does not match the digest of the presented Intent (§7.4.3 layer 2). The receipt is authentic but was issued for a different Intent and MUST NOT be relied upon for the presented one. |

Implementations MAY define additional result codes prefixed with `x_` for implementation-specific diagnostic refinement. Non-prefixed result codes not defined in this specification or a successor specification **MUST NOT** be emitted by a conformant Verifier.

#### 8.4.2 Verifier procedure

A Verifier that encounters a receipt whose `kid` does not resolve in the current JWKS cache **MUST**:

1. Fetch the JWKS at the time of verification (respecting cache headers but ignoring any cached result that does not include the `kid`).
2. If the fetched JWKS still does not include the `kid`, the receipt **MUST** be reported as `UNVERIFIABLE`.

A Verifier **MUST NOT**:

- Accept a receipt whose signature cannot be verified against any key currently present in the published JWKS (report `INVALID` or `UNVERIFIABLE` as appropriate).
- Cache "key not found" results for longer than the JWKS's declared cache lifetime; otherwise rotation would be invisible to the Verifier.
- Contact the Issuer at verification time for anything other than the JWKS. The receipt itself carries all information required for verification (per Invariant 4).
- Collapse `COMPROMISED_KEY` into either `VERIFIED` or `INVALID`. The compromised state is semantically distinct from both and must be reported as such.

Where the JWKS is briefly unreachable (network partition, Issuer outage), a Verifier MAY use its most recently-cached JWKS up to the cache's expiration. Verifiers **MUST** fail closed (report `UNVERIFIABLE`) if the JWKS is unreachable AND no non-expired cached copy exists.

---

## 9. Composition Boundary Invariants

### 9.1 What a Composition Boundary MUST do

A **Composition Boundary** is the specific location in a system where the RDP is invoked (per §2.4). Every AEAP-conformant Composition Boundary **MUST** satisfy the following invariants:

1. **Invoke the RDP strictly between Intent formation and Execution Surface action.** The Composition Boundary **MUST NOT** permit the Execution Surface to perform the Consequential Action before the RDP returns a Verdict. The Composition Boundary **MUST NOT** invoke the RDP after the Consequential Action has already taken effect (per Invariant 2 of §2.2).
2. **Forward exactly one Intent per Consequential Action.** The Composition Boundary **MUST NOT** bundle multiple Consequential Actions into a single RDP invocation (per Invariant 1 of §2.2). If a workflow proposes multiple Consequential Actions in close succession, each action is a separate Intent and receives a separate RDP evaluation.
3. **Honor the Verdict atomically.** On ALLOW, the Composition Boundary MAY permit execution (per §6.2). On ESCALATE, the Composition Boundary **MUST** hold the Consequential Action pending resolution and route to the escalation surface declared in the Verdict's `escalation` field. On BLOCK, the Composition Boundary **MUST** prevent execution and surface the typed `reason_code` to the Actor.
4. **Validate the Execution Receipt before relying on it.** On ALLOW or ESCALATE, the Composition Boundary **MUST** validate the receipt's signature (per §8.4), confirm that `iid` matches the submitted Intent, confirm that `verdict` matches the Verdict being enforced, and confirm that `exp` has not elapsed, before taking any action in reliance on the receipt.
5. **Attach the receipt to downstream artifacts where lineage is needed.** The receipt is the authoritative portable attestation of the authorization decision; the Composition Boundary SHOULD attach it to orchestration state, ledger entries, audit stores, and any artifact a downstream Verifier may need to validate.
6. **Surface typed denials to the Actor.** On BLOCK, the Composition Boundary **MUST** surface the `reason_code` (per §6.5) to the Actor rather than failing silently or returning a generic error. The Actor needs the typed reason to construct a correct supersession Intent, if any.

### 9.2 What a Composition Boundary MUST NOT do

Equally normative, a Composition Boundary **MUST NOT**:

- Second-guess the RDP's Verdict. The Composition Boundary enforces; it does not re-evaluate. A Composition Boundary that permits execution against a BLOCK Verdict or against an unresolved ESCALATE is non-conformant. Declining to execute an ALLOW Verdict is NOT a conformance violation (ALLOW establishes authorization, not an obligation to execute — see §6.2); the Composition Boundary MAY decline for any reason outside AEAP's scope.
- Modify the Intent between RDP evaluation and Execution Surface action. Any modification constitutes a new Intent that requires its own evaluation.
- Pass the receipt to the Execution Surface in a form that the Execution Surface could present as its own authorization (e.g., by stripping the signature and signing with a different key). The receipt's integrity is cryptographically bound to the Issuer; forwarding integrity is the Composition Boundary's responsibility.
- Re-use a receipt across multiple Consequential Actions. Each receipt binds to exactly one `iid` (per §3.5).
- Store receipts beyond the retention policy declared by the Protected Party. Receipts contain Actor and action metadata that MAY be subject to data-retention regulations; the Composition Boundary SHOULD treat receipts as governed records.

### 9.3 Fail-closed opt-out at the boundary

Per Invariant 5 (§2.2), the RDP fails closed by default. A Protected Party MAY choose to permit fail-open behavior at the Composition Boundary under specific documented conditions (e.g., "if the RDP is unreachable, permit execution with a specific operator attestation").

Where a Composition Boundary implements such an opt-out, it **MUST**:

- Document the specific conditions under which the opt-out applies
- Record every opt-out event in an append-only audit log, including the operator identity and the specific condition that triggered the opt-out
- Attach a distinct marker to any downstream artifact to indicate that execution occurred under a fail-open condition, so Verifiers can distinguish executions backed by valid AEAP receipts from executions backed by operator-attested exceptions

AEAP does not forbid Composition-Boundary-level fail-open behavior, but it does require that such behavior be explicit, logged, and attributable. Silent fail-open is a conformance violation at the Composition Boundary level, even though the RDP itself may still be conformant.

---

## 10. Rollout Discipline

### 10.1 Shadow Mode

**Shadow Mode is an operational deployment posture, not an AEAP Verdict.** The `shadow` claim on a receipt and the `verdict` claim on a receipt are orthogonal dimensions: a Shadow Mode receipt may carry any of `allow`, `escalate`, or `block` as its Verdict, exactly as an Enforce Mode receipt may. The `shadow` claim describes whether the Composition Boundary enforced the Verdict against the Execution Surface; the `verdict` claim describes what the RDP decided.

**Shadow Mode** is the operational posture in which the RDP is invoked and produces Verdicts, but the Composition Boundary does not enforce the Verdicts against the Execution Surface. The Execution Surface proceeds with the Consequential Action regardless of Verdict; the RDP's output is recorded for Protected-Party review but does not gate execution.

Shadow Mode is RECOMMENDED as the initial deployment posture for a new RDP, a new Policy version, or an onboarding Protected Party. Shadow Mode allows the Protected Party to observe how the Policy evaluates real traffic without exposure to the operational consequences of enforcement errors.

An Extended-conformant Composition Boundary **MUST** support Shadow Mode. Shadow Mode receipts (where the Issuer continues to issue receipts despite non-enforcement) **MUST** carry a `shadow` claim with the value `true` to prevent downstream systems from relying on a Shadow Mode receipt as if it were an enforced authorization.

### 10.2 Enforce Mode

**Enforce Mode** is the operational posture in which the Composition Boundary gates the Execution Surface on the RDP's Verdict per §9. Enforce Mode is the default posture for production AEAP deployments and the posture that corresponds to the normative requirements elsewhere in this specification.

Enforce Mode receipts do NOT carry a `shadow` claim (or carry it with value `false`, which is semantically equivalent to omission).

### 10.3 Transition between modes

A Protected Party MAY transition the Composition Boundary between Shadow Mode and Enforce Mode at any time. The transition is a Composition Boundary configuration change and does not require an AEAP protocol operation.

Implementations SHOULD provide a transition workflow that:

- Clearly indicates to operators which mode is currently active
- Records every mode transition in an audit log with operator attribution
- Permits per-action-type mode selection (e.g., "enforce `payment.*` but shadow `deploy.*`") where the Protected Party requires graduated rollout

The receipt's `shadow` claim is the authoritative indicator of which mode a given Verdict was produced under; Verifiers that need to distinguish Shadow Mode from Enforce Mode receipts **MUST** check the `shadow` claim.

---

## 11. Interop Profile: Human Agency Protocol (HAP)

*The AEAP Profile disclaimer in §1.5 applies to this Profile: inclusion of HAP as an AEAP Profile does not imply that AEAP depends upon, endorses, or is governed by HAP's specification. This Profile is a translation surface for consuming HAP mandate tickets as AEAP Evidence Signals.*

### 11.1 Scope and version tracking

The **AEAP-HAP Profile** defines the translation surface between AEAP and the Human Agency Protocol (HAP). An RDP Implementation that declares AEAP-HAP conformance accepts HAP mandate tickets as Evidence Signals under its base-protocol AEAP Conformance Level.

**External specification tracked.** AEAP-HAP Profile v1 is validated against **HAP v0.7**. AEAP does not maintain a shadow definition of HAP's internal schema; the HAP specification is authoritative on the representation and semantics of HAP mandate tickets. The AEAP-HAP Profile specifies only how AEAP consumes HAP mandate tickets at the Composition Boundary. When HAP evolves to a new version, the AEAP-HAP Profile is revised to track it, and the new Profile revision is versioned independently (AEAP-HAP v2, v3, etc.).

**Nature of the external artifact.** A HAP mandate ticket is itself an externally-issued authorization attestation — not merely an identity credential. The ticket represents bounded human authorization pre-granted for a specific class of consequential action, signed by a HAP Authority Server. AEAP consumes the mandate ticket as an authority Evidence Signal that attests to this pre-granted authorization; AEAP's own Verdict is still produced by the Protected Party's Policy evaluating the Intent against the mandate ticket alongside any other required Evidence Signals.

### 11.2 Semantic field categories consumed

An AEAP-HAP-conformant RDP processing a HAP mandate ticket **MUST** extract and surface the following semantic categories to Policy evaluation. The internal field names used by HAP to encode each category are authoritative to the HAP specification; the Profile normatively specifies the semantic requirement at the AEAP boundary.

| AEAP semantic category | Normative requirement at AEAP boundary | Policy visibility |
|---|---|---|
| **Mandate identifier** | Unique identifier for the specific mandate ticket | REQUIRED |
| **Issuer** | Identity of the HAP Authority Server that signed the mandate | REQUIRED |
| **Human principal** | Identity of the human whose authorization the mandate represents | REQUIRED |
| **Delegated actor / agent binding** | Identifier of the agent authorized to act under this mandate | REQUIRED |
| **Authorized action descriptor** | The action class or category the mandate permits | REQUIRED |
| **Action target constraint** | The target entity, resource, or scope the mandate permits action against | REQUIRED |
| **Bounds** | Numeric or structured limits on the authorization (amounts, counts, rate limits) | REQUIRED where present in the mandate |
| **Validity interval** | Not-before and expiration timestamps | REQUIRED |
| **Commitment mode** | Whether the mandate authorizes a single use, a bounded series, or a time window | REQUIRED where distinguishing modes affects Policy evaluation |
| **Revocation state** | Current revocation status (active, revoked, suspended) as of evaluation time | REQUIRED |
| **Audience** | The target system(s) the mandate was issued for | REQUIRED where the mandate declares an audience constraint |
| **Signature algorithm and key resolution** | How the mandate's signature is verified and against which keys | REQUIRED |

### 11.3 Normative behavior at the AEAP boundary

An AEAP-HAP-conformant RDP evaluating an Intent whose Evidence Signals include a HAP mandate ticket **MUST**:

1. Verify the mandate's cryptographic signature against the HAP issuer's published key material per HAP's own key-resolution rules.
2. Confirm the mandate has not expired per its validity interval, is not revoked per its current revocation state, and (where applicable) matches the audience constraint.
3. Confirm the mandate's authorized action descriptor and target constraint cover the Intent's `action_type` and `action_target`. If the Intent proposes an action outside the mandate's scope, the mandate does NOT satisfy the authorization requirement for this Intent; the Policy decides the Verdict accordingly.
4. Confirm the Intent's `actor` matches the mandate's delegated-actor binding, if the Policy requires actor-binding verification.
5. Surface each of the semantic categories in §11.2 to Policy evaluation using provenance "HAP" and attestation referencing the mandate's signature.
6. Record the mandate identifier, issuer, and verification result in the receipt's evidence envelope (§7.4.2).

**Interoperability requirement.** Two independent AEAP-HAP-conformant implementations evaluating the same HAP mandate ticket against the same Intent under the same Policy **MUST** reach the same AEAP-relevant interpretation (same semantic categories extracted with the same values, same mandate-vs-Intent compatibility result).

### 11.4 Non-scope

The AEAP-HAP Profile does not require the RDP to:

- Issue HAP mandate tickets (that is the role of HAP Authority Servers)
- Participate in HAP's mandate revocation workflows beyond consuming revocation state as evidence
- Represent AEAP receipts in HAP's vocabulary
- Define or override HAP's own field names, representation, or trust model

Where a Protected Party requires bidirectional HAP integration, that integration is implementation-defined above the AEAP-HAP translation surface.

---

## 12. Interop Profile: Okta Cross App Access and ID-JAG

*The AEAP Profile disclaimer in §1.5 applies to this Profile: inclusion of Okta XAA and ID-JAG as an AEAP Profile does not imply that AEAP depends upon, endorses, or is governed by Okta's specification. This Profile is a translation surface for consuming ID-JAG grants as AEAP Evidence Signals.*

### 12.1 Scope and version tracking

The **AEAP-XAA Profile** defines the translation surface between AEAP and Okta's Cross App Access (XAA) and Identity-JAG grants. An RDP Implementation that declares AEAP-XAA conformance accepts ID-JAG grants as Evidence Signals under its base-protocol AEAP Conformance Level.

**External specification tracked.** AEAP-XAA Profile v1 is validated against the Okta Cross App Access documentation and the ID-JAG assertion format current as of this specification's publication. AEAP does not maintain a shadow definition of ID-JAG's internal schema; Okta's specification is authoritative on the representation and semantics of ID-JAG grants. The AEAP-XAA Profile specifies only how AEAP consumes ID-JAG grants at the Composition Boundary. When Okta's XAA / ID-JAG specification evolves, the AEAP-XAA Profile is revised to track it and versioned independently.

**Nature of the external artifact.** An ID-JAG grant is an identity delegation assertion that is subsequently exchanged for a scoped access token at the resource authorization server. The ID-JAG itself attests delegated identity and authorization scope; it is not itself a decision on whether a specific consequential action may execute. AEAP consumes the ID-JAG as evidence of delegated identity and authority; AEAP's own Verdict is still produced by the Protected Party's Policy.

### 12.2 Semantic field categories consumed

An AEAP-XAA-conformant RDP processing an ID-JAG grant **MUST** extract and surface the following semantic categories to Policy evaluation. The internal field names used by ID-JAG to encode each category are authoritative to the Okta XAA / ID-JAG specification; the Profile normatively specifies the semantic requirement at the AEAP boundary.

| AEAP semantic category | Normative requirement at AEAP boundary | Policy visibility |
|---|---|---|
| **Grant identifier** | Unique identifier for the specific ID-JAG assertion | REQUIRED |
| **Issuer** | Identity of the Identity Provider that signed the ID-JAG | REQUIRED |
| **Delegated subject** | Identity of the user whose delegation the grant represents | REQUIRED |
| **Delegation / actor chain** | The chain of acting principals from the originating user through intermediate delegations to the final acting agent | REQUIRED where present (`act` chain) |
| **Authorized scopes** | The scopes the grant permits the delegated actor to request | REQUIRED |
| **Target resource / audience** | The resource server(s) or audience the grant was issued for | REQUIRED |
| **Validity interval** | Issued-at and expiration timestamps | REQUIRED |
| **Signature algorithm and key resolution** | How the grant's signature is verified and against which keys | REQUIRED |

### 12.3 Normative behavior at the AEAP boundary

An AEAP-XAA-conformant RDP evaluating an Intent whose Evidence Signals include an ID-JAG grant **MUST**:

1. Verify the ID-JAG grant's signature against the issuer's published key material per Okta's key-resolution rules.
2. Confirm the grant has not expired per its validity interval.
3. Confirm the grant's audience includes the RDP (or the AEAP boundary acting on behalf of a resource server the RDP governs).
4. Confirm the grant's authorized scopes cover the AEAP action implied by the Intent's `action_type`. The Policy declares how AEAP action-types map to XAA scopes.
5. Where an `act` (delegation chain) is present, surface the chain to Policy evaluation so that the Policy can reason about originating-vs-immediate actor identity.
6. Surface each of the semantic categories in §12.2 to Policy evaluation using provenance "Okta XAA" and attestation referencing the ID-JAG grant's signature.
7. Record the grant's identifier (`jti` or equivalent), issuer, and verification result in the receipt's evidence envelope.

**Interoperability requirement.** Two independent AEAP-XAA-conformant implementations evaluating the same ID-JAG grant against the same Intent under the same Policy **MUST** reach the same AEAP-relevant interpretation.

### 12.4 Composition with AEAP

AEAP-XAA is additive to base-protocol AEAP. An Intent evaluated under AEAP-XAA still produces an AEAP Verdict, an AEAP receipt, and composes under the AEAP Default Composition Algebra (§4.5). The ID-JAG grant is Evidence that the Policy references; the Policy, not the grant, decides the Verdict. AEAP-XAA does not displace or substitute for AEAP's base-protocol conformance obligations.

---

## 13. Interop Profile: Alibaba / Cisco Agent Operation Authorization (AOA)

*The AEAP Profile disclaimer in §1.5 applies to this Profile: inclusion of AOA as an AEAP Profile does not imply that AEAP depends upon, endorses, or is governed by the AOA specification. This Profile is a translation surface for consuming AOA tokens as AEAP Evidence Signals.*

### 13.1 Scope and version tracking

The **AEAP-AOA Profile** defines the translation surface between AEAP and the Alibaba / Cisco Agent Operation Authorization (AOA) framework. An RDP Implementation that declares AEAP-AOA conformance accepts AOA tokens as Evidence Signals under its base-protocol AEAP Conformance Level.

**External specification tracked.** AEAP-AOA Profile v1 is validated against the AOA IETF draft family and the Alibaba Open Agent Auth reference implementation current as of this specification's publication. AEAP does not maintain a shadow definition of AOA's internal schema; the AOA specification (and its evolving IETF drafts) is authoritative on the representation and semantics of AOA tokens. The AEAP-AOA Profile is revised and versioned independently as AOA stabilizes.

**Nature of the external artifact.** An AOA token (sometimes called an Agent Operation Authorization Token, or AOAT) is a rich authorization artifact that carries operation authorization, identity binding, policy references, evidence, delegation chain, and audit information. AOA tokens attest that a specific agent operation has been authorized by a human principal through a delegation chain. AEAP consumes the AOA token as an authority Evidence Signal; AEAP's own Verdict is still produced by the Protected Party's Policy.

### 13.2 Semantic field categories consumed

An AEAP-AOA-conformant RDP processing an AOA token **MUST** extract and surface the following semantic categories to Policy evaluation. The internal field names used by AOA to encode each category are authoritative to the AOA specification; the Profile normatively specifies the semantic requirement at the AEAP boundary.

| AEAP semantic category | Normative requirement at AEAP boundary | Policy visibility |
|---|---|---|
| **Token identifier** | Unique identifier for the specific AOA token | REQUIRED |
| **Issuer** | Identity of the AOA issuer that signed the token | REQUIRED |
| **Human principal** | Identity of the human whose authorization the token represents | REQUIRED |
| **Agent identity binding** | Cryptographic binding of the token to the specific agent authorized to use it | REQUIRED |
| **Authorized operation** | The specific agent operation the token permits | REQUIRED |
| **Resource / target** | The resource or target entity the authorized operation acts upon | REQUIRED |
| **Scope** | Any scope constraints applied to the authorization | REQUIRED where present |
| **Policy references** | References to the policies the token was issued under | REQUIRED where present |
| **Delegation chain** | The chain of delegations from the human principal to the acting agent | REQUIRED where present |
| **Embedded evidence / audit information** | Any additional attestation or audit data the token carries | REQUIRED to be preserved to the receipt's evidence envelope where present |
| **Validity interval** | Not-before and expiration timestamps | REQUIRED |
| **Signature algorithm and key resolution** | How the token's signature is verified and against which keys | REQUIRED |

### 13.3 Normative behavior at the AEAP boundary

An AEAP-AOA-conformant RDP evaluating an Intent whose Evidence Signals include an AOA token **MUST**:

1. Verify the AOA token's signature against the issuer's published key material per AOA's key-resolution rules.
2. Confirm the token has not expired per its validity interval.
3. Confirm the token's agent identity binding matches the Intent's `actor`.
4. Confirm the token's authorized operation and resource/target cover the Intent's `action_type` and `action_target`.
5. Where a delegation chain is present, surface the chain to Policy evaluation so that the Policy can reason about originating-vs-immediate authority.
6. Where embedded evidence or audit information is present, preserve it to the receipt's evidence envelope.
7. Surface each of the semantic categories in §13.2 to Policy evaluation using provenance "AOA" and attestation referencing the AOA token's signature.
8. Record the token identifier, issuer, agent binding, and verification result in the receipt's evidence envelope.

**Interoperability requirement.** Two independent AEAP-AOA-conformant implementations evaluating the same AOA token against the same Intent under the same Policy **MUST** reach the same AEAP-relevant interpretation.

### 13.4 Non-scope

The AEAP-AOA Profile does not require the RDP to issue AOA tokens, to participate in AOA revocation workflows beyond consuming revocation state as evidence, or to override AOA's own field names or trust model.

---

## 14. Versioning and Backward Compatibility

### 14.1 Specification versioning

This specification follows semantic versioning for its own evolution:

- **Major version (X.y.z)** indicates a breaking change — receipts issued under version X cannot be verified by implementations conformant only to version X-1, or evaluation behavior under version X is intentionally different from version X-1 for identical inputs.
- **Minor version (x.Y.z)** indicates new OPTIONAL capabilities that do not invalidate existing conformant behavior — new OPTIONAL claims, new reason codes, new Receipt Signature Profiles, new Interop Profiles. Minor-version additions MUST NOT change the semantics of previously-defined elements, and an implementation conformant to version X.Y.z MUST remain conformant to all elements of version X.(Y-1).z. New capabilities introduced in a minor version are OPTIONAL for implementations declaring conformance to that minor version.
- **Patch version (x.y.Z)** indicates editorial corrections, non-normative clarifications, or expanded informative notes that do not change normative requirements.

Published versions are frozen. Changes to a published version require a new version number per the rules above.

### 14.2 Receipt Signature Profile versioning

Each Receipt Signature Profile (§8.1) is versioned independently of the specification. A Profile version indicator appears as the first component of the compact receipt serialization (§7.4.1).

A Profile is formally deprecated by the specification by:

1. Publishing a successor Profile with a new version indicator
2. Declaring a deprecation window of at least 180 days during which both Profiles are supported
3. At the end of the deprecation window, moving the deprecated Profile to "retired" status

Issuers **MAY** continue to verify receipts signed under a retired Profile for the duration of those receipts' `exp` windows (per §8.3), but **MUST NOT** sign new receipts under a retired Profile.

### 14.3 Policy versioning in cross-RDP contexts

Where a Protected Party operates multiple RDPs (e.g., a production RDP and a shadow RDP for Policy validation, or RDPs across regional deployments), Policy versioning MUST remain globally coherent. A Policy version string MUST identify the same Policy content across all RDPs operated by the Protected Party; divergent Policies with the same version identifier are a conformance violation at the Protected Party level, even if each individual RDP remains conformant.

Where multiple Protected Parties exist in a system and each operates their own RDP(s), Policy version strings are scoped to the issuing Protected Party; cross-Protected-Party Policy version comparisons are not meaningful.

---

## 15. Security Model and Responsibility Boundaries

This section defines the security properties AEAP Core guarantees, the properties it explicitly does NOT guarantee, and the responsibility boundaries between the three parties that must cooperate for an AEAP-authorized action to produce the intended security outcome. The section is informative in its framing and normative where explicit MUST / MUST NOT language appears.

Any security claim about an authorization system decomposes into four distinct questions:

1. **Receipt authenticity.** Did a legitimate RDP issue this receipt?
2. **Decision integrity.** Was the authorization decision sound given the Policy and Evidence?
3. **Enforcement integrity.** Did the Composition Boundary actually enforce the Verdict?
4. **Operational accountability.** Can an operator later reconstruct what was authorized, under what Policy, with what Evidence, and whether the action was executed?

These are different questions requiring different controls. AEAP Core addresses some directly, informs others, and leaves still others to the implementation and the Executor. This section codifies which is which.

### 15.1 Responsibility boundaries

AEAP's security properties depend on correct behavior at three distinct responsibility layers:

**AEAP Core (this specification).** Defines invariants, verdict semantics, cryptographic contracts, canonicalization rules, replay-window requirements, verifier result codes, and the Composition Boundary primitive. The specification is the normative description of what MUST be true for conformant implementations. The authority of a claim at this layer is established by Appendix A conformance.

**RDP Implementation (the operator of a Runtime Decision Point).** Operates signing keys per §8; publishes the JWKS; signs receipts; maintains the replay-prevention store for `intent_id` uniqueness per §3.5; maintains audit records; performs key rotation and compromise handling per §8.4. For the Aiyoplane-operated hosted RDP, the operator is Aiyoplane, Inc.; for a self-operated RDP, the operator is the Protected Party or an agent acting on its behalf.

**Protected Party and Executor.** The Protected Party authors the Policy and selects the Evidence Signals and their trust declarations. The Executor is the party or system that performs the Consequential Action downstream of an ALLOW Verdict; the Executor is responsible for enforcing the Composition Boundary (§9) in its deployment, maintaining downstream idempotency (§15.3.1), and — where applicable — reporting observed execution outcomes to close the receipt lineage (§7.5).

A security property that holds at the Core layer does not automatically hold at the Implementation layer, and a property that holds at the Implementation layer does not automatically hold at the Executor layer. Each layer must be reasoned about separately.

### 15.2 Properties AEAP Core guarantees

For a conformant AEAP implementation at Core Conformance Level or above (§1.4), the following properties are guaranteed by the specification:

- **Receipt authenticity.** An Execution Receipt that successfully verifies against the Issuer's published JWKS using the Receipt Signature Profile declared in the receipt was produced by a party in possession of the corresponding signing key material at the moment the receipt was signed (§7, §8).
- **Receipt integrity.** The claims within a verified receipt have not been modified since signing. RFC 8785 JCS canonicalization combined with Ed25519 signatures over the canonical bytes make substitution of any claim detectable (§7.4).
- **Verifier result-code precision.** A conformant verifier MUST distinguish, at minimum, the result codes defined in §8.4: `VERIFIED`, `VERIFIED_EXPIRED`, `COMPROMISED_KEY`, `UNVERIFIABLE`, `INVALID`, and `INTENT_MISMATCH`. A verifier that collapses "the signature mathematically verifies" and "the signing key is still trustworthy for reliance" into a single boolean is non-conformant.
- **Policy evaluation determinism.** The same Policy, evaluated against the same canonical Intent and the same canonical Evidence Signals, MUST produce the same Verdict (§4.3).
- **Fail-closed defaults.** In any ambiguous, indeterminate, or erroneous condition — Policy silent on the Intent class, Policy evaluator unable to reach a conclusion, required Evidence Signal missing or expired, Policy store unreachable — the RDP MUST produce BLOCK, not ALLOW (§2.2 invariant 5).
- **Intent uniqueness and replay resistance.** A given `intent_id` MUST NOT be re-evaluated within the replay window defined by §3.5. Attempts to replay a previously-evaluated Intent within the window MUST produce BLOCK with the appropriate reason code.
- **Ternary verdict semantics.** The Verdict output of an evaluation is one of ALLOW, ESCALATE, or BLOCK (§6). BLOCK MUST NOT execute; unresolved ESCALATE MUST hold the action pending resolution; ALLOW MAY permit, subject to the Composition Boundary and §15.2 enforcement conditions.
- **Composition Boundary primitive.** AEAP Core defines the Composition Boundary (§9) and specifies its invariants. A conformant Composition Boundary MUST enforce the Verdict semantics above and MUST NOT silently convert an unresolved ESCALATE or a BLOCK into execution.

### 15.3 Properties AEAP Core explicitly does not guarantee

The following properties are intentionally outside AEAP Core's guarantees. Each is a legitimate security concern; naming the boundary explicitly is the mechanism by which the responsibility for each is correctly attributed.

#### 15.3.1 Exactly-once execution

AEAP Core prevents the same Intent from being re-evaluated within the replay window (§3.5). It does NOT prevent the authorized action from being executed more than once.

A conforming example of the gap: an Executor receives ALLOW, successfully performs the action, crashes before recording completion, and retries. The receipt still verifies — it is the same valid receipt — and the action can duplicate.

Downstream idempotency is the Executor's responsibility. Common patterns include idempotency keys tied to the receipt's `intent_id`, atomic state transitions keyed on `intent_id`, and payment-rail-level or database-level deduplication keyed on `intent_id`.

AEAP Core deliberately does not expand into an exactly-once execution protocol. Execution semantics are the concern of the systems that perform the execution; AEAP's concern is the authorization decision that gates them.

#### 15.3.2 Pre-expiration revocation of individual receipts

AEAP Core does NOT provide a mechanism for revoking an individual receipt before its `exp`. Portable offline verification against the published JWKS is a deliberate design property; introducing a mandatory online status check at verification time would reverse that property for every Executor.

Protected Parties requiring earlier withdrawal of a specific receipt MUST use an implementation-specific control. Two patterns are recognized by this specification:

**Short-lived receipts with re-evaluation.** The Protected Party configures the Policy to issue short `exp` (typically seconds to minutes). The Executor is required to re-submit the Intent for a fresh RDP evaluation per sensitive action. This pattern requires no protocol change; it is a Policy configuration choice using AEAP Core as specified.

**Implementation-specific online status mechanism.** An Issuer MAY expose an online status endpoint that an Executor opts into for specific high-value actions. The Executor explicitly accepts the availability dependency where the use case justifies it, rather than every Executor inheriting the dependency.

A future standardized AEAP profile MAY formalize the second pattern. Such a profile is NOT part of this specification and does NOT change any requirement in AEAP Core.

#### 15.3.3 Policy correctness

AEAP Core evaluates whatever Policy is authored and declared by the Protected Party. The specification guarantees that evaluation is deterministic and that the Verdict and reason code accurately reflect the result of that evaluation; it does NOT guarantee that the Policy itself expresses the rules the Policy author intended.

A conforming example of the gap: a Policy with a decimal-placement error in an amount threshold produces a deterministic ALLOW on a payment two orders of magnitude larger than the Protected Party intended. The receipt faithfully attests to the evaluation. The authorship error is a Protected Party responsibility.

Policy correctness is a Protected Party operational concern. Tooling that supports Policy authoring, review, simulation against historical Intents, versioning, and rollback is implementation infrastructure around the protocol. It is not a protocol guarantee.

#### 15.3.4 Evidence Signal truthfulness

The three-component Evidence envelope (§5.1) — Value / Provenance / Attestation — requires every Evidence Signal evaluated by the Policy to carry its own trust chain. A conformant AEAP implementation MUST evaluate the Attestations the Policy declares against the Policy's declared trust requirements.

AEAP Core does NOT guarantee:

- That the Attestation issuer is honest.
- That the Attestation reflects the current real-world state of the external system it describes.
- That the Attestation issuer's own key infrastructure is sound.

Selecting which Evidence Signal sources to trust and which Attestations to require is a Protected Party design decision made in the Policy.

#### 15.3.5 Enforcement at the Composition Boundary in a specific deployment

AEAP Core defines the Composition Boundary primitive (§9) and specifies the invariants it MUST satisfy. A reference Composition Boundary implementation that correctly implements the pattern is a correct implementation of the specification.

AEAP Core does NOT guarantee that the Composition Boundary in a given deployment is the only path to the gated Consequential Action. If a Protected Party gates a payment through an AEAP-attached Composition Boundary but leaves an unauthenticated administrative endpoint, a sibling API, or a direct database write that reaches the same consequence, the boundary has been bypassed structurally.

Verifying that enforcement is exhaustive — that the gated action is reachable only through the Composition Boundary — is the Executor's responsibility.

#### 15.3.6 Complete execution lineage

AEAP Core defines the Receipt (§7) and the OPTIONAL `execution-result` lineage-closure callback (§7.5). The intent-to-receipt portion is a protocol guarantee: for every authorization decision the RDP makes, a complete, signed, verifiable record exists.

AEAP Core does NOT guarantee that the observed outcome of the authorized action is reported back. The callback shape is defined; cooperative Executor participation is required to close the trail. An Executor that performs the action without reporting the outcome does not produce a non-conformant receipt, but the Protected Party's execution lineage is incomplete.

### 15.4 Receipt validity over time

The semantic question of what a receipt means over time is addressed as follows:

An AEAP ALLOW receipt attests that at the moment of RDP evaluation, the specific Intent satisfied the specific Policy against the specific Evidence Signals available at that moment, and the authorization is valid for the duration specified by `exp`.

A Policy change that occurs after an ALLOW does NOT retroactively invalidate previously issued receipts unless the Protected Party's Policy explicitly scopes authorization validity to fresher Evidence than the receipt's `exp` interval permits. Protected Parties requiring the latter MUST use the short-lived receipt pattern described in §15.3.2.

A receipt whose `exp` has passed MUST NOT be used to initiate a new Consequential Action (§8.3). A verifier that encounters a receipt past its `exp` MUST return `VERIFIED_EXPIRED` rather than `VERIFIED` (§8.4).

### 15.5 Signature verification versus trust

A successful signature verification establishes that the receipt was signed by a party in possession of the corresponding key material at the moment of signing. It does NOT, by itself, establish that the key is currently trustworthy for reliance.

A verifier MUST consult the Issuer's published JWKS for current key status (§8.4) and MUST return `COMPROMISED_KEY` rather than `VERIFIED` for receipts signed by a key whose current JWKS entry carries a compromised status, regardless of whether the signature mathematically verifies.

Relying parties MUST treat signature verification as a necessary but not sufficient condition for reliance. The full chain — signature verification, key-status consultation, `exp` check, and `intent_id` binding to the proposed action — is the complete reliance check.

---

## Appendix A: Conformance Test Vectors

This Appendix defines the normative conformance test vectors against which candidate AEAP implementations are validated. An implementation is AEAP-conformant at a declared Conformance Level and declared set of Profiles when it passes all test vectors applicable to that Level and those Profiles.

The test vectors are published as a companion artifact under Apache 2.0 at the URL declared in the Author Contacts (Appendix D). This Appendix defines the categories of test vectors and the structural requirements they must meet; the vectors themselves are maintained as machine-readable test fixtures.

### A.1 Test vector categories

**Intent validation vectors (§3).** For each required and optional Intent field, test vectors cover: well-formed Intents, missing required fields, type violations, non-atomic bundling attempts, canonicalization round-trips through RFC 8785 JCS, and supersession-chain structures.

**Policy evaluation vectors (§4).** For a reference Policy expressed in a documented DSL, test vectors cover: ALLOW/ESCALATE/BLOCK decision paths, Policy composition under the mandatory algebra, Policy silence (POLICY_SILENT_ON_ACTION_TYPE and POLICY_INDETERMINATE), determinism (same inputs → same Verdict), Policy version transitions, and Policy digest reproducibility.

**Evidence Signal vectors (§5).** Test vectors cover: value/provenance/attestation decomposition, each of the five signal categories, carried/collected/observed timings, freshness evaluation against Policy-declared tolerances, and Policy-declared source preferences across multiple timings.

**Verdict output vectors (§6).** Test vectors cover: each canonical reason code, Core-vs-Extended behavior under ESCALATE conditions (including `ESCALATE_UNSUPPORTED`), reason-code preservation across composition, and the strict one-intent-id-one-evaluation invariant under replay attempts.

**Receipt format vectors (§7, §8).** Test vectors cover v2 (Ed25519) receipt generation and verification for base-protocol conformance: signature validation; `exp` enforcement; `shadow` claim semantics; supersession-chain reconstruction; evidence-envelope-ref dereferencing. Each receipt vector pairs a known-good signing input with the expected canonicalized bytes and expected signature output; implementations that produce different bytes or different signatures are non-conformant. v1 (HMAC) development-profile vectors MAY be provided separately for reference-implementation and local-development testing, but MUST NOT contribute to any AEAP base-protocol conformance determination (per §8.1).

**Interop Profile vectors (§11, §12, §13).** For each supported Profile, test vectors cover the translation surface between the external authorization system's artifact (HAP mandate ticket, ID-JAG grant, AOA token) and AEAP's Evidence Signal three-component structure.

### A.2 Pass criteria

A candidate implementation is AEAP-conformant at a declared Conformance Level (Core, Extended, or Full) and declared set of Profiles when:

1. The implementation produces identical outputs (Verdicts, receipts, error codes) to the expected outputs declared by each applicable test vector
2. The implementation rejects every malformed input vector with the expected error code
3. For receipt-signing vectors, the signatures the implementation produces verify against the expected signatures using the vector's declared verification algorithm and key material
4. The implementation's conformance report explicitly names the version of the test vector suite it was tested against

Implementations SHOULD publish their conformance reports alongside their release artifacts. Full-conformant implementations **MUST** publish such reports (§1.4).

### A.3 Interoperability requirement

The core interoperability guarantee that Appendix A enforces is:

> **Two implementations conformant at the same Conformance Level, evaluating the same canonical Intent against a Policy identified by the same Policy digest, with the same canonical set of Evidence Signal values, MUST produce the same Verdict, the same reason code, and (for the signing-input portion) the same canonicalized bytes.**

Signatures themselves may differ (different keys produce different signatures over the same input), but the signing input bytes — the RFC 8785-canonicalized claims object encoded as base64url — **MUST** be identical. A test vector suite that cannot distinguish conformant implementations from non-conformant ones on this basis is itself non-conformant and MUST be revised.

---

## Appendix B: Change Log

### 0.9.2-DRAFT (October 2026)

Additive: adds §15 Security Model and Responsibility Boundaries. §15 codifies the responsibility boundary between AEAP Core, the RDP Implementation, and the Protected Party / Executor; names the security properties AEAP Core guarantees; explicitly names the properties AEAP Core does NOT guarantee (exactly-once execution, pre-expiration revocation of individual receipts, policy correctness, Evidence Signal truthfulness, enforcement exhaustiveness at the Composition Boundary in a given deployment, complete execution lineage without cooperative Executor participation); clarifies receipt validity over time; and clarifies signature verification versus trust. No change to any previously-defined element of AEAP Core; no change to receipt format, signing, verification, or canonicalization rules; conformant 0.9.1-DRAFT implementations remain conformant to 0.9.2-DRAFT.

### 0.9.1-DRAFT (October 2026)

Initial public draft of AEAP. Published for review and comment ahead of v1.0, which will be gated on Appendix A conformance test vectors passing identically against the reference implementations. Feedback welcomed per Appendix D.

---

## Appendix C: Glossary

The following terms are defined in the main text and listed here for cross-reference. Each entry points to its primary definition section.

- **Actor** — §1.2
- **AEAP Default Composition Algebra** — §4.5.1
- **Attestation** (Evidence Signal component) — §5.1.1
- **Composition Boundary** — §1.2, §2.4, §9
- **Consequential Action** — §1.2
- **Core / Extended / Full conformance** — §1.4
- **Enforce Mode** — §10.2
- **Escalation ID** — §6.3.1
- **Evidence Envelope** — §7.4.2
- **Evidence Signal** — §1.2, §5
- **Execution Receipt** — §1.2, §7
- **Execution Surface** — §1.2
- **Intent** — §1.2, §3
- **Intent Digest** — §3.4.1
- **Issuer** — §1.2, §8
- **JWKS Key Lifecycle States** (`current`, `rotating-out`, `retired`, `compromised`) — §8.2.1
- **Policy** — §1.2, §4
- **Policy Author** — §1.3
- **Policy Digest** — §4.2.2
- **Policy Version** — §4.2.1
- **Profile (AEAP Profile)** — §1.5
- **Protected Party** — §1.2
- **Provenance** (Evidence Signal component) — §5.1.1
- **RDP Implementation** — §1.3
- **Receipt Signature Profile** — §8.1
- **Receipt Verifier** — §1.3
- **Runtime Decision Point (RDP)** — §1.2, §2
- **Shadow Mode** — §10.1
- **Supersession** — §3.5.2
- **Value** (Evidence Signal component) — §5.1.1
- **Verdict** — §1.2, §6
- **Verifier Result Codes** (`VERIFIED`, `VERIFIED_EXPIRED`, `COMPROMISED_KEY`, `UNVERIFIABLE`, `INVALID`, `INTENT_MISMATCH`) — §8.4.1

---

## Appendix D: Author Contacts

**Specification maintainer:** Aiyoplane, Inc.
**Primary contact:** info@aiyoplane.com
**Public discussion:** https://aiyoplane.com/trust (links to current public discussion forum)
**Specification source:** https://github.com/aiyoplane/aeap (public repository, Apache 2.0 for test vectors, CC BY 4.0 for specification prose)
**Conformance test vector artifact:** https://github.com/aiyoplane/aeap-test-vectors (companion artifact)

Comments, errata, and proposed changes to this specification MAY be submitted via the specification source repository's issue tracker or by email to the primary contact. Substantive change proposals SHOULD include a rationale, the specific sections affected, and (where applicable) a draft of the proposed replacement language.

The specification maintainer commits to responding to substantive change proposals within 30 days of receipt, either with an acceptance into the current working draft, a request for additional detail, or a reasoned decline with explanation.

---

*End of AEAP v0.9.2-DRAFT. The specification is complete through §15 and Appendices A-D. Conformance test vectors are maintained as a companion artifact (Appendix D).*
