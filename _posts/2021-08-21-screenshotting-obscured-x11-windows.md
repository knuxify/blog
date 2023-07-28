---
layout: post
author: knuxify
title: How to take screenshots of partially/fully obscured windows in Xorg/X11
tags: linux tips x11
excerpt_separator: <!--more-->
category: quickies
---

*I initially posted this on my [Fediverse account](https://dither.ddns.net/notice/AA9v6W6uROGC5vub9E), and I'm now reposting it here for visibility. The all-lowercase text adds to the charm, so I'll keep it.*

…you can’t, not conventionally anyways. but there is a workaround! just start up an x session with xnest (that way you can still access the window you want to do stuff on). this will simply open a window with your new x session inside of it, and that should be enough 

<!--more-->

1. install ``xorg-server-xnest xwininfo imagemagick``
2. run ``Xnest :1 -geometry 1000x1400`` (substitute 1000x1400 with whatever resolution you want your new session to be. probably larger than your target window)
3. find the window id of the window you want to capture with ``DISPLAY=:1 xwininfo -tree``
4. take a screenshot with ``DISPLAY=:1 import -winid <window id you got in the previous command> targetfile.png``

if you get “resource temporarily unavailable” try a different window id

if all goes well, you should have a new png file that contains the screenshot! if you get a buggy screenshot, move the xnest window around a bit and try again
