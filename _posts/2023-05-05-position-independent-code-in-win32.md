---
title: What is Position Independent Code ?
categories: [programming, win32 , study]
tags: [lowlevel]
---

# Wtf is that again

I write this topic for explain what is and why is it important for me during my learning at windows low-level malware developpement (for virology study).

____

## What is Position Independent Code

Position Independent Code (PIC) is machine code that can be executed independently of its location in memory. 

This means that the code can be loaded at any address in memory and work correctly without the need for specific adjustments.

Absolute addresses are avoided whenever possible and relative addresses are used instead. 
PICs are often used for shared libraries because they can be loaded at different addresses in memory for each program that uses them.

___

Exemple with this C code.

```c
#include <windows.h>

// ...

// Allows to link a lib directly from the code
#pragma comment (lib, "user32.lib")

// ...

// Call windows msgbox ascii version function
MessageBoxA(NULL, "Hello :)", "oh", 0);

// ...

```

Ok. This is simply link a lib and call a function present on that lib at compilation.

> What is going on if we inject a shellcode of this kind code into curent or remote process memory ?

In order to be `independent`, we need to achieve that **dynamicly**. Because here the code use absolute address instead of relative address.

Join the game "Position Independent Code".

A light working version of this code can be :

```c
#include <windows.h>

// The GetProcAddress function is used to get the addresses of Win32 API functions
FARPROC GetProcAddressPortable(HMODULE hModule, const char* lpProcName) {
    return GetProcAddress(hModule, lpProcName);
}

// Entrypoint prototype for WINDOWS subsystem
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
    // Load dynamic library 'user32.dll'
    HMODULE hUser32 = LoadLibrary(TEXT("user32.dll"));

    if (hUser32 != NULL) {
        // Get address of MessageBoxA function position-independently
        FARPROC pMessageBoxA = GetProcAddressPortable(hUser32, "MessageBoxA");

        if (pMessageBoxA != NULL) {
            // Call MessageBoxA position-independently
            ((int (WINAPI *)(HWND, LPCSTR, LPCSTR, UINT))pMessageBoxA)(NULL, "Hello, World!", "Position Independent Code", MB_OK);

            // Free lib ptr alloc
            FreeLibrary(hUser32);
        }
    }

    return 0;
}
```

Here we use win32 api function to get relative address from OS library. As we can see here :

```c
((int (WINAPI *)(HWND, LPCSTR, LPCSTR, UINT))pMessageBoxA)(NULL, "Hello, World!", "Position Independent Code", MB_OK);
```

So,

- **pMessageBoxA** is a function pointer that points to the address of the **MessageBoxA** function.
- `((int (WINAPI *)(HWND, LPCSTR, LPCSTR, UINT))pMessageBoxA)` is a cast of the function pointer to the appropriate function type. In other words, we're telling the compiler that pMessageBoxA is a pointer to a function that returns an int, takes four arguments `(HWND, LPCSTR, LPCSTR, UINT)`, and follows the **WINAPI** calling convention.
- `(NULL, "Hello, World!", "Position Independent Code", MB_OK)` are the arguments we pass to the `MessageBoxA` function. `NULL` is the HWND (the parent window handle, here we don't have a parent window), `"Hello, World!"` is the message to display in the message box, `"Position Independent Code"` is the title of the message box, and `MB_OK` is a flag indicating that the message box should have an `"OK"` button.

Putting it all together, we call the `MessageBoxA` function with the supplied arguments using the `pMessageBoxA` function pointer. This allows the function to be called **position-independently**.

