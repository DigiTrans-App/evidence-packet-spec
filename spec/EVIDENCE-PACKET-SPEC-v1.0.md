# Evidence Packet Open Standard

## Specification for Trust Infrastructure for Enterprise AI

| Field | Value |
|-------|-------|
| **Status** | Public Draft |
| **Version** | 1.0.0 |
| **Date** | 2026-07-07 |
| **Authors** | DigiTrans Trust Infrastructure Working Group |
| **License** | Apache 2.0 (specification text), MIT (reference implementations) |
| **URI** | `https://evidencepacket.org/spec/v1.0` |
| **Schema** | `https://evidencepacket.org/schema/v1.0.0/evidence-packet.json` |

---

## Preface

This document defines the **Evidence Packet** — an open, portable, cryptographically verifiable proof object for governed AI workflow executions. Any organization, platform, or tool MAY implement this specification to produce or consume Evidence Packets without royalty or license restriction.

### Conformance Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

### Design Principles

1. **Verify without trust** — Any party can verify packet integrity without trusting the issuer
2. **Prove without exposing** — Contains proof of governance, never raw sensitive data
3. **Portable across clouds** — No vendor lock-in; works with any AI provider or governance system
4. **Compose across boundaries** — Packets can reference other packets across organizations
5. **Fail closed** — Incomplete or invalid packets indicate governance failure, not success

---

## 1. Abstract

An Evidence Packet is the atomic unit of trust for enterprise AI. It records the complete governance chain for a single AI workflow execution: what data was classified, what policy decided, what protections were applied, who approved it, which model was invoked through what governed path, and what the outcome was — all without exposing raw sensitive data.

Evidence Packets enable:
- **Provable compliance** without self-attestation
- **Independent verification** without privileged access
- **Portable trust** across organizational boundaries
- **Continuous monitoring** through packet stream analysis
- **Audit-ready evidence** that satisfies regulatory requirements

---

## 2. Terminology

| Term | Definition |
|------|-----------|
| Evidence Packet | A self-contained, versioned, hash-sealed proof object for one governed AI workflow execution |
| Trust Control Loop | Six-stage deterministic process: Classify → Decide → Transform → Approve → Route → Prove |
| Packet Hash | SHA-256 of the JCS-canonicalized packet with `packet_hash` field set to empty string |
| Hash Chain | Linked sequence where each event references the previous event's hash |
| Finalization | Sealing a packet: computing final hash, setting status to `complete`, making content immutable |
| Approved Model Path | A reviewed, authorized AI runtime route for specific data classification and purpose |
| Sensitive Data Unit (SDU) | A reference (hash or token) to protected data — never the data itself |
| Domain | One of seven logical sections within a packet representing a facet of the trust chain |
| Conformance Level | Degree of specification compliance: Basic, Standard, or Full |

---

## 3. Packet Envelope

### 3.1 Required Envelope Fields

Every conforming Evidence Packet MUST include:

```json
{
  "spec_version": "1.0.0",
  "packet_id": "epkt_<globally-unique-id>",
  "packet_type": "ai_workflow",
  "status": "complete",
  "environment": "production",
  "tenant_id": "<tenant-identifier>",
  "created_at": "<ISO-8601-timestamp>",
  "finalized_at": "<ISO-8601-timestamp>",
  "issuer": {
    "platform_id": "<issuing-platform>",
    "platform_version": "<version>",
    "organization_id": "<org-id>"
  }
}
```

### 3.2 Packet ID Format

- Prefix: `epkt_`
- MUST be globally unique
- RECOMMENDED format: `epkt_<tenant-short>_<timestamp-hex>_<random-hex>`
- MUST be assigned at creation and MUST NOT change

### 3.3 Spec Version

Follows semantic versioning (`MAJOR.MINOR.PATCH`):
- MAJOR: breaking schema changes requiring new parsers
- MINOR: additive fields, backward-compatible
- PATCH: clarifications without schema changes

### 3.4 Packet Types

| Type | Description |
|------|-------------|
| `ai_workflow` | Standard governed AI workflow execution |
| `multi_step` | Composite workflow referencing child packets |
| `verification_report` | Independent verification result |
| `continuity_event` | Lifecycle event (retention, legal hold) |

