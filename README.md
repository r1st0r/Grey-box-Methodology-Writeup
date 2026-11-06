# Web App Security Assessment — Grey-box Methodology (Redacted)

Nov 6, 2026 · @Kirill Trofimov

This is a redacted write-up of a grey-box security assessment of a multi-tenant retail catalogue and point-of-sale (POS) SaaS web application. The target's identity, domain, and addresses are withheld; testing was authorized and run against a pre-production instance with no real customer data. No exploitable vulnerabilities were found. The value of the write-up is the methodology: which classes were tested, how, and what the application did right.

## Scope and methodology

Authorized assessment of a pre-production instance with no live customer data, which permitted active testing (injection, request tampering, brute-force attempts) without risk to real tenants.

| Item | Detail |
| --- | --- |
| Target | Multi-tenant retail/POS SaaS web app (identity redacted) |
| Environment | Pre-production / test, no real customer data |
| Authorization | Written permission from the product owner |
| Approach | Grey-box — external recon plus authenticated testing with a self-provisioned standard user account |
| In scope | External perimeter; HTTP(S) application; authenticated API surface for a non-admin user |
| Out of scope | Backend source code; live POS devices; destructive testing; social engineering |
| Tooling | nmap, Burp Suite, manual analysis of the client-side JavaScript bundles |

**Redaction note:** domain, IP addresses, product and company names, exact version strings, and screenshots have been removed. The technology stack is described generically; no detail here identifies the target.

## Target overview (anonymized)

The application is a React single-page app built with Vite, served as a static bundle. The backend is ASP.NET Core (errors returned as RFC 7807 `application/problem+json`), fronted by a Caddy reverse proxy with HTTP/2 and HTTP/3. SSH is exposed but key-only, with no keys stored by the application.

Any unmatched path returns the SPA login page — expected client-side-router behaviour, not an information leak. Server and framework versions are suppressed in response headers, and no CVE was identifiable from the exposed surface.

### API surface shape

Recovered from the JavaScript bundles; all endpoints are same-origin and relative. Generalized paths:

- Auth: login, MFA verify, session check, logout
- Tenant-scoped resources under `/api/my/*`
- Admin resources under `/api/admin/customers/{id}/*`

Authentication is cookie-based with a `__Host-` prefixed session cookie; an API-key (`Bearer`) path is also present.

## Testing matrix

Each class was actively tested. No vulnerability was confirmed.

| Class | How it was tested | Observed result | Conclusion |
| --- | --- | --- | --- |
| Broken object-level auth (IDOR) | Standard user requesting admin objects and other tenants' identifiers | 403 Forbidden | Not exploitable |
| Broken function-level auth | Non-admin user calling admin endpoints | 403 on all | Enforced server-side |
| Password-change abuse | Tampered payload on the change-password call | Keyed off the session cookie, not a body user id | Not exploitable |
| SQL injection | Injection payloads in request parameters | Responses unchanged | Not present |
| CSRF | Removed Origin header; sent a foreign Origin | 403 in both cases | Strict Origin validation |
| Session / cookie | Inspected Set-Cookie | `__Host-` prefix (Secure, host-only, Path=/) | Correctly hardened |
| Mass assignment | Added unexpected fields (role, isAdmin, tenant id) to JSON bodies | 403 | Not exploitable |
| Cross-site scripting | Stored/reflected `img/onerror` payload in input fields | Did not execute | Output encoding effective |
| Info disclosure | Malformed JSON and wrong types | Clean 400, no stack trace | No leak |
| MFA brute force | Repeated codes on the verify endpoint | 405 without MFA enabled (no pending state) | Inconclusive — retest with 2FA on |
| Login rate limit | Repeated login attempts | 429 Too Many Requests | Rate limiting present |

## Security controls observed

The absence of findings is backed by concrete controls:

- **Session cookie** — `__Host-` prefix forces Secure, host-only, Path=/; cannot be set over HTTP or scoped to a subdomain.
- **CSRF defense** — strict Origin validation; missing or foreign origins rejected with 403, closing browser-based CSRF.
- **Transport** — HSTS with a one-year max-age and includeSubDomains; HTTP/3 available.
- **CSP** — `default-src 'none'; frame-ancestors 'none'`, plus `X-Frame-Options: DENY`.
- **Other headers** — nosniff, `Referrer-Policy: no-referrer`, a restrictive Permissions-Policy, `Cross-Origin-Resource-Policy: same-origin`, `Cache-Control: no-store`.
- **Error handling** — uniform `problem+json`, no stack traces or version disclosure.
- **Authorization** — checked ahead of business logic; admin endpoints 403 for standard users.
- **Version suppression** — neither the proxy nor the backend exposes a version number.

## Takeaways

A clean result is still a result. This assessment systematically ruled out the standard external and authenticated vulnerability classes against a well-built target, and documented the controls responsible. A thorough negative finding, with reproducible method, is as much a demonstration of security skill as a single lucky bug.

Surface not covered this round, and where attention would go next:

- Authenticated business logic — orders, pricing, and quantity flows (negative values, out-of-order state transitions, race conditions).
- A device command interface — the highest-value target for command injection, requiring source or a live device.
- API key management — key scope and tenant binding.
- MFA rate limiting — retest with 2FA enabled and an active pending state.

**Methodology, not luck:** the point of this write-up is the repeatable process, not the (absent) exploit.
