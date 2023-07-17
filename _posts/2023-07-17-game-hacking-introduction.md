---
title: Game Hacking Introduction
categories: [programming, game]
tags: [lowlevel]
---

<style>
r { color: Red }
b { color: Blue }
g { color: Green }
</style>

![Coordinates](/assets/supra.jpg)
_Supraland used exemple game_

Today we're going to talk about understanding how works low-level Game Hacking.

I was playing to **Supraland** game and while playing I had the idea to find the **XYZ coords** of my player **in the space**. This gave me the idea to write this article.

___
# Summary
- Low-Level Game Hacking ?
    - Internal
    - External
- Memory manipulation
- Hack the planet
    - Practical exemple
- Code injection

# Game Hacking ?

> To understand this article at least, you need at least a knowledge base in the computer field called "system" so low level.
{: .prompt-info }

Low-level game hacking involves the direct manipulation of games code and memory to gain specific advantages, modifications, or exploits.

For this there are two approaches in my opinion to achieve this. The `internal` and `external` approach.

## Internal
An internal game hack is a code that can be executed in the same address space than the program you want to reverse. 

>By exemple, a `dll_injection.dll` into the game `Supraland.exe` to ...

## External
An external game hack is a external program that read data in remote way from another external program.

> By exemple, `hack.exe` opens `Supraland.exe` in memory to ...

___

# Memory manipulation

`TODO`

___
# Hack the planet !

![Coordinates](/assets/xyz.png)
_XYZ Coordinates_

Firstly i've beginning with the Z axis which is in the game logic, the height in the space.

To find this we've going to "brute-force" an unknown value by filtering his state, and so what we know about this value in the memory.

### Important thing to know
#### Value
> Your ammo count could be represented by a numeric value.

#### Address
> An address represents a specific position in computer memory where a particular value is stored.

#### Pointer
> Pointers are often used when values are stored in dynamic memory locations or when they are spread across multiple addresses.

#### Static Address Pointer
> There're pointers that do not change between sessions game. (**restart**)

___

## Practical exemple

> We're going to RE a process with the tool named `Cheat Engine`. I use that tool to scan remote addresses and pointers.

#### My procedure to get **Z axis static address pointer** in a game:

> Getting static pointer can be a tedious thing to do.
{: .prompt-warning }

1. **Scan all addresses** in memory with an unknown value 
2. **Filter** value by increasing value or decreasing it **in game**.
3. Once a address is found:
    1. Check if in game we see that we can freeze the value from memory tool
    2. Generate a pointermap with this address
4. Restart the game and keep addresses and values in memory tool
5. Start the whole process again until step 3.2
6. Generate pointer scan for this address `(the second value we found)`
7. Locate the current value from game to pointermap scan
8. Restart the game and continue to filter all thoses thing until we've ~100 ~50 pointer and restart the process until we've got a static address pointer.

___

Let's imagine we found the pointer we wanted. It can looks like to :

> "Supraland-Win64-Shipping.exe" + 04C135DF -> 0C 10 155 6DA 12<br>
`Base Address` + `Address found` -> `offsets`
{: .prompt-info}

# [Work in Progress]<br>...
{:style="text-align: center;"}