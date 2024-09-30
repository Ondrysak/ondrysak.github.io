---
title: BuckeyeCTF 2024 - instructions
description: Writeup for instructions web challenge!
date: 2024-09-30
tldr: rev some wasm
draft: false
tags: ["ctf", "writeup", "web", "rev", "rust", "wasm"]
---

## Introduction 

At first glance, the challenge presents a seemingly static webpage with no apparent input options. After having a teammate explain what the fuck I am even looking at I get what later turns out to be a crucial piece of information the website is using `leptos`. Well If you like me do not feel any closer to knowing what to do with that information here is a hint.

https://leptos.dev/

> Leptos is a full-stack framework for building web applications in `Rust`.

![Ok](/images/weird-flex-but-ok.gif)

What’s important for us is that Leptos compiles its code into `WebAssembly (WASM)`, which is then delivered to the browser. The `JavaScript` here is quite minimal, so it's clear that the core of this challenge is hidden within the `wasm`.

## Setting up Ghidra to analyze the wasm 

We fail to find any flags that would be just simple strings, so we have to actually try to see what the code is doing.

`Ghidra` does not support `wasm` disassembly and decompilation out of the box, but after a bit of googling we find this neat ghidra plugin.

https://github.com/nneonneo/ghidra-wasm-plugin

We install the plugin, restart ghidra and now we are cooking!


## Just find the damn flag

There is an awful amount of code in the `wasm` file but eventually we bump into a function that seems to be doing a `XOR` decryption. Fortunately for `Ghidra` does a fairly good job decompiling it for into somewhat readable code.

```rust

void instructions::__App::{{closure}}::{{closure}}::{{closure}}::hf5a03e21b8bffa2b
               (undefined4 param1)

{
  int iVar1;
  undefined8 *param1_00;
  undefined8 *param1_01;
  int local_14;
  int local_10;
  undefined4 uStack_c;
  undefined8 local_8;
  
  param1_00 = (undefined8 *)__rust_alloc(0x41,1);
  if (param1_00 == (undefined8 *)0x0) {
    alloc::alloc::handle_alloc_error::he71533634a7a5ac5(1,0x41);
    do {
      halt_trap();
    } while( true );
  }
  *(undefined *)(param1_00 + 8) = 0xac;
  param1_00[7] = 0x188633df6d71cf8d;
  param1_00[6] = 0x730666847040127e;
  param1_00[5] = 0xc1685efdd3a997f1;
  param1_00[4] = 0xa4bfe80405367a2;
  param1_00[3] = 0x87d6022df5e04203;
  param1_00[2] = 0xe08329fe0f66c221;
  param1_00[1] = 0x77ef87be7bf01995;
  *param1_00 = 0x92145802662eb9ec;
  param1_01 = (undefined8 *)__rust_alloc(0x80,1);
  if (param1_01 == (undefined8 *)0x0) {
    alloc::alloc::handle_alloc_error::he71533634a7a5ac5(1,0x80);
    do {
      halt_trap();
    } while( true );
  }
  param1_01[0xf] = 0x117498b7d809291d;
  param1_01[0xe] = 0xb8027b754536d4d1;
  param1_01[0xd] = 0x531a058f804b3bc6;
  param1_01[0xc] = 0x8182ec6a54d92e36;
  param1_01[0xb] = 0x6237e5590bbcd9d6;
  param1_01[10] = 0xc1066649c8bf9111;
  param1_01[9] = 0xc465a4084c82485f;
  param1_01[8] = 0x9631e4652164dd1;
  param1_01[7] = 0x7cb403ba5543aeeb;
  param1_01[6] = 0x456202e64471734c;
  param1_01[5] = 0x9e5e30ccb8dba786;
  param1_01[4] = 0x553bceb7750c5793;
  param1_01[3] = 0xe3a336728c8d1d36;
  param1_01[2] = 0xd3e81d935051f754;
  param1_01[1] = 0x5b0b1d04a9c28e5;
  *param1_01 = 0xff243b79005ada8e;
  iVar1 = 0;
  local_10 = __rust_alloc(0x41,1);
  if (local_10 == 0) {
    alloc::raw_vec::handle_error::h604585f1a2687b06(1,0x41);
    do {
      halt_trap();
    } while( true );
  }
  while( true ) {
    *(byte *)(local_10 + iVar1) =
         *(byte *)((int)param1_01 + iVar1) ^ *(byte *)((int)param1_00 + iVar1);
    if (iVar1 == 0x40) break;
    ((byte *)(local_10 + iVar1))[1] =
         ((byte *)((int)param1_01 + iVar1))[1] ^ ((byte *)((int)param1_00 + iVar1))[1];
    iVar1 = iVar1 + 2;
  }
  core::str::converts::from_utf8::hf3dc2fd78e88c588(&local_14,local_10,0x41);
  if (local_14 == 0) {
    local_8 = CONCAT44(local_8._4_4_,param1);
    uStack_c = 0x41;
    local_14 = 0x41;
    std::thread::local::LocalKey<T>::with::h3d86c5da19e014f4
              (s_/public/ferris-1.svg$_ram_001006a4 + 0x14,&local_14);
    __rust_dealloc(param1_01,0x80,1);
    __rust_dealloc(param1_00,0x41,1);
    return;
  }
  uStack_c = 0x41;
  local_14 = 0x41;
  core::result::unwrap_failed::h9c8291f73d3ee71a
            (s_Invalid_UTF-8_ram_00100728,0xd,&local_14,&DAT_ram_00100738,&DAT_ram_00100748);
  do {
    halt_trap();
  } while( true );
}
```
One option we have here is the obvious one, we rev the rest of the code figure out what calls this function and what do we need to do trigger that condition. Easy enough but reversing `rust` is a pain in the ass, so we left that option for later.

