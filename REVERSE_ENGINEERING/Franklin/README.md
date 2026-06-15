# Franklin - BoroCTF 2026 (Rev) Writeup

## Challenge Information

* **Name:** Franklin
* **Category:** Reverse Engineering
* **Points:** 200
* **Description:** "A customized idea by me, about me, with me, for you."

## Initial Analysis

The challenge provided a single file named `Franklin`.

Running `file` immediately revealed that it was a TrueType font:

```bash
file Franklin
```

Output:

```text
Franklin: TrueType Font data, 20 tables, 1st "FFTM"
```

At first glance, the file appeared to be a normal font. Basic checks such as:

```bash
strings Franklin | less
```

did not reveal any obvious flag or hidden text.

Since the challenge category was **Reverse Engineering** and the provided artifact was a font, the next step was to inspect the internal font tables.

## Extracting Font Tables

Using FontTools:

```bash
ttx Franklin
```

This generates an XML representation of the font.

Among the extracted tables, the most interesting one was **GSUB** (Glyph Substitution Table).

GSUB is commonly used for:

* Ligatures
* Character substitutions
* Custom glyph replacement rules

## Discovering the Hidden Ligature

Inspecting the GSUB table revealed a ligature rule:

```text
asterisk <-
b o r o C T F braceleft
f R four n k l one n
underscore
f zero n seven
braceright
```

This means that when the glyph `asterisk (*)` is rendered, the font replaces it with a sequence of glyphs.

Converting glyph names into characters gives:

```text
boroCTF{
fR4nkl1n
_
f0n7
}
```

Resulting in:

```text
boroCTF{fR4nkl1n_f0n7}
```

## Flag

```text
boroCTF{fR4nkl1n_f0n7}
```

## Takeaway

This challenge demonstrates how TrueType/OpenType fonts can hide data using the **GSUB (Glyph Substitution)** table.

Instead of storing the flag directly in strings or metadata, the author embedded it inside a ligature rule that expands a single character into the full flag.

A nice introductory reverse-engineering challenge involving font internals.

