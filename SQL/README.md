# websec

A growing collection of web security attack workflows and reference guides — for use during penetration testing, labs (e.g. PortSwigger Web Security Academy), CTF competitions, and real-world engagements.

## Contents

### SQL Injection

| Technique | File | When to Use |
|-----------|------|-------------|
| **UNION Attack** | `sqli-union-attack.md` | Output reflected in page |
| **Visible Error-Based** | `sqli-visible-error-based.md` | Error messages leak data |
| **Blind — Boolean-Based** | `blind-sqli-boolean-based.md` | Response behavior changes (no output, no errors) |
| **Blind — Conditional Errors** | `blind-sqli-conditional-errors.md` | Error/no-error is the only signal |
| **Blind — Time-Based** | `blind-sqli-time-based.md` | No output, no errors, no behavior change — only timing |
| **DB Fingerprinting Cheat Sheet** | `sqli-db-fingerprint-cheatsheet.md` | Multi-DB payloads, extraction chains, decision tree |

### Decision Tree

```
Send ' →
  Error visible?          → Visible Error-Based
  Output in page?         → UNION Attack
  Response changes?       → Boolean-Based Blind
  No change at all?       → Time-Based Blind
  Input inside XML body?  → WAF Bypass (see cheat sheet §7)
  Async / WAF blocks all? → OOB/OAST (see cheat sheet §6)
```

*More categories coming: XSS, SSRF, authentication bypass, IDOR, etc.*

## Usage

Quick-reference cheat sheets during security assessments, labs, or CTF competitions. Each file covers the full workflow from identification to exploitation, with multi-DB payloads and real-world tips.

> ⚠️ **Disclaimer:** The techniques documented here are for educational purposes and authorized security testing only. Do not use them against systems you do not have explicit permission to test.
