
---
title: hkcert CTF 2024 - BabyCracker
description: Reversing a binary baby challenge. 
date: 2024-11-09
tldr: Solve of a binary rev challenge.
draft: false
tags: ["ctf", "writeup", "rev", "binary"]
---

## Static analysis 

```bash
$ file cracker
cracker: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c40da391f00958199cc717ccbb4919d20f23718c, for GNU/Linux 3.2.0, stripped
```

```bash
notdi@NTB-PC1NJ74B:/mnt/c/Users/notdi/Downloads/CTF_SAT_09.11/rev/BabyCracker$ strings cracker
/lib64/ld-linux-x86-64.so.2
__cxa_finalize
strchr
__libc_start_main
strstr
strlen
__isoc99_scanf
printf
libc.so.6
GLIBC_2.7
GLIBC_2.2.5
GLIBC_2.34
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
PTE1
u+UH
hkcert24{
NJ'5
-F%6
Enter the flag:
+C8h
n:.b
trBd3
~U>JB
You are now verified!
;*3$"
GCC: (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
Ubuntu clang version 14.0.0-1ubuntu1.1
.shstrtab
.interp
.note.gnu.property
.note.gnu.build-id
.note.ABI-tag
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.dynamic
.got.plt
.data
.bss
.comment
```




## Decompile

```C
__int64 __fastcall main(int a1, char **a2, char **a3)
{
  int i; // [rsp+Ch] [rbp-124h]
  int v5; // [rsp+18h] [rbp-118h]
  int v6; // [rsp+1Ch] [rbp-114h]
  char user_input[256]; // [rsp+20h] [rbp-110h] BYREF
  char **v8; // [rsp+120h] [rbp-10h]
  int v9; // [rsp+128h] [rbp-8h]
  int v10; // [rsp+12Ch] [rbp-4h]

  v10 = 0;
  v9 = a1;
  v8 = a2;
  printf("Enter the flag: ");
  __isoc99_scanf("%s", user_input);
  if ( strstr(user_input, needle) != user_input )
    goto FAIL;
  v6 = strlen(user_input);
  if ( strchr(user_input, 125) != &user_input[v6 - 1]
    || user_input[v6 - 2] != 49
    || user_input[v6 - 5] + user_input[v6 - 4] + user_input[v6 - 3] != 300
    || 2 * user_input[v6 - 5] + user_input[v6 - 4] + 2 * user_input[v6 - 3] != 496
    || 2 * user_input[v6 - 5] + 3 * user_input[v6 - 4] + user_input[v6 - 3] != 607 )
  {
    goto FAIL;
  }
  v5 = 0;
  for ( i = 0; i < v6 - 14 && (*((char *)some_xor + i) ^ user_input[i + 9]) == xor_result[i]; ++i )
    ++v5;
  if ( v5 == 28 && i == v6 - 14 )
  {
    printf("You are now verified!\n");
    return 0;
  }
  else
  {
FAIL:
    printf("Bye\n");
    return (unsigned int)-1;
  }
}
```

The idea is simple we ask the user for input and if the input is the flag we get the `You are now verified!` output. We did rename some of the variables as we decompiled this, to make it a bit easier to read.

## First check 

Quickly glancing over the decompiled pseudocode we see three checks being in place, the first one being a the `strstr(user_input, needle) != user_input`

```
.data:0000000000004050 needle          dq offset aHkcert24     ; DATA XREF: main+43↑r
.data:0000000000004050                                         ; "hkcert24{"
```

if the condition mentioned above is true we fail, meaning we need the input to start with `hkcert24{` - not surprising...

## Second check 

Next we check for certain characters in the input at certain indexes 

```C
if ( strchr(user_input, 125) != &user_input[v6 - 1]
    || user_input[v6 - 2] != 49
    || user_input[v6 - 5] + user_input[v6 - 4] + user_input[v6 - 3] != 300
    || 2 * user_input[v6 - 5] + user_input[v6 - 4] + 2 * user_input[v6 - 3] != 496
    || 2 * user_input[v6 - 5] + 3 * user_input[v6 - 4] + user_input[v6 - 3] != 607 )
```

We need all of these to match. Otherwise we get the `Bye` message and fail again. 

## Third check 

The final check is this piece of code we are xoring the user output with a buffer in the `.rodata` section with and comparing the result with another buffer in `.rodata` section of the binary. If this succeeds, we succesfully recovered the flag.

```C
 v5 = 0;
  for ( i = 0; i < v6 - 14 && (*((char *)some_xor + i) ^ user_input[i + 9]) == xor_result[i]; ++i )
    ++v5;
  if ( v5 == 28 && i == v6 - 14 )
  {
    printf("You are now verified!\n");
    return 0;
  }
```

## Solve script

Putting all of the info above together we hack together a quick solve script to find the solution.


```python
# Step 1: Extract data from unk_200E (prefix_hkcert24)
prefix_hkcert24 = [
    0xCE, 0x21, 0xDB, 0x64, 0xD1, 0x50, 0xE0, 0x1B,
    0x0D, 0x3E, 0xFB, 0x0A, 0x52, 0x2F, 0x94, 0x9D,
    0xAF, 0xB1, 0x58, 0x6B, 0x8A, 0xEE, 0xC1, 0xF0,
    0xFC, 0x19, 0x0A, 0xE3
]

# Step 2: Extract data from lookup_buffer
lookup_buffer = [
    0xBD, 0x10, 0xB6, 0x50, 0xBD, 0x35, 0xBF, 0x78,
    0x7F, 0x0A, 0x98, 0x61, 0x1F, 0x1C, 0xCB, 0xA9,
    0xF0, 0xD9, 0x6C, 0x05, 0xEE, 0xAC, 0xB8, 0x98,
    0x9D, 0x77, 0x3C, 0xBC
]

# Step 3: Initialize the user_input array
user_input = [''] * 42  # Total length of the flag is 42 characters

# Step 4: Set the known parts of the flag
# Positions 0 to 8 correspond to "hkcert24{"
user_input[0:9] = list('hkcert24{')

# Compute positions 9 to 36 using the XOR formula
for i in range(28):
    xor_result = prefix_hkcert24[i] ^ lookup_buffer[i]
    user_input[i + 9] = chr(xor_result)

# Set the last five characters based on the solved values
user_input[37] = 'c'
user_input[38] = 'h'
user_input[39] = 'a'
user_input[40] = '1'
user_input[41] = '}'

# Step 5: Verify the constraints from the code
length_of_flag = len(user_input)
a = ord(user_input[length_of_flag - 5])
b = ord(user_input[length_of_flag - 4])
c = ord(user_input[length_of_flag - 3])

# Constraint checks
assert a + b + c == 300, "Constraint 1 failed"
assert 2*a + b + 2*c == 496, "Constraint 2 failed"
assert 2*a + 3*b + c == 607, "Constraint 3 failed"

# Step 6: Construct and print the flagx`
flag = ''.join(user_input)
print(flag)
```

and we get the solution 

```bash
$ python3 solve.py
hkcert24{s1m4le_cr4ckM3_4_h4ndByhan6_cha1}

```
