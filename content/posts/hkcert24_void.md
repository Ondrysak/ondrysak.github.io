
---
title: hkcert CTF 2024 - Void
description: Reversing whitespace obfusacated javascript
date: 2024-11-09
tldr: Solve of a javascript rev challenge.
draft: false
tags: ["ctf", "writeup", "rev", "javascript", "invisible"]
---

## Initial analysis

> I made a simple webpage that checks whether the flag is correct... Wait, where are the flag-checking functions?
    

After inspecting the source of the web page - we pretty much see nothing... scrolling all the way down we see a fairly dense piece of javascript, looking around a bit we see eventually figure out that this is "packed" by `invisible`.


http://aem1k.com/invisible/

https://x.com/aemkei/status/1843756978147078286


## Downloading the source code 

```bash
curl https://c09-void.hkcert24.pwnable.hk/
```

## Patch to print the decoded payload

We notice that the payload is being build and `eval` use used on it, in theory we could try to build an external unpacker, but since they way this works is it just builds the the payload into a string `f` using `String.fromCharCode` the idea was that we can just patch the original unpacker to print the payload instead of executing it.

```javascript
function \u3164(){return f="",p=[]  
,new Proxy({},{has:(t,n)=>(p.push(
n.length-1),2==p.length&&(p[0]||p[
1]||eval(f),f+=String.fromCharCode
(p[0]<<4|p[1]),p=[]),!0)})}//aem1k
```

after we patch we get the following 

```javascript
function \u3164(){return f="",p=[]  
,new Proxy({},{has:(t,n)=>(p.push(
n.length-1),2==p.length&&(p[0]||p[
1]||console.log(f),f+=String.fromCharCode
(p[0]<<4|p[1]),p=[]),!0)})}//aem1k
```

## Executing the patched javascript

```javascript
const flag = document.getElementById('flag');
flag.focus();

handleKeyPress = event => event.key === 'Enter' && check();

function check() {
    if (flag.value === 'hkcert24{j4v4scr1p7_1s_n0w_alm0s7_y3t_4n0th3r_wh173sp4c3_pr09r4mm1n9_l4ngu4g3}') {
        flag.disabled = true;
        flag.classList.add('correct');
    } else {
        flag.classList.add('wrong');
        setTimeout(() => flag.classList.remove('wrong'), 500);
    }
}
```

there does not seem to be any additional obfusaction after that and we get the flag right away.

```
hkcert24{j4v4scr1p7_1s_n0w_alm0s7_y3t_4n0th3r_wh173sp4c3_pr09r4mm1n9_l4ngu4g3}
```

