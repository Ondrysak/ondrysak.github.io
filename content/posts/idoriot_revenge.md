---
title: ImaginaryCTF 2023 - Idoriot-revenge
description: Writeup for Idoriot-revenge web challenge
date: 2023-12-18
tldr: The idiot who made it, made it so bad that the first version was super easy. It was changed to fix it.
draft: false
tags: ["ctf", "writeup", "web"]
---

# idoriot-revenge

The idiot who made it, made it so bad that the first version was super easy. It was changed to fix it.

## Leak the source by logging in

Welcome, User ID: 947778829
Source Code



```php
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
$admin = $db->query('SELECT * FROM users WHERE username = "admin" LIMIT 1')->fetch();

// Check user_id
if (isset($_GET['user_id'])) {
    $user_id = (int) $_GET['user_id'];
    // Check if the user is admin
    if ($user_id == "php" && preg_match("/".$admin['username']."/", $_SESSION['username'])) {
        // Read the flag from flag.txt
        $flag = file_get_contents('/flag.txt');
        echo "<h1>Flag</h1>";
        echo "<p>$flag</p>";
    }
}

// Display the source code for this file
echo "<h1>Source Code</h1>";
highlight_file(__FILE__);
?>
```


## conditions we are trying to satisfy

### user_id

This time the user_id is supplie via GET param?! wtf 

`$_GET['user_id']`

`$user_id == "php"` pretty much equal to `$user_id == 0`

After converting the `$_GET['user_id']` to an integer using `(int)`, any non-numeric value in the `$_GET['user_id']` will be converted to `0`. This is because `(int)` will try to parse the string until it encounters a non-numeric character, and any characters after that will be ignored. Therefore, any non-numeric value would satisfy the condition `$user_id = "php";` after the conversion to `(int)` because both sides will have a value of `0`.

For example, if you have a URL like: `http://idoriot-revenge.chal.imaginaryctf.org/index.php?user_id=abc`, the value of `$user_id` after the conversion will be `0`, and the condition `$user_id == "php";` will evaluate to `true`.

### username

```
preg_match("/".$admin['username']."/", $_SESSION['username'])
```

username contains the substring `admin` easy we just can create username like `kakadmin` to pass.

## get the flag



http://idoriot-revenge.chal.imaginaryctf.org/index.php?user_id=kokot

```
Flag
ictf{this_ch4lleng3_creator_1s_really_an_idoriot}
```