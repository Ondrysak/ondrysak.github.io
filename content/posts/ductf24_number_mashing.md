---
title: DownUnderCTF 2024 - number mashing
description: Writeup for number mashing rev challenge
date: 2024-07-08
tldr: Reversing ARM executable with ghidra to find an integer overflow.
draft: false
tags: ["ctf", "writeup", "rev", "overflow"]
---


## number mashing /beginner

We get a binary that asks us to enter two numbers and we always get a `Nope!` after we make a few attempts. We start reversing the binary with `ghidra` as its built for arm. In a bit we figure out that this actually has no mathematical solution, thus we have to think outside of the box a bit... eventually we spot a possible overflow as marked in the pseudocode below.


```C
  if (((numba1 == 0) || (numba2 == 0)) || (numba2 == 1)) {
    puts("Nope!");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  numba1_div_numba2 = 0;
  if (numba2 != 0) {
    numba1_div_numba2 = numba1 / numba2; // <----- OVERFLOW HERE
  }
  if (numba1_div_numba2 != numba1) {
    puts("Nope!");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
```

With that out of the way we just need to figure out what numbers to use to cause an overflow tha would result in

```
numba1/numba2 == numba1_div_numba2 == numba1
```

We are obviously looking for some corner case - this is where I got the idea of doing this

```
INT32_MIN/-1 = 2147483648 ~= INT32_MIN
```

due to `INT32` having only a single zero the limits are not symmetrical - `INT32_MIN + INT32_MAX == -1`. We use to solve this challenge.

```
nc 2024.ductf.dev 30014
Give me some numbers: -2147483648 -1
Correct! DUCTF{w0w_y0u_just_br0ke_math!!}
```

