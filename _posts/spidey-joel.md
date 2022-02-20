---
layout: post
title: "Reversing: Spiderman 2000 - Re-enabling Joel Jewett Cheat Code"
description: "Re-enabling an old cheat code"
created: 2017-08-18
modified: 2017-08-18
tags: [krystalgamer, spiderman 2000, spiderman, pc, joel jewett, cheat code, enable, reversing]
comments: true
---

## Disabled on the port 


For some reason the cheat code is disabled on the PC version, if you input it nothing happens. While some speculate that's due to he not being part on the porting process the same applies for the other ports which he's in-game. Thankfully, the devs were lazy and didn't completely remove him from the game(its models and unlocking logic is still there).

## Finding him in the code 


He's cheat code is `RULUR` so the first thing I did was search his cheat code in IDA. Xrefing it only shows one result which is an array of cheat codes with the following structure: `CHEAT CODE - CHEAT DESCRIPTION`.

![Cheat Array](images/spidey/cheat_array.png)

On the left you can see the two xrefs, since the one related to the cheat code is the first I decided to find where it leads me.

Here's the pseudo-code of the function that references it:

```c
signed int __cdecl sub_47C440(int a1)
{
  int v1; // ebx@1
  int v2; // edi@1
  char **v3; // esi@1

  v1 = -1;
  v2 = 0;
  v3 = off_5513E0;
  while ( !strcmp((_BYTE *)a1, *v3) )
  {
    v3 += 2;
    ++v2;
    if ( (signed int)v3 >= (signed int)aToonSpidey )
      goto LABEL_6;
  }
  v1 = v2;
  if ( v2 == 4 )
    return -1;
LABEL_6:
  if ( sub_47C240(v1) )
    return v1;
  return -1;
}
```

## What's going on?


The while loop walks through the cheat code list and each one is compared to the one sent to the function. If it's not equal it keeps going until the end of the array is reached and then calls `sub_47C240` with the argument -1. If the strings are equal(you entered a valid cheat code) it stops and `sub_47C240` is called with the index of the cheat code, unless the index is 4 in that case the function returns **immediately** -1.

### What the hell is on index 4??


The `RULUR` cheat code. Now it's time to patch the binary and remove that check!

![Jump replace](images/spidey/jump_patch.png)

The possibilities to fix this are endless but what I decided to do was simply `NOP` the jump.

Lastly, `sub_47C240` is the actual function where the cheat is activated if the argument is -1 it simply returns 0, else it modifies some global flags.

### Here's a video I made showcasing the cheat code working


{.video https://www.youtube.com/embed/H-bs5sTNDmc"}
*This post wasn't meant to come this earlier but the one one I was writing is taking more time*
