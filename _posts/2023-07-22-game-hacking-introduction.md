---
title: Game Hacking Introduction (in French)
categories: [programming, game]
tags: [lowlevel]
---

# Cas n°1 : Supraland

> Supraland est un jeu sous windows (x64) de casse-tête en vu FPS et en 3D utilisant le moteur de jeu Unreal 4.

![Coordinates](/assets/supra.jpg)
_Supraland used exemple game_

# Introduction

A la base j'étais en train de jouer à ce jeu. Et comme le jeu est prévu pour qu'on test un maximum le jeu, je cherchais à exploiter la physique du moteur sans utiliser de logiciel tier. 

Histoire de voir si le jeu été propice à des **glitchs**. 

Ayant fait le tour du jeu selon moi, m'est venu l'idée de pousser la recherche plus loin en cherchant l'axe Z du personnage de façon bas niveau, **out of the box**.

# Par où commencer ?

![ok](/assets/meme/jd25yqv8xsf31.jpg){: width="400" height="200" }

Pré-requis :

- [x] Cheat Engine
- [ ] ~~De la patience~~

# Qu'est-ce que c'est Cheat Engine ?

Cheat Engine est un logiciel utilisé pour modifier les valeurs en mémoire des jeux vidéo. Il permet aux joueurs de changer des attributs comme la santé ou l'argent. Il peut aussi être utilisé pour des expérimentations en cybersécurité.

ça va être l'outil parfait pour obtenir ce que l'on souhaite.

# Premier pas : La quête des coordonnées GPS

![Coordinates](/assets/xyz.png)
_XYZ Coordinates_

Dans un jeu 3D, XYZ représente les coordonnées spatiales d'un objet ou d'un personnage dans un espace tridimensionnel. La lettre X fait référence à la position horizontale, la lettre Y à la position verticale et la lettre Z à la position en profondeur. Ensemble, ces coordonnées permettent de déterminer l'emplacement précis de l'objet ou du personnage dans l'environnement 3D du jeu.

On peut tout à fait interchanger l'ordre de ces valeurs. 
C'est ce que je fais personnellement en mettant :

- **X (horizontale)**
- **Y (profondeur)**
- **Z (hauteur)**

Je préfère comme ça. Ce sont que des noms, si j'ai envie d'en appeler un *Jean-Marie* et l'autre *Gwendoline* c'est pareil.

### Pourquoi l'axe Z et pas un autre ?

Bonne question, il est plus facile de commencer avec cet axe plutôt qu'un autre. Car lors de la recherche il est plus facile de faire varier uniquement la hauteur du personnage qu'une unique translation sur l'axe X ou Y (plot twist c'est pas possible enfaite).

Comment faire varier l'axe Z ? Tout simplement en faisant des scan avec cheat engine lorsque notre personnage se trouve a côté d'une monté ou d'un escalier. 
Il suffira de monter & descendre entre chaque **scan** dans le but d'éliminer les valeurs qui ne suivent pas la logique de déplacement verticale que vous appliqué en jeu.

De ce fait, la coordonnée d'axe Z va incrémenter ou décrémenter.

## Information avant d'attaquer la forme

Dans l'univers du Hacking de jeu vidéo, il y a deux moyen d'arriver à lire la mémoire d'un programme ou jeu. 

Il existe ce qu'on appel 2 type de cheat en informatique :

### Cheat interne

Un cheat interne est un code qui peut être exécuté dans le même espace d'adressage que le programme que vous souhaitez hack.

>Par exemple, avec une lib `dll_injection.dll` injecté directement dans le processus `Supraland.exe` [...]

### Cheat externe
Un cheat externe est un programme externe qui lit les données à distance à partir du processus actif d'un autre programme (le jeu).

> Par exemple, `hack.exe` ouvre `Supraland.exe` en mémoire pour [...]

___

## Cheat Engine

Pour se faire, on va ouvrir **Cheat Engine** et ouvrir également le processus du jeu Supraland. *(Supraland-Win64-Shipping.exe)*

