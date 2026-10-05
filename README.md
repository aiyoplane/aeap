# AEAP — Aiyo Execution Authorization Protocol

**An open protocol for authorization decisions at the boundary between autonomous intent and consequential execution.**

AEAP defines a Runtime Decision Point that evaluates a proposed action against a policy authored by the party being protected, returns a ternary verdict (ALLOW, ESCALATE, or BLOCK), and emits a cryptographically-signed, portable Execution Receipt that any downstream system can verify offline. The protocol is agnostic to the identity of the acting party, the rail on which value moves (if any), the wallet or signer involved (if any), and the specific category of consequential action being authorized.

AEAP is designed for use at any boundary where an autonomous system — a human-initiated workflow, an AI agent, or a machine-to-machine process — proposes to perform an action with real-world consequences. Payment, DevOps deployment, procurement, data write, inter-agent delegation, and other consequential-action classes are all first-class use cases.

---

## Current status

**AEAP v0.9.1-DRAFT** — initial public draft, published for review and comment ahead of v1.0.

- 📄 **Specification:** [`AEAP.md`](./AEAP.md)
- 🧪 **Conformance test vectors:** [`test-vectors/`](./test-vectors/) — pending v1.0; see directory README for status
- 📋 **Change log:** [`CHANGELOG.md`](./CHANGELOG.md)
- 🤝 **Contributing:** [`CONTRIBUTING.md`](./CONTRIBUTING.md)
- 📧 **Contact:** info@aiyoplane.com

v1.0 is gated on the Appendix A conformance test vector artifact being authored and passed identically by at least two independent reference implementations.

---

## Reference implementations

Two independent reference implementations of the RDP composition pattern demonstrate Core conformance. The two share no vendored code — they are maintained as separate codebases precisely so that passing the same test vectors from both demonstrates AEAP's interoperable implementability.

**Node:**
- [`@aiyoplane/mcp-authz`](https://github.com/aiyoplane/mcp-authz) — Runtime Decision Point composition at an MCP server
- [`@aiyoplane/verify`](https://www.npmjs.com/package/@aiyoplane/verify) — Offline Ed25519 verifier for AEAP execution receipts

**Python:**
- [`aiyoplane-mcp-authz`](https://pypi.org/project/aiyoplane-mcp-authz/) — Python port of the MCP composition reference
- [`aiyoplane-verify`](https://pypi.org/project/aiyoplane-verify/) — Python port of the receipt verifier

**CI/CD:**
- [AEAP Receipt Verifier GitHub Action](https://github.com/marketplace/actions/aeap-receipt-verifier) — Verify AEAP receipts inside GitHub Actions workflows

**macOS CLI:**
- [`brew install aiyoplane/tap/aiyo`](https://github.com/aiyoplane/homebrew-tap) — Homebrew distribution of the verifier CLI

Implementations shared by third parties are welcomed under the same conformance criteria that apply to the reference implementations. See the specification's §1.1 and §1.4 for authorship-vs-authority discussion.

---

## Quick concepts

AEAP's architecture decomposes an authorization decision into named primitives:

- **Intent** — a structured representation of exactly one proposed Consequential Action (§3)
- **Policy** — declarative rules authored by the Protected Party specifying authorization conditions (§4)
- **Evidence Signal** — information the RDP evaluates (value, provenance, attestation) in applying the Policy (§5)
- **Verdict** — the ternary output: ALLOW, ESCALATE, or BLOCK (§6)
- **Execution Receipt** — a cryptographically-signed, portable attestation of the authorization decision (§7)
- **Composition Boundary** — the specific location where the RDP is invoked (§9)

Five invariants hold across every AEAP evaluation (§2.2):

1. One authorization decision per Consequential Action
2. The decision occurs strictly between Intent and Execution
3. Policy is authored by the Protected Party
4. Authorization decisions MUST be independently verifiable
5. The RDP fails closed by default

---

## Licensing

AEAP uses a two-license split that respects the different natures of specification prose and executable test content:

- **Specification prose** (`AEAP.md` and other documentation): [Creative Commons Attribution 4.0 International (CC BY 4.0)](./LICENSE-SPECIFICATION)
- **Conformance test vectors** (in the `test-vectors/` directory): [Apache License 2.0](./test-vectors/LICENSE-TEST-VECTORS)

See each license file for the full terms.

Reference implementations in the Aiyoplane GitHub organization (npm, PyPI, GitHub Marketplace Action, Homebrew tap) are published under Apache License 2.0 in their own repositories.

---

## About

AEAP was authored and is maintained by **Aiyoplane, Inc.** The authority of AEAP as a protocol is established by adoption, interoperability demonstrated by Appendix A conformance, and the collective practice of the implementing ecosystem — not by the identity of its author. See §1.1 of the specification for the full authorship-vs-authority statement.

Learn more: [aiyoplane.com](https://aiyoplane.com)
Trust surface and public artifacts: [aiyoplane.com/trust](https://aiyoplane.com/trust)
Contact: info@aiyoplane.com

---

**Verify First. Execute Second.**
