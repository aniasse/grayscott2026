# Jour 2 — Optimiser un stencil (Gray-Scott)

*🏠 [Accueil](../README.md) · ← [Jour 1](day1.md)*

Le jour 2 applique les concepts du jour 1 à un vrai cas : rendre une simulation
**Gray-Scott** de plus en plus rapide, sans changer son résultat.

---

## 1. Le modèle de Gray-Scott

Deux substances chimiques **U** et **V** qui **diffusent** dans l'espace et
**réagissent** ensemble. De ce duo simple émergent des **motifs de Turing** :
taches, labyrinthes, coraux… Les équations :

```
∂u/∂t = Du·∇²u − u·v² + F·(1−u)
∂v/∂t = Dv·∇²v + u·v² − (F+k)·v
```

Le motif se construit au fil du temps :

<p align="center">
  <img src="img/gs_evolution_1.png" width="200" alt="début"/>
  <img src="img/gs_evolution_2.png" width="200" alt="milieu"/>
  <img src="img/gs_evolution_3.png" width="200" alt="fin"/>
  <br/><em>Évolution de la concentration de V (début → fin de simulation).</em>
</p>

---

## 2. Le stencil et le Laplacien

Le terme `∇²` (Laplacien) se calcule par un **stencil** : chaque point se met à jour
en fonction de ses **voisins**.

```
        u[i-1,j]
u[i,j-1]  u[i,j]  u[i,j+1]      ∇²u ≈ (voisins) − 4·u[i,j]
        u[i+1,j]
```

C'est l'opération centrale — et, on l'a vu, typiquement **memory-bound**.

---

## 3. Mesurer AVANT d'optimiser

Règle d'or : **on n'optimise pas à l'aveugle**. On commence par **chronométrer**
et trouver le point chaud (la boucle qui mange le temps). Optimiser le reste = perte
de temps (loi d'Amdahl).

```
écrire → MESURER → optimiser le point chaud → re-mesurer → …
```

---

## 4. Le data layout (disposition mémoire)

Comme le stencil est memory-bound, **la façon de ranger les données en mémoire**
compte énormément. On stocke la grille 2D dans **un tableau plat contigu** (et non
un tableau de structures) : les accès deviennent réguliers → le cache et le SIMD
sont efficaces.

**À retenir :** bon layout = données contiguës = mémoire heureuse.

---

## 5. La vectorisation automatique

Avec un layout propre et `-O3 -march=native`, le compilateur **vectorise** la boucle
de stencil tout seul (plusieurs points calculés par instruction). Gros gain, zéro
ligne de code en plus.

---

## 6. Le voyage d'optimisation (Ex 9 : `91 → 95`)

Le même Gray-Scott, réécrit étape par étape — c'est le cœur du jour 2 :

| Version | Idée | Concept |
|---|---|---|
| `91` très naïf | baseline lente | — |
| `92` naïf | un peu mieux | — |
| `93` layout efficace | données contiguës | **memory-bound** |
| `94` auto-vectorisé | le compilo vectorise | **SIMD** |
| `95` Laplacien optimisé | stencil affiné | |

➡️ On **mesure** le gain à chaque marche : c'est la démonstration vivante du jour 1.

---

## 7. Le blocking (cache tiling)

Sur une grande grille, on parcourt trop de mémoire pour le cache. Le **blocking**
découpe le travail en **petits blocs** qui tiennent dans le cache → on réutilise les
données tant qu'elles sont « chaudes ».

```
Grille entière           Découpée en blocs
┌───────────────┐        ┌────┬────┬────┐
│               │   →    │ B1 │ B2 │ B3 │   chaque bloc tient
│               │        ├────┼────┼────┤   dans le cache
└───────────────┘        │ B4 │ B5 │ …  │
```

**À retenir :** une optim **mémoire** (pas calcul) — précieuse quand on est memory-bound.

---

## 8. Le parallélisme avec TBB

**TBB** (Intel Threading Building Blocks) = paralléliser sur tous les cœurs, façon
C++ moderne (≈ OpenMP, autre style). On distribue les lignes/blocs de la grille sur
les cœurs. Combiné à la vectorisation : **SIMD dans chaque cœur × N cœurs**.

---

## 9. Sortie des données & images (Ex 7-8)

- **Ex 7 — DataOutput** : sauvegarder les résultats au format **HDF5** (le standard
  I/O en HPC), avec un test qui valide l'écriture/lecture.
- **Ex 8 — ImagePlotting** : relire le HDF5 et produire des **images PNG** (les
  motifs ci-dessus). C'est le **post-traitement / la visualisation**.

```
simulation → output.h5 (HDF5) → gray_scott_image → pics/*.png
```

---

## 10. MAQAO — analyser la performance

**MAQAO** (outil UVSQ) regarde le **binaire au niveau assembleur** et te dit :
où passe le temps (hotspots), si la boucle est **vectorisée**, si tu es **memory-
ou compute-bound**, et **quoi corriger**. C'est le pont entre « ça compile » et
« ça va vite ».

> Comme un banc de diagnostic de garagiste : il ouvre le capot et pointe le goulot.

---

## 🧩 La pyramide d'optimisation (le résumé du jour 2)

```
1. bon DATA LAYOUT      → la mémoire suit
2. VECTORISATION        → chaque cœur calcule par paquets (SIMD)
3. BLOCKING             → on garde les données dans le cache
4. PARALLÉLISME (TBB)   → tous les cœurs en même temps
5. (GPU / multi-nœuds)  → au-delà d'une machine
   ↳ et on MESURE à chaque étape (MAQAO)
```

➡️ Pour refaire tourner tout ça chez toi : [guide d'installation](../INSTALL.md).
