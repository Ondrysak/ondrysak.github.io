Title: Making world a better place with Sentry!
Date: 2018-05-07 12:01
Modified: 2018-05-07 13:01
Category: technology
Tags: technology, sentry
Slug: Sentry
Authors: Ondřej Naňka
Summary: What is Sentry, what does it do?


Who doesn't hate errors, Sentry is here to help!

Sentry is open-source error tracking that helps developers monitor and fix crashes in real time. Iterate continuously. Boost efficiency. Improve user experience.

You are asking for sure, how to use Sentry in my python app, ask no more! Here is a simple example.

First u need to make an account, the web ui is where you are gonna look if you recieve an alert!

[Signup](https://sentry.io/signup/)

If you haven’t already, start by downloading Raven. The easiest way is with pip 

```bash
pip install raven --upgrade
```

Now all you need to do is add these two lines to your code!

```python
from raven import Client
client = Client('https://<key>:<secret>@sentry.io/<project>')
```

It's done, congrats you just made world a better place.

Other supported platforms are 

+ C#
+ Cocoa
+ Cordova
+ Electron
+ Elixir
+ Go
+ Java
+ JavaScript
+ Minidump
+ Node
+ Perl
+ PHP
+ Python
+ React Native
+ Ruby
+ Rust

Sentry is also integrated with the following:

+ Asana
+ Bitbucket
+ GitHub
+ HipChat
+ Heroku
+ Jira
+ SessionStack
+ Slack
+ Splunk


More info in the docs you fool!

[Docker docs](https://sentry.io/welcome/)

