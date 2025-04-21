---
title: UMASSCTF 2025 - Bullet Dodge
description: Writeup for the Bullet Dodge reverse engineering challenge
date: 2025-04-21
tldr: Unity reverse engineering and patching to get the flag
draft: false
tags: ["ctf", "writeup", "rev", "unity", "dnspy"]
---

## Introduction

The challenge "Bullet Dodge" from UMASSCTF 2025 looks impossible at first: you need to reach a score of **1,000,000** to win. That's not just difficult—it feels intentionally unachievable. The implication? There must be a backdoor.

We start digging into the Unity game's internals using `dnSpy`, focusing on `Assembly-CSharp.dll`, where all the juicy stuff lives in Unity C# games.

## Static Analysis with dnSpy

For this challenge, we used [dnSpyEx](https://github.com/dnSpyEx/dnSpy), a modern fork of the original dnSpy project that’s actively maintained and includes better .NET 5+ support, IL editing, and decompilation improvements.

This tool is essential for reverse engineering Unity games, as it allows you to:
- Browse and decompile `Assembly-CSharp.dll` in a user-friendly UI
- Edit C# code directly and recompile it live
- Inspect Unity types like `MonoBehaviour`, `GameObject`, `TextMeshProUGUI`, and more

dnSpyEx made it easy to locate the `b()` method that renders the flag, understand how obfuscation is handled, and quickly patch the code to trigger a win state. It’s basically the IDA Pro of Unity reversing — but free and way friendlier.

Two interesting classes

```csharp
using System;
using System.Text;
using TMPro;
using UnityEngine;

public class f : MonoBehaviour
{
        public TextMeshProUGUI b;

        public a a;

        public bool d;

        public int e;

        public bool p = true;

        public static string q = "kp";

        private int o = 1;

        private string h = "";

        private void Start()
        {
                b.text = j(l("U0NPUkU6IA==")) + k(e);
                d = true;
        }

        public string j(byte[] a)
        {
                return Encoding.UTF8.GetString(a);
        }

        private void Update()
        {
                string text = j(l("QlU=")) + w(j(l("Snp3PQ=="))) + w("LiQ=");
                if (d)
                {
                        e += o;
                        b.text = j(l("U0NPUkU6IA==")) + k(e);
                }
                string inputString = Input.inputString;
                for (int i = 0; i < inputString.Length; i++)
                {
                        h += inputString[i];
                        if (h.Length > text.Length)
                        {
                                h = h.Substring(1);
                        }
                        else if (h.Length == text.Length)
                        {
                                k(4);
                        }
                        else
                        {
                                j(l("b21vbW9tbW9t"));
                        }
                        if (h.ToUpper() != text)
                        {
                                k(13);
                        }
                        else
                        {
                                g();
                        }
                }
                if (e >= s())
                {
                        a.b();
                        o = 0;
                        p = false;
                }
        }

        public byte[] l(string a)
        {
                return Convert.FromBase64String(a);
        }

        public string w(string e)
        {
                byte[] array = l(e);
                for (int i = 0; i < array.Length; i++)
                {
                        array[i] = r((byte)q[i], array[i]);
                }
                return j(array);
        }

        public string k(int a)
        {
                return a.ToString();
        }

        private void g()
        {
                e += s();
        }

        public byte r(byte a, byte b)
        {
                return (byte)(a ^ b);
        }

        public int s()
        {
                return 100000;
        }
}
```
which will reaching to this one when we win really really hard to do `a.b();`

```csharp
using System;
using System.Text;
using TMPro;
using UnityEngine;

public class a : MonoBehaviour
{
        public TextMeshProUGUI z;

        public f k;

        public g r;

        public string q = "uZ7pKmEoFbYL5vR2X1nJwGTa9qMc3dC=";

        public void b()
        {
                z.text = d(f(w(d(f(h(z))))));
                g();
        }

        public string w(string e)
        {
                byte[] array = kr(e);
                for (int i = 0; i < array.Length; i++)
                {
                        array[i] = y((byte)q[i], array[i]);
                }
                return j(array);
        }

        public byte y(byte a, byte b)
        {
                return (byte)(a ^ b);
        }

        public string d(byte[] a)
        {
                return Encoding.UTF8.GetString(a);
        }

        public void g()
        {
                if (k.p && r.po)
                {
                        base.gameObject.SetActive(value: true);
                }
        }

        public byte[] f(string a)
        {
                return Convert.FromBase64String(a);
        }

        public string j(byte[] a)
        {
                return Encoding.UTF8.GetString(a);
        }

        public string h(TextMeshProUGUI a)
        {
                return a.text;
        }

        public byte[] kr(string a)
        {
                return Encoding.UTF8.GetBytes(a);
        }
}   
```


We quickly find the method responsible for displaying the win flag:

```csharp
public void b() {
    this.z.text = this.d(this.f(this.w(this.d(this.f(this.h(this.z))))));
    this.g();
}
```

This `b()` method appears to be called upon winning, and it runs a sequence of transforms to decode a flag:
- Base64 decode (`f`)
- UTF-8 decode (`d`)
- XOR with hardcoded key (`w`)

There's also a condition that enables the win screen:

```csharp
public void g() {
    if (this.k.p && this.r.po) {
        base.gameObject.SetActive(true);
    }
}
```

## The Cheat Code

The game includes a cheat-code mechanic that checks keyboard input against an obfuscated string built from a mix of base64 decoding and XOR. We could reverse engineer the full code — but instead, let’s go straight to the good stuff.

## Patching to Win

Instead of trying to guess the full cheatcode and figuring out how to input it, we rewrite the `Update()` method to instantly win. Here's the patched version:

```csharp
private void Update() {
    string text = "hovno"; // New fake cheatcode
    this.g();
    this.a.b();
    this.o = 0;
    this.p = false;

    foreach (char c in Input.inputString) {
        this.h += c.ToString();
        this.g();
        this.a.b();
        this.o = 0;
        this.p = false;
    }

    if (this.e >= this.s()) {
        this.a.b();
        this.o = 0;
        this.p = false;
    }
}
```

This patched `Update()`:
- Forces `g()` (add points) and `b()` (display flag)

Run the patched game,and boom—the flag pops.

![Original chal.png](/images/bullet.png)

## Static Analysis with Strings

Alternatively, you can solve this challenge **without patching anything or even executing the binary at all** , just by extracting strings and following the transformation logic from code.

Using the `strings` utility, we can extract relevant encoded constants:

```bash
strings level0 | rg ".*=$"
Iw8GMh5cC1gTCAsDbUYIYRV0XywmLRIsbTd0FX1XcwA=
uZ7pKmEoFbYL5vR2X1nJwGTa9qMc3dC=
```

These are used in the `b()` flag-displaying method. We reverse the decoding chain with Python:

```python
import base64

# h(z) initial value
encoded_input = "Iw8GMh5cC1gTCAsDbUYIYRV0XywmLRIsbTd0FX1XcwA="
xor_key = "uZ7pKmEoFbYL5vR2X1nJwGTa9qMc3dC="  # plain string

# f() → Base64 decode
step1_bytes = base64.b64decode(encoded_input)

# d() → UTF-8 decode
step2_str = step1_bytes.decode("utf-8")

# w() → XOR string with key, key is string of chars
step2_bytes = step2_str.encode("utf-8")
xor_key_bytes = xor_key.encode("utf-8")

# XOR each byte
xored = bytes([b ^ xor_key_bytes[i] for i, b in enumerate(step2_bytes)])

# j() → UTF-8 decode
step3_str = xored.decode("utf-8")

# f() → Base64 decode
final_bytes = base64.b64decode(step3_str)

# d() → UTF-8 decode
flag = final_bytes.decode("utf-8")

print("FLAG:", flag)
```

Running the above gives:

```
FLAG: UMASS{R4N_FR0M_B1LL_o7}
```

## Conclusion

We successfully beat "Bullet Dodge" not by dodging bullets, but by diving into Unity internals with `dnSpy` and patching the C# code. Or, if you're lazy and smart, by just extracting the strings and tracing the decode steps statically.

