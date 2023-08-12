---
title: Life cycle of a pointer (in French)
categories: [programming]
tags: [lowlevel]
---

Voici quelques exemples de code en C pour illustrer le cycle de vie des variables pointeur :

**Exemple 1 : Allocation automatique sur la pile**

Dans cet exemple, la variable `ptr` est déclarée automatiquement dans la fonction `foo`. Sa durée de vie se limite à la portée de cette fonction, et la mémoire allouée pour cette variable est automatiquement libérée lorsque la fonction se termine. Si `ptr` est retourné depuis `foo`, il ne sera plus valide une fois que `foo` sera terminé.

```c
int* foo() {
    int x = 10;
    int* ptr = &x;
    return ptr;
}

int main() {
    int* p = foo(); // p pointe vers une variable locale qui n'existe plus
    // ...
    return 0;
}
```


**Exemple 2 : Allocation dynamique sur le tas**

Dans cet exemple, la variable `ptr` est allouée dynamiquement sur le tas en utilisant la fonction `malloc`. Sa durée de vie se prolonge jusqu'à ce que la mémoire allouée soit explicitement libérée en appelant la fonction `free`. Si `ptr` est retourné depuis `foo`, la mémoire allouée doit être libérée dans la fonction appelante.


```c
int* foo() {
    int* ptr = (int*) malloc(sizeof(int));
    *ptr = 10;
    return ptr;
}

int main() {
    int* p = foo(); // p pointe vers une variable allouée sur le tas
    // ...
    free(p); // libère la mémoire allouée pour la variable
    return 0;
}
```


Il est important de noter que dans cet exemple, la fonction `foo` renvoie un pointeur vers une variable allouée sur le tas. La responsabilité de libérer cette mémoire est donc transférée à la fonction appelante. Si la fonction appelante ne libère pas la mémoire, cela peut entraîner des fuites de mémoire.

**Exemple 3 : Utilisation d'un pointeur sur une variable locale**

Dans cet exemple, un pointeur `ptr` est initialisé pour pointer vers une variable locale `x` dans la fonction `foo`. Cependant, après la fin de la fonction `foo`, la variable `x` n'existe plus, donc `ptr` ne pointe plus vers une adresse mémoire valide.


```c
int* foo() {
    int x = 10;
    int* ptr = &x; // ptr pointe vers une variable locale
    return ptr;
}

int main() {
    int* p = foo();
    // utilise p, qui pointe vers une variable locale qui n'existe plus
    // cela peut provoquer un comportement indéfini
    return 0;
}
```

Dans cet exemple, `ptr` pointe vers une variable locale `x` qui est allouée sur la pile. Lorsque `foo` se termine, la mémoire allouée pour `x` est automatiquement libérée, donc `ptr` ne pointe plus vers une adresse mémoire valide. L'utilisation de `p` dans `main` peut provoquer un comportement indéfini, tel qu'un segmentation fault, car elle accède à une zone mémoire qui n'est plus valide.

___ 
>et maintenant dans la même optique comment se servir d'un pointer passé en paramètre d'une fonction ?

___

Lorsque vous passez un pointeur en paramètre d'une fonction en C, vous pouvez modifier la valeur pointée par le pointeur dans la fonction appelée. La façon dont vous modifiez la valeur dépend du type de pointeur que vous passez, soit un pointeur vers une valeur simple, soit un pointeur vers une structure ou un tableau.

Voici quelques exemples pour illustrer comment se servir d'un pointeur passé en paramètre d'une fonction :

**Exemple 1 : Pointeur vers une valeur simple**

Dans cet exemple, nous passons un pointeur `ptr` en paramètre d'une fonction `foo` qui prend un pointeur vers un entier. Nous pouvons modifier la valeur pointée par `ptr` en utilisant l'opérateur `*` pour déréférencer le pointeur.


```c
void foo(int* ptr) {
    *ptr = 10; // modifie la valeur pointée par le pointeur
}

int main() {
    int x = 0;
    int* ptr = &x;
    foo(ptr);
    printf("%d\n", x); // affiche "10"
    return 0;
}
```

Dans cet exemple, nous passons un pointeur `ptr` vers un entier à la fonction `foo`. Nous modifions la valeur pointée par `ptr` en utilisant l'opérateur `*` pour déréférencer le pointeur. Lorsque la fonction `foo` se termine, la valeur de `x` a été modifiée.

**Exemple 2 : Pointeur vers un tableau**

Dans cet exemple, nous passons un pointeur `ptr` vers un tableau d'entiers à la fonction `foo`. Nous pouvons accéder aux éléments du tableau en utilisant l'arithmétique des pointeurs.


```c
void foo(int* ptr, int size) {
    for (int i = 0; i < size; i++) {
        *(ptr + i) = i; // modifie chaque élément du tableau
    }
}

int main() {
    int arr[5] = {0};
    int* ptr = arr;
    foo(ptr, 5);
    for (int i = 0; i < 5; i++) {
        printf("%d ", arr[i]); // affiche "0 1 2 3 4"
    }
    printf("\n");
    return 0;
}
```

Dans cet exemple, nous passons un pointeur `ptr` vers un tableau d'entiers à la fonction `foo`. Nous utilisons l'arithmétique des pointeurs pour accéder à chaque élément du tableau et modifier sa valeur. Lorsque la fonction `foo` se termine, les valeurs du tableau ont été modifiées.

**Exemple 3 : Pointeur vers une structure**

Dans cet exemple, nous passons un pointeur `ptr` vers une structure `personne` à la fonction `foo`. Nous pouvons accéder aux champs de la structure en utilisant l'opérateur `->` pour déréférencer le pointeur.

```c
struct personne {
    char nom[20];
    int age;
};

void foo(struct personne* ptr) {
    strcpy(ptr->nom, "Jean"); // modifie le champ nom de la structure
    ptr->age = 25; // modifie le champ age de la structure
}

int main() {
    struct personne p;
    struct personne* ptr = &p;
    foo(ptr);
    printf("%s %d\n", p.nom, p.age); // affiche "Jean 25"
    return 0;
}
```


Dans cet exemple, nous définissons une structure `personne` qui a deux champs : `nom` et `age`. Nous passons un pointeur `ptr` vers une structure `personne` à la fonction `foo`. Nous modifions les champs de la structure en utilisant l'opérateur `->` pour déréférencer le pointeur. Lorsque la fonction `foo` se termine, les champs de la structure ont été modifiés.

Dans `main`, nous affichons les champs de la structure `personne` en utilisant l'opérateur `.` pour accéder aux champs. Nous utilisons la structure `personne` `p` directement, car elle a été modifiée par la fonction `foo`.

Il est important de noter que lorsque vous passez un pointeur en paramètre d'une fonction, vous devez vous assurer que le pointeur pointe vers une zone mémoire valide. Si vous utilisez un pointeur qui ne pointe pas vers une zone mémoire valide, cela peut provoquer un comportement indéfini, tel qu'un segmentation fault ou une corruption de la mémoire.