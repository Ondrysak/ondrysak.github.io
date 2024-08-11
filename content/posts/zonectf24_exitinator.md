---
title: CTFZone 2024 Quals - exitinator
description: Writeup for exitinator rev challenge.
date: 2024-08-11
tldr: Byte rotation via libc gadgets.
draft: false
tags: ["ctf", "writeup", "rev", "libc", "rop"]
---

# exitinator

It was a scorching hot Saturday, the kind of day that makes you want to graba few teammates and hide away indoors with the air conditioning on full blast. This was the easiest `rev` challenge, so I decided to tackle it while team was busy decoding SSTV signals in the hardware part of the competition.

> You cannot escape. Flag is CTFZone{flag_from_binary}

You can download the binary file here: [exitinator.zip](https://krakenfiles.com/view/5zjWWNXJwm/file.html)

## Static analysis - attempting to reverse the binary 

We are fortunate and the binary has debugging symbols making our job a bit easier, let's start by having a look at the `main` function.

```C
int __fastcall __noreturn main(int argc, const char **argv, const char **envp)
{
  size_t v3; // rax
  int i; // [rsp+4h] [rbp-4Ch]
  void *libc; // [rsp+8h] [rbp-48h]
  exit_function_list *entry; // [rsp+18h] [rbp-38h]
  unsigned __int8 flag[22]; // [rsp+20h] [rbp-30h] BYREF
  unsigned __int64 v8; // [rsp+38h] [rbp-18h]

  v8 = __readfsqword(0x28u);
  *(_QWORD *)flag = 0x9BA1B93173893C99LL;
  *(_QWORD *)&flag[8] = 0x33A13699915F2781LL;
  *(_QWORD *)&flag[14] = 0x86AB39137D3733A1LL;
  libc = get_libc_base();
  v3 = strlen((const char *)flag);
  memcpy(&buffer[80], flag, v3);
  memcpy(&buffer[256], "Correct flag u win", 0x12uLL);
  memcpy(&buffer[336], "Ha-ha u loooose", 0xFuLL);
  set_table(libc);
  get_decoded_xorx();
  entry = get_exit_func_entry();
  overwrite_entry(entry, table[0], buffer);
  for ( i = 0; i < strlen((const char *)flag); ++i )
    overwrite_entry(entry, table[i % 6 + 3], &buffer[i]);
  overwrite_entry(entry, table[9], buffer);
  exit(5);
}
```

we see a few more functions we need to understand before attempting to solve this. Let's go top down and look into `get_libc_base` next, it indeed seems to be just returning `libc` base.

```C
void *__cdecl get_libc_base()
{
  return &memcpy - 201024;
}
```

next we move to `set_table` function which seems to be setting up a table of pointers into `libc` except for the last one which points somewhere in our binary instead.

```C
void __cdecl set_table(void *libc)
{
  table[0] = (char *)libc + 552816;
  table[1] = (char *)libc + 733328;
  table[2] = (char *)libc + 555728;
  table[3] = (char *)libc + 1585922;
  table[4] = (char *)libc + 1580596;
  table[5] = (char *)libc + 1432563;
  table[6] = (char *)libc + 240608;
  table[7] = (char *)libc + 1163270;
  table[8] = (char *)libc + 1112219;
  table[9] = &algn_11A9[6];
}
```

this does not tell us much yet lets continue exploring the functions being used, next up we have `get_decoded_xorx` and  `get_exit_func_entry` - one thing to notice is that it only performs its magic once and returns the same value saved in `xorx` for any subsequent call. One thing to notice is that we are getting a `exit_function_list` and using that to compute the value. Thats interesting but of no significant impact yet.

```C
uint64_t __cdecl get_decoded_xorx()
{
  void (*func_ptr)(void); // [rsp+8h] [rbp-18h]

  if ( !xorx )
  {
    func_ptr = get_exit_func_entry()->fns[0].func.at;
    xorx = __ROR8__(func_ptr, 17) ^ ((unsigned __int64)get_ld_base() + 21376);
  }
  return xorx;
}

exit_function_list *__cdecl get_exit_func_entry()
{
  return (exit_function_list *)((char *)get_libc_base() + 2117568);
}
```


now the `overwrite_entry` function seems to be the meat and potatoes of this challenge. We seem to be computing some pointers and stacking them in some list. But we need to investigate further to understand why and how, notice that so far we do not see the functions being executed in any way, which is suspicious as after `overwrite_entry` is executed multiple times in a loop the only other thing that is called is `exit(5)` at the very end of `main`.

```C
void __cdecl overwrite_entry(exit_function_list *entry, void *addr, void *arg)
{
  unsigned __int64 v3; // rbx
  void (*new_func)(void); // [rsp+28h] [rbp-18h]

  v3 = (((unsigned __int64)addr + 256) ^ get_decoded_xorx()) << 17;
  new_func = (void (*)(void))(v3 | ((((unsigned __int64)addr + 256) ^ get_decoded_xorx()) >> 47));
  entry->fns[idx].func.at = new_func;
  entry->fns[idx].flavor = 4LL;
  entry->fns[idx].func.on.arg = arg;
  if ( idx == 31 )
    entry->idx = 32LL;
  --idx;
}
```

##  Registering a functions to be executed on exit

After reading a the code above a bit we start stepping trough the code and notice that all the stuff we computed earlier actually gets executed **after** `exit(5)` is called, that's unusual to say at least. Reading the glibc source reveals a but more about what is happening here.

```C
/* Copyright (C) 1991-2024 Free Software Foundation, Inc.
   This file is part of the GNU C Library.
   The GNU C Library is free software; you can redistribute it and/or
   modify it under the terms of the GNU Lesser General Public
   License as published by the Free Software Foundation; either
   version 2.1 of the License, or (at your option) any later version.
   The GNU C Library is distributed in the hope that it will be useful,
   but WITHOUT ANY WARRANTY; without even the implied warranty of
   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
   Lesser General Public License for more details.
   You should have received a copy of the GNU Lesser General Public
   License along with the GNU C Library; if not, see
   <https://www.gnu.org/licenses/>.  */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pointer_guard.h>
#include <libc-lock.h>
#include <set-freeres.h>
#include "exit.h"
/* Initialize the flag that indicates exit function processing
   is complete. See concurrency notes in stdlib/exit.h where
   __exit_funcs_lock is declared.  */
bool __exit_funcs_done = false;
/* Call all functions registered with `atexit' and `on_exit',
   in the reverse of the order in which they were registered
   perform stdio cleanup, and terminate program execution with STATUS.  */
void
attribute_hidden
__run_exit_handlers (int status, struct exit_function_list **listp,
		     bool run_list_atexit, bool run_dtors)
{
  /* First, call the TLS destructors.  */
  if (run_dtors)
    call_function_static_weak (__call_tls_dtors);
  __libc_lock_lock (__exit_funcs_lock);
  /* We do it this way to handle recursive calls to exit () made by
     the functions registered with `atexit' and `on_exit'. We call
     everyone on the list and use the status value in the last
     exit (). */
  while (true)
    {
      struct exit_function_list *cur;
    restart:
      cur = *listp;
      if (cur == NULL)
	{
	  /* Exit processing complete.  We will not allow any more
	     atexit/on_exit registrations.  */
	  __exit_funcs_done = true;
	  break;
	}
      while (cur->idx > 0)
	{
	  struct exit_function *const f = &cur->fns[--cur->idx];
	  const uint64_t new_exitfn_called = __new_exitfn_called;
	  switch (f->flavor)
	    {
	      void (*atfct) (void);
	      void (*onfct) (int status, void *arg);
	      void (*cxafct) (void *arg, int status);
	      void *arg;
	    case ef_free:
	    case ef_us:
	      break;
	    case ef_on:
	      onfct = f->func.on.fn;
	      arg = f->func.on.arg;
	      PTR_DEMANGLE (onfct);
	      /* Unlock the list while we call a foreign function.  */
	      __libc_lock_unlock (__exit_funcs_lock);
	      onfct (status, arg);
	      __libc_lock_lock (__exit_funcs_lock);
	      break;
	    case ef_at:
	      atfct = f->func.at;
	      PTR_DEMANGLE (atfct);
	      /* Unlock the list while we call a foreign function.  */
	      __libc_lock_unlock (__exit_funcs_lock);
	      atfct ();
	      __libc_lock_lock (__exit_funcs_lock);
	      break;
	    case ef_cxa:
	      /* To avoid dlclose/exit race calling cxafct twice (BZ 22180),
		 we must mark this function as ef_free.  */
	      f->flavor = ef_free;
	      cxafct = f->func.cxa.fn;
	      arg = f->func.cxa.arg;
	      PTR_DEMANGLE (cxafct);
	      /* Unlock the list while we call a foreign function.  */
	      __libc_lock_unlock (__exit_funcs_lock);
	      cxafct (arg, status);
	      __libc_lock_lock (__exit_funcs_lock);
	      break;
	    }
	  if (__glibc_unlikely (new_exitfn_called != __new_exitfn_called))
	    /* The last exit function, or another thread, has registered
	       more exit functions.  Start the loop over.  */
	    goto restart;
	}
      *listp = cur->next;
      if (*listp != NULL)
	/* Don't free the last element in the chain, this is the statically
	   allocate element.  */
	free (
cur);
    }
  __libc_lock_unlock (__exit_funcs_lock);
  if (run_list_atexit)
    call_function_static_weak (_IO_cleanup);
  _exit (status);
}
void
exit (int status)
{
  __run_exit_handlers (status, 
&__exit_funcs, true, true);
}
libc_hidden_def (exit)

```

so basically what we are doing is calling the gadgets in the `overwrite_entry` added them as the program exits deffering all of the execution until the code above gets executed.

> Call all functions registered with `atexit` and `on_exit` ,in the reverse of the order in which they were registered perform stdio cleanup, and terminate program execution with STATUS.  

We are however registering them manually by modifying the structure with `overwrite_entry` so they are called in normal order.

## Dynamic analysis - it's time for gdb

Let's load the binary in `gdb` we are actually using `pwndbg` too as it has some cool additional features. We need to handle loading the included `so` file to ensure it works as the binary heavily relies on hardcoded offsets.

```bash
ls -lhta
-rwxrwxrwx 1 notdi notdi  29K Aug  3 21:00 exitinator
-rwxrwxrwx 1 notdi notdi 232K Jul 30 17:30 ld-linux-x86-64.so.2
-rwxrwxrwx 1 notdi notdi 2.1M Jul 30 17:30 libc.so.6
```

now its time to get our hands dirty with `gdb`

```bash
gdb --ex=run --args env LD_LIBRARY_PATH=$PWD ./exitinator
pwndbg> checksec
File:     /rev/exitinator/exitinator
Arch:     amd64
RELRO:    Full RELRO
Stack:    No canary found
NX:       NX enabled
PIE:      PIE enabled
RUNPATH:  b'./'
```

first lets set a breakpoint in `overwrite_entry` and see if its really doing what we think - registering functions to be executed when the program exits.

```bash
pwndbg> break overwrite_entry
Breakpoint 1 at 0x55555555568e
pwndbg> continue
# move to next ret instruction to make sure we are end of the function and one function should be registered
pwndbg> nextret
pwndbg> print entry
$1 = (exit_function_list *) 0x7ffff7faffc0
```
now we inspect the structure to see if there really is something registered 


```bash
pwndbg> print *entry
$2 = {
  next = 0x0,
  idx = 32,
  fns = {{
      flavor = 4,
      func = {
        at = 0x1f0dd65749acc1c6,
        on = {
          fn = 0x1f0dd65749acc1c6,
          arg = 0x0
        },
        cxa = {
          fn = 0x1f0dd65749acc1c6,
          arg = 0x0,
          dso_handle = 0x0
        }
      }
    }, {
      flavor = 0,
      func = {
        at = 0x0,
        on = {
          fn = 0x0,
          arg = 0x0
        },
        cxa = {
          fn = 0x0,
          arg = 0x0,
          dso_handle = 0x0
        }
      }
    } <repeats 30 times>, {
      flavor = 4,
      func = {
        at = 0x1f0dd6684e4cc1c6,
        on = {
          fn = 0x1f0dd6684e4cc1c6,
          arg = 0x555555558360
        },
        cxa = {
          fn = 0x1f0dd6684e4cc1c6,
          arg = 0x555555558360,
          dso_handle = 0x0
        }
      }
    }}
}
```

we know `overwrite_entry` gets executed many times as we iterate over the `buffer` in `main` so let's check if doing `continue` a few times make the list grow.

```bash
pwndbg> info break
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x000055555555568e <overwrite_entry+8>
        breakpoint already hit 5 times
$3 = {
  next = 0x0,
  idx = 32,
  fns = {{
      flavor = 4,
      func = {
        at = 0x1f0dd65749acc1c6,
        on = {
          fn = 0x1f0dd65749acc1c6,
          arg = 0x0
        },
        cxa = {
          fn = 0x1f0dd65749acc1c6,
          arg = 0x0,
          dso_handle = 0x0
        }
      }
    }, {
      flavor = 0,
      func = {
        at = 0x0,
        on = {
          fn = 0x0,
          arg = 0x0
        },
        cxa = {
          fn = 0x0,
          arg = 0x0,
          dso_handle = 0x0
        }
      }
    } <repeats 27 times>, {
      flavor = 4,
      func = {
        at = 0x1f0dd64f174ac1c6,
        on = {
          fn = 0x1f0dd64f174ac1c6,
          arg = 0x555555558362
        },
        cxa = {
          fn = 0x1f0dd64f174ac1c6,
          arg = 0x555555558362,
          dso_handle = 0x0
        }
      }
    }, {
      flavor = 4,
      func = {
        at = 0x1f0dd64b90c4c1c6,
        on = {
          fn = 0x1f0dd64b90c4c1c6,
          arg = 0x555555558361
        },
        cxa = {
          fn = 0x1f0dd64b90c4c1c6,
          arg = 0x555555558361,
          dso_handle = 0x0
        }
      }
    }, {
      flavor = 4,
      func = {
        at = 0x1f0dd64bc6a8c1c6,
        on = {
          fn = 0x1f0dd64bc6a8c1c6,
          arg = 0x555555558360
        },
        cxa = {
          fn = 0x1f0dd64bc6a8c1c6,
          arg = 0x555555558360,
          dso_handle = 0x0
        }
      }
    }, {
      flavor = 4,
      func = {
        at = 0x1f0dd6684e4cc1c6,
        on = {
          fn = 0x1f0dd6684e4cc1c6,
          arg = 0x555555558360
        },
        cxa = {
          fn = 0x1f0dd6684e4cc1c6,
          arg = 0x555555558360,
          dso_handle = 0x0
        }
      }
    }}
}
```

indeed the list of functions is growing, we are on the right path. Next lets try to inspect the `table` to figure out what are functions being called, we may just try to do that from the `entry` but the pointers are mangled so going from non-mangled pointers in `table` is easier. One thing I totally missed is that we are adding a static offset `0x100` to the pointers in `table` in `overwrite_entry` so we need to account for that. Let's get started.


```bash
pwndbg> print table
$4 = {0x7ffff7e31f70 <_IO_getline_info+176>, 0x7ffff7e5e090 <strchr+16>, 0x7ffff7e32ad0 <_IO_proc_close+400>, 0x7ffff7f2e302, 0x7ffff7f2ce34, 0x7ffff7f08bf3 <gethostbyname_r+163>, 0x7ffff7de5be0 <setlocale+1328>, 0x7ffff7ec7006 <ttyname+54>, 0x7ffff7eba89b, 0x5555555551af, 0x0 <repeats 90 times>}
# however the pointers are off by 0x100 so this does really help yet
pwndbg> print table[0]+0x100
$6 = (void *) 0x7ffff7e32070 <gets>
pwndbg> print table[1]+0x100
$7 = (void *) 0x7ffff7e5e190 <strcmp>
pwndbg> print table[2]+0x100
$8 = (void *) 0x7ffff7e32bd0 <puts>
pwndbg> print table[3]+0x100
$9 = (void *) 0x7ffff7f2e402
pwndbg> print table[4]+0x100
$10 = (void *) 0x7ffff7f2cf34
pwndbg> print table[5]+0x100
$11 = (void *) 0x7ffff7f08cf3 <gethostbyname_r+419>
pwndbg> print table[6]+0x100
$12 = (void *) 0x7ffff7de5ce0 <setlocale+1584>
pwndbg> print table[7]+0x100
$13 = (void *) 0x7ffff7ec7106 <ttyname_r+134>
pwndbg> print table[8]+0x100
$14 = (void *) 0x7ffff7eba99b
pwndbg> print table[9]+0x100
$15 = (void *) 0x5555555552af <gadjet>
```

ok, thats nice the first three pointers are actuall pointing to beginning of some reasonable functions which even make sense for our challenge, but whats up with the remaining 6 and the last one? Let's look into `gadjet` first as we see it defined in our binary. It's the function that does the final comparison of the changes we made to the `buffer` where we save the `gets` input and applying modifications after.

```C
void __cdecl gadjet(void *buffer)
{
  if ( !strcmp((const char *)buffer, (const char *)buffer + 80) )
    puts((const char *)buffer + 256);
  else
    puts((const char *)buffer + 336);
}
```

Now we have a mental map of whats happening there the order functions/gadgets are executed on `exit(5)` goes like this 

- `gets` is called to read the flag from user into `buffer`
- gadgets from `libc` are called while iterating over the input stored in `buffer`
- finally we call the `gadjet` function above to check if we should print `Correct flag u win` or `Ha-ha u loooose`

Now let's try to inspect the gadgets in libc to see what they actually are. We notice they are all pointing to `ror` instructions.

> ROR Instruction : ROR destination, count. The ROR instruction shifts each bit to the right, with the lowest bit copied in the Carry flag and into the highest bit. 

```bash
pwndbg> x/2i table[3]+0x100
   0x7ffff7f2e402:      ror    BYTE PTR [rdi],0x95
   0x7ffff7f2e405:      ret
pwndbg> x/2i table[4]+0x100
   0x7ffff7f2cf34:      ror    BYTE PTR [rdi],0x89
   0x7ffff7f2cf37:      ret
pwndbg> x/2i table[5]+0x100
   0x7ffff7f08cf3 <gethostbyname_r+419>:        ror    BYTE PTR [rdi],0x85
   0x7ffff7f08cf6 <gethostbyname_r+422>:        ret
pwndbg> x/2i table[6]+0x100
   0x7ffff7de5ce0 <setlocale+1584>:     ror    BYTE PTR [rdi],0x84
   0x7ffff7de5ce3 <setlocale+1587>:     ret
pwndbg> x/2i table[7]+0x100
   0x7ffff7ec7106 <ttyname_r+134>:      ror    BYTE PTR [rdi],0x88
   0x7ffff7ec7109 <ttyname_r+137>:      ret
pwndbg> x/2i table[8]+0x100
   0x7ffff7eba99b:      ror    BYTE PTR [rdi],0x8e
   0x7ffff7eba99e:      ret
```

## Conclusion - Solve script 

Now we finally understand the mechanism used to encode the flag, fortunately for us the rotation operation can be easily reversed, simply by rotating the other direction. For convinience we can try implementing that all that in python, even tho PIE is enabled and pointers are mangled we managed to restore all the information needed by using `gdb` to dynamically analyze the binary after we understood enough about it by doing static analysis first. ChatGPT did a solid job of writing the code for us.


```python
def rolb(value, shift):
    """Rotate left operation on a byte (8 bits)."""
    shift = shift % 8  # Ensure the shift is within the bounds of 0-7
    return ((value << shift) & 0xFF) | (value >> (8 - shift))

def main():
    # Initialize the flag array with the provided values
    flag = bytearray(128)
    flag[0:8] = (0x9BA1B93173893C99).to_bytes(8, byteorder='little')
    flag[8:16] = (0x33A13699915F2781).to_bytes(8, byteorder='little')
    flag[14:22] = (0x86AB39137D3733A1).to_bytes(8, byteorder='little')[0:8]
    
    # Rotation values
    rors = [0x95, 0x89, 0x85, 0x84, 0x88, 0x8e]
    
    # Length of the flag (based on non-zero initialized values)
    n = len(flag[:22])
    
    # Perform the rotation on each byte of the flag
    for i in range(n):
        flag[i] = rolb(flag[i], rors[i % 6])
    
    # Convert the byte array to a string and print the result
    print(flag[:22].decode('latin1'))

if __name__ == "__main__":
    main()
```

Now we just execute the solve script and hope

```bash
 python3 .\solve.py
3x171n470r_d3l437_bruh
$ ./exitinator
3x171n470r_d3l437_bruh
Correct flag u win
```

Thats it we got the flag!