---
layout: post
title: "Update on the blog"
description: "What's been going on"
created: 2018-01-08
modified: 2018-01-08
tag: [update, new year, 2018]
comments: true
---


# Why I haven't posted


Lazyness. Quickest answer but that's not the whole story. It's my first year in college and the experience has been to say the least overwhelming. At first everything is chill and fine until the exams come in. My grades were actually great but the amount of time that I spent studying leaves almost to no time to reverse engineer or to write. The time that I get to myself I spent watching youtube videos, series or playing.

Don't get me wrong I still love to reverse engineer and explore the game engines of my favorite titiles but the circunstances lead to a lack of time to do so.

# What has changed

Finally built my rig! Here are the specs:

* Ryzen 3 1200 (OC'd to 3.6Ghz)
* MSI Geforce GTX 1050 TI LP
* AEGiS G.SKILL 8GB DDR4 3000MHz RAM (supposedly XMP ready but i'm still unable to set its speed to 2933 so at the moment I'm locked at 2133)
* MSI B350M BAZOOKA
* Seasonic S12II Bronze 520W
* 1Tb HDD Seagate Barracuda
* A case from NOX with a windows :)
* BenQ GL2450 24'' FHD Monitor

Peripherals:
* Fnatic Gear Rush G1 (MX Brown)
* MSI Interceptor Mouse
* Gamecom Plantronics 380

I have lots of ideas for my projects and I'm excited to start working on them, specially with my new workflow.

## Resonance


To start, I'm dropping capstone and and keystone from my Resonance project due to lack of communication from the devs. Some PR have been sitting on their github since 2015 and [aquynh](https://github.com/aquynh), the main developer, seems more interested with people using its projects rather than actually improving them. He's quick to add the projects using his engines to their respective websites but what irritates me is the fact some PRs are just left to die. The turning point was when I suggested a new disassembly flag which would let the developer choose how the 0x66 prefix would've been disassembled when working with x86_64. My suggestion came from an amazing talk from Christopher Domas about the x86 instruction set, you can watch it [here](https://www.youtube.com/watch?v=KrksBdWcZgQ&feature=youtu.be&t=32m49s), worth every minute. Basically AMD and Intel disassemble the 0x66 prefix differently (AMD does it correctly while Intel doesn't) which can lead a malicious program following different paths when executing on a processor from those brands. The worst part is that all disassemblers make the same mistake as Intel. To solve the problem I suggested a AMD disassembly mode which would do it correctly and the Intel(default) which would have done things as they're before. All code would've portable to the newer **patched** version but since then I've got nothing but radio silence from the developer. I even tried to contact him on twitter but he completly ignored me.

What irks me more is the fact that 3 months after the PR has been there sitting I decided to push the changes I've done to Resonance until February to the main repo. One of the changes included the Keystone assembler engine and to my surprise my project had been added to their *Showcases*. Not only my suggestions have been ignored but my projects have been broadcaat has an example. That means the main dev DID see my changes, so hasn't he replied to my PR. Also he seems to go to every project that uses capstone, keystone or unicorn and opens a issue saying "cool!"/"awesome!" but refuses to give any support.

I'll stop talking about crap projects and showcase two that seem more promising: Zydis and asmjit. Both have much more interesting licenses and their devs seem to be more open to the suggestions.


## Custom Texture Loader


I started re-writing it and moved from MVSC to GCC and NASM. The only thing really tying me to Visual Studio was the inline assembly and naked functions and with that problem solved nothing is stopping me. Visual Studio is an amazing IDE but it ran so slowly and poorly and poorly on my old laptop that it drove me nuts that I've grown a certain aversion to it. Also I want to leave it clear that if I intent to do some Windows development I'll surely use VS as it is the best.


## TRG disassembler


Eh... Not really interested in this. As the file format seemed to be so bloated and stupidly arranged I kinda lost interested. I've said this a million times in the past but if someone with experience is willing to help reversing this game I will be really glad since I'll be able to split my work and make much progress. The problem is that when I'm working on a part of the game I'm always questioning what the other is doing and without any help I need to play both of them which is tiresome.


## MSDMAN


If the name isn't clear then wait for the release ;).


# End note


I'm also learning more about binary exploiting and pwning since it's an area that really interests me, so expect some posts about CTFs and binary reversing that aren't games. See you on the next post!

P.S - Happy (LATE) new year!
