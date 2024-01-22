---
title: MAPNACTF 2024 - Novel Reader Challenges
description: Writeup for Novel Reader and Novel Reader 2 web challenges
date: 2024-01-20
tldr: Exploiting path traversal and a logic flaw in word balance calculation to read flags from a novel reading application.
draft: false
tags: ["ctf", "writeup", "Web"]
---

# Novel Reader Challenges

## Novel Reader

### Challenge Overview

The challenge involved a web application for reading novels, with a specific restriction that only public novels could be read.

### Exploit

The key to this challenge was URL decoding. The code snippet:

```python
name = unquote(name)
if(not name.startswith('public/')):
    return {'success': False, 'msg': 'You can only read public novels!'}, 400
```

suggested that we could use URL encoding to bypass the restriction. The path had to start with `public/`, so we performed path traversal using this payload: `public/%252e%252e/%252e%252e/flag.txt`.

### Retrieving the Flag

By making a request to `/api/read/public/%252e%252e/%252e%252e/flag.txt`, we successfully retrieved the flag:

```
Flag: MAPNA{uhhh-1-7h1nk-1-f0r607-70-ch3ck-cr3d17>0-4b331d4b}
```

## Novel Reader 2

### Challenge Overview

The second part of the challenge required reading "A-Secret-Tale.txt" using the same directory traversal technique. However, the flag was located near the end of the tale, and we needed more credits to access it.

### Exploit

The application allowed charging through an API, but we could only accumulate a maximum of 11 word_balance, which was insufficient. The breakthrough came when we discovered we could supply a negative `nwords` value to the `/api/charge` endpoint, exploiting the poor validation of credit and setting `words_balance` to -1.

### Retrieving the Flag

This exploit resulted in reading the novel from the beginning to almost the end, due to the `buf[0:-1]` slice in the code:

```python
buf = readFile(name).split(' ')
buf = ' '.join(buf[0:session['words_balance']])+'... Charge your account to unlock more of the novel!'
```

And thus, we retrieved the second flag:

```
{
"msg": "Once a upon time there was a flag. The flag was read like this: MAPNA{uhhh-y0u-607-m3-4641n-3f4b38571}.... Charge your account to unlock more of the novel!",
"success": true
}
```

### Conclusion

These challenges demonstrated the importance of thoroughly validating user inputs and the potential dangers of improper handling of URL encoding and arithmetic operations in web applications.
