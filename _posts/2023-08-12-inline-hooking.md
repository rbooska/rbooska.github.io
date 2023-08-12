---
title: x64 Inline Hooking (in French)
categories: [programming, win32, hack]
tags: [lowlevel]
---


Hello tout le monde ! Aujourd'hui je voulais vous partager un billet de blog sur le **Hooking** dit **"inline"**. 

> Disclamer: Attention, cet article est a but purement éducatif et (l'auteur) me désolidarise de toute responsabilités.
{: .prompt-warning }

___
# Introduction 📖

Dans l'univers de la *programmation*, nous avons deux type de langages.

Les langages dits:
- **Interprété / Managé**
- **Natif**

Nous allons nous concentrer ici sur le type dit **Natif** dans un environnement **Windows 10/11 x64 bit** *(x86_64)*. 

Il va être question de faire de l'instrumentation.

>En [informatique](https://fr.wikipedia.org/wiki/Informatique "Informatique"), l'**instrumentation du code** est une opération consistant à ajouter des [instructions machine](https://fr.wikipedia.org/wiki/Instruction_machine "Instruction machine") supplémentaires à un [programme informatique](https://fr.wikipedia.org/wiki/Programme_informatique "Programme informatique") sans nécessiter la modification du [code source](https://fr.wikipedia.org/wiki/Code_source "Code source") original.
{: .prompt-info }

# Hooking ? 🪝

C'est une technique avancé en programmation qui consiste à modifier une fonction existante dans un programme afin de :

- **Modification du comportement d'une fonction** : 
	- modifier les arguments passés à une fonction
	- modifier le retour d'une fonction
	- empêcher l'exécution d'une fonction
- **Tracer l'exécution d'une fonction** : 
	- tracer les appels à une fonction
	- tracer les arguments passés à une fonction
	- tracer le retour d'une fonction
- **Désactiver une fonction** : 
	- désactiver une fonction qui est malveillante
	- qui ne fonctionne pas correctement.

On voit le genre bref.
## Commençons ➡️

> Pour se faire nous allons utiliser un langage natif permettant une gestion flexible de la *mémoire*. Nous allons utiliser du **C** avec les **API Win32**.
{: .prompt-tip }

Dans un premier temps nous allons tester la fonction normalement puis commencer l'hameçonnage.

Voici à quoi va ressembler notre code de base :
```c
#include <Windows.h> // API Win32
#include <stdio.h>   // Pas utile fondamentalement (printf)

// pointeur vers notre fonction de hook
BYTE* pMsgbox;
// Octets de l'ancienne fonction MessageBoxA sauvegardé 
// avant hook pour pouvoir déinstaller le hook
BYTE oldBytes[12] = { 0 };

// prototype de la fonction à hook
int hook_msgbox(HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT Type);

// définition de la fonction de hook (avec le même profile que la fonction à hook)
int hook_msgbox(HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT Type)
{
    printf("[*] hook_msgbox called\n");
    delete_hook();
    return MessageBoxA(hWnd, "Hooked MessageBoxA function ! :)", "* Hooked *", Type);;
}

// désinstalle le hook
// void delete_hook();

// installe le hook
// void set_hook();

int main()
{
    printf("x64 windows inline hook test !\n\n");

    // obtenir un pointer vers une fonction, ici MessageBoxA (popup)
    pMsgbox = GetProcAddress(GetModuleHandleA("user32.dll"), "MessageBoxA");
    if (pMsgbox == 0)
        return 1;

    // loop 5 fois pour tester le hook puis retire le hook
    int i = 0;
    while (i < 5)
    {
        set_hook();
        MessageBoxA(0, "Test", "Test", 0x40);
    }

    delete_hook();
}
```

Une fois que la lecture est faite, il faut qu'on se concentre sur l'écriture des deux fonctions qui vont déclencher un changement de comportement.


## Installer le hook 🌱

Bien ! nous arrivons au vif du sujet.

Le but ici est de remplacer les 12 premiers octets de la fonction que l'on veux hook par un saut écris directement en opcode assembleur x64.

```nasm
mov rax @hook_msg
jmp rax
```
___
>En assembleur réel, vous utiliseriez plutôt l'adresse réelle de la fonction, par exemple `mov rax, 0x12345678` où `0x12345678` serait l'adresse mémoire réelle de `hook_msg`.
Mais nous sommes en C et nous pouvons faire cette abstraction. 
{: .prompt-tip }

Explication : 

**Ligne 1**: `mov rax, @hook_msg`

>Cette instruction copie (ou "charge") l'adresse de la fonction `hook_msg` dans le registre `rax`. Les registres sont des emplacements de mémoire très rapides intégrés dans le processeur pour stocker temporairement des valeurs. Le registre `rax` est généralement utilisé comme registre de retour (dans le contexte des appels de fonctions) ou pour stocker des valeurs temporaires. Dans cet exemple, l'adresse (ou pointeur) de la fonction `hook_msg` est chargée dans le registre `rax`.

**Ligne 2**: `jmp rax`

>Cette instruction est un "**saut inconditionnel**" (**jump** en anglais) vers l'adresse stockée dans le registre `rax`. En d'autres termes, elle dirige l'exécution du programme vers l'**adresse** de la fonction `hook_msg`.

___ 
En résumé, ces deux instructions font ce qui suit :

1. L'adresse de la fonction `hook_msg` est chargée dans le registre `rax`.
2. Le programme effectue un saut inconditionnel vers l'adresse contenue dans `rax`, ce qui a pour effet d'exécuter la fonction `hook_msg`.

Donc dans l'ordre il faut :
- Modifier la protection de la zone mémoire que l'on souhaite éditer (*PAGE_EXECUTE_READWRITE*)
- Copier les 12 premiers octets (backup)
- Editer les octets comme ça :
	- (où `pMsgbox` est le pointeur de nos 12 premiers octets)
		- pMsgbox[0] -> `0x48` -> `mov`
		- pMsgbox[1] -> `0xB8` -> `rax`
		- pMsgbox[2] -> `@hook_msgbox` <- adresse vers la fonction `msg_hook()`
			- un **DWORD64** est codé sur *64 bit* -> **8 octets**.
			- un **DWORD** est codé sur *32 bit* -> **4 octets**.
		- pMsgbox[10] -> `0xFF` -> `jmp`
		- pMsgbox[11] -> `0xE0` -> `rax`
- Re-mettre la protection de la zone mémoire que l'on a édité (*dwOldProtect*)

Je précise tout de même avant d'aller plus loin. 

En environnement Win32 il existe plusieurs type de variables que l'on a l'habitude d'utiliser. Effectivement on utilise ces types pour spécifier la taille des données que que l'on manipule.

```c
void set_hook()
{
    DWORD dwOldProtect;
    // Modification des autorisations mémoire pour permettre l'écriture
    VirtualProtect(pMsgbox, 12, PAGE_EXECUTE_READWRITE, &dwOldProtect);

    // Sauvegarde des premiers octets de la fonction cible
    *(DWORD64*)(oldBytes) = *(DWORD64*)(pMsgbox);
    *(DWORD*)(oldBytes + 8) = *(DWORD*)(pMsgbox + 8);

    // Injection de code de hooking
    *(BYTE*)(pMsgbox) = 0x48;
    *(BYTE*)(pMsgbox + 1) = 0xB8;
    *(DWORD64*)(pMsgbox + 2) = hook_msgbox;
    *(BYTE*)(pMsgbox + 10) = 0xFF;
    *(BYTE*)(pMsgbox + 11) = 0xE0;

    // Restauration des autorisations mémoire d'origine
    VirtualProtect(pMsgbox, 12, dwOldProtect, &dwOldProtect);
}
```


Explications détaillées des étapes :

1. **Modification des Autorisations Mémoire** : La fonction `VirtualProtect` modifie les autorisations mémoire de la fonction cible pour les définir en tant que lecture/écriture/exécution (`PAGE_EXECUTE_READWRITE`). Cela permettra de modifier le code de la fonction cible.
    
2. **Sauvegarde des Octets Originaux** : Les premières instructions de la fonction cible (`MessageBoxA`) sont copiées dans la variable `oldBytes` pour être restaurées plus tard.
    
3. **Injection de Code de Hook** : Les instructions d'origine de la fonction cible sont modifiées pour les remplacer par les opcodes nécessaires pour effectuer un saut vers `hook_msgbox`.
    
    - `0x48` : Opcode pour l'instruction `mov rax`.
    - `0xB8` : Opcode pour l'instruction `mov imm64` (chargement d'une adresse mémoire 64 bits dans `rax`).
    - `hook_msgbox` : Adresse de la fonction `hook_msgbox`.
    - `0xFF` : Opcode pour l'instruction `jmp rax`.
4. **Restauration des Autorisations Mémoire** : Une fois que le code de hooking est injecté, les autorisations mémoire d'origine sont restaurées à l'aide de `VirtualProtect`.

>Attention, ce type de hooking est considéré comme étant **bourin**. Il peut être tout à fait inefficace dans certains cas. Ainsi il faudra s'en remettre à utiliser une autre technique similaire.
{: .prompt-danger }

## Désinstaller le hook ❌

Plus simple, il s'agit d'utiliser la sauvegarde d'octets que l'on a effectué lors de l'installation du hook.

Ainsi :
- Modifier la protection de la zone mémoire que l'on souhaite éditer (*PAGE_EXECUTE_READWRITE*)
- Re-copier les 12 premiers octets (backup)
- Re-mettre la protection de la zone mémoire que l'on a édité (*dwOldProtect*)

```c
void delete_hook()
{
    DWORD dwOldProtect;
    VirtualProtect(pMsgbox, 12, PAGE_EXECUTE_READWRITE, &dwOldProtect);

    *(DWORD64*)(pMsgbox) = *(DWORD64*)(oldBytes);
    *(DWORD*)(pMsgbox + 8) = *(DWORD*)(oldBytes + 8);

    VirtualProtect(pMsgbox, 12, dwOldProtect, &dwOldProtect);
}
```
___

# Conclusion

C'était une rapide introduction au hooking inline en C/ASM.

Merci d'avoir suivis cet explication !

チュー <3