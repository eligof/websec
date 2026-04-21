# WebSec

Field reference for real-world web application security. Built from
engagements and bug bounty work — what to look for on live targets,
payloads that actually land in production, filter/WAF bypasses, and
how to turn a bug into impact.

Not a tutorial. Not lab walkthroughs. Notes written so future-me can
grep for a technique at 2am during a test.

---

## How to use this

Each folder covers one vulnerability class. Inside:

- **Recon** — how to spot the sink on an unfamiliar app
- **Payloads** — working strings, grouped by DB/browser/framework
- **Bypasses** — what to try when the obvious payload eats a 403
- **Impact** — how to escalate from "it reflects" to "it owns the app"

---

## Injection

| Class | Status | Notes |
|-------|--------|-------|
| [SQL Injection](./SQL/) | Covered | UNION, error-based, boolean/time blind, DB fingerprinting |
| NoSQL Injection | TODO | Mongo `$where`/`$regex`, auth bypass, JS injection |
| Command Injection | TODO | Blind OOB, filter bypass, argument injection |
| LDAP / XPath / SSTI | TODO | Template engines (Jinja2, Twig, Freemarker, Velocity) |
| XXE | TODO | In-band, OOB, blind via DTD, SSRF pivot |

## Client-side

| Class | Status | Notes |
|-------|--------|-------|
| [Cross-site Scripting (XSS)](./XSS/) | In progress | Reflected, stored, DOM, context breakouts, CSP bypass |
| CSRF | TODO | SameSite gaps, JSON CSRF, Flash/XHR tricks |
| CORS misconfig | TODO | Null origin, reflected ACAO, credentialed leaks |
| Clickjacking | TODO | Frame busting bypass, drag-and-drop, keystroke |
| Prototype pollution | TODO | Client and server side, gadget chains |

## Auth & session

| Class | Status | Notes |
|-------|--------|-------|
| Broken auth | TODO | Password reset, MFA bypass, username enum, credential stuffing |
| JWT attacks | TODO | `alg:none`, kid injection, jku/x5u, RS256->HS256 confusion |
| OAuth / OIDC | TODO | redirect_uri games, PKCE downgrade, token leakage |
| SAML | TODO | Signature wrapping, XSW, replay |
| Session management | TODO | Fixation, predictable IDs, logout handling |

## Access control & logic

| Class | Status | Notes |
|-------|--------|-------|
| IDOR / BOLA | TODO | UUID guessing, HPP, verb tampering, GraphQL field-level |
| Privilege escalation | TODO | Horizontal/vertical, mass assignment, role tampering |
| Business logic | TODO | Race conditions, price/quantity tampering, multi-step bypass |
| Information disclosure | TODO | Error verbosity, debug endpoints, `.git`/`.env`, source maps |

## Server-side request & file

| Class | Status | Notes |
|-------|--------|-------|
| SSRF | TODO | Cloud metadata (IMDSv1/v2, Azure IMDS, GCP), DNS rebinding, gopher |
| File upload | TODO | Extension/MIME bypass, polyglots, SVG XSS, path traversal on name |
| Path traversal / LFI / RFI | TODO | Encoding tricks, null byte, wrapper abuse, log poisoning |
| Insecure deserialization | TODO | Java (ysoserial), .NET, PHP (phpggc), Python / Ruby gadgets |

## Request-layer

| Class | Status | Notes |
|-------|--------|-------|
| HTTP request smuggling | TODO | CL.TE, TE.CL, TE.TE, H2.CL, H2.TE, client-side desync |
| Web cache poisoning / deception | TODO | Unkeyed headers, path confusion, illegal cache keys |
| HTTP Host header attacks | TODO | Password reset poisoning, routing-based SSRF, cache key injection |
| WebSocket attacks | TODO | CSRF over WS, auth-context confusion |

## API

| Class | Status | Notes |
|-------|--------|-------|
| REST API testing | TODO | Mass assignment, verb tampering, content-type confusion |
| GraphQL | TODO | Introspection, batching, alias DoS, field suggestions, IDOR |
| gRPC / SOAP | TODO | Reflection, WSDL parsing, XXE via SOAP |

## Misc / advanced

| Class | Status | Notes |
|-------|--------|-------|
| Web LLM attacks | TODO | Prompt injection, indirect injection, tool abuse |
| Race conditions | TODO | Single-packet attack, TOCTOU in business flows |
| Supply chain | TODO | Dependency confusion, typosquatting, malicious postinstall |
| Tooling reference | TODO | Burp, ffuf, nuclei, sqlmap, httpx, katana — flags I actually use |

---

## Conventions

- **gotcha** — thing that burned me once, now documented
- **high-impact** — the variant of the bug class that lands bounties
- **waf-bypass** — filter / WAF evasion for that specific sink
- **writeup** — pointer to a real-world report or disclosure

Payloads are raw. Copy, adapt the target param, fire.
