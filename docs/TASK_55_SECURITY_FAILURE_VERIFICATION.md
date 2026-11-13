# Task 55: Comprehensive Security and Failure-Path Verification Report

**Target Deployment Environment:**
- **Frontend URL:** `https://magicpin-frontend.onrender.com`
- **Backend URL:** `https://magicpin-backend.onrender.com`
- **Active Branch:** `feature/frontend-core`
- **Verified Commit SHA:** `5143dfd74e66f2dbffbf1089369e1dd5116cdd2f`
- **Evaluation Date:** 2026-10-31
- **Final Security Verdict:** **PASS WITH WARNINGS**

---

## 1. Executive Summary

A comprehensive application security and failure-path verification was conducted on the Magicpin AI Agent application across its live production deployment on Render and its local codebase. The verification evaluated authentication robustness, CSRF defense, multi-tenant database isolation, rate limiting under concurrency, reverse proxy client IP extraction, input validation boundaries, centralized error shielding, and external AI provider failure modes.

### Verification Highlights:
- **Authentication & Sessions:** Strong Argon2id password hashing, 32-byte cryptographically secure session tokens stored exclusively as SHA-256 hashes, session revocation enforcement, and `HttpOnly`/`SameSite=Lax`/`Secure` cookie attributes verified.
- **CSRF Defense:** Session-bound HMAC-SHA256 tokens validated on all state-changing merchant endpoints using constant-time comparison.
- **Input Validation & Sanitization:** Strict Pydantic validation across all public and protected routes. Invalid UUIDs, malformed JSON, missing fields, and out-of-bounds payloads cleanly reject with HTTP 422 `VALIDATION_ERROR` without stack trace leakage.
- **Error Handling & Information Disclosure:** Centralized FastAPI exception handlers sanitize unexpected errors into generic HTTP 500 JSON payloads, preventing database error messages or internal tracebacks from reaching clients.
- **AI Failure Resilience:** External LLM timeouts (Groq/Grok) fail gracefully. Chat messages persist as `"failed"` and return retryable HTTP 502 responses; customer support inquiries persist as `"needs_review"` with `ai_status="failed"`, ensuring human-in-the-loop merchant review is never blocked.
- **Automated Verification:** 316 automated tests passed cleanly (15 security tests, 104 unit tests, 75 API tests, 37 integration tests on a dedicated disposable PostgreSQL 16 container, and 85 frontend tests with a zero-warning TypeScript/Vite production build).
- **Warnings Identified:** Two non-blocking configuration warnings were identified and documented: (1) CORS origin hostname resolution discrepancy in `render.yaml`, and (2) trusted proxy CIDR configuration for upstream Render load balancers. Both have concrete mitigation paths and do not compromise demonstration security.

---

## 2. Deployment and Audit Metadata

| Attribute | Specification | Verification Source |
|---|---|---|
| **Repository** | `https://github.com/ArjunReddyVelma/magicpin-ai-agent` | Git remote origin |
| **Active Branch** | `feature/frontend-core` | `git status` |
| **Commit SHA** | `5143dfd74e66f2dbffbf1089369e1dd5116cdd2f` | `git rev-parse HEAD` |
| **Merge Pull Request** | PR #13 (`refactor: remove WhatsApp integration and stabilize migration tests`) | GitHub merge commit |
| **Backend Runtime** | Python 3.12.8 / Uvicorn (FastAPI) | Render Free Web Service |
| **Database Engine** | PostgreSQL 16.x | Render PostgreSQL (Oregon) |
| **Frontend Runtime** | Vite 5.4.10 / React 18.3.1 SPA | Render Global CDN Rewrite |
| **Active LLM Provider** | Groq (`openai/gpt-oss-120b`) | Live Render Environment / Task 54 |
| **Audit Scope** | Non-destructive live probes, static code audit, disposable DB automated test suite | Application Security Audit |

---

## 3. Non-Destructive Live Probes & Behavioral Evidence

Live network probes were executed against the deployed backend (`https://magicpin-backend.onrender.com`) to evaluate security behavior in production:

