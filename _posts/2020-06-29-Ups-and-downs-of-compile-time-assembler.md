---
layout: post
title:  "Ups and downs of compile time assembler"
date:   2020-06-20 19:00:00 +0000
tags: [ Cpp, C++, metaprogramming, DSL, domain, specific, language, karta, assembler, compile, time, compilation]
---

Hey ho,
Over two months ago I shared a teaser for the project I was about to start. Contrary to my other brilliant-yet-useless projects, this one got some traction.
The name of the game is "x86 assembler at compilation time". 

{% include twitters.html %}

This is karta. At the very beginning I envisioned it just like I promoted it - the syntax would be close to x86 Intel syntax asm, but nooot quite. Having spent a lot of time prototyping it though, it now looks closer to:
```cpp
using namespace karta::x86;
// Second operand's type determines immediate size.
// That's not something I like, but I guess it has it's pros.
mov(eax, std::uint8_t(8)); 
mov(eax, cs[eax]);
mov(edx, cs[1234]);
mov(ecx, edx);
mov(eax, ds[edx + eax * _2]); 
```
In my opinion that's way nicer than the previous syntax (which screamed *I AM A TEMPLATE* to the user). Also, all of that is legal karta syntax.

Hold up - why bother with all of that? The project probably requires a lot of tedious work? What if you make a mistake? How are the compile times?
So..

### Why bother?
Having spent a bit of time looking at code of some JIT engine which constructed the instructions byte by byte I wished it was more readable. I'm a total rookie with asm encodings, so looking at DSL for x86 asm would certainly be more pleasant. 
That's how karta came about. 

### Taking the first step
I started by probing the interest on Twitter for DSL asm library - in hindsight it was a good decision, as I felt it kept me accountable. The probe got pretty popular (by my Twitter standards).
Thus I got to coding my first instruction - mov.
After implementing I think two variants ("mov reg, imm" and "mov reg, reg") I noticed that I'm in dire needs of tests - after all, the amount of possible instruction encodings was enormous. This is why I'm currently working on a subproject named tcg, which - given specification of instructions ("what kind of arguments does it take") - can generate .nasm files, convert them to .obj files and generate unit tests (currently based on Catch2) with karta call result being the value that's compared against.
There is one tiiiiny issue - test cases for unfinished mov implementation take half an hour to generate.
It's a tip of the iceberg - are many possible encodings. I'm not gonna wait around for my builds to finish! 
Karta itself is not an offender here - the snippets are pretty lightweight.. But in case of mov there were 100 000 test cases that the compiler had to chew through in single translation unit. Ouch!

### ... and immediately fumbling over
So this is what I am currently struggling with. I want tcg to scale, as the biggest hurdle so far was getting memory encoding right. I believe that if tests would be quick to build, I could iterate up to full x86 asm parity within two months of work.

Of course there are other issues on the horizon - some of which are not that big of an issue anymore. One of them was cross-calling to C++ code (e.g: call(std::printf);). There's an issue: it can't be constexpr. The address of an entity is not known until link time and that's (probably) why reinterpret\_cast is banned in constexpr context (reinterpret\_cast to const char\* would do the trick for me, but.. tough luck). Therefore while it might be allowed, I'm not sure if it would be your daily way to use karta.

### How you can help

It would be easier for me if I knew what qualities would make karta valid for your use case (assuming permissive license):
1. Do you have an use case for karta in some of your projects?
2. If so, do you care about everything being constexpr?
3. Can your project use C++20?

Answers to these questions would guide my work a bit - right now I'm the sole potential user, so the project reflects my needs. :)

Thanks for taking your time to read through all of that. Have a good day!

