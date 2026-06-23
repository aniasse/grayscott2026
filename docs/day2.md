# Jour 2 — Optimisation d'un stencil (Gray-Scott)

*[Accueil](../README.md) · ← [Jour 1](day1.md) · inventaire exhaustif → [Référence des modules](day2-reference.md)*

Le second jour applique les principes du [Jour 1](day1.md) à un cas concret :
l'accélération progressive d'une simulation **Gray-Scott**, à résultat numérique
constant. Le matériel décline une même simulation en variantes, chacune isolant **une
seule** technique d'optimisation afin de la mesurer indépendamment.

Chaque section ci-dessous présente le **principe** (ce qui est appris) puis
l'**utilisation** (comment le mettre en œuvre). Les commandes supposent
l'environnement installé et compilé (voir [installation](../INSTALL.md)) ; elles
s'exécutent depuis le répertoire `build/`.

---

## 1. Le modèle de Gray-Scott

**Principe.** Deux substances **U** et **V** diffusent et réagissent chimiquement. De
ce système émergent des **motifs de Turing**. Équations gouvernantes :

```
∂u/∂t = Du·∇²u − u·v² + F·(1−u)
∂v/∂t = Dv·∇²v + u·v² − (F+k)·v
```

<p align="center">
  <img src="img/gs_evolution_1.png" width="200" alt="initial"/>
  <img src="img/gs_evolution_2.png" width="200" alt="intermédiaire"/>
  <img src="img/gs_evolution_3.png" width="200" alt="final"/>
  <br/><em>Figure — Concentration de V aux stades initial, intermédiaire et final.</em>
</p>

**Utilisation.** Les paramètres physiques et numériques sont passés en ligne de commande
(module `4-Parameters`) :

| Option | Signification |
|---|---|
| `-n` | nombre d'images produites |
| `-e` | nombre de pas de temps entre deux images |
| `-r` / `-c` | nombre de lignes / colonnes de la grille |
| `-o` | fichier de sortie (HDF5) |
| `-k` `-f` `-u` `-v` `-t` | taux de disparition, d'alimentation, diffusions, pas de temps |
| `-y` `-x` | hauteur / largeur de bloc (blocking) |
| `-p` `-m` | fichier de performances (.toml) / nombre de mesures |

```bash
./9-Simulation/autovec_gray_scott_O3 -n 10 -e 200 -r 512 -c 512 -o output.h5
```

---

## 2. Le stencil et le Laplacien

**Principe.** Le terme `∇²` (Laplacien) est évalué par un **stencil 3×3** : chaque point
est mis à jour en fonction de ses voisins.

```
        u[i-1,j]
u[i,j-1]  u[i,j]  u[i,j+1]      ∇²u ≈ (somme pondérée des voisins) − 4·u[i,j]
        u[i+1,j]
```

Ce noyau constitue le cœur du calcul ; sa faible **intensité arithmétique** le rend
typiquement *memory-bound*.

---

## 3. Mesurer avant d'optimiser

**Principe.** L'optimisation s'appuie sur la mesure, non sur l'intuition. Le cours
fournit une infrastructure dédiée (`1-FirstPerformanceTest`, `2-BenchmarkFunction`,
`3-FunctionTimer`) reposant sur deux exigences : la **répétition** des mesures et
l'**épinglage des threads sur les cœurs** (`pin_thread_to_core`), qui éliminent le bruit
et les migrations de threads.

**Utilisation.** Chronométrage direct, ou enregistrement des performances dans un fichier :

```bash
time ./9-Simulation/autovec_gray_scott_O3 -n 10 -e 30 -r 1080 -c 1920
./9-Simulation/autovec_gray_scott_O3 -n 10 -e 30 -r 1080 -c 1920 -p perf.toml
```

---

## 4. Disposition mémoire (AoS → SoA)

**Principe.** À résultat identique, l'organisation des données en mémoire détermine la
performance (`5-DataLayout`, intégré en `9-Simulation/91→93`) :