| Probe ID | Target Endpoint | HTTP Method | Input / Condition | Expected Response | Actual Live Response | Security Assessment |
|---|---|---|---|---|---|---|
| **PRB-01** | `/v1/healthz` | `GET` | Standard liveness check | `HTTP 200 {"status":"ok"}` | `HTTP 200 {"status":"ok"}` | **PASS** — Service responsive |
| **PRB-02** | `/v1/readyz` | `GET` | Database readiness check | `HTTP 200 {"status":"ready",...}` | `HTTP 200 {"status":"ready","checks":{"database":"ok"}}` | **PASS** — DB pool healthy |
| **PRB-03** | `/v1/auth/me` | `GET` | Missing session cookie | `HTTP 401 UNAUTHORIZED` | `HTTP 401 {"error":{"code":"UNAUTHORIZED","message":"Authentication required"}}` | **PASS** — Fail-closed auth |
| **PRB-04** | `/v1/support/messages` | `GET` | Missing session cookie | `HTTP 401 UNAUTHORIZED` | `HTTP 401 {"error":{"code":"UNAUTHORIZED","message":"Authentication required"}}` | **PASS** — Protected inbox |
| **PRB-05** | `/v1/conversations` | `POST` | Missing session cookie | `HTTP 401 UNAUTHORIZED` | `HTTP 401 {"error":{"code":"UNAUTHORIZED","message":"Authentication required"}}` | **PASS** — State mutation blocked |
| **PRB-06** | `/v1/auth/login` | `POST` | Invalid dummy credentials (`nonexistent_sec_test@example.com`) | `HTTP 401 INVALID_CREDENTIALS` | `HTTP 401 {"error":{"code":"INVALID_CREDENTIALS","message":"Invalid email or password"}}` | **PASS** — No account enumeration |
| **PRB-07** | `/v1/auth/login` | `POST` | Malformed JSON payload (`{"email": "invalid`) | `HTTP 422 VALIDATION_ERROR` | `HTTP 422 {"error":{"code":"VALIDATION_ERROR","message":"body -> 10: JSON decode error"}}` | **PASS** — Clean parse error |
| **PRB-08** | `/v1/auth/register` | `POST` | Invalid email format (`not-an-email`) | `HTTP 422 VALIDATION_ERROR` | `HTTP 422 {"error":{"code":"VALIDATION_ERROR","message":"body -> email: Value error, Invalid email format"}}` | **PASS** — Schema enforcement |
| **PRB-09** | `/v1/public/support/merchants/not-a-valid-uuid` | `GET` | Malformed non-UUID path parameter | `HTTP 422 VALIDATION_ERROR` | `HTTP 422 {"error":{"code":"VALIDATION_ERROR","message":"path -> merchant_id: Input should be a valid UUID..."}}` | **PASS** — Type boundary enforced |
| **PRB-10** | `/v1/public/support/merchants/00000000-0000-0000-0000-000000000000` | `GET` | Valid UUID format, nonexistent merchant | `HTTP 404 MERCHANT_NOT_FOUND` | `HTTP 404 {"error":{"code":"MERCHANT_NOT_FOUND","message":"Merchant not found."}}` | **PASS** — Controlled 404 |
| **PRB-11** | `/v1/public/support/sessions` | `POST` | Empty JSON object `{}` | `HTTP 422 VALIDATION_ERROR` | `HTTP 422 {"error":{"code":"VALIDATION_ERROR","message":"body -> merchant_id: Field required"}}` | **PASS** — Missing field rejected |
| **PRB-12** | `/v1/public/support/sessions` | `POST` | Nonexistent merchant UUID | `HTTP 404 MERCHANT_NOT_FOUND` | `HTTP 404 {"error":{"code":"MERCHANT_NOT_FOUND","message":"Merchant not found."}}` | **PASS** — Foreign key boundary |
| **PRB-13** | `/v1/public/support/sessions/not-a-uuid/messages` | `POST` | Malformed session UUID path parameter | `HTTP 422 VALIDATION_ERROR` | `HTTP 422 {"error":{"code":"VALIDATION_ERROR","message":"path -> session_id: Input should be a valid UUID..."}}` | **PASS** — Type boundary enforced |
| **PRB-14** | `/v1/public/support/sessions/00000000-.../messages` | `POST` | Missing `X-Customer-Token` header | `HTTP 401 CUSTOMER_TOKEN_REQUIRED` | `HTTP 401 {"error":{"code":"CUSTOMER_TOKEN_REQUIRED","message":"Customer token is required in X-Customer-Token header."}}` | **PASS** — Header token required |
| **PRB-15** | `/v1/public/support/sessions/00000000-.../messages` | `POST` | Invalid token value (`bad-token-xyz`) | `HTTP 404 SESSION_NOT_FOUND` | `HTTP 404 {"error":{"code":"SESSION_NOT_FOUND","message":"Customer support session not found."}}` | **PASS** — Opaque token rejected |
| **PRB-16** | `/v1/auth/me` | `GET` | Spoofed cookie (`mp_session=fake_invalid_token`) | `HTTP 401 UNAUTHORIZED` | `HTTP 401 {"error":{"code":"UNAUTHORIZED","message":"Invalid or expired session"}}` | **PASS** — Token hash mismatch |
| **PRB-17** | `/v1/auth/login` | `OPTIONS` | Origin `https://magicpin-frontend` | `HTTP 200 OK` | `HTTP 200 OK` (CORS headers returned) | **PASS** — Configured origin accepted |
| **PRB-18** | `/v1/auth/login` | `OPTIONS` | Unauthorized origin (`https://malicious-attacker.com`) | `HTTP 400 Disallowed CORS origin` | `HTTP 400 Disallowed CORS origin` | **PASS** — Unauthorized origins blocked |

