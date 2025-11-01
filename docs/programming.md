# Programming your Agon computer

Whilst there is a lot of great software you can download and run on your Agon, you can write your own software too. And this is probably the reason you bought your Agon!

There are several programming languages available, something for everyone, from beginner to expert... or those just reliving their 1980's memories.

Some of the languages run directly on your Agon computer. Some run on your Windows, Mac or Linux PC for transfer to your Agon. Plus, there are some which your can run both on the Agon and your PC.

Most of the languages available allow you to code in traditional 16bit or 24bit _ADL mode_, where the full amount of memory is available, not limited to 64k.

In order to help speed up development, there is also an emulator so your can program and test _most_ code quickly on your PC, before transferring to the Agon.

It doesn't matter which model you own from the Agon platform range of computers, inside they are all the same and can run exactly the same software. The only thing which might make your software be less compatible is if you utilise features from the latest OS builds where another user may not have updated their system. If you intend on sharing your programs, make sure you specifiy the minimum system of MOS/VDP required.

All languages listed here come with examples to help you get started.



## The Languages

### Basic

This most popular language is installed by many Agon owners and is a great way to quickly start programming your Agon computer. 

This is BBC BASIC from Richard Russell, and it has a direct line of heritage back to the original BBC Micro, and is based on the source code of the version from 1987 that ran on the Z80 second processor.

The Agon port of BBC Basic provides a familiar yet powerful language used by beginners and experts alike. It runs directly on the Agon computer and is often started automatically by many users with a command in in their system boot script `./MOS.md#boot-script`.

BBC Basic is available in two versions: Z80 version with a memory limit of 64k, and ADL version using all memory available.

More information on [Z80 version](https://github.com/breakintoprogram/agon-bbc-basic) or [ADL version](https://github.com/breakintoprogram/agon-bbc-basic-adl)

Or, for a quick update of your current version:

Download the [Z80 version](https://github.com/breakintoprogram/agon-bbc-basic/releases/latest)

Download the [ADL version](https://github.com/breakintoprogram/agon-bbc-basic-adl/releases/latest)

In addition, there is a port of Richard Russell's BASIC V available. Although it runs in Z80 mode (ie. limited to 64k of memory), BASIC V includes several new keywords, and supports a handful of useful new programming structures.

Download [BASIC V](https://github.com/breakintoprogram/agon-bbc-basic-v)


### Z80 Assembly

If you have ever written software in z80 Assembly for the Sinclair ZX81, ZX Spectrum, MSX, Amstrad CPC, TRS-80, Nintendo Game Boy, or even the Texas Instruments calulators, then this will be familliar territory for you.

You can write your code in whatever text editor you like, then assmble your source code with `ez80asm`, a z80 assembler which runs on the Agon itself, or on Mac, Windows, Linux, and even Raspberry Pi.

Download [ez80asm assembler](https://github.com/AgonPlatform/agon-ez80asm)


### C or C++

Many programmers enjoy coding in `C` or `C++`. This language is also available and there are a few options for coding on your desktop PC, then compiling and transferring onto your Agon to run.

A whole development toolchain is available for for Mac/Linux/Raspberry Pi toolchain called `AgonDev`. It can also be run on Windows using WSL. 

AgonDev is still in active development and is getting more advanced every day.

Download [AgonDev](https://github.com/AgonPlatform/agondev)


### B Simple

_B_ was a language which, unsurpisingly, was around before _C_. This interpretation, `B Simple`, allows you to create programs that will be familliar to anyone who has used C, but without all the advanced features of C++. B Simple can be programmed on the Agon itslef, or on your desktop PC under Mac, Linux, Windows, or Raspberry Pi. 

B Simple's approach is to compile your code into z80 assembler, then use the power of `ez80asm` (mentioned above) to assemble the compiled code into the final binary executable. 

Whilst B Simple has a limited set of variable types, it is fast and easy to use. Plus, it can easily be expanded by writing custom libraries in z80 assembler.

Download [B Simple](https://github.com/nihirash/bsimple-ez80-compiler)

### Forth

Some Agon users might reminisce on the days long gone of owning (or wishing they owned) a _Jupiter Ace_, with its niche _Forth_ programming language. 

`Agon Forth` runs on the Agon itself and provides a great way to learn the language.

Download [Agon Forth](https://github.com/lennart-benschop/agon-forth)

## The Emulator

If you create your programs on your PC (or Mac/Linux), the Emulator enables you to test them right there, before dowbnloading to your Agon.
There are some limitations, such as using additional hardwarde (eg. i2c devices, or io ports) but most other features are fully functional, and is updated regularly to keep in line with the latest Agon MOS/VDP operating system.

Download the latest version of [The Agon Emulator](https://github.com/tomm/fab-agon-emulator)

## Getting your code onto the Agon

If you have been building your Agon program on a PC (or Mac/Linux), there is a tool available called `Hexload` to help transfer the binary over to your Agon.

Further information and download [Hexload](https://github.com/AgonPlatform/agon-hexload)


Good luck with your programming. There is lots of community help and support available on the [Agon & Console8 Community Discord server](https://discord.com/invite/sN3vXru26s)

There are several getting started guides on the internet too, as well as YouTube videos, such as those from [AgonBits](https://www.youtube.com/@AgonBits)


