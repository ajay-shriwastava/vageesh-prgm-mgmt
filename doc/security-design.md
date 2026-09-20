can# Vageesh — Security & MCP Client Design

## Status: In Progress
Last updated: 2026-09-02

---

## Scope of This Document

Three parallel workstreams were identified. This document covers all three with their current status.

| Workstream | Status |
|---|---|
| 1. Okta JWT Validation | Decision done — ready to implement |
| 2. MCP Client | Design in progress — open questions remain |
| 3. MCP Server | Not started — discussion deferred |

---

## Workstream 1 — Okta JWT Validation (No MCP)

### Decision
Replace the JWT stub in `app/dependencies.py` with real Okta JWT validation.
No MCP involved. Standard OAuth 2.0 / OIDC.

### Setup
- Free Okta developer account (developer.okta.com)
- OIDC Web Application registered in Okta Admin Console
- Required env vars: `OKTA_ISSUER`, `OKTA_CLIENT_ID`, `OKTA_AUDIENCE`

### What Changes
```
TODAY (stub)
  get_current_user() accepts any bearer token
  returns {"id": "stub-user"}

AFTER
  get_current_user() validates JWT signature
  against Okta JWKS endpoint:
    https://{domain}.okta.com/oauth2/default/v1/keys
  checks: expiry, issuer, audience
  returns real claims: sub, email, groups
```

### What Stays the Same
- `Authorization: Bearer <token>` header — unchanged
- `localStorage["symphony_token"]` in frontend — unchanged
- JWT format — unchanged

### Files to Change
- `app/dependencies.py` — replace stub with JWKS validation
- `.env.example` — add Okta vars
- `requirements.txt` — add `python-jose[cryptography]`, `httpx`

---




## References
- Okta developer account: https://developer.okta.com
- GCP Secret Manager: https://cloud.google.com/secret-manager
- MCP protocol: https://modelcontextprotocol.io
- GCP Workload Identity Federation: https://cloud.google.com/iam/docs/workload-identity-federation