---

## 4. Multi-Tenant Isolation Verification

Multi-tenant isolation ensures that no merchant can view, query, modify, or approve data belonging to another merchant.

### 4.1 Architectural Isolation Mechanism
The application enforces tenant isolation strictly at the dependency and repository levels:
1. **Server-Derived Identity (`backend/app/core/dependencies.py`):**
   ```python
   def get_current_merchant(
       user: User = Depends(get_current_user),
       db: Session = Depends(get_db),
   ) -> MerchantProfile:
       """Derives merchant ownership EXCLUSIVELY from authenticated session.
       Never accepts or trusts client-supplied merchant_id for authorization."""
       merchant = merchant_repo.get_by_user_id(user.id)
       ...
   ```
   No client-supplied `merchant_id` query parameter, header, or body field is accepted for merchant authorization. The merchant identity is derived entirely from the server-validated session cookie (`UserSession.user_id`).

2. **Scoped Repository Queries:**
   - `ConversationRepository.get_by_id`: Scoped by `where(Conversation.id == conversation_id, Conversation.merchant_id == merchant_id)`.
   - `CustomerSupportRepository.get_message_for_update`: Scoped by `where(CustomerSupportMessage.id == message_id, CustomerSupportMessage.merchant_id == merchant_id)`.
   - `ActionRepository.get_by_id`: Scoped by `where(Action.id == action_id, Action.merchant_id == merchant_id)`.
   - Attempting to access another merchant's resource yields `None`, which triggers an immediate `HTTP 404 NOT FOUND` (`MESSAGE_NOT_FOUND`, `CONVERSATION_NOT_FOUND`, or `ACTION_NOT_FOUND`). This prevents ID-enumeration and avoids leaking resource existence across tenants.

### 4.2 Multi-Tenant Verification Status
- **Live Multi-Tenant Probe:** **NOT VERIFIED (Live)**. In accordance with safety rules, creating a second live demo merchant account requires explicit user authorization. Zero unauthorized user accounts were created on Render.
- **Static Code Analysis:** **PASS**. All queries, update paths, and delete routes enforce tenant boundaries.
- **Automated Test Suite:** **PASS**. Full cross-tenant isolation tests passed against PostgreSQL:
  - `test_cross_merchant_chat_isolation`
  - `test_cross_merchant_context_prompt_isolation`
  - `test_cross_merchant_context_isolation`
  - `test_cross_merchant_conversation_isolation`
  - `test_cross_merchant_session_isolation`
  - `test_merchant_cannot_deliver_other_merchant_inquiry`
  - `test_tenant_isolation_prevents_cross_merchant_action_access`
  - `test_production_multi_merchant_e2e_journey`

---

## 5. Authentication and Session Lifecycle Security

