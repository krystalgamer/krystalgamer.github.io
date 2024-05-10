---
layout: post
title: "Spider-Man (Neversoft) decompilation project Progress Checkpoint - May 2024"
description: "Progress report for the Spider-Man decompilation project"
created: 2024-05-09
modified: 2024-05-10
tags: [spider-man, decompilation, ida, spider-man 2000]
comments: false
---

In 2019, I started the decompilation the PC version of the Spider-Man game [developed by Neversoft](https://en.wikipedia.org/wiki/Spider-Man_(2000_video_game)), which most is commonly called Spider-Man 2000. Work didn't last long, I made few commits the last being done on 10th of August 2020.

Fast-forward to 2023 and my interest in going back to the decompilation had re-ignited, all thanks to a few community members that actively still create mods and explore the game. Here's a non-exhaustive list of the most active ones:


- [Anganoff](https://www.youtube.com/@Anganoff)
- [T.C Kral-Yusuf](https://www.youtube.com/@tc.kralyusuf)
- [Zedek](https://github.com/ZedekThePlagueDoctor)

After fixing some issues with my mods and improving the codebase I decided to restart the decompilation once again.


# Restarting the project

Restarting was a no-brainer has I had completely shifted my approach to the project and learned a lot more about it. Even though I knew the game was written in C++, I had previously decided to use C.
I had no strong reason to do so besides the fact I was more comfortable with C. This time I did not make the same mistake.


Previously, I was live-testing the code. I would re-write a method, hook into the game code and run the game and see if it worked. Not only, this method was tedious it left the door open for the correctness of the code - *did I mess up the signdness? did mess the edge case?*. This time I would follow a process similar to matching decompilation projects, but instead of trying to recreate the game as close to the byte I would strive for function parity.
This means that each function/sub-routine would be recreated trying to match as close as possible the original version. For cases where it didn't match, they'd need to be analyzed  as it's quite easy to make a change in code that changes the register allocation or instruction order.


## Case for matching decompilation

Having a project that when built produces a binary that is an exactly replica of the game is great as it means all the original behaviours (bugs included) were preserved. One could also develop a sense on how the original developers wrote code and thus be more efficient at decompiling more pieces of the code.


## Case against matching decompilation

It is a laborious, fruitless and tedious process. Even when using the same compiler toolchain used by the original developers there's a lot of "code massages" until it produces the exact same output. After all that work one thinks to themselves - *Did the developers go through all this effort to write this routine? It looks nothing like real code.*
When I say "code massages" they are not limited to changing the code but also the build flags - it's totally doable that some object files where built with different optimizations.

Also not having non-matching assembly does not mean the bugs are not preserved as it's more likely a bug in a game is caused by logic rather than a compiler spitting a different instruction.


There's no bigger frustration than struggling to get a function to match. When I was decompiling a super-small routine of a PSX game I had the logic pinned down, it was taking parameters and writing to some global variable. My code was doing it but had an extra copy to a register which I was going crazy for. After a while a realized the issue, I had defined the function as returning an `int` if I changed to `void` it would be a perfect match. This to say that there's so many variables at play that getting a matching decompilation is just too much work.



## A proper middle ground

Let's say we're decompiling a function we can either do matching decompilation or equivalent decompilation. The matching process has been described, but how does one evaluate if two pieces of assembly are equivalent. Two factors: sign correctness and memory accesses.

Let's assume we're decompiling the following function `void foo(MyStruct *arg)` and there's the following piece of code

```asm

...
mov ebx, arg ; would actually be something like: mov ebx, [esp+X]
movzx eax, [ebx+0x42]
...

; also seen as

...
mov ebx, arg
xor eax, eax
mov ax, [ebx+0x42]
...
```

One can deduce at the position `0x42` of `MyStruct` there's a 16-bit **unsigned integer**. If the instruction had been `movsx` (move with sign-extension) then it'd be a different story.

Another example, let's say for the function `void bar(MyStruct *src, MyStruct *dst)` you see the following:
```asm
mov ebx, src
mov eax, [ebx+0x20]

mov ecx, dst
mov [ecx+0x24], eax
mov [ecx+0x28], eax
```

Which would be the proper decompilation?
```c

dst->field_24 = src->field_20;
dst->field_28 = src->field_20;


// OR

int tmp = src->field_20;
dst->field_24 = tmp;
dst->field_28 = tmp;
```

The answer is the second! For the first one it's totally possible that in-between the writes the value changes, therefore the assembly generated would contain a read for each write. BUT! It's totally possible to make the compiler generate the same assembly for both, given you tweak with the flags enough or use the `restrict` keyword to convince the compiler there's no way the value would change in-between those writes.



Having these concepts laid down I restarted the decompilation project.


# May Checkpoint

The work is publicly accessible on my [GitHub](https://github.com/krystalgamer/spidey-decomp).

The [Macintosh](../spidey-apple) version of the game contains symbols - function names, parameters types and names. This has been used to identify functions of the game, the compiler used for the PC version has quite a few routines that have been inlined which is annoying. A cool thing about this version is that is contains boundaries for the object files. When the code inside `foobar.cpp` starts there will be a dummy section called `.sinit_foobar_cpp` so it has been extremely helpful into getting a proper recreation of the source directory.

The game shares the same engine with Tony Hawk Pro Skater 2 and that game has had 2 demos for the PlayStation that contains symbols - they contain more information than Macintosh version has they also contain classes/structure definitions with offsets. Since both of these games (ab)use inheritance it has been a godsent helping outlining where base class ends and where child class begins. For reference here's the inheritance chart of the player class `CPlayer -> CSuper -> CBody -> CItem (-> CClass)`. `CClass` doesn't seem to be present in the PC and Mac versions, either it was fully inlined or removed when ported.



So far 75 data structures have been added to the codebase. I've also created validation macros to make sure both structure size and memory accesses don't get messed up by padding or accidental mistakes. They are `VALIDATE_SIZE` that takes a structure/class name and the expected size and `VALIDATE` which takes a structure/class name, a field name and the expected offset. For both macros, if the check fails it'll print a message.

Re-written functions:

- Memory allocation methods
- Most of `CItem`
- Most of `CBody`
- Most of `CSuper`
- Geometry Transformation engine methods
- Utility methods

By starting to write code for the "lower" level methods it made me able to identify functions that have been inlined. I've also started to develop this intuition on how the compiler works. For example, when there's a conditional jump, it jumps towards the un-met condition (or as I used to say "jump to further ahead in the source file").

The following code:

```c

int MyFunction(int arg)
{
  if (arg)
    return 200;
  return 45;
}
```

Would generate the following assembly:
```asm

mov eax, [esp+4]
test eax, eax
jz label
mov eax, 200
jmp end


label:
mov eax, 45
end:
ret
```

Has you can see the conditional jump is taken if the argument is zero which is the opposite of the expression inside the if-statement!




Recently to help identify which functions are matching and which need to be revisited I developed a tagging system which is `@Ok`, `@NotOk` and `@TODO`. The idea is to be easily searchable and make it easier to generate high-level status reports.

Finally, I've also made the decompilation setup reproducible which is basically IDA Pro, Ghidra and the same version of Visual Studio by the developers (it was identified using [Detect-It-Easy](https://github.com/horsicq/Detect-It-Easy)). In the beginning I was working on a different machine that for a while didn't have access to.


## Broadcasting my progress

In late 2023, I've decided to [stream every time I'd work the project](https://www.youtube.com/@kRySt4LGaMeR). The goal was simple to educate people on the complexity of this endeavor. Since I own a Discord community with close to a thousand members, it's not uncommon to get the question - *what % of work is left to do?* - which is quite tricky to answer.

Decompilation process goes as follows. Identify function in code and the data structures it uses. Then, recreate data structures in code and finally recreate the function. It's a lot of work until the code is done. For a lot of streams I was just identifying fields in data structures and naming them if it was possible to infer from the scenario.


Livestreaming has also come with the benefit that I can share with the audience all of the rabbit holes I fall into during the process, such as automation with IDAPython.


## Focusing on consistency

As described in this post, decompilation project is tedious and laborious. Therefore it's easy to lose motivation and leave it in backburner for a long-time. The solution I have found is consistency, everyday try to work on it a bit as all effort will compound. An example of this are days where the time I worked on the project was just outlining fields in the classes, there will be a day where I won't need to open the `Structures` tab in IDA and that would mean I have the internal structures all figured out.

I've also worked in removing all the inconveniences that can hinder me working on it, such as the fact I required a specific computer to work on. Now that I have a portable and reproducible environment I have less excuses not to tackle this project. A cool side-effect of working on it regularly is that I feel more motivated, like a self-fulfilling prophecy the more I work the more I feel motivate to do more.


I've talked about this with [MrMartinIden](https://gitlab.com/MrMartinIden) who is also working on decompilation project for [Ultimate Spider-Man PC Version](https://gitlab.com/MrMartinIden/openusm) and we both agree that this type of projects require much more than motivation to thrive.


# Looking Forward


There's still aspects that I'd love to improve such as function equality evaluation. I'd love to automate the process. Here's possibilities that I've thought so far:
- x86 symbolic execution engine that would validate whether the same memory accesses
- Lift x86 to a Z3 style theorem and try to see if both sides (original vs re-written) are equal
- Lift both original and re-written version to LLVM IR and see if they are reduced to same output


For the first two instances the idea of dealing with loops makes it daunting and I haven't found a proper way to deal with it. For the third option I have little experience with LLVM so I'm not sure how feasible it is. If someone has any expertise in this topic please reach out to me! My contact information is in the [About page](../about).