| Variante | Structure | Effet |
|---|---|---|
| très naïve | `vector<vector<Cell>>` (AoS imbriqué) | lignes non contiguës, accès dispersés |
| naïve | `vector<Cell>` plat (AoS contigu) | contiguë, mais champs entrelacés |
| efficace | **SoA** (`matU`, `matV`… séparés) + `__restrict__` + double tampon | accès réguliers, vectorisable |

Leviers : passage **AoS → SoA** (Structure of Arrays), mot-clé **`__restrict__`** (absence
d'alias), et parcours respectant la contiguïté mémoire.

**Utilisation.** Comparaison directe des trois dispositions :

```bash
for t in very_naive_gray_scott_O3 naive_struct_gray_scott_O3 naive_gray_scott_O3; do
  echo "== $t ==" ; time ./9-Simulation/$t -n 5 -e 100 -r 512 -c 512 -o /dev/null
done
```

---

## 5. Vectorisation (SIMD)

**Principe.** Sur une disposition efficace, la vectorisation automatique traite plusieurs
points par instruction (`6-Vectorization`, intégré en `9-Simulation/94→95`). La
transformation déterminante est le calcul du **Laplacien en passe séparée** sur
l'intérieur du domaine, qui supprime les branches de bord et rend la boucle vectorisable.
L'**allocation alignée** (`AlignedAllocator`) optimise les chargements vectoriels.

La vectorisation est activée par les **drapeaux de compilation**, à code constant :

| Cible | Drapeaux |
|---|---|
| `naive_gray_scott_O3` | `-O3` |
| `autovec_gray_scott_O3` | `-O3 -march=native -mtune=native -ftree-vectorize -funroll-loops` |
| `laplacian_gray_scott_O3_vectorize` | idem |

**Utilisation.** Mesure du gain apporté par la vectorisation :

```bash
for t in naive_gray_scott_O3 autovec_gray_scott_O3 laplacian_gray_scott_O3_vectorize; do
  echo "== $t ==" ; time ./9-Simulation/$t -n 5 -e 100 -r 512 -c 512 -o /dev/null
done
```

---

## 6. Blocking (pavage du cache)

**Principe.** Sur une grande grille, le volume parcouru dépasse la capacité du cache. Le
**blocking** (`14-Blocking`) découpe le calcul en blocs dimensionnés pour le cache,
maximisant la réutilisation des données chargées (optimisation mémoire, non calcul).

**Utilisation.** La taille de bloc est paramétrable (`-y`, `-x`) ; un script recherche la
valeur optimale :

```bash
./14-Blocking/142-AutoVectorization/… -n 5 -e 100 -r 1024 -c 1024 -y 64 -x 512
bash 14-Blocking/scriptFindBestBlock.sh        # balayage des tailles de bloc
```

---

## 7. Pavage spatio-temporel (pyramides)

**Principe.** Une **pyramide** (`15-AdvancedBlocking`) calcule plusieurs pas de temps au
sein d'un même bloc de cache. Le stencil réduisant le domaine valide à chaque pas, la
zone calculable rétrécit en forme de pyramide — d'où un blocage **temporel** qui amortit
davantage les accès mémoire.

**Utilisation.** Le nombre d'itérations par pyramide et la bordure sont paramétrables
(`-z`, `-b`) :

```bash
bash 15-AdvancedBlocking/scriptFindBestPyramid.sh
```

---

## 8. Parallélisme multi-cœur (TBB)

**Principe.** Intel **Threading Building Blocks** répartit le calcul sur l'ensemble des
cœurs (`TBB/`). Trois stratégies sont comparées : par lignes (`GrayScottLoopTBB`), par
blocs (`GrayScottBlockTBB`) et par pyramides (`GrayScottPyramidTBB`). Combinée à la
vectorisation, la performance s'écrit : SIMD par cœur × nombre de cœurs.

**Utilisation.** Le nombre de threads se contrôle par variable d'environnement :

```bash
export TBB_NUM_THREADS=8      # ou OMP_NUM_THREADS selon la configuration
time ./TBB/GrayScottLoopTBB/… -n 5 -e 100 -r 1024 -c 1024 -o /dev/null
```

---

## 9. GPU

**Principe.** Cinq familles, du plus bas au plus haut niveau (`GPU/`) :

| Famille | Approche |
|---|---|
| **CUDA** | noyaux explicites ; optimisations par **streams** (recouvrement), **CUDA Graphs** (réduction du coût de lancement en itératif), exécution asynchrone, pyramides GPU |
| **nvc++** | offload GPU du C++ standard (`std::execution::par`) |
| **Thrust** | algorithmes GPU haut niveau |
| **CUPP** | encapsulation C++ de CUDA |

Le point clé est la réduction des **allers-retours CPU↔GPU** : en itératif, les CUDA
Graphs capturent une séquence de noyaux et la rejouent en une seule soumission.

**Utilisation.** Nécessite un GPU NVIDIA et la chaîne CUDA (sur le cluster :
`module load cuda`). Vérification préalable des capacités matérielles :

```bash
./GPU/TestCudaCapabilities/…           # détection du GPU
time ./GPU/CUDA/GrayScottCudaLaplacianGraph/… -n 10 -e 200 -r 1080 -c 1920 -o output.h5
```

---

## 10. Entrées/sorties et visualisation

**Principe.** Les résultats sont écrits au format **HDF5** (`7-DataOutput`, standard d'E/S
en HPC), avec un test de validation. La performance des E/S est elle-même étudiée
(`PerformanceHDF5`), y compris depuis la mémoire GPU (`DataOutputCpuGpu`). La
visualisation convertit le HDF5 en **images PNG** (`8-ImagePlotting`).

**Utilisation.** Chaîne complète simulation → données → images :

```bash
# Test du format de données (Ex 7)
ctest -R TestGrayScottDataFormat --output-on-failure

# Simulation -> HDF5
./9-Simulation/autovec_gray_scott_O3 -n 10 -e 200 -r 512 -c 512 -o output.h5

# HDF5 -> PNG (Ex 8)
mkdir -p pics
./8-ImagePlotting/gray_scott_image -i output.h5 -o pics/
```

---

## 11. Calcul distribué (décomposition de domaine)

**Principe.** `29-DistributedComputing` introduit la **MetaImage** : une grande image est
découpée en **tuiles** traitées indépendamment, les pyramides gérant le recouvrement
temporel entre tuiles. L'architecture sépare **données**, **traitement**
(`ProcessingManager` CPU ou CUDA) et **mémoire** (`RessourceMemoryManager`), ouvrant la
voie au multi-GPU et au multi-nœud.

**Utilisation.**

```bash
bash 29-DistributedComputing/scriptGrayScottMetaImage.sh
```

---

## 12. Analyse de performance (MAQAO)

**Principe.** **MAQAO** analyse le binaire au niveau assembleur : points chauds, taux de
vectorisation, régime (memory-bound / compute-bound), recommandations. Il établit le lien
entre correction fonctionnelle et performance.

**Utilisation.** Compilation avec informations de débogage (`-g`), puis analyse :

```bash
maqao oneview -R1 -- ./9-Simulation/autovec_gray_scott_O3 -n 5 -e 100 -r 512 -c 512 -o /dev/null
# rapport HTML généré dans RESULTS/
```

---

## 13. Industrialisation

**Principe.** `27-Deliverable` fournit des recettes de conteneurisation **Apptainer** et
**Docker** (environnements de compilation, GPU, production) pour un déploiement
reproductible de la solution optimisée.

---

## Synthèse — l'arc d'optimisation

```
MESURE (timer, benchmark, pin-to-core)
   │
   ├─ DISPOSITION MÉMOIRE   AoS → SoA + __restrict__          (5, 9)
   ├─ VECTORISATION         Laplacien sans branche + drapeaux  (6, 9)
   ├─ BLOCKING              spatial (14) puis temporel (15)
   ├─ MULTI-CŒUR            TBB : loop / block / pyramid        (TBB)
   ├─ GPU                   CUDA (streams, graphs), nvc++, Thrust
   └─ DISTRIBUÉ             décomposition en tuiles             (29)
        ↳ E/S HDF5 + visualisation PNG (7, 8) · analyse MAQAO · industrialisation (27)
```

Chaque étape est isolée, mesurable, et conserve un résultat numérique identique —
application directe des principes du [Jour 1](day1.md). Inventaire module par module :
[Référence détaillée](day2-reference.md).
