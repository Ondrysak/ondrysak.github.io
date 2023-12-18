---
title: ImaginaryCTF 2023 - web
description: Writeup for web forensic challenge
date: 2023-12-18
tldr: We recovered this file from the disk of a potential threat actor. Can you find out what they were up to?
draft: false
tags: ["ctf", "writeup", "forensic"]
---

# web

We have are given a `.mozzila` folder. Need to find something???

## Grepping for flag

Grepping the whole folder for flag format gives us nothing intresting 

## Grepping for URLS

Grepping for urls shows bunch of random links even after we filter our some of the obvious decoys we still od not see anything interesting

```
grep -rEo "(http|https)://[a-zA-Z0-9./?=_%:-]*" | sort -u | grep -v -e wayfair -e wiki -e nike -e kors -e rakuten -e office -e armour -e saks -e ulta
```

## Yo teach app seems promising 


```
{"JWT":false,"Profile":false}
{"name":"hacker42","uuid":"c876d4f0-31bc-4a62-866a-0d8fd26e75d9"}
how did they get you to be so realistic lol. you almost fooled me 
```


```
┌──(kali㉿kali)-[/mnt/ctfk/forensic_web/.mozzila/firefox]
└─$ dumpzilla --Passwords 8ubdbl3q.default       

=============================================================================================================
== Decode Passwords     
============================================================================================================
=> Source file: /mnt/ctfk/forensic_web/.mozzila/firefox/8ubdbl3q.default/logins.json
=> SHA256 hash: 15dedec5a0290af97085e31a7db0a2ab71fa98982b8e2cc266c3271c01eb714f

Web: https://yoteachapp.com
Username: 
Password: UeMBYIbgPqNiSWzOVguTbccMOnLirDoEGTjgiqNrbOvwzynbyN


=============================================================================================================
== Passwords            
============================================================================================================
=> Source file: /mnt/ctfk/forensic_web/.mozzila/firefox/8ubdbl3q.default/logins.json
=> SHA256 hash: 15dedec5a0290af97085e31a7db0a2ab71fa98982b8e2cc266c3271c01eb714f

Web: https://yoteachapp.com
User field: 
Password field: 
User login (crypted): MDIEEPgAAAAAAAAAAAAAAAAAAAEwFAYIKoZIhvcNAwcECJs6PTFwzrMiBAiRmXcD4tn3bw==
Password login (crypted): MGIEEPgAAAAAAAAAAAAAAAAAAAEwFAYIKoZIhvcNAwcECBZPCW+NjkpUBDieso9w5lPvD85RNcErLbGTXdamyji7ZKcL9FHxjnvt1WqwcVCsOETgCWCgwCg1jJmAW/MYugOoqQ==
Created: 2023-07-09 18:53:56
Last used: 2023-07-09 18:53:56
Change: 2023-07-09 18:53:56
Frequency: 1


===============================================================================================================
== Total Information
==============================================================================================================

Total Decode Passwords     : 1
Total Passwords            : 1
```
The credentials we recovered are not working to login to the app so we keep looking until we find 

URL: https://yoteachapp.com/supersecrethackerhideout

Using the recovered password as admin password we get into a room with admin access and see the following message

Good
```
[*] flag found: ictf{behold_th3_forensics_g4untlet_827b3f13}
```