### 3.5 Status Lifecycle

```
draft → in_review → complete → exported
              ↘ voided
```

A packet in `complete` status MUST have a valid `packet_hash`. A packet in `draft` status MUST NOT be presented as governance evidence.

---

## 4. Seven Domains

The packet body consists of seven evidence domains. Each domain records one facet of the trust chain.

### 4.1 Workflow Domain (REQUIRED)

Records WHAT was executed.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | string | MUST | Unique workflow identifier |
| `workflow_name` | string | MUST | Human-readable name |
| `workflow_version` | string | MUST | Semantic version |
| `business_owner_ref` | string | MUST | Team or person accountable |
| `business_purpose` | string | MUST | Why this AI workflow exists |
| `risk_tier` | enum | MUST | `low`, `medium`, `high`, `critical` |
| `regulatory_scope` | array | SHOULD | Applicable regulations |
| `data_source_refs` | array | SHOULD | References to input data sources |

### 4.2 Actor Domain (REQUIRED)

Records WHO initiated it.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `actor_id` | string | MUST | Unique actor identifier |
| `actor_type` | enum | MUST | `human`, `service`, `pipeline`, `scheduled` |
| `actor_role` | string | MUST | Role at time of invocation |
| `auth_method` | string | SHOULD | Authentication mechanism used |
| `permissions_verified` | array | SHOULD | Permissions checked |

### 4.3 Data Context Domain (REQUIRED)

Records WHAT data was assessed — without the data itself.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `classification_summary` | string | MUST | Human-readable classification |
| `sensitive_data_types` | array | MUST | Categories found (PII, PHI, etc.) |
| `max_sensitivity` | enum | MUST | `none`, `internal`, `confidential`, `restricted`, `regulated` |
| `raw_sensitive_values_present` | boolean | MUST | **MUST be `false`** |
| `source_hashes` | array | SHOULD | SHA-256 of input sources |
| `data_residency` | string | SHOULD | Where data resides |

**INVARIANT:** `raw_sensitive_values_present` MUST be `false`. Any implementation producing `true` generates an INVALID packet. This is the foundational privacy guarantee.

### 4.4 Protection Domain (REQUIRED when sensitive data present)

Records HOW data was protected before AI processing.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mode` | enum | MUST | `tokenization`, `masking`, `redaction`, `encryption` |
| `token_refs` | array | MUST* | Token references with field paths |
| `detokenization.approval_required` | boolean | MUST | Whether reversal needs approval |
| `raw_sensitive_data_stored` | boolean | MUST | **MUST be `false`** |

*Required when `mode` is `tokenization`.

Each token reference:
```json
{
  "field_path": "$.patient.name",
  "token_ref": "dtok_<hex>_<hex>_<hex>",
  "source_hash": "sha256:<64-hex-chars>"
}
```

### 4.5 Policy Domain (REQUIRED)

Records WHAT decision was made and WHY.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `policy_id` | string | MUST | Which policy was evaluated |
| `decision` | enum | MUST | `allow`, `deny`, `transform`, `escalate`, `conditional_allow` |
| `decision_reason` | string | MUST | Human-readable rationale |
| `risk_score` | number | SHOULD | Computed risk (0-100) |
| `control_mappings` | array | SHOULD | Mapped framework controls |

### 4.6 Human Review Domain (REQUIRED when review is triggered)

Records WHETHER human oversight occurred and the outcome.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `required` | boolean | MUST | Whether review was triggered |
| `status` | enum | MUST* | `approved`, `rejected`, `pending`, `exception_approved` |
| `approver_id` | string | MUST* | Who approved (when approved) |
| `approval_timestamp` | string | MUST* | When (ISO-8601) |
| `rationale_ref` | string | SHOULD | Reference to approval record |

*Required when `required` is `true`.

### 4.7 AI Path Domain (REQUIRED)

Records WHICH model was invoked through WHAT governed route.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider` | string | MUST | AI provider (e.g., `aws-bedrock`) |
| `model_id` | string | MUST | Specific model identifier |
| `route_id` | string | MUST | Approved model path reference |
| `region` | string | MUST | Deployment region |
| `guardrails_applied` | array | SHOULD | Active guardrails |
| `routing_decision` | enum | MUST | `approved`, `blocked`, `exception` |
| `invocation_metadata` | object | SHOULD | Tokens, latency, stop reason |
| `guardrail_trace` | object | SHOULD | Guardrail intervention details |

