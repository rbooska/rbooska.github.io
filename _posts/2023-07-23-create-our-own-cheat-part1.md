---
title: C++ simple cheat template (in French)
categories: [programming, game]
tags: [lowlevel]
---

# C++ Cheat boiler plate 

Une fois l'adresse mémoire statique trouvé et testé, par exemple :

> "Supraland-Win64-Shipping.exe" + 04C135DF -> 0C 10 155 6DA 12<br>
`Base Address` + `Address found` -> `offsets`
{: .prompt-info}

Il est grand temps de développer notre propre outil en C/C++. Pour cela...

> "Adresse mémoire DMA *(Direct Memory Access)*" : Chemin d'une adresse mémoire accesible en direct. C'est ce qu'on a qualifié d'adresse statique plus haut. <br>S'obtient par scan AoB *(Array of Bytes)* ou par adresse mémoire & offsets directement.
{: .prompt-tip}

Pour se faire, nous allons faire un cheat externe qui va utiliser les API Win32 sous Windows. Il va :
- ouvrir le processus du jeu Supraland 
- obtenir l'adresse mémoire de base du module
- faire une lecture mémoire sur une adresse mémoire DMA
- faire une écriture mémoire sur l'adresse mémoire lue
    - comme c'est un float (Z) ça donnera dans ce genre: `write_float(readFloat(z_axis) + 10.0f)`
- regarder en jeu si on est monté d'un cran en jeu

Voici le schéma nominal du programme, et dans une boucle on verrait le personnage monter à l'infini.

Voici les fichiers que nous allons créer:

`main.cpp`{: .filepath}<br>
`cheat.cpp & cheat.h`{: .filepath}<br>
`memory.cpp & memory.h`{: .filepath}<br>
`offsets.cpp & offsets.h`{: .filepath}<br>

