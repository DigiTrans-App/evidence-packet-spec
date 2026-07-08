---
layout: default
title: Evidence Packet Open Standard v1.0
---

# Evidence Packet Open Standard

**The atomic unit of trust for enterprise AI.**

[Full Specification](#specification) | [Python SDK](https://pypi.org/project/digitrust-sdk/) | [Live Verification](https://controlplane.digitranshq.com/api/trust-center/verify-page)

---

## What is an Evidence Packet?

An Evidence Packet is a portable, cryptographically verifiable proof object that records the complete trust chain for a governed AI workflow execution. It captures what happened, who authorized it, what controls were applied, what model was used, and what the outcome was — without exposing raw sensitive data.

## Quick Start

```bash
pip install digitrust-sdk
```

```python
from digitrust_sdk import DigiTrustClient

client = DigiTrustClient(endpoint="https://controlplane.digitranshq.com")
packet = client.govern(workflow="my-workflow", model="claude-3", payload={"data": "..."})
```

---

## Specification

The full specification is available in the repository: [EVIDENCE-PACKET-SPEC-v1.0.md](https://github.com/DigiTrans-App/evidence-packet-spec/blob/main/spec/EVIDENCE-PACKET-SPEC-v1.0.md)

---

## License

Specification text: Apache 2.0 | Reference implementations: MIT
