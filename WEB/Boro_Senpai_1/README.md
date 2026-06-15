# boro-senpai 1 — IDOR Writeup

**CTF:** boroCTF  
**Challenge:** boro-senpai 1  
**Category:** Web  
**Points:** 100  
**Flag:** `boroCTF{3l_psY_c0ngR00!}`

---

## Description

> Hououin Kyouma here. I need your help finding my assistant — the self-proclaimed neuroscience prodigy. She had an old @channel handle. Find it, and her profile is yours.

A fake @channel (2chan/4chan-style imageboard) themed around Steins;Gate.

---

## Reconnaissance

Visiting the challenge URL revealed a Steins;Gate-themed @channel imageboard. The logged-in user was `hououin_kyouma` — the alter ego of Okabe Rintaro from Steins;Gate.

Key clues from the description:
- **Hououin Kyouma** = Okabe Rintaro's mad scientist alias
- **"my assistant, the neuroscience prodigy"** = Makise Kurisu (Christina)
- **@channel handle** = reference to the in-universe 2channel-like imageboard in Steins;Gate

Reading the imageboard threads revealed Kurisu's old @channel handle:

```
KuriGohanandKamehameha
```

This is her canonical embarrassing handle from the Steins;Gate lore.

---

## Vulnerability: IDOR (Insecure Direct Object Reference)

The profile URL structure was:
```
/profile/{username}
```

Our authenticated session was `hououin_kyouma`, so our profile was at:
```
/profile/hououin_kyouma
```

The profile page stated: *"Your profile is visible only to you"* — suggesting private profiles.

However, the server only used the **URL parameter** to decide which profile to return, with **no authorization check** to verify the requesting user owned that profile.

---

## Exploitation

Simply navigating to Kurisu's profile URL:

```
https://w03xj6cjsucj.boroctf.com/profile/KuriGohanandKamehameha
```

The server returned her private profile without any verification, revealing the flag:

```
boroCTF{3l_psY_c0ngR00!}
```

---

## Root Cause

The server-side logic was equivalent to:

```python
# VULNERABLE CODE (pseudocode)
@app.route('/profile/<username>')
def profile(username):
    user = db.get_user(username)
    return render_template('profile.html', user=user)
    # Missing: if username != session['user']: abort(403)
```

No check was made to verify: *"Is the logged-in user authorized to view THIS profile?"*

---

## Fix

A proper authorization check should be implemented:

```python
# FIXED CODE (pseudocode)
@app.route('/profile/<username>')
@login_required
def profile(username):
    if username != session['current_user']:
        abort(403)  # Forbidden
    user = db.get_user(username)
    return render_template('profile.html', user=user)
```

---

## IDOR Overview

**IDOR (Insecure Direct Object Reference)** is an access control vulnerability where an application uses user-controllable input to access objects directly without verifying authorization.

It appears in the [OWASP Top 10](https://owasp.org/Top10/) under **A01: Broken Access Control**.

**Real-world impact:**
- Reading other users' private messages, medical records, or financial data
- Modifying or deleting other users' data
- Full account takeover if sensitive tokens are exposed

---

## Timeline

1. Read challenge description → identified Makise Kurisu as the target
2. Searched Steins;Gate lore → found handle `KuriGohanandKamehameha`
3. Observed URL pattern `/profile/hououin_kyouma`
4. Changed URL to `/profile/KuriGohanandKamehameha`
5. Retrieved private profile → flag captured ✅

---

## References

- [OWASP: Insecure Direct Object Reference](https://owasp.org/www-chapter-ghana/assets/slides/IDOR.pdf)
- [PortSwigger: IDOR](https://portswigger.net/web-security/access-control/idor)
- [Steins;Gate Wiki: Makise Kurisu](https://steins-gate.fandom.com/wiki/Makise_Kurisu)

---

*Writeup by [@elkhaoudari](https://github.com/elkhaoudari) *
