---
layout: post
title: "Reversing: Spiderman 2000 - Apple to the Rescue!"
description: "MacOs version of the game contains important information for the analysis of the game"
created: 2017-09-26
modified: 2017-10-23
comments: true
tags: [spiderman, powerpc, spiderman 2000, macos, symbols, reverse engineering, ida pro]
---

# Hard to find
{: .center}

Before trying to figure out how the textures are stored I downloaded the game for every platform available, hoping I'd get some symbols left accidentally by the devs. They were: PS1, N64 and Dreamcast according to the [wikipedia article](https://en.wikipedia.org/wiki/Spider-Man_(2000_video_game)) and other sources.

Some while ago I was reading the [PCGamingWiki entry](https://pcgamingwiki.com/wiki/Spider-Man_(2001)) on the game and it looks like the game was also released for the MacOS. The dev team behind it is Aspyr Media but even they don't acknowledge its existence. Checking their website it's not on any of their game's lists: [normal](http://www.aspyr.com/games) or [legacy](http://www.aspyr.com/games/legacy) while Spiderman 2 MacOS, the **master-shit** is. No fucking idea.

Another weird thing I noticed is that the Google Sidebar that displays info about the game actually reports it was ported to Mac. Unfortunately I have no idea where it's getting the information from.

Fun fact: Remember I said it is not present on the Wikipedia page, right? Well.. It is on the [List of Macintosh games](https://en.wikipedia.org/wiki/List_of_Macintosh_games), go figure.

# Finding them
{: .center}

To be honest finding them was really easier. A quick google search lead me to a website that seems dedicated to host old content. The website in question is [MyAbandonWare](https://www.myabandonware.com/game/spider-man-3qc#Mac). After that was matter of time to figure out how open `.sit` files. And was again with an unknown file format, `toast`. Thankfully, 7-zip is able to extract it. One weird thing is that it contains 2 spidey executables, one of them is big and the other is about 20Kb and contains some strings regarding the executing it( similar to the "This program can not be run in DOS Mode" ).

# What's the big deal about them
{: .center}

They contain symbols. Symbols are accessory information(function names, variables,..) used to aid the debugging process and they are almost always stripped. But for some reason non-Windows games have the fame to usually not being stripped. Awesome!!

If you look at my texture extractor you'll see most variables don't have a name because I have no ideia of what they do. Although they play a part in getting the texture and I see how they work, I wasn't able to extrapolate their intended use.
Now that I have access to the function names correlating instructions and strings I'm able to correctly name the functions in the Windows version making the reversing **sooo much easier**. Because of that now I know that the textures are "twiddled", a method used to increased performance in rendering. Even though this is not a problem in PCs it was kept..lazy devs.

# What now?
{: .center}

At the moment I'm finishing a TRG disassembler but right after I'm done I'll refactor my old tools and document the PSX file format. If this information hadn't come out I wouldn't even think of updating them. 

# Download
{: .center}

Download [here]({{ site.github.url }}/files/Spider-Man)

{% include disqus.html %}