| Security Control | Implementation Standard | Source Location | Audit Finding |
|---|---|---|---|
| **Password Hashing** | Argon2id (`Type.ID`, 19 MiB RAM, 2 iterations, 1 parallelism, 16-byte salt) | `backend/app/core/security.py` | **PASS** — Resilient against GPU cracking |
| **Session Generation** | 32-byte cryptographically secure random token (`secrets.token_urlsafe(32)`) | `backend/app/core/security.py` | **PASS** — 256-bit entropy |
| **Token Storage** | SHA-256 hash stored in `user_sessions.token_hash`. Raw token is never persisted | `backend/app/core/security.py` | **PASS** — Database compromise cannot leak plaintext session tokens |
| **Cookie Flags** | `HttpOnly=True`, `SameSite=Lax`, `Secure=True` (in production), `Path=/` | `backend/app/core/dependencies.py` | **PASS** — Mitigates XSS extraction & cross-site leak |
| **Session Expiration** | 7 days (`SESSION_TTL_MINUTES=10080`), enforced at request time against `expires_at` | `backend/app/core/dependencies.py` | **PASS** — Expired sessions reject with 401 |
| **Session Revocation** | Explicit `revoked_at` column checked on every request; set on logout | `backend/app/core/dependencies.py` | **PASS** — Immediate invalidation on logout |
| **Session Rotation** | Re-authenticating issues new session; refresh rotates token hash | `backend/app/api/v1/routes/auth.py` | **PASS** — Mitigates session fixation |
| **Brute-Force Defense** | Login rate limiter bounds attempts per IP/email (5 attempts / 900s) | `backend/app/core/security.py` | **PASS** — Blocks credential stuffing |

---

## 6. CSRF Defense Architecture

State-changing requests (`POST`, `PATCH`, `DELETE`) by authenticated merchants require cross-site request forgery protection:
- **Token Format:** Session-bound HMAC-SHA256: `csrf:{session_id}:{user_id}` generated with server secret `CSRF_SECRET`.
- **Decoupled from Cookie:** The CSRF token is not derived from or identical to the raw session cookie value, preventing breach attacks and cookie-reading leakage.
- **Verification (`verify_csrf`):**
  - Extracts `X-CSRF-Token` header.
  - Computes expected HMAC for the current authenticated session.
  - Compares using `hmac.compare_digest` (constant-time verification) to prevent timing side channels.
  - Rejects missing token with HTTP 403 `CSRF_TOKEN_MISSING`.
  - Rejects invalid token with HTTP 403 `CSRF_TOKEN_INVALID`.
- **Public Customer Support:** Public customer interactions (submitting inquiries) use stateless bearer tokens (`X-Customer-Token`) rather than ambient browser cookies, making CSRF defense unnecessary for public customer routes.

---

## 7. Rate Limiting and Concurrency Architecture

Public customer support endpoints are exposed without user authentication, requiring robust rate limiting:
- **Session Creation Limit:** 10 sessions per IP per hour (`RATE_LIMIT_CUSTOMER_SESSION_PER_HOUR`).
- **Inquiry Submission Limit:** 15 messages per IP per minute (`RATE_LIMIT_CUSTOMER_INQUIRY_PER_MINUTE`) AND 5 messages per session per minute (`RATE_LIMIT_CUSTOMER_SESSION_INQUIRY_PER_MINUTE`).
- **PostgreSQL Advisory Locks:** Dual-key concurrency protection acquires transaction-scoped advisory locks derived deterministically via 64-bit BigInt hashing (`hashtext()`).
- **Deadlock Prevention:** Lock keys are sorted in ascending numerical order (`key_a, key_b = sorted([lock_1, lock_2])`) before acquisition, mathematically eliminating AB-BA deadlocks under concurrent requests.
- **Audit Table Optimization:** Rate limit events are logged to `audit_events`. High-throughput queries utilize the migration 0004 index: `CREATE INDEX idx_audit_events_rate_limit ON audit_events ((details ->> 'rate_limit_key'), created_at DESC)`.
- **Fail-Closed Semantics:** In production (`APP_ENV="production"`), if the rate limiter encounters a database connectivity or lock failure, it fails closed with HTTP 503 rather than allowing unbounded request flooding.

---

## 8. Client IP & Reverse Proxy Extraction Security

