# Jour 1 — Fondations de la performance

*[Accueil](../README.md) · suite → [Jour 2](day2.md)*

Le premier jour établit le vocabulaire et les principes. L'optimisation suppose de
comprendre le fonctionnement du matériel et la localisation des pertes de temps.

---

## 1. Le CPU

Un processeur est composé de **plusieurs cœurs** indépendants. La fréquence
d'horloge stagne depuis ~2005 (limite thermique) ; la montée en performance passe
désormais par l'**augmentation du nombre de cœurs**, ce qui rend le parallélisme
incontournable.

```
CPU
 └─ cœur ── registres · ALU (calcul) · unités vectorielles (SIMD) · cache
 └─ cœur ── …
 └─ … (ex. 20 cœurs par socket sur TAOUEY)
```

**Principe :** la performance provient du nombre de cœurs, non de la fréquence brute.

---

## 2. La compilation

La compilation traduit le code source C++ en code machine **avant l'exécution**, avec
optimisation. Elle comprend quatre étapes :

```
source .cpp ─① préproc. ─② compilation ─③ assemblage ─④ édition de liens ─→ exécutable
              (#include)   (→ assembleur,   (→ code        (+ bibliothèques)
                            OPTIMISATION)    machine .o)
```

L'optimisation et la vectorisation sont produites à l'étape ②. Options recommandées :
**`-O3 -march=native`** (optimisation pour le processeur cible). Les langages compilés
(C++) sont rapides ; les langages interprétés (Python) privilégient la flexibilité.

**Principe :** sans optimisation (`-O0`), le code peut être un ordre de grandeur plus lent.

---

## 3. La vectorisation (SIMD)

*Single Instruction, Multiple Data* : une instruction traite plusieurs valeurs
simultanément.

```
Scalaire :  a0=b0+c0 ; a1=b1+c1 ; … (8 instructions)
Vectorisé : [a0..a7] = [b0..b7] + [c0..c7]   (1 instruction)
```

La vectorisation est générée automatiquement par le compilateur lorsque la boucle est
régulière (accès contigus, absence de dépendances). Gain typique : ×4 à ×16 selon la
largeur des registres (SSE 128 bits, AVX2 256 bits, AVX-512 512 bits).

---

## 4. Concurrence et parallélisme

Deux notions distinctes :

> **Concurrence** : structurer un programme pour *gérer* plusieurs tâches simultanées.
> **Parallélisme** : *exécuter* plusieurs calculs en même temps.

| | Concurrence | Parallélisme |
|---|---|---|
| Objectif | masquer l'**attente** (E/S, réseau) | accélérer le **calcul** |
| Matériel | possible sur **un seul cœur** | exige **plusieurs cœurs** |
| Domaine | serveurs, E/S asynchrones | calcul intensif |

La concurrence est un outil de structuration ; le parallélisme en est le résultat
lorsque le matériel le permet.

---

## 5. Memory-bound et compute-bound

Caractérisation du facteur limitant d'un calcul :

- **Compute-bound** : limité par la vitesse des unités de calcul → l'ajout de cœurs
  améliore la performance.
- **Memory-bound** : limité par la bande passante mémoire ; les cœurs attendent les
  données → l'ajout de cœurs n'apporte qu'un gain marginal.

L'**intensité arithmétique** (opérations par octet chargé) détermine le régime. Les
**stencils** présentent une faible intensité arithmétique et sont généralement
memory-bound. Outil d'analyse de référence : le *roofline model*.

---

## 6. La pureté (purity)

Une fonction est **pure** lorsque sa sortie dépend uniquement de ses entrées, sans
effet de bord (aucune modification d'état partagé).

> Un calcul pur ne comporte pas de dépendance cachée : il est **parallélisable sans
> risque de** ***data race***.

Application : un schéma à **double tampon** (lecture d'un tableau, écriture d'un autre)
est pur et trivialement parallélisable. Un schéma **en place** (lecture et écriture du
même tableau) introduit des dépendances et n'est pas parallélisable sans précaution.

---

## 7. Tests unitaires en calcul scientifique

Les arrondis flottants interdisent la comparaison exacte. Trois règles :

1. **Comparaison avec tolérance** (`|a − b| < ε`), jamais d'égalité stricte sur des
   flottants.
2. **Vérification d'invariants** lorsque le résultat exact est inconnu : conservation
   d'énergie, symétrie, absence de `NaN`, solution analytique d'un cas simple.
3. **Équivalence séquentiel / parallèle** : une divergence révèle une *data race*.

---

## Synthèse

```
CPU (cœurs multiples, unités SIMD)
  │  la PURETÉ autorise la parallélisation sans bug
  ├─ VECTORISATION : au sein d'un cœur              (×8–16)
  ├─ PARALLÉLISME  : entre les cœurs                (×N cœurs)
  └─ régime MEMORY-BOUND → gain plafonné par la bande passante

La COMPILATION produit l'optimisation ; les TESTS en garantissent la justesse.
```

Suite : [Jour 2 — optimisation d'un stencil](day2.md).
