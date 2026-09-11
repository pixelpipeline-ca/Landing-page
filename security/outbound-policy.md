# Agent Security Gateway — Outbound Policy

## Purpose

The outbound policy validates responses produced by the Business Agent before they are returned to the Security Agent and ultimately to the user.

It provides a second security boundary:

User → Security Agent → Inbound Policy → Business Agent → Outbound Policy → Security Agent → User

The Business Agent therefore never communicates directly with the user.

---

## Security Objective

The outbound layer prevents unsafe downstream content from being propagated back to the user.

It protects against:

- Sensitive or restricted information leakage
- Credentials, secrets, or internal security information
- Internal system or agent instructions
- Prompt/instruction leakage
- Content attempting to manipulate or override another agent
- Unexpected downstream responses

The Business Agent response is treated as untrusted data and must pass outbound validation.

---

## Outbound Classification

The Business Agent `Result` is passed to an outbound `Classify` node.

### SAFE

Content is normal business or informational content that is safe to return.

Examples:

- General knowledge
- Approved business information
- Normal retrieved records
- Non-sensitive operational responses

Flow:

Business Agent → Classify → SAFE → Respond to Agent

The original Business Agent `Result` is returned.

---

### BLOCK

Content contains information that must not be returned.

Examples:

- Credentials or secrets
- Sensitive/restricted information
- Internal system instructions
- Security configuration
- Hidden agent instructions
- Instructions attempting to manipulate another agent or security control

Flow:

Business Agent → Classify → BLOCK → Security Response

Returned contract:

decision = BLOCK
reason   = Outbound response blocked by security policy
response = The downstream response was blocked by security policy.

The original Business Agent response is NOT returned.

---

### OTHER / UNKNOWN

Any response that cannot be confidently classified as SAFE must fail safely.

Flow:

Business Agent → Classify → OTHER → Security Response

The original Business Agent response is NOT returned.

This implements fail-closed behavior for ambiguous classifications.

---

## ALLOW Path

For a normal inbound request:

User
  ↓
Security Agent
  ↓
Inbound Policy: ALLOW
  ↓
Business Agent
  ↓
Outbound-Classify-Allow
  ├── SAFE  → Return Business Agent Result
  ├── BLOCK → Block response
  └── OTHER → Block response

---

## REDACT Path

When unnecessary sensitive information is detected in the inbound request:

User + PII
  ↓
Security Agent
  ↓
Inbound Policy: REDACT
  ↓
Sanitized / normalized request
  ↓
Business Agent
  ↓
Outbound-Classify-Redact
  ├── SAFE  → Return Business Agent Result
  ├── BLOCK → Block response
  └── OTHER → Block response

The Business Agent receives only the sanitized request.

Example:

Original request:
"My name is John Smith, email john@example.com.
What is the capital of Germany?"

Business Agent receives:
"Provide the capital city of Germany."

The unnecessary PII never reaches the Business Agent.

---

## Validated Scenarios

### Normal Request

Input:

"What is the capital of Canada?"

Expected:

Inbound  = ALLOW
Outbound = SAFE
Response = Business Agent Result

Status: PASS

### PII Redaction

Input contains unnecessary name, email, or phone information plus a benign request.

Expected:

Inbound  = REDACT
PII      = removed before Business Agent
Outbound = SAFE
Response = sanitized Business Agent result

Status: PASS

---

## Security Properties

The current POC provides:

1. Inbound prompt-injection detection
2. Privilege-escalation detection
3. Risk-based blocking
4. PII minimization before downstream processing
5. Business Agent isolation
6. Outbound response classification
7. Fail-closed handling for unknown outbound classifications
8. No direct User ↔ Business Agent communication

This creates bidirectional policy enforcement around the Business Agent:

Inbound:
User → Policy → Business Agent

Outbound:
Business Agent → Policy → User

---

## POC Limitation

Outbound classification is intentionally minimal for the POC.

The current implementation performs classification and blocking rather than deterministic field-level output redaction.

A production implementation can extend this layer with:

- DLP policies
- Microsoft Purview
- Deterministic PII detection
- Secret scanning
- Response redaction
- Content safety policies
- Audit logging
- Policy telemetry
- Evaluation and regression datasets

---

## Design Principle

> No request reaches the Business Agent without inbound policy enforcement, and no Business Agent response reaches the user without outbound policy enforcement.

The Security Gateway remains the sole interaction boundary exposed to the user.