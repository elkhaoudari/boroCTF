# G63p_G0d

**Category:** Misc
**Difficulty:** Easy

## Description

A text file was provided. The challenge name strongly hinted that `grep` would be useful.

## Initial Analysis

The file appeared to contain a large amount of seemingly random text. Opening it manually was impractical, so the first step was to inspect its contents from the command line.

Since the flag format was known to be:

```text
boroCTF{...}
```

the most direct approach was to search the file for occurrences of the string `boroCTF`.

## Solution

Using `grep`, we searched the file for the flag pattern:

```bash
grep -R "boroCTF" .
```

The command immediately returned the only matching line, revealing the flag.

A more precise approach would be:

```bash
grep -o 'boroCTF{[^}]*}' filename
```

This extracts only the flag itself without any surrounding text.

## Flag

```text
boroCTF{G63p_G0d}
```

## Lessons Learned

* Always pay attention to challenge names; they often contain hints.
* Command-line tools can solve some challenges in seconds.
* `grep` is one of the most useful tools for CTFs involving logs, source code, configuration files, and large text datasets.

## Tools Used

* grep
* cat
* less
* Linux command line

## Commands

```bash
grep -R "boroCTF" .
```

```bash
grep -o 'boroCTF{[^}]*}' filename
```

