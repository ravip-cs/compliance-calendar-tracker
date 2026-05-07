# Security Policy — Compliance Calendar & Tracker

**Project:** Compliance Calendar & Tracker  
**Stack:** Spring Boot (Java) · React (Frontend) · Flask (AI Service / Python)  
**Security Reviewer:** [Your Name]  
**Last Updated:** May 2026  
**Version:** 1.0.0

---

## Table of Contents

1. [Supported Versions](#supported-versions)
2. [Reporting a Vulnerability](#reporting-a-vulnerability)
3. [Security Architecture Overview](#security-architecture-overview)
4. [Attack Surface Map](#attack-surface-map)
5. [Authentication & Authorization](#authentication--authorization)
6. [API Security](#api-security)
7. [AI Service Security](#ai-service-security)
8. [Input Validation Policy](#input-validation-policy)
9. [Dependency Management](#dependency-management)
10. [Secrets Management](#secrets-management)
11. [Security Testing Checklist](#security-testing-checklist)
12. [Contact](#contact)

---

## Supported Versions

| Version | Supported |
|---------|-----------|
| main (latest) | ✅ Active security review |
| < main | ❌ Not supported |

---

## Reporting a Vulnerability

**Do NOT open a public GitHub Issue for security vulnerabilities.**

To report a vulnerability:

1. Email the security reviewer directly (see [Contact](#contact))
2. Include: affected component, reproduction steps, potential impact
3. You will receive an acknowledgment within **48 hours**
4. A fix target will be communicated within **5 business days**
5. Credit will be given in release notes (if desired)

Do not exploit vulnerabilities against production data.

---

## Security Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                     React Frontend                        │
│   (JWT stored in memory / HttpOnly Cookie)                │
└────────────────────────┬─────────────────────────────────┘
                         │ HTTPS
                         ▼
┌──────────────────────────────────────────────────────────┐
│              Spring Boot Backend (Port 8080)              │
│   JWT Auth Filter → Controllers → JPA/Hibernate → DB     │
└────────────────────────┬─────────────────────────────────┘
                         │ Internal HTTP (no auth)
                         ▼
┌──────────────────────────────────────────────────────────┐
│           Flask AI Service (Port 5000)                    │
│   /describe · /recommend · /report · /health             │
│   RAG Pipeline → LLM API                                 │
└──────────────────────────────────────────────────────────┘
```

**Critical risk:** Flask AI service runs with `debug=True` and has no authentication layer protecting it from direct external access.

---

## Attack Surface Map

| Surface | Component | Risk Level |
|---------|-----------|------------|
| JWT Authentication | Spring Boot | 🔴 High |
| Flask AI Endpoints | Python/Flask | 🔴 High |
| REST API (CRUD) | Spring Boot | 🟠 Medium |
| React Input Fields | Frontend | 🟠 Medium |
| `.env` File in Repo | Root directory | 🔴 Critical |
| RAG Pipeline Inputs | Flask/RAG | 🟠 Medium |
| Dependency Chain | Java + Python | 🟡 Low-Medium |

---

## Authentication & Authorization

### Current Implementation
- JWT tokens used for Spring Boot API authentication
- Token secret and expiry must be managed via environment variables

### Required Controls
- [ ] JWT secret must be minimum 256-bit random value
- [ ] Tokens must expire within 15–60 minutes; refresh tokens scoped separately
- [ ] Implement `Authorization: Bearer <token>` validation on **all** protected routes
- [ ] Reject tokens with `alg: none` (algorithm confusion attack)
- [ ] Flask AI endpoints must validate a shared internal secret header (`X-Internal-Key`)
- [ ] Failed login attempts must be rate-limited (max 5/min per IP)

### Forbidden
- Storing JWT in `localStorage` (XSS-accessible)
- Hardcoding JWT secret in source code
- Using symmetric algorithms without rotation policy

---

## API Security

### Spring Boot REST Endpoints
All endpoints must enforce:
- Input length limits (reject payloads > 10KB for standard fields)
- Parameterized queries only — no raw SQL string concatenation
- `Content-Type: application/json` validation
- CORS restricted to known frontend origin (not `*`)
- HTTP method enforcement (GET endpoints must not accept POST)

### Flask AI Endpoints

| Route | Method | Auth Required | Risk |
|-------|--------|---------------|------|
| `/` | GET | None | Low |
| `/health` | GET | None | Low |
| `/describe` | POST | ⚠️ Missing | High |
| `/recommend` | POST | ⚠️ Missing | High |
| `/report` | POST | ⚠️ Missing | High |

**Action required:** All POST endpoints must require `X-Internal-Key` header.

---

## AI Service Security

### Prompt Injection Risk
The Flask RAG pipeline accepts user-controlled text that is sent to an LLM. Without sanitization, attackers can inject instructions to:
- Override system prompts
- Exfiltrate context data
- Generate policy-violating content

### Required Mitigations
- Sanitize user input before appending to LLM prompts (strip `Ignore previous instructions`, role override patterns)
- Set maximum input length: 2000 characters
- Wrap LLM calls in try/except; never return raw LLM errors to client
- Log all AI inputs for audit trail (exclude PII)
- Implement output filtering before returning AI response to frontend

---

## Input Validation Policy

All user-submitted fields must be validated server-side:

| Field Type | Validation Rule |
|------------|-----------------|
| Dates | ISO 8601 format only; reject relative strings |
| Text fields | Max 500 chars; strip HTML tags |
| Email | RFC 5322 regex + domain check |
| Task names | Alphanumeric + limited punctuation; reject `<`, `>`, `"`, `'`, `;` |
| File uploads | Whitelist MIME types; max 5MB; rename on server |
| AI prompt input | Max 2000 chars; reject known injection patterns |

---

## Dependency Management

### Java (pom.xml)
- Run `mvn dependency:check` before each release
- Use OWASP Dependency-Check Maven Plugin
- No SNAPSHOT dependencies in production builds

### Python (requirements.txt)
- Pin all versions (e.g., `flask==3.0.3` not `flask>=3`)
- Run `pip audit` or `safety check` in CI pipeline
- Review Flask, LangChain, and vector DB libraries quarterly

---

## Secrets Management

**CRITICAL FINDING:** `.env` file is committed to the repository root and is publicly visible.

### Immediate Actions Required
1. Rotate ALL keys currently in `.env` immediately
2. Add `.env` to `.gitignore` — verify with `git rm --cached .env`
3. Use GitHub Secrets for CI/CD; environment variables for local dev
4. Never commit API keys, JWT secrets, or DB credentials

### `.env` Template (safe to commit)
```env
# Copy this to .env and fill in values. Never commit .env
JWT_SECRET=CHANGE_ME
DB_URL=CHANGE_ME
OPENAI_API_KEY=CHANGE_ME
INTERNAL_API_KEY=CHANGE_ME
```

---

## Security Testing Checklist

- [ ] SQL Injection: all database-bound inputs tested
- [ ] XSS: all React input fields and display components tested
- [ ] JWT manipulation: algorithm confusion, expired token replay
- [ ] API authorization: IDOR checks on task/event endpoints
- [ ] Flask endpoint direct access: all AI routes tested without Spring Boot proxy
- [ ] Prompt injection: all AI input fields tested
- [ ] Rate limiting: login endpoint, AI endpoint
- [ ] `.env` secrets: verified not accessible via public URL
- [ ] CORS policy: verified only allowed origins accepted
- [ ] Dependency scan: zero critical CVEs

---

## Contact

| Role | Name | Contact |
|------|------|---------|
| Security Reviewer | [Your Name] | [your-email@domain.com] |
| Project Lead | [Team Lead Name] | [lead@domain.com] |

---

*This document is maintained as part of the capstone project security review cycle. Update after each sprint's security testing session.*
