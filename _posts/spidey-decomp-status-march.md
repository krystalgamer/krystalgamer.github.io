---
layout: post
title: "Spider-Man (Neversoft) decompilation project Progress Checkpoint - March 2025"
description: "Progress report for the Spider-Man decompilation project - March 2025"
created: 2025-03-14
modified: 2025-03-14
tags: [spider-man, decompilation, ida, spider-man 2000, september]
comments: false
---

Shortly after the [previous progress checkpoint](../spidey-decomp-status-september), I took a break and came back in the [middle of February](https://www.youtube.com/watch?v=E5qEV5CK_9k). Nevertheless, progress has been steady and I've learned enough of interesting things that I think are worth sharing.

As always, you can check the [repository](https://github.com/krystalgamer/spidey-decomp) and all progress has been livestreamed on my [youtube channel](https://www.youtube.com/@kRySt4LGaMeR).

# High level overview

Between the previous checkpoint and 14th of March 2025 there have been made 313 commits, totalling 1653.

- @Ok 1306 (+276)
- @NotOk 196 (+9)
- @BIGTODO 55 (-2)
- @MEDIUMTODO 274 (+13)
- @SMALLTODO 206 (-87)

Recently I've focused more on finishing each source file individually, which is great because during each session there's less context switching. Additionally, the quirky behaviours that I document tend to be localized which I believe is influenced by the engineer that wrote that component.

# Anatomy of a while-loop

Microsoft Visual C++ has this very peculiar way of generating code for `while` loops, it turns them into a `if (cond)` and `do ... while(cond);` combo.


So, this piece of code:

```c

while (x == 5)
{
    // code
}
```

Would become:

```c
if (x == 5)
{
    do
    {
        // code
    } while (x == 5);
}
```

Or in x86 assembly:

```asm

cmp eax, 5
jnz out_of_the_loop

start_code:

; code


cmp eax, 5 ; same condition as before
jz start_code

out_of_the_loop:
```


This was something that I quickly internalized and didn't give much thought, until I was reversing `dcmemcard.cpp`. In `DCCard_Wait` there's this block of code that looked like this:

```asm

condition:
mov eax, [global_variable]
cmp eax, 3
jz condition
```



How could there be a loop condition that doesn't have the exact same check before the code starts? `for` loop code generation is basically the same as the one of `while` loops, so that couldn't be it.

The following code:

```c

// global_variable must be volatile or this will be optimized out
// or create an infite loop
while (global_variable)
    ;

```

Would generate:

```asm
mov eax, [global_variable]
cmp eax, 3
jnz out_of_condition


condition:
mov eax, [global_variable]
cmp eax, 3
jz condition

out_of_condition:
```

It was then that it struck me, the code was doing some kind of busy waiting and it's related to Dreamcast memory cards. It was a `do...while` loop without a body.

```c

do
{
} while (global_variable);
```


[Link](https://github.com/krystalgamer/spidey-decomp/blob/7598bdf8182abb0c7adf94e6d0110639cf32bc90/dcmemcard.cpp#L176-L178) to the actual code. LTI Gray Matter, the company that ported the game to PC, must've removed the body of the loop when porting since it no longer served a purpose.


# Strange inlining in PowerPC version

In previous entries of this series I've mentioned how functions in the Windows version have been heavily inlined. On the PowerPC (or Mac) version, inlining seems to be absent. I use the word *seems* purposefully because I can only recount one instance where I noticed it - the destructor of `DCKeyFrame`.


It has the following definition:

```
class DCKeyFrame
{
    DCKeyFrame *pNext;
};
```

And the destructor is as follows:

```c++

DCKeyFrame::~DCKeyFrame(void)
{
    delete this->pNext;
}
```

Well, on the PowerPC version it generated some deeply nested code.

```c++

DCKeyFrame::~DCKeyFrame(void)
{
    if (this->pNext)
    {
        if (this->pNext->pNext)
        {
            if (this->pNext->pNext->pNext)
            {
                /*
                    it continues
                */
                delete(this->pNext->pNext);
            }

            delete(this->pNext);
        }

        delete(this->pNext);
    }
}

```

# Code Archeology

On the other hand, if it wasn't the PowerPC version there would be whole functions and extra-logic that would be gone. Here's a small refresher on how inlining works on MSVC.
Inlining only works on the same source file, therefore if I want a function to be always inlined I'd have to declare it an a header - [example](https://github.com/krystalgamer/spidey-decomp/blob/7229bb94db3e39c83b2ed723a5729e623d2963bd/ps2funcs.h#L136-L152). The only other exception would be if a function has it is "depth limit" and stopped inlining code.

Here's an example of logic that would be lost if not for the Mac version. Take a look at `OpenMusicFile` from `PCMusic.cpp`:

```c++
INLINE u8 OpenMusicFile(char *pName, bool a2)
{
	char v2[256];

	strcpy(v2, "Voice\\");
	strcat(v2, pName);
	DXERR_printf("\t\tMUSIC PLAYING %s\r\n", pName);

	gMusicFileHandle = gdFsOpen(v2, 0);
	gMusicBinkHandle = BinkOpen(gMusicFileHandle, BINKFILEHANDLE | BINKIOSIZE);

	if (!gMusicBinkHandle)
		return 0;

	if (a2)
		BinkSetVideoOnOff(gMusicBinkHandle, 0);
	else
		BinkSetVideoOnOff(gMusicBinkHandle, 1);
	return 1;

}
```

It is only called from one other place: `PCMUSIC_Play`, and the second argument is `true`. Therefore the line `BinkSetVideoOnOff(gMusicBinkHandle, 1);` is gone. 


Even though this is not a common occurrence it tends to show up more in file loading/handling routines and I believe it was developers making the code more flexible but not making use of it. Thankfully, in most instances inlining does not remove code but only binds parameters.


```c++
INLINE void CCarnage::PlayXA(
		i32 a2,
		i32 a3,
		i32 a4)
{
	if (Rnd(100) <= a4)
		Redbook_XAPlayPos(a2, a3, &this->mPos, 0);
}
```

In this case the inline would work much more like a macro, where it simply replaces all parameters with the passed values.


## Visual C++ and Link-Time Optimization

An interesting tid-bit is that if two functions from different object files have the exact same code, then the linker will only emit only one instance.


This is the case for `MJ_RelocatableModuleClear` - [link](https://github.com/krystalgamer/spidey-decomp/blob/7229bb94db3e39c83b2ed723a5729e623d2963bd/mj.cpp#L22-L34) - and `Submariner_RelocatableModuleClear` - [link](https://github.com/krystalgamer/spidey-decomp/blob/7229bb94db3e39c83b2ed723a5729e623d2963bd/submarin.cpp#L53-L64). It's for situtations like this that I'm glad the Mac version preserves all of it as-is.



## Leftover code?

There are also some funny findings, such as in the `CGrenadeExplosion` constructor where there's a random call `Rnd()` - [link](https://github.com/krystalgamer/spidey-decomp/blob/54e4244c5ff5ac624a159c119d5af27ad3a3c9de/exp.cpp#L148)


# Learnings from a matching decompilation

There is nothing more frustating than recreating a function and the assembly is about 99% a close match to the original. Most of them are due to re-ordered memory accesses (functionally the same) which I still don't understand what causes it - classes such as `CVector` are more subject to this for some unknown reason.

Code like this:

```c++

this->mVec.vx = 100;
this->mVec.vy = 50;
this->mVec.vz = 3;

this->mField = 99;
```

Would become something like:
```c++

this->mVec.vz = 3;
this->mVec.vx = 100;
this->mField = 99;

this->mVec.vy = 50;
```

Although ignoring them would be more benefitial to finish the project quicker, this is the type of knowledge that can help other decomp projects and also gives us an insight on how the compiler works. Here's a couple of interesting cases.


## Swapped registers (ecx & edx)

For `buAnalyzeBackupFileImage` I had written the following code:

```c++
i32 buAnalyzeBackupFileImage(
		SBackupFile* a1,
		void* a2)
{

	i32* pA2 = static_cast<i32*>(a2);
	i32 v2 = pA2[0];

	a1->mBackupSize = v2;

	a1->pCardHead = reinterpret_cast<SCardHead*>(&pA2[1]);

	return 0;
}
```

It basically takes data from a buffer `a2` and populates the `SBackupFile` structure. The generated code was exactly the same but registers `ecx` and `edx` were swapped. Thanks to Mac version I could see that the write to `mBackupSize` was actually done by `memcpy`, which is an intrinsic on MSVC!

By doing the following change:

```c++
i32 v2;

memcpy(&v2, pA2, 4);
a1->mBackupSize = v2;
```

It was finally matching! [Commit link](https://github.com/krystalgamer/spidey-decomp/commit/977f54a2099fefc9d941c3103d84b3e9819539e6).


## Fitting 1 byte into a 4 byte register

This one envolves a global variable `u8 gObjFileRegion` and function call `Spool_GetModel(u32 Checksum, i32 Region)`. The variable is used by multiple classes during construction and passed as the second argument to the function. As you can see it's an unsigned 1-byte value, therefore it's zero extended when passed.

For all of its usages this is how the assembly looked like:

```asm
xor eax, eax
mov al, [gObjFileRegion]

push 0x12345678; first argument is always a constant

call Spool_GetModel
```


For some reason for the constructor of `CSonicBubble` it was slightly different.

```asm
mov eax, [gObjFileRegion]
and eax, 0xFF

push 0x12345678; first argument is always a constant

call Spool_GetModel
```

I tried so many different things and eventually gave up. Couple hours later couldn't stop thinking about it, so I decided to check what was so different from other usages to yield a different result. Nothing in the function bodies - they were all `Spool_GetModel(0x12345678, gObjFileRegion);`. The difference was in the definition, all other places had it `extern u8 gObjFileRegion`, while for the file of `CSonicBubble` it was where it was defined! I moved the definition of the variable to another file - [commit link](https://github.com/krystalgamer/spidey-decomp/commit/bb82a230f759435dd50f8230f9933d15bf674c58) - and voila! 

From this experience I could deduce that the compiler is quite conservative at generating memory accesses for external variables as it can only know that the variable was **at most** 1 byte long. Where for the file it was declared it knew it had allocated 4 bytes to it (3 of them zeroed), therefore could generate code that is slightly more optimized.

## The missing exception handler

Since early in development the compiler started to generate exception handlers that were matching so I never gave them a deeper thought. Even when I had to write a [SEH handler](https://github.com/krystalgamer/spidey-decomp/blob/bb82a230f759435dd50f8230f9933d15bf674c58/SpideyDX.cpp#L367-L386) it was quite simple. It all changed when writing the destructor for `DCObject`. The PC version had a handler setup that was freeing a field that was already free'd during the destructor. It was super confusing.

```c++

class DCObject
{
    DCObject *field_E4;
};


class DCObjectList : DCObject
{
};

DCObject::~DCObject(void)
{
    // DCObject::~DCObject(this->field_E4)
    delete this->field_E4;
    this->field_E4 = 0;
    
    
    // DCObjectList::~DCObjectList(this->field_E4)
    DCObjectList* pObjList = reinterpret_cast<DCObjectList*>(this->field_E4);
    delete pObjList;
}

EH_HANDLER
{
    delete this->field_E4;
}
```

Before we dig deeper it's really weird that two distinct destructors are called for the same variable. The handler was also doing something that's already done during the destructor which added to the confusion. To better understand what was going on I had to dig deeper into how handlers work, [this article](https://www.openrce.org/articles/full_view/21) written by Igorsk was a godsend. The most important part for this is the field `nTryBlocks` from the `FuncInfo` structure, as it specificies how many try blocks exist in the function. For this destructor it was set to 0, which made total sense (why would object destruction ever cause an exception) but also added to the confusion. 

A couple of hours later I decided to take another look, this time focusing on other destructors and quickly realized they all had exception handlers that called  the base class destructor. This gave me an idea, if I add a field that's not a pointer and has a non-default destructor will it be added to the exception handler? Yes, it was. Therefore I made appropriate changes to the code.



```c++
class DCObject
{
    DCObjectList field_E4;
};


class DCObjectList
{
    DCObject *pObject;
};

DCObjectList::~DCObjectList(void)
{
    delete this->pObject;
}

DCObject::~DCObject(void)
{

    // DCObject::~DCObject(this->field_E4)
    delete this->field_E4->pObject;
    this->field_E4->pObject = 0;


    // DCObjectList::~DCObjectList(&this->field_E4)
    delete this->field_E4;
}
```

Now the code was matching and there was an exception handler that did do `delete this->field_E4;`. To be honest there was a small difference, my handler is simply a jump to the destructor while the PC version is fully inlined something that I couldn't make happen, even with `forceinline`.


The main reason why this took so long for me to figure out was due to the fact that both classes reference each other. If I had paid closer attention to the Mac version I could've realized earlier that the call was supposed to be `DCObjectList::~DCObjectList(&this->field_E4)` and not `DCObjectList::~DCObjectList(this->field_E4)` and solved this much earlier.

## Binary result

This is the tale of the function `u8 DCCard_Exists(u32)`. The function epilogue was as follows:

```asm
test eax, eax
setz al
retn
```

Which IDA translated to:

```c++
return var == 0;
```

Looks good, but the generated code was strange.

```asm
xor edx, edx
test eax, eax
setz dl
mov eax, edx
retn
```

For some reason, it was using an extra register, `edx` to perform the comparison and then copy that value to `eax`. Tried so many combinations, including the ternary operator and what worked was as follows:

```c++

if (var)
    return 0;

return 1;
```

My theory on why `var == 0` yields so much verbose code is because it somehow considers `eax` as clobbered and therefore must clear the top 24-bits. By explicitly unrolling all possibilities and the compiler knowing `eax` already has the top 24 bits zeroed, it was able to simplify the code. Here's the [code](https://github.com/krystalgamer/spidey-decomp/blob/7229bb94db3e39c83b2ed723a5729e623d2963bd/dcmemcard.cpp#L79-L82) in case you're curious to check it out.


# Thank you(s)

- [madebr](https://github.com/krystalgamer/spidey-decomp/commit/b6e49f2274fec7dcb6184e418f0b2cb170340254#diff-eef4fba09c963157c939ecc33e2a2f26b0f39ee8e17b49ce4bcc9da42599525aR83) noticed that I committed a bug to the repository. I had accidentally stubbed out a function by trying to make the code compilable in multiple operating systems, 

- decomp.me Discord server - it's a great place to discuss decompilation projects and get help. Shoutout to [Chippy](https://github.com/1superchip) for entertaining my ideas. 

- Igorsk - for the exception handler post and for the help on the Reverse Engineering Discord server.
