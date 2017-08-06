---
title: Zero-width space contamination (work in progress)
date: 2017-08-06 20:08:02
tags:
---

One day I was running `docker-compose up -d`, minding my own business, when suddenly:

```
Traceback (most recent call last):
  File "bin/docker-compose", line 3, in <module>
  File "compose/cli/main.py", line 67, in main
  File "compose/cli/main.py", line 117, in perform_command
  File "compose/cli/main.py", line 922, in up
  File "compose/project.py", line 395, in up
  File "compose/service.py", line 327, in ensure_image_exists
  File "compose/service.py", line 347, in image
  File "site-packages/docker/utils/decorators.py", line 21, in wrapped
  File "site-packages/docker/api/image.py", line 262, in inspect_image
  File "site-packages/docker/api/client.py", line 202, in _url
  File "urllib.py", line 1306, in quote_plus
  File "urllib.py", line 1299, in quote
KeyError: u'\u200b'
Failed to execute script docker-compose
```

I would like to write about this error, since searching for it on Google gave me hardly any results. Looking for `KeyError: u'\u200b'` yielded four results, and none of them were related to docker-compose. Actually, they were all related to Haiku operating system, which wasn’t much help. hopefully, now this blogpost will join them.

In my case it was running `docker-compose`, but as I’ve found, it can be anything, really. So if you’re reading this, I hope I can save you a few hours of debugging.

# Analysis

Error message seems ambiguous, but let’s take a look at this `\u200b` symbol. Quick search yields following site:

[http://www.fileformat.info/info/unicode/char/200B/index.htm](http://www.fileformat.info/info/unicode/char/200B/index.htm)

Well, _zero-width space_ certainly sounds like something that could ruin your day.

Now let’s analyze the bottom of the stack trace:

```
File "urllib.py", line 1306, in quote_plus
File "urllib.py", line 1299, in quote
```

We can find, that _urllib_ is a utility for fetching some web data and parsing urls. `quote` and `quote_plus` functions (todo: methods?) suggest, that someone was working on some kind of urls. Now ask yourself a question: have you changed any configuration files (or any other configuration or code) recently, that contained urls? If yes, than it may be time to check if your configuration wasn’t contaminated with some zero-width spaces.

How do you check it? Unfortunately, as far as I know, most text editors doesn’t support showing it. But you can use some HEX editor, e. g. HEX-editor plugin for Notepad++.

On fileformat.info we can find, that HEX code for UTF-8 zero width space is `0xE2 0x80 0x8B`. We can now look for it in HEX-editor:

(image)

There it is! Removing it should fix your problems.

But where did I get this zero-width space from? In my case it was Jenkins. I copied build number and pasted it into my docker-compose.yml. It seems Jenkins puts some zero-width spaces inside the build numbers, huh.

(images)

# tl;dr

Some url in your configuration or code got contaminated with zero-width space -- a character that will most likely not show in any editor. Check all urls that you recently modified (probably copy-pasted) for that character.

# Final thoughts

One more thought: after fixing this error, I noticed, that it actually showed in git diff command. Pro tip: to avoid such strange problems, watch carefully what you commit!

Some more info on zero-width space:

[https://en.wikipedia.org/wiki/Zero-width_space](https://en.wikipedia.org/wiki/Zero-width_space)

todo: check spelling and stuff