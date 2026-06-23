# Jour 1 — Les fondations de la performance

*🏠 [Accueil](../README.md) · suite → [Jour 2](day2.md)*

Le premier jour pose le vocabulaire. Rien de magique : pour aller vite, il faut
d'abord comprendre **comment une machine calcule** et **où le temps se perd**.

---

## 1. Le CPU

Un processeur, ce n'est pas « un » cerveau, mais **plusieurs cœurs** indépendants.
On a arrêté d'augmenter la fréquence vers 2005 (trop de chaleur), alors on a mis
**plus de cœurs**. C'est toute la raison d'être du parallélisme.

```
CPU
 └─ cœur ── registres · ALU (calcul) · unités vectorielles (SIMD) · cache
 └─ cœur ── …
 └─ … (ex. 20 cœurs par socket sur TAOUEY)
```

**À retenir :** la puissance vient du **nombre de cœurs**, pas de la vitesse brute.

---

## 2. La compilation

**Compiler = traduire ton C++ en code machine, une fois, avant l'exécution** — et
l'optimiser au passage. Quatre étapes :

```
ton .cpp ─① préproc ─② compilation ─③ assemblage ─④ liens ─→ exécutable
            (#include)   (→ assembleur,    (→ code      (+ bibliothèques)
                          OPTIMISATION ici)  machine .o)
```

C'est à l'étape ② que naît la vectorisation. Le bon réflexe : **`-O3 -march=native`**
(optimiser pour *ce* processeur). Compilé (C++) = rapide ; interprété (Python) = souple
mais lent.

**À retenir :** sans optimisation (`-O0`), ton code peut être **10× plus lent**.

---

## 3. La vectorisation (SIMD)

*Une instruction qui traite plusieurs nombres d'un coup.* Au lieu d'additionner 1
valeur, le cœur en additionne 8 (ou 16) à la fois.

```
Sans SIMD :  a0=b0+c0 ; a1=b1+c1 ; … (8 instructions)
Avec SIMD :  [a0..a7] = [b0..b7] + [c0..c7]   (1 instruction)
```

C'est **gratuit** : le compilateur le fait si la boucle est « propre ». Gain : ×4 à ×16.

---

## 4. Concurrence vs Parallélisme

La confusion classique. La formule à retenir :

> **Concurrence = *gérer* plusieurs tâches à la fois. Parallélisme = les *faire* en même temps.**

| | Concurrence | Parallélisme |
|---|---|---|
| But | gérer l'**attente** (I/O, réseau) | aller plus vite en **calcul** |
| Matériel | marche sur **1 cœur** | exige **plusieurs cœurs** |
| Exemple | 1 serveur web | un gros calcul découpé |

> Image : la concurrence, c'est un barista qui jongle entre 3 commandes ;
> le parallélisme, c'est 3 baristas qui bossent en même temps.

---

## 5. Memory-bound vs Compute-bound

**La** question de perf : ton code attend-il **le calcul** ou **la mémoire** ?

- **Compute-bound** : limité par la vitesse de calcul → ajouter des cœurs aide.
- **Memory-bound** : les cœurs **attendent les données** → ajouter des cœurs n'aide
  presque plus (ils se battent pour la bande passante).

Beaucoup de codes scientifiques (dont les **stencils**) sont **memory-bound**.
C'est pour ça qu'on ne gagne pas toujours ×N en mettant N cœurs.

**Outil de référence :** le *roofline model*.

---

## 6. La pureté (purity)

Une fonction est **pure** si sa sortie ne dépend **que de ses entrées**, sans effet
de bord (pas d'état partagé modifié). Pourquoi c'est crucial en HPC ?

> Un calcul **pur n'a pas de dépendances cachées** → il est **parallélisable sans
> risque** (pas de *data race*).

Exemple concret : un calcul qui **lit un tableau et en écrit un autre** (double
tampon) est pur → parallèle facilement. Le même calcul **en place** (lire/écrire le
même tableau) crée des dépendances → dangereux en parallèle.

---

## 7. Les tests unitaires (version HPC)

En calcul scientifique, on ne teste pas une égalité exacte — les **arrondis
flottants** l'interdisent. Trois réflexes :

1. **Jamais `==` sur des flottants** → comparer avec une **tolérance** (`|a-b| < ε`).
2. **Tester des invariants** quand on ne connaît pas le résultat exact :
   conservation d'énergie, symétrie, pas de `NaN`, solution analytique d'un cas simple.
3. **Séquentiel == parallèle** → si ça diffère, tu as une **data race**.

---

## 🧩 Comment tout s'emboîte

```
CPU (des cœurs, chacun avec du SIMD)
  │  la PURETÉ rend le calcul parallélisable sans bug
  ├─ VECTORISATION : à l'intérieur d'un cœur          (×8-16)
  ├─ PARALLÉLISME  : entre les cœurs                   (×N cœurs)
  └─ mais si MEMORY-BOUND → le gain plafonne (bande passante)

La COMPILATION fabrique tout ça ; les TESTS garantissent que ça reste juste.
```

➡️ Suite : [Jour 2 — optimiser un vrai stencil](day2.md).
