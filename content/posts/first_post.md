---
title: ImaginaryCTF 2023 - Blurry
description: Writeup for Blurry forensic challenge
date: 2023-12-18T02:01:58+02:00
tldr: Gaussian blurring is reverisble, oops!
draft: false
tags: ["ctf", "writeup", "forensics"]
---

## Blurry
we are given a blurry image of an QR code - we probably need to undo the blur

![Original chal.png](/images/chall.png)


Since we are cheap fukerz we will go with https://www.gimp.org/, after some fiddling with the default sharpening tools we are not getting anywhere.

Let's install some more complex sharpen filters https://gmic.eu/

Assuming a gaussian blur after fiddling with

```
Details -> Sharpen [Deblur] 
```

we find the right blur radius `7.7` and just push the iterations of the deblur filter until we get something squarish and readable, just going by eye here since we know how a QR code should look and we get the flag.

![Deblurred back into readable](/images/deblurred.png)


```
ictf{blurR1ng_is_n0_m4tch_4_u_2ab140c2}
```