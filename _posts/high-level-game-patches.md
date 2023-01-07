---
layout: post
title: "High level mods/patches for console video-games"
description: "This blogpost explores my journey of bringing higher level languages (in this case C) to console video-game modding, as an alternative to hand-tailored assembly mods."
created: 2023-01-07
modified: 2023-01-07
tags: [later]
comments: false
---

This blogpost explores my journey of bringing higher level languages (in this case C) to console video-game modding, as an alternative to hand-tailored assembly mods.

# Prelude

I had done a [mod](https://github.com/krystalgamer/usm-debug-menu), in C, for Ultimate Spider-man on the PC. This game shares the engine and a lot of its codebase with the Spider-Man 2 on the PlayStation 2. My goal was to bring the same mod to this game, but I had no intention of manually converting it to MIPS assembly. Targetting the PS2 was simple, just had to get a compiler that supported MIPS-R900 (and hopefully the PS2 ABI), the issue was deciding what would be the way of injecting the resulting machine code into the game.

Even though modifying the original ELF is convenient, having to rebuild the game ISO every time I made a change seemed too daunting. There are some fast ISO builders such as [mkpsxiso](https://github.com/Lameguy64/mkpsxiso), but I still opted not to go this route. The idea I settled on was using the console cheat-engines to inject my modifications. These engines are at their core arbitrary data writers, if I take my compiled code and convert it to cheat-code it will then proceed to write it to the game process. This has the added advantage that the original game is left untouched and iterating over is a completely external process.


# Targeting the Playstation 2

The first step was to get a compiler that targeted the PS2, I got it from [ps2dev](https://github.com/ps2dev). By default, C compilers include libc and its initialization routines into the produced binary, which are extremely useful but unwanted when someone is trying to create the smallest working executable. Thus the code was compiled using `-nostartfiles -nostdlib`. If I needed any libc functionality I could either [reimplement it](https://github.com/krystalgamer/sm2-debug-menu/blob/0c86a82da6787532a8f89e06ac7be4549c7f2022/menu.c#L60-L70) or [find whether the game has it included](https://github.com/krystalgamer/sm2-debug-menu/blob/0c86a82da6787532a8f89e06ac7be4549c7f2022/menu.c#L76-L77). This "bring your own libc" methodology helped keeping the output and toolchain as slim as possible.

The next step was to convert the ELF file into a PCSX2 (PlayStation 2 emulator) compatible cheat-code. Thankfully, on the PCSX2 forums there are some threads ([here](https://forums.pcsx2.net/Thread-How-PNACH-files-work-2-0) and [here](https://forums.pcsx2.net/Thread-How-pnach-files-work)) documenting how they work. Here's the gist of them:

```text
patch=(type),(processor),(address),(size),(value)
```

- type: 0 to write only once, 1 to freeze the value
- processor: EE/IOP, EE is for game code while IOP is used for PS2 libraries
- address: where to write
- size: byte/short/word/extended, extended will determine the correct size to write
- value: the value to write

Having this information I prototyped a [python script](https://github.com/krystalgamer/sm2-debug-menu/blob/83b1afce4bac4c9e0b48da1a069c8454b5d26a89/pnach_creator.py) that takes the `.text` and `.data` sections from the ELF and converts them into a cheat-code.


# Targeting the game

The cheat-code needs to be mapped to a memory region that wouldn't conflict with the game or the kernel. This spawned another question, "Should I compile the code as position-independent code(PIC) and then figure out where to map it or should I compile it to a specific position?".
The advantage of position-independent code is that if the memory region I choose becomes unsuitable I can easily relocate it to a new zone, but it increases the file size by requiring an extra section, the `.got`, that contains addresses for global variables.
On the other hand, position-specific code can be smaller since the compiler can use relative addressing, the issue is that it might require an extra compilation phase. The extra compilation phase depends on how I decide to map the cheat-code, top-down or bottom-up. Top-down means that the memory region grows downwards, from `0x4000` to `0x5000`, while bottom-up is the opposite, from `0x5000` to `0x4000`, this is the one that requires a two-phase compilation. Why would I do bottom-up mapping? 

Let's say we have the following three contiguous memory regions, 1 (free) - 2 (used) - 3 (free). Bottom-up mapping is useful if I decide to map to region 1, since it guarantees my cheat-code would never overlap region 2. Top-down would be useful for region 3. The reason bottom-up requires two compilation phases is that the first is used to determine the size of the code to write and calculate where to map it - `(BASE) = (start of region 2) - (size of the code)`. Using the following linker options `-Wl,--section-start=.text=(BASE)` the second compilation then generates code that must be mapped to the `BASE` address.

With this information, I updated my python script and created a [shell script](https://github.com/krystalgamer/sm2-debug-menu/blob/master/generate.sh) to automatically run these steps.

## Hooking game functions

Having an arbitrary code injector it was now time to have the game logic redirected. The script was then extended to support patching the game memory:

```python
patches = {
        'pause_menu_key_handler': (0x00657244, PatchType.NORMAL, 'Vtable entry'),
        'debug_menu_render': (0x00519234, PatchType.JAL, 'Unused nullsub before perf info'),
        'game_unpauser_unpause_hook': (0x00353C60, PatchType.JAL, 'Calling game_pause after unpause makes the pause front_end appear so hook the unpause call'),
        'sub_3E9A90': (0x00395DA8, PatchType.JAL, 'Nops the call to sub_3E9A90 when debug menu is enabled because it handles START and SELECT for some reason')
        }
```

It uses a python dictionary where the keys are the function/variable names from *my code* and the value is a tuple - the first value is the address where the patch is going to be written to, the second is the type of patch and the third is for logging/documentation purposes. The function/variable names are used to fetch their respective memory positions, this is done because the executable is compiled with debug symbols. Currently, it supports two patch types:

- `NORMAL` - writes to the memory position the address of the symbol specified in the key
- `JAL` - a JAL(Jump and Link) is similar to x86 `CALL`, it replaces an existing `JAL dest_addr` with `JAL my_addr`. It has its special type since the jump is relative and thus must be calculated on a case-by-case basis.

## Logging

To make my life simpler I'm eager to add as much logging as possible. Luckily, the PlayStation 2 BIOS contains a logging syscall and the emulator [supports it](https://github.com/PCSX2/pcsx2/pull/4123).


## Final result

Here's how the menu ended up and if you want to check the source [here it is](https://github.com/krystalgamer/sm2-debug-menu).

{.video https://www.youtube.com/embed/zXpfMl3o-ho}


# Interesting development tidbits 

During development I faced some interesting situations that I documented along the way and found important enough to warrant to be talked about.



## PS2DEV's GCC doesn't respect the old ABI

This one caught me off-guard. The reason it's not a bigger issue in the community it's because the toolchain is aimed for homebrew and not for this type of work. Making changes to GCC was an option but I decided to fix the arguments myself. I only encountered issues when calling functions with a mix of integers and floats in the arguments. The problem is as follows:

```c
void foo(int x, float y) {
	/* code goes here*/
}

void bar(float w, int z) {
	/* code goes here*/
}
```

The older GCC used by the game developers passes the arguments as follows:

```text
foo:
x in $a0 and y in $f12

bar:
w in $a0 and z in $f12
```

As you can see the order is irrelevant.


The newer GCC used by PS2DEV passes as follows:
```text
foo:
x in $a0 and y in $f13

bar:
w in $f12 and z in $a1
```

The registers used are sensible to the function prototype and convey extra information about how the prototype was written.

The PS2 contains a handy instruction which is `mtc1`, it loads a value from a general-purpose register to a floating-point register as-is. To keep things simple I passed all arguments in general-purpose registers.

```c

void nglListAddString(void *font, const char *str, int color, int x, int y, int z,  int x_scale, int y_scale){

	asm __volatile__(
		"subu $sp, $sp, 4\n"
		"sw $ra, 0($sp)\n"

		"mtc1 $a3, $f12\n"
		"mtc1 $a4, $f13\n"
		"mtc1 $a5, $f14\n"
		"mtc1 $a6, $f15\n"
		"mtc1 $a7, $f16\n"

		 "li $t9, 0x00511858\n"
		 "jal $t9\n"

		 "nop\n"
		 "ld $ra, 0($sp)\n"
		 "add $sp, $sp, 4\n"

		);
}

typedef union Converter{
	int i;
	float f;
}Converter;

int __inline cfi(float f){
	Converter c;
	c.f = f;
	return c.i;
}

/* call functions as follows */
nglListAddString(*nglSysFont, current_menu->title, green_color, cfi(render_x*1.0f), cfi(render_height * 1.0f), cfi(0.f), cfi(1.f), cfi(1.f));
```


## GCC's unwarranted behaviour

In order to keep the code as small as possible I was compiling the code with `-Os`. Everything was working fine until I started to remove some printfs and started to get some crashes. Moving function calls around also seemed to randomly fix the problem, this was an indication that somehow memory/stack corruption was happening. After a lot of testing, I figured out that if `-O2`/`-O3`/`-Os` were used then the problem would appear. The issue was caused by [Interprocedural analysis](https://www.ibm.com/docs/en/i/7.2?topic=techniques-interprocedural-analysis-ipa) or IPA. One of its functions is to determine whether registers are polluted across function calls and if not then re-use them. Here's an example:

```c
x = foo()
bar(x)
another(x)
```

Which can be translated into the following assembly:
```asm
jal foo
move $a0, $v0 ; the result from foo is in $v0
move $s0, $a0 ; save the value in a special register
jal bar
move $a0, $s0 ; restore the value
jal another
```

Let's say the first argument is passed in register `$a0`, if IPA determines that calling `bar` won't modify `$a0` then it doesn't need to reassign the value before calling `another`. Thus can simplify the assembly to the following:

```asm
jal foo
move $a0, $v0 ; the result from foo is in $v0
jal bar
jal another
```

As you can see it's extremely helpful because it saves instructions and reduces code size.

The problems start arising when mixed with inline assembly. As I mentioned in the previous section I had to create some wrapper functions to fix the registers and according to IPA those functions were seen as empty functions. Why? Because I didn't provide any clobber information. This clobber information is an **optional** parameter that informs the compiler which registers and memory were modified inside the inline assembly block. It is great because if you provide clobber information the compiler can perform further optimizations. The issue is that if you provide *no* information it assumes *nothing* was clobbered. Keep in mind this is an **optional** parameter and if you don't provide it the compiler assumes the best-case scenario (nothing clobbered), while in my opinion it should assume the worst-case scenario (everything was clobbered). In order to provide accurate clobber information I'd need to reverse the functions I was calling and functions they call, which is an insane amount of work. I then had two possible fixes, reduce the optimization level or add `__attribute__((noipa))` to the functions with inline assembly.

## Hours of debugging can save minutes of reading documentation

This was one of the most time consuming things to debug ever. I managed to succesfully add the menu to the game, but for some reason when I added a toggle for it it just didn't work. If the default state was off and I toggled it on it would flash for a brief moment and disappear. If the default was on and I toggled it off it would flash and stay up. After poking around my code I decided to take a look at the emulator and for some reason the breakpoints on write weren't being triggered. It finally occurred to me that it could be related to the cheat code. Instead of going to documentation and realizing that the `type` field in the cheat code changes the write-once nature to freezing the value, I worked around it in the most cumbersome way possible. I modified the cheat code generator not to include the `.data` section and started assume all variables were zero-initialized, this means that it **must** be mapped to a region that is zeroed. Thankfully the one I was using was already like that so it wasn't that much of trouble.


During this time I even started messing with [linker scripts](https://github.com/krystalgamer/sm2-debug-menu/blob/dabfc3d1d16a7f12dc957fed56f79c936ebe3782/linker_script), the only noteworthy change was that I moved the whole `.data` section inside `.text` so that I only took as much data as needed to the cheat code.


# Final thoughts

Using higher level languages to develop these kind of modifications is trickier than just using assembly. There are a lot of footguns and because of that it is easy to be diverted from the original goal. Despite being rough around the edges, it really boosts productivity. I can easily make a change and reload a save-state and the mod is automatically injected. Not having to restart the game is such a time saver and allows for so many iterations and exploration attempts.


There's some things that I also which to explore further. Such as, creating "slimmer" versions of compilers that can target consoles. This version would come with no standard library support and thus would reduce a lot of the complexity of setting these toolchains. Additionally, older developer consoles used to have more RAM, on retail these memory regions would mirror others. Most emulators support this extended RAM, it's just a matter of toggling a checkbox, I believe this extended memory could be a great resource for mod-makers and tinkerers.
