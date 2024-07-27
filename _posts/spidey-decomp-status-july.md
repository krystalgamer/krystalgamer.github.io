---
layout: post
title: "Spider-Man (Neversoft) decompilation project Progress Checkpoint - July 2024"
description: "Progress report for the Spider-Man decompilation project - July"
created: 2024-07-27
modified: 2024-07-27
tags: [spider-man, decompilation, ida, spider-man 2000]
comments: false
---

Since my [last post](../spidey-decomp-status-may) I've made quite a bit of progress and would love share it. Additionally, if you're interested in following the progress in real time you can [check my livestreams](https://www.youtube.com/@kRySt4LGaMeR/streams) or [track the repository](https://github.com/krystalgamer/spidey-decomp).

# Progress overview

Since May more than 500 commits have been pushed to the repository and reached the 700 commit mark. Here's the key metrics:

- @Ok - 507
- @NotOk - 167
- @BIGTODO - 63
- @MEDIUMTODO - 56
- @SMALLTODO - 50

You might've noticed that I broke down the `TODO` into different categories. The deciding factor for the attributed type  comes from my estimated cognitive load (line count, accessing known globals variables, accessing unknown global variables, complex arithmetic being performed). This has helped significantly to prioritize my work.


But how much % of the game is decompiled? Still hard to say. IDA counts ~4000 functions for the Windows version, but a lot of them have been inlined. For the PowerPC version it also counts 4000, but here very little to no inlining happened (I seriously cannot tell if the developers just manually did it) but it also has a bunch of wrappers for WinAPI and DirectX. Once I wrote a script that ignored nullsubs (functions that are just the return statement) and functions that IDA identified as library functions and got ~3000 functions. The issue is that IDA can misidentify user code as library code due to similarities at assembly level (has happened with my own code) and in this game's case it didn't catch the fact zlib was statically linked.

Taking either those numbers you get a completion of 12% to 16%. My short-term plan to finally have an answer in this regard is to take all the function symbols from the PowerPC version (ignoring the wrapper parts), declare placeholders in code and use the repo tagging system to measure completion.


# Deciphering global variables

Global variable access is scary because I'm always wondering whether the `dword_XXXXXX` that is recognized by the disassembler (and decompiler) is a solo variable or a field in a bigger structure. Because of that I've been avoiding recreating any functions that interact with globals.

The key to figure these variables was to look at Init/Find/Clear functions. Thanks to them I figured out the start of the array of `SPSXRegion` objects and multiple linked lists related to game entities.

In the case of arrays, Microsoft Visual C++ compiler is quite unique at generating code. Let's say you write the following:
```c
Foo foos[10];

for (int i = 0; i < 10; i++)
{
	foos[i].bar = 200;
}
```

The generated assembly will roughly translate to:

```c

for (Foo* foo = &foos[0]; foo < &foos[11]; foo++)
{
	foo->bar = 200;
}
```

It can be a little jarring at first to deal with this since technically `&foos[11]` is an undefined area. There's a very high chance `&foos[11]` is the address of another global variable that the compiler placed there. Although it might clobber decompiled pseudo-code, it's something that becomes second nature after a while. On the other hand, on the PowerPC version the compiler sticks closely to the written code.


Figuring out the multiple linked lists was surprisingly simple. There were logs, such as `DoAssert(dword_XXXXXX == 0, "BaddyList is NULL?");`. Not only it gave me a hint that's a list but that it's most likely a `CBaddy`, nice.


# Dealing with shifted pointers

In some instances (such as iterating an array of objects) the compiler will pivot the pointer to one of the members. So instead of doing `[eax+0x44]` and `[eax+0x48]`, it'll do `[eax-4]` and `[eax]`.

```c

struct MyStruct
{
	int foo;
	unsigned char bar;
};

MyStruct structs[30];

for (int i = 0; i<30; i++)
{
	structs[i].bar = 8;
	structs[i].foo = 2;
}
```

Will generate the following code:

```c

for (unsigned char *ptr = &structs[i].bar;
	ptr < (&structs + 30 * sizeof(MyStruct) + offsetof(MyStruct.bar));
	ptr += sizeof(MyStruct))
{
	*ptr = 8;
	*(int*)(ptr-4) = 2;
}
```

This example is quite cheeky as it combines the previous knowledge of MSVC slightly changing the loop condition with the shifted pointers. Here's a more cleaned up version:

```c

for (int i = 0; i<30; i++)
{
	unsigned char *ptr = &structs[i].bar;
	*ptr = 8;
	*(int*)(ptr-4) = 2;
}
```

My goal with it was to show that the topics I'm presenting here rarely show in isolation.

The problem with this optimization is that it confuses the decompiler and the pseudo-code generated is terrible to look at. In order to match the accesses I was looking at the assembly and calculating manually which field was being accessed. Doing it a couple times was enough to annoy me and then it dawned on me that I've seen this somewhere. I knew these were called "shifted pointers" so I plug into my search engine `ida pro shifted pointers` and find an appropriately named article *Igor’s tip of the week #54: Shifted pointers* - [link](https://hex-rays.com/blog/igors-tip-of-the-week-54-shifted-pointers/).

Following what the article says you can get the following pseudo-code:

```c

for (int i = 0; i<30; i++)
{
	MyStruct* __shifted(0x4, MyStruct) ptr = &structs[i].bar;

	ADJ(ptr)->bar = 8;
	ADJ(ptr)->foo = 2;
}
```

Which you might agree is fully readable and easily understandable. `__shifted(0x4, MyStruct)` simply means that the pointer is pointing to a `MyStruct` and has been offseted 4 bytes in this case it's the offset of `bar`. All accesses through it are also decorated with `ADJ()` to let the the person know it has been manually adjusted.



# What in the sub eax, 0?

A snippet that was becoming more common the more I decompiled functions was the following:

```asm

sub eax, 0 ;if another register held zero it could also be "sub eax, register"
jz _label1
dec eax
jz _label2
dec eax
jz _label3
...
```

IDA decompiled it as a series of `if...else if` statements, that I could not get to match.

```c

if (eax-0 == 0)
{
}
else if(--eax == 0)
{
}
else if(--eax == 0)
{
}
```

At certain point I realized that it was too obtuse to write code like this and that developers wouldn't make their lives so hard by doing this. Additionally, it violated the behavior that I had experienced before with [conditional jumps](../spidey-decomp-status-may/). That's when it occurred to me that it's way more likely to be a `switch` statement. It indeed was. For some reason I assumed the compiler would only generate jump tables for `switch` statements.


I still don't understand why the compiler prefers a `sub eax, 0` instead of the more understandable `test eax, eax` but I digress. This is another thing that after learning about becomes second nature and becomes part of the compiler lore that I'm slowly learning.

# Validating Virtual Methods

Virtual method order is relevant. Changing their declaration order will generate slightly different code, because their position in the [vtable](https://en.wikipedia.org/wiki/Virtual_method_table) (virtual method table) will change. Even though my initial declaration can be correct, it's totally possible to mess up during a refactor and end up with non-matching code. Therefore I added vtable index validation to my validation framework.

I [struggled to get the validation working](https://www.youtube.com/live/Ech4CtUO-c4?si=4BE-fzSSUFRLdunp&t=2006), mainly because the compiler refused to compile `reinterpret_cast<u32*>(&Class::VirtualMethod)`, so I had to use some tricks with variable arguments to make it work. The validation abuses the fact the compiler creates thunk functions when you do `&Class::VirtualMethod`. It materializes a function whose calling convention is `thiscall` and takes an undetermined number of arguments. Which is the following:

```asm
mov eax, [ecx] ;get the vtable address
jmp [eax+INDEX] ; jmp to the address in the INDEX position
```

My validator [ensures the bytes](https://github.com/krystalgamer/spidey-decomp/blob/b7d19c6dc2ead02565c8fe5efcf62ca191cacb9c/validate.cpp#L47-L74) at the given address are the opcodes for the assembly above and that the index matches.


# Decompiler and Compiler quirks

Throughout my journey I've came across some odd behaviors from the decompiler (IDA) and the compiler (MSVC) that I thought were important to note but their rarity didn't allow me to fully dig into the details.

1. IDA is quite good at identifying the usage of comma operator. There was [this function](https://github.com/krystalgamer/spidey-decomp/blob/66e0bb56fa0b5ab98160dfe40e1b817457531836/baddy.cpp#L437-L439) that I was getting extremely close to getting a matching decompilation but there was something that caused my version to be more verbose. Even though IDA's output contained the comma operator, I disregarded it at first because it seemed way too contrived. Turns out it was indeed the way to getting a matching decompilation.

2. When recreating `Shell_ContainsSubString` the decompiled output in the Windows version was too messy and the PowerPC version was much cleaner (as usual). I knew the name of function, parameter names but the PowerPC decompilation seemed off. The [function](https://github.com/krystalgamer/spidey-decomp/blob/66e0bb56fa0b5ab98160dfe40e1b817457531836/shell.cpp#L862-L883) is that typical finding the needle in haystack, which can be trivially implemented as such:

```c
int contains_substring(const char *hay, const char *needle)
{
	for (
		int hayIndex = 0;
		hay[hayIndex];
		hayIndex++)
	{
		for (
			int needleIndex = 0;
			;
			needleIndex++)
		{
			if (needle[needleIndex] != hay[hayIndex+needleIndex])
				break;

			if (needle[needleIndex] == 0)
				return 1;
		}

	}

	return 0;
}

```

The way IDA was decompiling was the following:

```c
int contains_substring(const char *hay, const char *needle)
{
	int needleIndex = 0;
	for (
		int hayIndex = 0;
		hay[hayIndex];
		hayIndex++)
	{
		for ( ; ; needleIndex++)
		{
			if (needle[needleIndex] != hay[hayIndex+needleIndex])
				break;

			if (needle[needleIndex] == 0)
				return 1;
		}

	}

	return 0;
}

```

Which implies the characters of `needle` must exist in `hay` sequentially but not a **strict sequence**. I looked at the disassembly and indeed the decompilation was wrong. Two registers got aliased to the same variable in the decompilation for some reason.

## Deoptimizing MSVC


Microsoft's Visual C++ Compiler is quite decent at optimizing redundant checks and avoiding generating duplicate assembly. Here's an example for redundant checks:

```c
void my_func()
{
	if (this->bar == 1)
	{
		/* do something */
	}

	if (this->bar == -1)
	{
		/* do something else */
	}

}
```

Will be optimized to:

```c
void my_func()
{
	if (this->bar == 1)
	{
		/* do something */
	}
	else if (this->bar == -1)
	{
		/* do something else */
	}

}
```

The optimization is simple, if the first condition is true, there's not reason to evaluate the second.
Regarding the duplicate assembly, here's an example:

```c

if (...)
{
	/* code here */
	this->bar = 5;
}
else
{
	/* other code here */
	this->bar = 5;
}
```

Will be optimized to:

```c

if (...)
{
	/* code here */
}
else
{
	/* other code here */
}

this->bar = 5;
```

Since the assignment to `bar` happens at the end in both scenarios, the compiler moved it to the common route which avoid duplicate instruction sequences.

These features are great... But they make my code not match the game's code. I believe I have the compiler flags pinned down since I have so many matching functions (Maximum Speed and Consistent Floats). Could these object files been compiled with different flags? I'm doubtful. It's way more likely the developers wrote code in way that prevented the optimization.


For the first case, the solution is quite easy. For the first condition write `bar` to a local variable and compare against it and leave the second check as is.

```c
void my_func()
{
	int localBar = this->bar;
	if (localBar == 1)
	{
		/* do something */
	}

	if (this->bar == -1)
	{
		/* do something else */
	}

}
```




The second type of optimization is the trickiest to fool without causing big changes to the output code so I tend not to overthink it. Luckily it hasn't been many the cases where I had to fool the compiler to output less ideal code.

# Building on other platforms

Even though I'm using a really old toolchain to get as close as possible to the original code, I also plan to compile and run the code in other environments. Making it compile and run on linux was quite simple, omit `windows.h` in non-windows environments and replace `WinMain` with `main`, all with macro magic.

While g++ worked like a breeze, there was some tweaking needed for Clang++ but nothing too extraordinary. A quirk that I've stumbled upon is that [g++/clang++ can get confused](https://github.com/krystalgamer/spidey-decomp/commit/5601a73e98c91f15420a351d666e79b4b5ded2a2) with expressions such as `0x54E-0x53C-4` and generate an error. I tend to use them when defining padding between class members when I don't know what lays in between.

```c

int field_53C;
unsigned char padAfter53C[0x54E-0x53C-4];
int field_54E;
```

The advantage of having an expression like this is that it makes it easier when I add a field between to adjust the padding. Microsoft's compiler [can do it just fine](https://godbolt.org/z/P8471jPz7) while both [g++](https://godbolt.org/z/K7c6af3YK) and [clang](https://godbolt.org/z/Mx361rzzG) fail. Two possible ways to avoid the problem is to constant fold myself or wrap each term in parentheses.

# Final thoughts

First of all I'd like to reiterate that this project is pretty laborious and can get boring. Everytime I work on it I set a goal of decompiling at least 10 functions, goal that I've been able to hit pretty consistently. Part of it comes from the fact that I review my stream footage to identify what I should keep doing and what derailed my progress.


Something that started to happen pretty casually was me linking to code snippets when talking about the game and it feels amazing. It really feels like everyday I know just a little more about the game and even though on a day over day basis the change is small compared to last progress report this doesn't look like the same project (in a good way!). I've told this to some people, but I do believe the game will be playable way before the code is fully decompiled. Even though, I've been delaying as much as possible doing any file loading or rendering routines it feels like the time is coming.

Finally, I'd like also to shoutout [foxtacles](https://github.com/foxtacles) - contributor to the [Lego Island decompilation](https://github.com/isledecomp/isle) - and [Ethan Roseman](https://github.com/ethteck) (ethteck) - contributor (creator?) of [decomp.me](https://decomp.me) - for letting me know about the tooling that has been developed to aid on decompilation projects. Even though only 2 months have gone by, a lot has changed. Let's see how the next two go!