Ce tool écris en *delphi* est **open-source** pour information.

Il va nous permettre de faire du **filtrage** dynamique sur des **adresses mémoire** et leurs **valeurs**.

En outre, on peut faire une recherche par **valeur** pour y trouver enfin une **adresse mémoire** pour à terme y trouver une **adresse statique**.

#### Qu'est-ce qui dit lui ?

#### Valeur
> Votre nombre de munitions peut être représenté par une valeur numérique.

#### Adresse
> Une adresse représente une position spécifique dans la mémoire de l'ordinateur où une valeur particulière est stockée.

#### Pointeur
> Les pointeurs sont souvent utilisés lorsque les valeurs sont stockées dans des emplacements de mémoire dynamiques ou lorsqu'elles sont réparties sur plusieurs adresses.

#### Pointeur d'adresse statique
> Il y a des pointeurs qui ne changent pas entre les sessions de jeu. (**redémarrage**)
> C'est pour cette raison précise qu'il existe des cheats partagé (payant ou non) sur internet. 

Ok c'est cool... ça ne nous dit toujours pas comment y arriver.

## Exemple pratique

![meme](/assets/meme/q3brjna79un71.jpg){: w="400"}

> Nous allons RE (reverse) un processus avec `Cheat Engine`.

#### Ma procédure pour obtenir le **pointeur d'adresse statique de l'axe Z** dans un jeu :

> Obtenir un pointeur statique peut être une chose fastidieuse à faire.
{: .prompt-warning }

1. **Balayer toutes les adresses** en mémoire avec une valeur inconnue
2. **Modifier** et **filtrer la valeur** (en augmentant, en diminuant, en modifiant la valeur, etc...)
     1. pour augmenter la valeur -> sauter dans les airs _(ou grimper un peu)_
     2. pour diminuer la valeur -> attendez que le joueur recule là où il a sauté _(ou descende à partir de là)_
3. Une fois qu'une adresse est trouvée :
     1. Vérifiez si dans le jeu, nous voyons que nous pouvons geler la valeur de l'outil de mémoire
     2. Générer un pointermap avec cette adresse
4. Redémarrez le jeu et conservez les adresses et les valeurs dans l'outil de mémoire
5. Recommencez tout le processus jusqu'à l'étape 3.2
6. Générez une analyse de pointeur pour cette adresse `(la deuxième valeur que nous avons trouvée)`
7. Localisez la valeur actuelle du jeu au pointermap scan
8. Redémarrez le jeu et continuez à filtrer toutes ces choses jusqu'à ce que nous ayons ~100 ~50 pointeur et redémarrez le processus jusqu'à ce que nous ayons un pointeur d'adresse statique.

Pas évident hein ?

C'est normal on cherche a isoler des adresses mémoires spécifique parmis peut être 1 million voire 1 milliard d'adresse mémoire !

## Obtenir les autres axes

Une fois l'axes Z trouvé il suffit d'aller voir dans la mémoire en explorant la plage mémoire en temps réel. <br>
Comme c'est un jeu en 3D, on s'apperçoit qu'on trouve deux autres valeur en *float*, elle correspond à **90%** aux autres axes. 

Donc on peut ajouter ou soustraire `0x04` au dernier offset de **l'adresse statique Z**.

## Et la caméra ?

Effectivement il est possible de récupérer les adresses responsable de la *caméra dans le jeu*. <br>
Et ça sera pour un **prochain article** où nous parlerons d'une entité mathématique qui s'appel le `quaternion`. 

> Le quaternion est un moyen de représenter ou de décrire l’orientation d’un objet dans un espace en 3 dimensions.
{: .prompt-warning }

![Quaternion](/assets/quater.svg){: w="300"}
![Quaternion](/assets/quater.jpg)
![Quaternion](/assets/quater.gif)
_Animation d'un quaternion_

# Conclusion

Et voilà comment obtenir la position GPS du personnage. 

C'était une introduction technique à de la manipulation mémoire dans un jeu vidéo.

[Suite à venir]<br>...
{:style="text-align: center; font-size: 25px"}