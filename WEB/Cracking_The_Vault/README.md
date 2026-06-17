# Cracking the Vault — Source Code Disclosure

**CTF:** boroCTF 2026
**Challenge:** Cracking the Vault
**Category:** Web
**Points:** 100

---

## Description

> Apparently this banking system is super secure and would never store the password somewhere in plain sight...

The challenge provided a single webpage titled:

```text
Very Secure Vault
```

Users were asked to enter a password in order to unlock the vault and reveal the flag.

---

## Initial Reconnaissance

Opening the page presented a simple login form:

```html
<input placeholder="Enter password">
<button>Unlock</button>
```

There was no visible hint regarding the password.

Since this was a beginner web challenge, the next step was inspecting the page source.

---

## Source Code Review

Viewing the HTML source immediately revealed the issue.

Inside the JavaScript code:

```javascript
const correctPassword = "passworduwu";
```

The application stored the vault password directly inside client-side JavaScript.

Further down:

```javascript
if (input === correctPassword) {
    messageEl.textContent =
        "✓ Correct! Flag: boroCTF{th3_p@th_l3ss_tr@vers3d}";
}
```

Not only was the password exposed, but the flag itself was also hardcoded in the source code.

---

## Exploitation

There were two possible solutions.

### Method 1 — Use the Password

Enter:

```text
passworduwu
```

The page responds:

```text
✓ Correct! Flag: boroCTF{th3_p@th_l3ss_tr@vers3d}
```

### Method 2 — Read the Source

Simply viewing the HTML source reveals both:

```javascript
const correctPassword = "passworduwu";
```

and

```text
boroCTF{th3_p@th_l3ss_tr@vers3d}
```

No interaction with the form is required.

---

## Vulnerability

This challenge demonstrates a classic mistake:

```text
Client-Side Secret Exposure
```

Sensitive data should never be trusted to remain hidden when delivered to a user's browser.

Anything sent to the client can be viewed through:

* View Source
* Developer Tools
* Browser Debugger
* Saved HTML Files

---

## Attack Flow

```text
Open Website
      ↓
View Source
      ↓
Inspect JavaScript
      ↓
Discover Password
      ↓
Discover Flag
      ↓
Challenge Solved
```

---

## Flag

```text
boroCTF{th3_p@th_l3ss_tr@vers3d}
```

---

## Root Cause

The application performed authentication entirely on the client side:

```javascript
if (input === correctPassword)
```

The server never validated the password.

Additionally, both the password and the flag were embedded directly into the JavaScript source.

---

## Remediation

A secure implementation should:

* Validate credentials on the server.
* Never expose secrets in client-side code.
* Store flags and sensitive data server-side.
* Return only the minimum information required by the client.

Example:

```python
if submitted_password == stored_password:
    return flag
```

instead of:

```javascript
const correctPassword = "passworduwu";
```

---

## Lessons Learned

* Always inspect page source during web challenges.
* Never assume hidden HTML or JavaScript is secure.
* Client-side validation can almost always be bypassed.
* Sensitive information belongs on the server, not in the browser.

---

## Tools Used

* Browser View Source
* Developer Tools

---

## Takeaway

This challenge was a beginner-friendly introduction to client-side security mistakes. The password and flag were both embedded directly in the HTML source, making the challenge solvable simply by reviewing the page code without interacting with the application.

