---
title: Personnal Windows Study
categories: [programming, win32 , study]
tags: [lowlevel]
---

# Alternative to ADVAPI32.dll registry functions

Hello folks !

One days i've waking up with the idea to write up somes of my studies on Windows.

This article is not intended for a specific audience. 

and that's what I'm going to do, starting with the desire to test the registry functions present in the ntdll library in alternative to advapi32.dll


The goal is to respect somes rules. 
We're going to follow these:
- Dynamic call NT functions
  - using dynamic low level ntdll calls
- Reimplement all needed functions
  - including somes win32 functions
- The less dependencies possible
- Code in C (i've don't disable CRT but i don't gonna use it)

___

## Dynamic api call on windows

Lets beginning with the classic GetProcAddress function reimplementation needed for getting a function pointer here for our NT functions.

```c
// string comparaison
int _strcmp(const char* str1, const char* str2)
{
    while (*str1 != '\0' && *str2 != '\0' && (*str1 == *str2)) {
        str1++;
        str2++;
    }

    return (*str1 - *str2);
}

// reimplementation of GetProcAddress in kernel32.dll 
// get a function pointer address by looping through ExportDirectory structure compairing function names.
FARPROC GetProcAddressCustom(HMODULE hModule, const char* lpProcName)
{
    PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)hModule;
    PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)((LPBYTE)pDosHeader + pDosHeader->e_lfanew);
    PIMAGE_EXPORT_DIRECTORY pExportDirectory = (PIMAGE_EXPORT_DIRECTORY)((LPBYTE)pDosHeader + pNtHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress);

    DWORD* pNames = (DWORD*)((LPBYTE)pDosHeader + pExportDirectory->AddressOfNames);
    WORD* pOrdinals = (WORD*)((LPBYTE)pDosHeader + pExportDirectory->AddressOfNameOrdinals);
    DWORD* pFunctions = (DWORD*)((LPBYTE)pDosHeader + pExportDirectory->AddressOfFunctions);

    for (DWORD i = 0; i < pExportDirectory->NumberOfNames; ++i)
    {
        const char* pName = (const char*)((LPBYTE)pDosHeader + pNames[i]);
        if (_strcmp(pName, lpProcName) == 0)
        {
            WORD wOrdinal = pOrdinals[i];
            DWORD dwFuncRVA = pFunctions[wOrdinal];
            FARPROC pFunction = (FARPROC)((LPBYTE)hModule + dwFuncRVA);
            return pFunction;
        }
    }

    return NULL;
}
```
{: .nolineno }

Right, here we have to use PE headers the module passed in parameter. So the purpose being loop in all functions and check their name by simple string comparaison.

The structs used here are present when we use windows.h header, by the way here a exemple of ImageExportDirectory:

```c
// exemple of structure used (not needed to right)
typedef struct _IMAGE_EXPORT_DIRECTORY {
    DWORD   Characteristics;
    DWORD   TimeDateStamp;
    WORD    MajorVersion;
    WORD    MinorVersion;
    DWORD   Name;
    DWORD   Base;
    DWORD   NumberOfFunctions;
    DWORD   NumberOfNames;
    DWORD   AddressOfFunctions;     // RVA from base of image
    DWORD   AddressOfNames;         // RVA from base of image
    DWORD   AddressOfNameOrdinals;  // RVA from base of image
} IMAGE_EXPORT_DIRECTORY, *PIMAGE_EXPORT_DIRECTORY;
```
{: .nolineno }

## Getting ntdll module handle

Now we'll need to get the ntdll module handle, and for that we can call:
```c
HMODULE ntdll;
GetModuleHandleExW(0, L"ntdll.dll", &ntdll);
```
{: .nolineno }

Like ntdll is loaded at runtime of any PE no need to load `ntdll.dll`.

Instead of that we'll getting ntdll module handle dynamicly by reading the `PEB` *(Process Environnment Block)*. 
Which is located the current ntdll module handle if we make a little parse.

lets begin with needed reimplementations

```c
wchar_t towlower(wchar_t ch)
{
    if (ch >= L'A' && ch <= L'Z')
    {
        return ch + (L'a' - L'A');
    }
    else
    {
        return ch;
    }
}

int wcsicmp(const wchar_t* str1, const wchar_t* str2)
{
    while (*str1 != L'\0' && *str2 != L'\0')
    {
        if (towlower(*str1) != towlower(*str2))
        {
            return towlower(*str1) - towlower(*str2);
        }

        str1++;
        str2++;
    }

    return towlower(*str1) - towlower(*str2);
}
```
{: .nolineno }

and the rest of the code here.

```c
#include <winternl.h>
#include <intrin.h>

// using intrinsect microsoft mecanism to get x64 PEB
PPEB GetPEB()
{
    // Retrieve the address of the current TEB using the special GS register
    PTEB pTeb = (PTEB)__readgsqword(0x30); 
    // Retrieve the pointer to the PEB structure from the TEB
    PPEB pPeb = pTeb->ProcessEnvironmentBlock; 
    // Return the pointer to the PEB structure
    return pPeb; 
}

HMODULE GetNtdllModuleHandle()
{
    // Retrieve the pointer to the PEB structure
    PPEB peb = GetPEB(); 
    // Get the address of the first loaded module in the module list
    PLDR_DATA_TABLE_ENTRY ldrEntry = (PLDR_DATA_TABLE_ENTRY)peb->Ldr->InMemoryOrderModuleList.Flink; 
    // Initialize the handle to NULL
    HMODULE hNtdll = NULL; 

    // Iterate over the loaded module list
    while (ldrEntry != NULL) 
    {
        // If the module name is available
        if (ldrEntry->FullDllName.Buffer != NULL) 
        {
            // If the module name matches "ntdll.dll"
            if (wcsicmp(ldrEntry->FullDllName.Buffer, L"ntdll.dll") == 0) 
            {
                // Retrieve the handle of the module
                hNtdll = (HMODULE)ldrEntry->DllBase; 
                // Exit the loop
                break; 
            }
        }

        // Move to the next element in the list
        ldrEntry = (PLDR_DATA_TABLE_ENTRY)ldrEntry->InMemoryOrderLinks.Flink; 
    }

    // Return the handle of the "ntdll.dll" module
    return hNtdll; 
}
// Test
int main()
{
    HMODULE hNtdll = GetNtdllModuleHandle();
    LPVOID ntopenkey = GetProcAddressCustom(hNtdll, "NtOpenKey");
    printf("ntdll.dll module handle: %p\n", hNtdll);
    printf("ntdll.dll NtOpenKey pointer address: %p\n", ntopenkey);
    return 0;
}
```
{: .nolineno }

> Here commented code to get ntdll module handle (x64) on Windows.
{: .prompt-info }


WIP....