FastAPI applications deployed behind reverse proxies (Render, Cloudflare, AWS ALB) risk IP spoofing if `X-Forwarded-For` is naively trusted:
- **Implementation (`backend/app/core/security.py` `get_client_ip`):**
  1. Inspects the immediate TCP peer address (`request.client.host`).
  2. If the peer IP is NOT in `TRUSTED_PROXIES`, forwarding headers (`X-Forwarded-For`) are **strictly ignored**. The immediate peer IP is returned. This prevents arbitrary public clients from spoofing client IPs.
  3. If the peer IP IS in `TRUSTED_PROXIES`, hops are traversed from **right to left** (most recent proxy to earliest).
  4. Each hop is validated using Python's `ipaddress` library (handling bracketed IPv6, port stripping, and CIDR subnet evaluation).
  5. The first hop from the right that is not a trusted proxy is selected as the authentic client IP.
- **Verification:** Unit tests `test_client_ip_extraction.py` (14 tests) and integration tests `test_rate_limiter_proxy.py` verified that untrusted peers cannot spoof IP headers to bypass rate limits.

---

## 9. Input Validation & Error Handling Boundaries

### 9.1 Schema Boundaries (Pydantic V2)
- All request payloads are bound to Pydantic models with strict typing:
  - `email`: Normalized lowercase with RFC regex validation.
  - `password`: Enforces minimum 8 characters, maximum 128 characters, uppercase, lowercase, number, and special character.
  - `business_name`: Stripped, minimum 1 character, maximum 100 characters.
  - `uuid.UUID`: Path parameters automatically validated by FastAPI/Pydantic; invalid characters reject at framework level with HTTP 422.
  - `customer_text`: Stripped, minimum 1 character, maximum 2000 characters.

### 9.2 Centralized Exception Handling (`backend/app/main.py`)
- `HTTPException`: Sanitized JSON response preserving defined status code and error code.
- `RequestValidationError`: Formats first validation failure into structured JSON (`{"error": {"code": "VALIDATION_ERROR", "message": "..."}}`).
- `Exception` (Unhandled): Catches any unexpected runtime exception, logs the stack trace server-side via standard logger, and returns a sanitized HTTP 500 response (`"An unexpected server error occurred. Please try again later."`). Stack traces, SQL queries, and database driver exceptions are shielded from the client.

---

## 10. AI / External Service Failure Paths

The application relies on external LLM APIs (Groq / Grok). Failure handling ensures system availability when upstream providers experience latency or downtime:

### 10.1 Chat Service (`ChatService.send_message`)
- LLM calls are bounded by `LLM_TIMEOUT_SECONDS=30` and `LLM_MAX_RETRIES=2`.
- When an LLM call fails or times out:
  1. The user message is saved in PostgreSQL with `status="failed"`.
  2. Conversation row locks are immediately committed and released, avoiding database thread starvation.
  3. Returns HTTP 502 `LLM_PROVIDER_ERROR` with structured payload:
     ```json
     {
       "error": {
         "code": "LLM_PROVIDER_ERROR",
         "message": "The AI service is temporarily unavailable. Your message has been saved as failed and can be retried.",
         "details": {
           "failed_message_id": "<uuid>",
           "retryable": true
         }
       }
     }
     ```
  4. The client can retry the failed turn using `retry_message_id` with a fresh idempotency key without losing draft text.

### 10.2 Customer Support Service (`CustomerSupportService.submit_customer_inquiry`)
- When an inquiry arrives, the message is persisted first to PostgreSQL (`commit=True`).
- If subsequent AI draft generation fails:
  1. Exception is caught and logged (`logger.error(...)`).
  2. The message remains in `status="needs_review"`.
  3. Message records `ai_status="failed"` and stores the truncated error (`ai_error=str(exc)[:500]`).
  4. Customer receives HTTP 201 Created with their inquiry safely logged.
  5. The merchant can review the inquiry in their inbox, manually type an answer, and deliver the reply to the customer. Upstream AI failure does not impair the human support channel.

---

## 11. Security Warnings & Remediation Recommendations

While the application security posture is robust for educational demonstration, two non-blocking warnings were identified during live audit:

### Warning 1: CORS Origin Resolution in `render.yaml`
- **Condition:** In `render.yaml`, `FRONTEND_ORIGIN` is configured as:
  ```yaml
  - key: FRONTEND_ORIGIN
    fromService:
      type: web
      name: magicpin-frontend
      property: host
  ```