Son arborescence:
```
cheat/
├─ src/
│  ├─ main.cpp
│  ├─ cheat.cpp
│  ├─ memory.cpp
│  ├─ offsets.cpp
│  ├─ cheat.h
│  ├─ memory.h
│  ├─ offsets.h
```
___
> On trouve ici tous les offsets et adresse à lire ou patcher.
{: .prompt-tip}
`offsets.h`{: .filepath}<br>
```c
static uintptr_t x_coordAddr = 0x041C1560;
static std::vector<unsigned int> x_offsets { 0x9C0, 0x6A0, 0x588, 0x20, 0x520, 0xC0, 0x1D0 };

static uintptr_t y_coordAddr = 0x041C1560;
static std::vector<unsigned int> y_offsets { 0xB88, 0x500, 0x20, 0x6A0, 0x630, 0xC0, 0x1D4 };

static uintptr_t z_coordAddr = 0x045F1C38;
static std::vector<unsigned int> z_offsets { 0x18, 0x110, 0x5D8, 0x20, 0x4C0, 0x550, 0x1D8 };

static uintptr_t o_coordAddr = 0x041C1560;
static std::vector<unsigned int> o_offsets { 0x9C0, 0x6A0, 0x588, 0x20, 0x520, 0xC0, 0x1A8 };


static uintptr_t gravityAddr = 0x041C1560;
static std::vector<unsigned int> gravity_offsets { 0xC48, 0x6A0, 0x38, 0x8, 0x488, 0x58, 0xCC };
```
> Ici on effectue tout ce qui est en rapport à la mémoire de façon générale
{: .prompt-tip}
`memory.h`{: .filepath}<br>
```c
#pragma once
#include <windows.h>
#include <stdio.h>
#include <psapi.h>
#include <vector>

DWORD FindProcessIdByName(WCHAR* processName);
uintptr_t GetRemoteProcessBaseAddress(DWORD processId);
uintptr_t FindDMAAddy(HANDLE hProc, uintptr_t ptr, std::vector<unsigned int> offsets);
uintptr_t FindDMAAddy(HANDLE hProc, uintptr_t ptr, const unsigned int* offsets, size_t offsetCount);
int countOffsets(unsigned int offsets[]);
```
`memory.cpp`{: .filepath}<br>
```c
#include "memory.h"
#include "tlhelp32.h"


DWORD FindProcessIdByName(WCHAR* processName)
{
    DWORD processId = 0;
    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

    if (snapshot != INVALID_HANDLE_VALUE)
    {
        PROCESSENTRY32 processEntry;
        processEntry.dwSize = sizeof(PROCESSENTRY32);

        if (Process32First(snapshot, &processEntry))
        {
            do
            {
                if (lstrcmpW(processEntry.szExeFile, processName) == 0)
                {
                    processId = processEntry.th32ProcessID;
                    break;
                }
            } while (Process32Next(snapshot, &processEntry));
        }

        CloseHandle(snapshot);
    }

    return processId;
}

uintptr_t GetRemoteProcessBaseAddress(DWORD processId)
{
    HANDLE hProcess = OpenProcess(PROCESS_QUERY_INFORMATION | PROCESS_VM_READ, FALSE, processId);
    if (hProcess == NULL)
    {
        printf("Failed to open the remote process. Error: %d\n", GetLastError());
        return 0;
    }

    HMODULE hModules[1024];
    DWORD cbNeeded;
    if (EnumProcessModules(hProcess, hModules, sizeof(hModules), &cbNeeded))
    {
        MODULEINFO moduleInfo;
        if (GetModuleInformation(hProcess, hModules[0], &moduleInfo, sizeof(moduleInfo)))
        {
            CloseHandle(hProcess);
            return (uintptr_t)moduleInfo.lpBaseOfDll;
        }
    }

    CloseHandle(hProcess);
    return 0;
}

uintptr_t FindDMAAddy(HANDLE hProc, uintptr_t ptr, std::vector<unsigned int> offsets)
{
    uintptr_t addr = ptr;
    for (unsigned int i = 0; i < offsets.size(); ++i)
    {
        ReadProcessMemory(hProc, (BYTE*)addr, &addr, sizeof(addr), 0);
        addr += offsets[i];
    }
    return addr;
}

uintptr_t FindDMAAddy(HANDLE hProc, uintptr_t ptr, const unsigned int* offsets, size_t offsetCount)
{
    uintptr_t addr = ptr;
    SIZE_T bytesRead;

    for (size_t i = 0; i < offsetCount - 1; ++i)
    {
        if (!ReadProcessMemory(hProc, (LPCVOID)addr, &addr, sizeof(addr), &bytesRead) || bytesRead != sizeof(addr))
        {
            return 0;  // �chec de la lecture de la m�moire, retourne 0 ou une valeur d'erreur appropri�e
        }

        addr += offsets[i];
    }

    uintptr_t lastOffset = offsets[offsetCount - 1];
    addr += lastOffset;

    return addr;
}

int countOffsets(unsigned int offsets[]) {
    return sizeof(offsets) / sizeof(offsets[0]);
}
```
> C'est ici qu'on fait le traitement nécessaire pour pouvoir lire une adresse mémoire externe
{: .prompt-tip}
`cheat.h`{: .filepath}<br>
```c
#pragma once
#include "memory.h"
#include "offsets.h"

void showValueFloat(HANDLE hprocess, uintptr_t addr, std::vector<unsigned int> offsets);
void showValueFloat(HANDLE hprocess, uintptr_t addr, std::vector<unsigned int> offsets, const char* text);
void showValueInt(HANDLE hprocess, uintptr_t addr, std::vector<unsigned int> offsets);
void showValueInt(HANDLE hprocess, uintptr_t addr, std::vector<unsigned int> offsets, const char* text);

float readFloat(HANDLE hprocess, uintptr_t addr, std::vector<unsigned int> offsets);
BOOL writeFloat(HANDLE hprocess, uintptr_t address, std::vector<unsigned int> offsets, float value);
BOOL writeFloat(HANDLE hprocess, uintptr_t address, std::vector<unsigned int> offsets, const char* text, float value);

void force_coords(HANDLE hProcess, uintptr_t addr, std::vector<unsigned int> offsets, float speed_delta, char symbol);
void block_Z(HANDLE hProcess, uintptr_t base_address);
```
`cheat.cpp`{: .filepath}<br>
```c
#include "cheat.h"

void showValueFloat(HANDLE hprocess, uintptr_t addr, std::vector<unsigned int> offsets, const char* text) {
    float value;
    uintptr_t ptr = FindDMAAddy(hprocess, addr, offsets);
    if (ReadProcessMemory(hprocess, (LPCVOID)ptr, &value, sizeof(value), NULL))
        printf("%s: %f\n", text, value);
    else
        printf("Failed to read the value at the final address. Error: %d\n", GetLastError());
}

void showValueInt(HANDLE hprocess, uintptr_t addr, std::vector<unsigned int> offsets, const char* text) {
    int value;
    uintptr_t ptr = FindDMAAddy(hprocess, addr, offsets);
    if (ReadProcessMemory(hprocess, (LPCVOID)ptr, &value, sizeof(value), NULL))
        printf("%s: %d\n", text, value);
    else
        printf("Failed to read the value at the final address. Error: %d\n", GetLastError());
}

float readFloat(HANDLE hprocess, uintptr_t addr, std::vector<unsigned int> offsets)
{
    float value = 0.0f;
    uintptr_t ptr = FindDMAAddy(hprocess, addr, offsets);
    if (!ReadProcessMemory(hprocess, (LPCVOID)ptr, &value, sizeof(value), NULL))
        printf("Failed to read the value at the final address. Error: %d\n", GetLastError());
    return value;
}

BOOL writeFloat(HANDLE hprocess, uintptr_t address, std::vector<unsigned int> offsets, float value)
{
    uintptr_t ptr = FindDMAAddy(hprocess, address, offsets);
    BOOL writeResult = WriteProcessMemory(hprocess, (LPVOID)ptr, &value, sizeof(value), NULL);
    if (writeResult == FALSE)
    {
        printf("Failed to write to process memory. Error: %d\n", GetLastError());
        return FALSE;
    }

    return TRUE;
}

BOOL writeFloat(HANDLE hprocess, uintptr_t address, std::vector<unsigned int> offsets, const char* text, float value)
{
    uintptr_t ptr = FindDMAAddy(hprocess, address, offsets);
    BOOL writeResult = WriteProcessMemory(hprocess, (LPVOID)ptr, &value, sizeof(value), NULL);
    if (writeResult == FALSE)
    {
        printf("Failed to write to process memory. Error: %d\n", GetLastError());
        return FALSE;
    }

    printf("%s: %f\n", text, value);

    return TRUE;
}


void force_coords(HANDLE hProcess, uintptr_t addr, std::vector<unsigned int> offsets, float speed_delta, char symbol) {
    float coord = readFloat(hProcess, addr, offsets);
    switch (symbol)
    {
    case '+':
        writeFloat(hProcess, addr, offsets, "-> written value", coord + speed_delta);
        break;
    case '-':
        writeFloat(hProcess, addr, offsets, "-> written value", coord - speed_delta);
        break;
    case '*':
        writeFloat(hProcess, addr, offsets, "-> written value", coord * speed_delta);
        break;
    default:
        break;
    }
}

void block_Z(HANDLE hProcess, uintptr_t base_address)
{
    float z = readFloat(hProcess, base_address + z_coordAddr, z_offsets);
    float gravity = readFloat(hProcess, base_address + gravityAddr, gravity_offsets);
    writeFloat(hProcess, base_address + z_coordAddr, z_offsets, z+0.001f);
    writeFloat(hProcess, base_address + gravityAddr, gravity_offsets, 0.0f);
}

// TODO: AOB Scan method
```
___
### Code final :
> C'est la boucle principale du programme
{: .prompt-tip}
`main.cpp`{: .filepath}<br>
```c
#include <windows.h>
#include <stdio.h>
#include "memory.h"
#include "offsets.h"
#include "cheat.h"
#include "camera.h"

const wchar_t* module_name = L"Supraland-Win64-Shipping.exe";
int cheat_speed_clock = 5;
float xy_speed = 100.0f;
float z_speed = 500.0f;



void cls()
{
    HANDLE hConsole = GetStdHandle(STD_OUTPUT_HANDLE);
    CONSOLE_SCREEN_BUFFER_INFO csbi;
    SMALL_RECT scrollRect;
    COORD scrollTarget;
    CHAR_INFO fill;
    if (!GetConsoleScreenBufferInfo(hConsole, &csbi))
        return;
    scrollRect.Left = 0;
    scrollRect.Top = 0;
    scrollRect.Right = csbi.dwSize.X;
    scrollRect.Bottom = csbi.dwSize.Y;
    scrollTarget.X = 0;
    scrollTarget.Y = (SHORT)(0 - csbi.dwSize.Y);
    fill.Char.UnicodeChar = TEXT(' ');
    fill.Attributes = csbi.wAttributes;
    ScrollConsoleScreenBuffer(hConsole, &scrollRect, NULL, scrollTarget, &fill);
    csbi.dwCursorPosition.X = 0;
    csbi.dwCursorPosition.Y = 0;
    SetConsoleCursorPosition(hConsole, csbi.dwCursorPosition);
}


int main()
{
    DWORD processId = FindProcessIdByName((WCHAR*)module_name);
    HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, processId);
    if (hProcess == NULL) {
        printf("Failed to open process with id %lu\n", processId);
        return 1;
    }

    uintptr_t baseAddress = GetRemoteProcessBaseAddress(processId);
    if (baseAddress == 0)
    {
        printf("Failed to get the base address of the remote process.\n");
        CloseHandle(hProcess);
        return 1;
    }

    struct Vector3 coords;
    BOOL lock = FALSE;
    while (true) {
        cls();
        printf("Supraland Cheat\n\n");
        printf("Commands :\n");
        printf("\t[K] : Enable/Disable trainer \n");
        printf("\t[P] : Block Z axis\n");
        printf("\t[SPACE] : Super jump\n");
        printf("\t[CTRL] : Reverse super jump\n");
        printf("\t[Num.5] : Increase X axis\n");
        printf("\t[Num.2] : Decrease X axis\n");
        printf("\t[Num.6] : Increase Y axis\n");
        printf("\t[Num.3] : Decrease Y axis\n");

        printf("\n\n");

        if (GetAsyncKeyState('M') & 0x8000) {
            cls();
            printf("Supraland Cheat\n\n");
            printf("Bye bye\n\n");
            break;  // Sort de la boucle si la touche 'Q' est enfoncée
        }

        if (GetAsyncKeyState('K') & 0x8000)
        {
            lock = !lock;
            Sleep(200);
        }

        if (!lock)
        {
            if (GetAsyncKeyState(VK_SPACE) & 0x8000)
                force_coords(hProcess, baseAddress + z_coordAddr, z_offsets, z_speed, '+');

            if (GetAsyncKeyState(VK_CONTROL) & 0x8000)
                force_coords(hProcess, baseAddress + z_coordAddr, z_offsets, z_speed, '-');

            if (GetAsyncKeyState(VK_NUMPAD5) & 0x8000)
                force_coords(hProcess, baseAddress + x_coordAddr, x_offsets, xy_speed, '+');

            if (GetAsyncKeyState(VK_NUMPAD2) & 0x8000)
                force_coords(hProcess, baseAddress + x_coordAddr, x_offsets, xy_speed, '-');

            if (GetAsyncKeyState(VK_NUMPAD6) & 0x8000)
                force_coords(hProcess, baseAddress + y_coordAddr, y_offsets, xy_speed, '+');

            if (GetAsyncKeyState(VK_NUMPAD3) & 0x8000)
                force_coords(hProcess, baseAddress + y_coordAddr, y_offsets, xy_speed, '-');

            if (GetAsyncKeyState(VK_NUMPAD8) & 0x8000) {
                //MoveInCameraDirection()
            }

            if (GetAsyncKeyState('P') & 0x8000) {
                printf("Blocking Z axis...");
                while (true)
                {
                    if (GetAsyncKeyState('O') & 0x8000)
                        break;
                    
                    block_Z(hProcess, baseAddress);
                }
            }
        }
        else
        {
            printf("\t!!! Commands locked !!!\n\n");
        }

        printf("Player Coords:\n");
        coords.x = readFloat(hProcess, baseAddress + x_coordAddr, x_offsets);
        coords.y = readFloat(hProcess, baseAddress + y_coordAddr, y_offsets);
        coords.z = readFloat(hProcess, baseAddress + z_coordAddr, z_offsets);

        showValueFloat(hProcess, baseAddress + x_coordAddr, x_offsets, "\tx");
        showValueFloat(hProcess, baseAddress + y_coordAddr, y_offsets, "\ty");
        showValueFloat(hProcess, baseAddress + z_coordAddr, z_offsets, "\tz");
        showValueFloat(hProcess, baseAddress + o_coordAddr, o_offsets, "\to");

        // pause between each loop
        if (cheat_speed_clock > 0)
            Sleep(cheat_speed_clock);
    }


    CloseHandle(hProcess);
    return 0;
}
```

# [Work in Progress]<br>...
{:style="text-align: center;"}