---

## 5. Evidence Chain

### 5.1 Structure

```json
{
  "evidence_chain": {
    "ledger_id": "<tenant-ledger>",
    "packet_hash": "sha256:<64-hex>",
    "previous_packet_hash": "sha256:<64-hex>",
    "hash_algorithm": "SHA-256",
    "canonicalization": "JCS",
    "timeline": [ ... ],
    "retention_policy": "<policy-ref>",
    "storage_uri": "<storage-location>"
  }
}
```

### 5.2 Hash Computation

The `packet_hash` MUST be computed as follows:

1. Set `evidence_chain.packet_hash` to empty string `""`
2. Serialize the packet using JSON Canonicalization Scheme (JCS) per [RFC 8785](https://datatracker.ietf.org/doc/html/rfc8785)
3. Compute SHA-256 of the UTF-8 encoded canonical form
4. Format as `sha256:<lowercase-hex-digest>`

### 5.3 Timeline Events

Each timeline event forms a hash chain:

```json
{
  "sequence": 1,
  "event_type": "classification_complete",
  "timestamp": "2026-07-07T12:00:01Z",
  "event_hash": "sha256:<hash-of-this-event>",
  "previous_event_hash": null,
  "actor_ref": "system:classifier",
  "event_ref": "evt_001"
}
```

**Chain rule:** For event at sequence N > 1, `previous_event_hash` MUST equal `event_hash` of event at sequence N-1.

### 5.4 Verification Algorithm

To verify a packet:

1. Verify `packet_hash` using the computation in §5.2
2. For each timeline event at sequence N > 1, verify `previous_event_hash` equals prior event's `event_hash`
3. Scan all string values for raw sensitive data patterns (social-security-no., credit card, private keys)
4. Verify `raw_sensitive_values_present` is `false`
5. Verify all required domains are present per packet type

If ALL checks pass → status: `verified`
If hash or chain fails → status: `failed`
If non-critical checks fail → status: `partial`

---

## 6. Privacy Invariants

These invariants MUST hold for every valid Evidence Packet:

| # | Invariant | Enforcement |
|---|-----------|-------------|
| P1 | No raw sensitive values in packet content | Pattern scan + boolean flag |
| P2 | No raw prompts stored in packet | `ai_path.raw_prompt_stored` MUST be `false` |
| P3 | No raw model responses stored in packet | `ai_path.raw_response_stored` MUST be `false` |
| P4 | Token references are opaque | Format: `dtok_<hex>` — no reversible encoding |
| P5 | Detokenization requires explicit approval | `protection.detokenization.approval_required` MUST be `true` |
| P6 | Source data referenced by hash only | `data_context.source_hashes` — never inline |

---

## 7. Conformance Levels

### 7.1 Level 1: Basic

Minimum viable Evidence Packet. MUST include:
- Packet envelope (§3.1)
- Workflow domain (§4.1)
- Data context domain with `raw_sensitive_values_present: false` (§4.3)
- Policy domain with decision (§4.5)
- Evidence chain with valid `packet_hash` (§5)

### 7.2 Level 2: Standard

Full governance proof. MUST include everything in Level 1 plus:
- Actor domain (§4.2)
- Protection domain (§4.4) when sensitive data present
- AI path domain (§4.7)
- Human review domain (§4.6) when review was triggered
- Timeline with hash chain (§5.3)
- All privacy invariants (§6)

### 7.3 Level 3: Full

Complete audit-ready packet. MUST include everything in Level 2 plus:
- All SHOULD fields populated
- Framework control mappings in policy domain
- Guardrail trace in AI path domain
- Output domain with hash and disposition
- Retention policy and storage references
- Previous packet hash (ledger chaining)

---

## 8. Extensions

Implementations MAY add extension domains under the `extensions` key:

```json
{
  "extensions": {
    "x-bedrock-trace": { ... },
    "x-compliance-mapping": { ... },
    "x-custom-controls": { ... }
  }
}
```

Extension keys MUST be prefixed with `x-`. Core spec fields MUST NOT be placed in extensions.

---

## 9. Remote Verification Protocol

### 9.1 Overview

Any party holding an exported Evidence Packet can verify it independently. No API keys, system access, or trust relationship required.

### 9.2 Verification Endpoint

Conforming platforms SHOULD expose:

```
POST /api/trust-center/verify
Content-Type: application/json

{ "packet": { ... } }
```

Response:
```json
{
  "verification_id": "vrfy_<id>",
  "overall_status": "verified",
  "integrity_valid": true,
  "raw_values_absent": true,
  "completeness_score": 100.0,
  "assertions": [ ... ],
  "verifier_statement": "..."
}
```

### 9.3 Twelve Verification Checks

| # | Check | Pass Condition |
|---|-------|---------------|
| 1 | Schema completeness | All required envelope fields present |
| 2 | Identity correlation | Tenant and trace IDs present |
| 3 | Packet hash integrity | Recomputed hash matches claimed hash |
| 4 | Event hash chain | All previous_event_hash values valid |
| 5 | Raw value absence | No social-security-no., credit card, key patterns found |
| 6 | Tokenization proof | Token refs valid, raw data not stored |
| 7 | Policy decision | Governance decision recorded |
| 8 | Data classification | Sensitivity assessment present |
| 9 | Model path authorization | Route approved, not blocked |
| 10 | Human review | Approval recorded when required |
| 11 | Output accountability | Output hash and disposition present |
| 12 | Retention policy | Lifecycle management specified |

---

## 10. Non-Replacement Rule

Evidence Packets supplement existing security and compliance controls. They do NOT replace:
- Identity and access management (IAM)
- Multi-factor authentication (MFA)
- SOC 2, ISO 27001, or other certifications
- Privacy impact assessments
- Legal counsel or regulatory filings
- Customer security obligations
- Network security, encryption at rest/transit
- Incident response procedures

This rule MUST be stated in every verification report and public trust posture endpoint.

---

## 11. API Contract

### 11.1 Create Packet

```
POST /api/evidence-packets
Authorization: Bearer <token>
```

### 11.2 Append Event

```
POST /api/evidence-packets/{packet_id}/events
Authorization: Bearer <token>
```

### 11.3 Finalize Packet

```
POST /api/evidence-packets/{packet_id}/finalize
Authorization: Bearer <token>
```

### 11.4 Export Packet

```
GET /api/evidence-packets/{packet_id}/export?format=json
Authorization: Bearer <token>
```

### 11.5 Verify Packet (Public)

```
POST /api/trust-center/verify
Content-Type: application/json
(No authentication required)
```

---

## 12. Security Considerations

- Packet hashes provide tamper evidence, not tamper prevention
- Implementations SHOULD store packets in append-only, versioned storage (e.g., S3 with versioning)
- Implementations SHOULD use KMS-managed keys for encryption at rest
- The verification endpoint MUST NOT require authentication (it verifies public proof)
- Implementations MUST rate-limit the verification endpoint to prevent abuse
- Token references MUST NOT be reversible without explicit approval flow

---

## 13. IANA Considerations

This specification defines the following media type:

- `application/evidence-packet+json` — An Evidence Packet in JSON format

---

## Appendix A: Complete Example

See `examples/complete-evidence-packet-v1.0.json` in this repository.

## Appendix B: JSON Schema

See `schemas/evidence-packet-v1.0.schema.json` for the normative machine-readable schema.

## Appendix C: Reference Implementations

- **Python SDK:** `pip install digitrust-sdk` (reference implementation)
- **Verification CLI:** `digitrust verify <packet.json>`
- **GitHub Action:** `digitrust/verify-evidence-packet@v1`

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-07-07 | Initial public draft |

---

*Copyright 2026 DigiTrans, Inc. Licensed under Apache 2.0.*
*This is an open specification. Contributions welcome at https://github.com/evidence-packet/spec.*
