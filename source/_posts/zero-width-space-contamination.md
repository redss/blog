---
title: Zero-width space contamination
date: 2017-08-06 20:08:02
tags:
---

One day I was running `docker-compose up -d`, minding my own business, when suddenly I got:

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

I would like to write about this error, since searching for it on Google gave me hardly any results. Looking for `KeyError: u'\u200b'` phrase yielded four results, and none of them were related to Docker Compose. Actually, they were all related to Haiku Operating System, which wasn't much help. Hopefully, now this blog post will join them.

I got this while running `docker-compose` command but as I’ve found, similar errors can occur anywhere. So if you’re reading this, I hope I can save you a few hours of debugging.

# Analysis

Let’s first take a look at this `\u200b` symbol. Quick search yields following site:

[http://www.fileformat.info/info/unicode/char/200B/index.htm](http://www.fileformat.info/info/unicode/char/200B/index.htm)

It's a **zero-width space** -- an invisible whitespace character that can be used to control line breaks. It also sounds like something that could ruin your day. 

At this point it's clear who the culprit is. But to pinpoint its location, we must dig deeper. Let’s analyze the bottom of the stack trace:

```
File "urllib.py", line 1306, in quote_plus
File "urllib.py", line 1299, in quote
```

We can find, that `urllib.py` is a Python module used mainly for fetching some web data, but also parsing URLs. `quote` and `quote_plus` functions suggest, that someone was working with URLs.

At this point it's obvious, that some zero-width space got itself into some URL. In fact, I did change one URL in `docker-compose.yml` before this problem appeared. Finding zero-width space should now be easy... right?

After some tinkering I've learned yet another lesson about zero-width spaces: most text editors doesn't support showing them.

{% asset_img zero-width-space-notepad-plus-plus.png "Notepad++ will not show zero-width space even with &#34;Show All Characters&#34; turned on." %}

In the end I decided to use _HEX-Editor_ plugin for Notepad++. On [fileformat.info](http://www.fileformat.info/info/unicode/char/200B/index.htm) we can find, that hex code for UTF-8 zero width space is `0xE2 0x80 0x8B`.

{% asset_img zero-width-space-hex.png "There the sucker is!" %}

Removing that character fixed my problem.

But where did I get this zero-width space from? In my case it was Jenkins -- I copied build number and pasted it into my `docker-compose.yml`. It turns out that Jenkins puts a zero-width space inside the build numbers, huh. I guess the moral of this story is to always be carefull with copying stuff from the web into source code.

One more tip: I've noticed that `git diff` will actually show zero-width spaces as `<U+200B>`. I don't know many people who use `git diff` on a daily basis, but it might prove to be a good way of finding zero-width spaces once you know they are in your source code.

Some more info on zero-width space:

[https://en.wikipedia.org/wiki/Zero-width_space](https://en.wikipedia.org/wiki/Zero-width_space)

# tl;dr

Some URL in your configuration or code got contaminated with [zero-width space](https://en.wikipedia.org/wiki/Zero-width_space) -- a character that will most likely not show in any text editor, but will break URL parsing. Check all URLs that you recently modified (probably copy-pasted) for that character, e. g. using some hex editor or `git diff`.