## Reimplementing the XOR decryption in python

I was certainly not feeling like reversing any more `rust` than I already did, I could already smell the flag anyway. Writing python however, who could ever say no that especially if you have an AI overlord ready to help you with that task!

```python
import struct

# Given values from param1_00 and the extra byte
param1_00 = [
    0x92145802662eb9ec,
    0x77ef87be7bf01995,
    0xe08329fe0f66c221,
    0x87d6022df5e04203,
    0x0a4bfe80405367a2,
    0xc1685efdd3a997f1,
    0x730666847040127e,
    0x188633df6d71cf8d,
]
param1_00_extra_byte = 0xac

# Given values from param1_01
param1_01 = [
    0xff243b79005ada8e,
    0x05b0b1d04a9c28e5,
    0xd3e81d935051f754,
    0xe3a336728c8d1d36,
    0x553bceb7750c5793,
    0x9e5e30ccb8dba786,
    0x456202e64471734c,
    0x7cb403ba5543aeeb,
    0x09631e4652164dd1,
    0xc465a4084c82485f,
    0xc1066649c8bf9111,
    0x6237e5590bbcd9d6,
    0x8182ec6a54d92e36,
    0x531a058f804b3bc6,
    0xb8027b754536d4d1,
    0x117498b7d809291d,
]

# Convert the param1_00 values into bytes
param1_00_bytes = b''
for value in param1_00:
    param1_00_bytes += struct.pack('<Q', value)
param1_00_bytes += bytes([param1_00_extra_byte])

# Convert the param1_01 values into bytes and limit to 65 bytes
param1_01_bytes = b''
for value in param1_01:
    param1_01_bytes += struct.pack('<Q', value)
param1_01_bytes = param1_01_bytes[:65]

# Perform the XOR operation
flag_bytes = bytes([b1 ^ b2 for b1, b2 in zip(param1_00_bytes, param1_01_bytes)])

# Decode and print the flag
try:
    flag = flag_bytes.decode('ascii')
    print(f"The flag is: {flag}")
except UnicodeDecodeError:
    print("Failed to decode the flag.")
```

and we get the flag!

```bash
python3 sol.py
The flag is: bctf{c0mp1l1n6_ru57_m4k35_my_4ud10_570p_w0rk1n6_2a14bdd6fa28e02d}
```

We did not manage to solve the challenge during the CTF and also this probably was not the intended solution. After searching a bit we find what this challenge was really about!

> The idea was that the code check your CPU usage and if it's high enough for long enough you get the flag!

also

> You could breakpoint the creation of pressure observer in JavaScript console and call it with payload to simulate critical pressure wait 30 second and get the flag!

```
([{state:"critical"}])
```

## Conclusion

Despite not solving the "instructions" challenge during the BuckeyeCTF 2024, our post-CTF analysis led us to a deeper understanding of WebAssembly and Rust reverse engineering. By inspecting the WASM code with Ghidra and implementing the XOR decryption in Python, we successfully extracted the flag—even if this most likely wasn't the intended solution