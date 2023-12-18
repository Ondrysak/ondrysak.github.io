---
title: ImaginaryCTF 2023 - snailchecker
description: Writeup for snailchecker rev challenge
date: 2023-12-18
tldr: Optimize me, if you dare. Or not. It might run if you try hard enough.
draft: false
tags: ["ctf", "writeup", "rev"]
---

# snailchecker 
Optimize me, if you dare. Or not. It might run if you try hard enough.

Attachments
https://imaginaryctf.org/r/Q2CAo#check.py

## original code

```python
#!/usr/bin/env python3
import math

def find_final_value(length,b):
    power_of_two = 2 ** int(math.log2(length))
    print("power_of_two")
    print(power_of_two)
    print("Final val candidate")

    #return power_of_two + last*2 + 1
    return (2 * b[3]) % (power_of_two/2)+(power_of_two/2) + 1

def enc_old(b):
 print("Start Input")
 print(b)
 a = [n for n in range(b[0]*2**24+b[1]*2**16+b[2]*2**8+b[3]+1)][1:]
 #print("Length")
 #print(len(a))
 #print(find_final_value(len(a),b))
 c,i = 0,0
 while len([n for n in a if n != 0]) > 1:
  i%=len(a)
  if (a[i]!=0 and c==1):
   a[i],c=0,0
  if (a[i] != 0):
   c+=1
  i += 1
 return sum(a)

import math

def enc_bad(b):
    print(b)
    num = b[0]*2**24+b[1]*2**16+b[2]*2**8+b[3] + 1
    return 2*(num - 2**math.floor(math.log2(num))) + 1


def enc(b):
    # Find value of L for the equation
    n = b[0]*2**24+b[1]*2**16+b[2]*2**8+b[3] + 1
    valueOfL = n - (2 ** (n.bit_length() - 1))
    return 2 * valueOfL - 1 

#for i in range(1,64):
#  print("sum1 -> "+str(enc_old([0,0,4,i])))
#  print("sum2 -> "+str(enc([0,0,4,i])))

print(r"""
    .----.   @   @
   / .-"-.`.  \v/
   | | '\ \ \_/ )
 ,-\ `-.' /.'  /
'---`----'----'
""")
# flag = input("Enter flag here: ").encode()

import itertools
import string
def foo(l):
     yield from itertools.product(*([l] * 4)) 

for flag in foo(string.printable):
  flag = "".join(flag)
  flag = bytes(flag, 'ascii')
  #print(flag)
  out = b''
  for n in [flag[i:i+4] for i in range(0,len(flag),4)]:
    out_new = bytes.fromhex(hex(enc(n[::-1])-1)[2:].zfill(8))
    #print(out)
    out += out_new
    #print(out)
  #print(b'j\xd0\xe0\xcad\xe0\xbe\xe6J\xd8\xc4\xde`\xe6\xbe\xda>\xc8\xca\xca^\xde\xde\xc4^\xde\xde\xdez\xe8\xe6\xde')
  #if out == b'L\xe8\xc6\xd2f\xde\xd4\xf6j\xd0\xe0\xcad\xe0\xbe\xe6J\xd8\xc4\xde`\xe6\xbe\xda>\xc8\xca\xca^\xde\xde\xc4^\xde\xde\xdez\xe8\xe6\xde':
  if b'z\xe8\xe6\xde'.startswith(out):
    print(out)
    print(flag)
    print("[*] Flag correct!")
    break
  else:
    #print("[*] Flag incorrect.")
    pass
````


## optimize the enc function

its disguised josephus problem, we can replace the enc function with constant time solution to josephus problem k=2

https://en.wikipedia.org/wiki/Josephus_problem

```python
#!/usr/bin/env python3

def enc(b):
    total_elements = b[0] * 2 ** 24 + b[1] * 2 ** 16 + b[2] * 2 ** 8 + b[3]
    print(total_elements)
    # Calculate the number of elements that will be included in the sum
    num_remaining = (total_elements + 1) // 2

    # Calculate the sum of the elements to be included
    sum_remaining = (num_remaining * (num_remaining + 1)) // 2

    return sum_remaining


print(r"""
    .----.   @   @
   / .-"-.`.  \v/
   | | '\ \ \_/ )
 ,-\ `-.' /.'  /
'---`----'----'
""")
flag = input("Enter flag here: ").encode()
out = b''
for n in [flag[i:i+4] for i in range(0,len(flag),4)]:
  out += bytes.fromhex(hex(enc(n[::-1]))[2:].zfill(8))

if out == b'L\xe8\xc6\xd2f\xde\xd4\xf6j\xd0\xe0\xcad\xe0\xbe\xe6J\xd8\xc4\xde`\xe6\xbe\xda>\xc8\xca\xca^\xde\xde\xc4^\xde\xde\xdez\xe8\xe6\xde':
 print("[*] Flag correct!")
else:
 print("[*] Flag incorrect.")

```


## flag by 4 byte chunks

and bruteforce the flag by 4 byte chunks

```
ictf
{jos
ephu
s_pr
oble
m_sp
eed?
booo
oooo
ost}
```

```
ictf{josephus_problem_speed?boooooooost}
```
there seems to be a typo, after twiddling a bit with the flag we manage to find the actual flag.
```
ictf{josephus_problem_speed_boooooooost}
```

