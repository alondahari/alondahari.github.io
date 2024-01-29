---
title: Moving the cursor to active window
categories:
  - Tooling
tags:
  - Mac
  - Dev Health
---

I have a long history of wrist and finger issues ever since I started on my coding career. Being a climber doesn't help.
With the influence of a co-worker I made the move about 6 months ago to use NeoVim (instead of VSCode) and created my custom [Dactyl-ManuForm](https://github.com/abstracthat/dactyl-manuform) keyboard.
I feel like I fell in love with coding all over again. I love improving and customizing my workflow, so it felt very natural and exciting.

Quickly, I started exploring how I can use my keyboard as much as possible, and my trackpad as little as possible, since it is the main culprit of my wrist pains. I switched most of the apps I use to ones that support keyboard usage and shortcuts.
There are two apps I still heavily use my trackpad, my email client (Apple's "Mail") and the browser ([ Arc ](https://arc.net/)). I just didn't find an email client that works well with all the keyboard functionality I need, and I don't even want to think about navigating the web with my keyboard alone.

So I still need to use my trackpad, but I wanted to minimize my usage as much as possible. I also use 2 large external monitors, so moving the cursor from one to the other requires a lot of movement. I wanted my cursor to automatically move to the active application (in focus), which I switch between using my keyboard and [Raycast](https://raycast.com).

After looking for a while with no good solution in sight, I finally came across [ Hammerspoon ](https://hammerspoon.org). Hammerspoon is a powerful automation tool for Mac, which allows you to use lua and run scripts on your OS. Since I already used lua a bunch to configure my NeoVim, it seemed like the perfect solution.

Hammerspoon has a plugin ecosystem (called spoons), and after browsing it for a bit (getting all sorts of ideas), I came across a [spoon that did exactly what I needed](https://www.hammerspoon.org/Spoons/MouseFollowsFocus.html).
After installing Hammerspoon and the extension, I was ready to rock. The app installs in your `.config` folder, and all you need to do is add a `init.lua` file to load your extensions and configs. Mine looks like this:

```lua
mouse_follows_focus = hs.loadSpoon("MouseFollowsFocus")
mouse_follows_focus:configure({})
mouse_follows_focus:start()
```

That's it! The spoon is installed in the `spoons` folder, so everything is contained right there to inspect. Enjoy!
