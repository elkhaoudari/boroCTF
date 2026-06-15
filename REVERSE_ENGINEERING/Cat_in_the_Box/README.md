# Cat in the ... Box? — boroCTF 2026

> Reverse Engineering | 200 Points | 216 Solves

## Challenge Description

> We love cats over here at boroCTF. We feel like we have a hidden connection to them.

The challenge provides a single executable named `cat`.

---

# Reconnaissance

The first step was gathering basic information about the binary.

```bash
file cat
checksec cat
strings cat | less
```

While inspecting the output of `strings`, several interesting hardcoded values appeared:

```text
a1b2c3
golden
silver
...
```

These values did not immediately reveal the flag, suggesting some form of runtime processing or obfuscation.

Since no obvious flag pattern existed, static analysis became the next logical step.

---

# Static Analysis

The binary was loaded into Ghidra for deeper inspection.

After reconstructing the program flow, the main logic looked roughly like this:

```c
for each string in candidate_list
{
    if (condition_1(string) &&
        condition_2(string))
    {
        selected_string = string;
        break;
    }
}
```

The program iterated through multiple candidate strings and applied two validation routines.

Only one string successfully passed both checks:

```text
ymweyc
```

This value was then used inside an XOR decoding function.

---

# Recovering Hidden Data

The XOR routine revealed two hidden strings:

```text
https://files.catbox.moe/
```

and

```text
.txt
```

The program concatenated these components with the validated key:

```text
https://files.catbox.moe/ymweyc.txt
```

At this point, it became clear that the flag was not embedded directly inside the binary.

Instead, the binary acted as a loader that dynamically fetched external content.

---

# Understanding Program Behavior

Further analysis showed the program constructing and executing a command similar to:

```bash
curl -s https://files.catbox.moe/ymweyc.txt
```

Rather than continuing to reverse every instruction, it was faster to reproduce the behavior manually.

---

# Fetching the Remote Content

Running:

```bash
curl https://files.catbox.moe/ymweyc.txt
```

returned:

```text
The cat wispers the flag to you ...
Segmentation fault (core dumped)

-----------------
SUPER SECRET AREA
-----------------
Yeah, I hardcoded the segfault.
Here's your real flag: boroCTF{lEts_gO_B3y0nd_b1nar1e$}
```

The challenge intentionally displayed a fake segmentation fault to mislead participants into thinking something had gone wrong.

However, the real flag was printed immediately afterward.

---

# Flag

```text
boroCTF{lEts_gO_B3y0nd_b1nar1e$}
```

---

# Technical Summary

| Step              | Result                          |
| ----------------- | ------------------------------- |
| Binary Inspection | No obvious flag                 |
| Static Analysis   | Identified validation logic     |
| String Recovery   | Recovered `ymweyc`              |
| XOR Decoding      | Revealed Catbox URL structure   |
| Dynamic Analysis  | Confirmed remote file retrieval |
| Final Retrieval   | Obtained flag                   |

---

# Lessons Learned

This challenge is a good reminder that:

* Flags are not always stored inside binaries.
* XOR obfuscation remains a common technique for hiding strings.
* Following program logic is often more effective than brute-forcing.
* External resources (URLs, APIs, remote files) should always be investigated during reversing.
* Apparent crashes may be intentional misdirection.

Although the challenge appeared to be a standard reverse engineering task, the actual objective was to understand the program's control flow well enough to reconstruct the hidden URL and retrieve the remote content.

The title *"Cat in the ... Box?"* was a subtle hint that the flag was hidden somewhere outside the binary itself—inside a "cat box."