- **Observed Behavior:** On Render, `property: host` evaluates to the internal service name `magicpin-frontend` (without `.onrender.com`). Backend normalization transforms this into `https://magicpin-frontend`. Consequently, direct cross-origin browser requests with `Origin: https://magicpin-frontend.onrender.com` receive HTTP 400 `Disallowed CORS origin`.
- **Demonstration Impact:** **Minimal / Mitigated**. In the live deployment, the frontend accesses the backend via Render CDN rewrite rules (`/v1/* -> https://magicpin-backend.onrender.com/v1/*`). All browser requests are same-origin to the frontend CDN, avoiding cross-origin preflight requests during standard user workflows.
- **Recommended Remediation:** In `render.yaml` or the Render dashboard, set `FRONTEND_ORIGIN` explicitly to `https://magicpin-frontend.onrender.com`.

### Warning 2: Reverse Proxy CIDR Configuration (`TRUSTED_PROXIES`)
- **Condition:** In `render.yaml`, `TRUSTED_PROXIES` is set to default `"127.0.0.1,::1"`.
- **Observed Behavior:** Render routes incoming traffic through internal proxy nodes (`10.0.0.0/8`). Because `10.0.0.0/8` is not currently in `TRUSTED_PROXIES`, `get_client_ip` treats the Render proxy as an untrusted peer and extracts the proxy IP rather than peeling `X-Forwarded-For`.
- **Demonstration Impact:** **Low**. Rate limiting continues to function; all demo requests are attributed to the shared Render ingress proxy rather than granular client IPs. In a high-traffic production environment, this would pool rate limits across all users sharing an ingress proxy.
- **Recommended Remediation:** Update `TRUSTED_PROXIES` in Render service environment settings to include `"127.0.0.1,::1,10.0.0.0/8"`.

---

## 12. Automated Test Execution Summary

A complete automated test execution was conducted locally using a disposable PostgreSQL 16 container (`postgres:16-alpine`, port 55436) that was created, migrated, tested, and destroyed:

```
================================ Backend Test Summary ================================
Security Tests (backend/tests/security/):        15 passed in 1.23s
Unit Tests (backend/tests/unit/):                104 passed in 3.33s
API Tests (backend/tests/api/):                  75 passed in 4.75s
Integration Tests (backend/tests/integration/):  37 passed, 2 skipped (smoke) in 6.23s
======================================================================================
Backend Total:                                   231 passed, 2 skipped in 15.54s

=============================== Frontend Test Summary ===============================
Frontend Unit/API Tests (node --test test/*.test.js): 85 passed in 200.6ms
Production Build (tsc && vite build):                 Built cleanly in 1.22s
======================================================================================
Frontend Total:                                  85 passed, 0 failed

================================ Grand Total Tests ==================================
Total Test Suite:                                316 passed, 0 failed, 2 skipped
======================================================================================
```

---

## 13. Academic Demonstration Security Assessment

| Threat Vector | Evaluated Safeguard | Risk Level | Assessment |
|---|---|---|---|
| **SQL Injection** | SQLAlchemy 2.0 parameterized queries exclusively. Zero raw SQL string interpolation | **Very Low** | Defended |
| **Cross-Site Scripting (XSS)** | React JSX auto-escaping; cookies marked `HttpOnly`; untrusted input treated as plain text | **Very Low** | Defended |
| **CSRF** | Session-bound HMAC-SHA256 tokens validated on all state-changing endpoints | **Very Low** | Defended |
| **Broken Object-Level Auth (BOLA)** | Tenant identity resolved exclusively from session; repository queries enforce `merchant_id` | **Very Low** | Defended |
| **Denial of Service (DoS)** | Dual-key PostgreSQL rate limiting with advisory locks and optimized index | **Low** | Defended |
| **Credential Stuffing** | Login rate limiting (5 attempts per 15 min per IP/email) with Argon2id hashing | **Low** | Defended |
| **LLM Outage / Prompt Injection** | System prompts strictly delimit merchant context; AI timeouts fail safe with merchant review preserved | **Low** | Defended |
| **Information Disclosure** | Centralized exception handling shields database errors, stack traces, and system internals | **Very Low** | Defended |

---

## 14. Verification Sign-Off

- **Role:** Senior Application Security Engineer & QA Automation Engineer
- **Status:** **COMPLETED**
- **Final Verdict:** **PASS WITH WARNINGS**
- **Readiness for Academic Demonstration:** **APPROVED**. The application demonstrates defensive engineering across all architectural layers, handles failure paths gracefully, preserves human-in-the-loop review during external service degradation, and maintains strict tenant and database boundaries.
