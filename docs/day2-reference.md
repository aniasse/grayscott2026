# Jour 2 — Référence détaillée des modules

*[Accueil](../README.md) · synthèse → [Jour 2](day2.md) · fondations → [Jour 1](day1.md)*

Analyse complète du matériel `day-2` du cours *Performance With Stencil*. Le projet
décline une même simulation Gray-Scott en de nombreuses variantes, chacune isolant
une technique d'optimisation et mesurable indépendamment. Les modules sont organisés
en sept ensembles.

Principe transverse : le programme principal (`9-Simulation/main.cpp`) est **générique**
et paramétré à la compilation (`-DGRAY_SCOTT_LIB_INCLUDE`). Chaque variante de calcul
est compilée en **bibliothèque partagée** distincte, ce qui permet de comparer les
implémentations à code de pilotage identique.

---

## A. Infrastructure de mesure

L'optimisation commence par une mesure fiable.

| Module | Contenu | Notion |
|---|---|---|
| `1-FirstPerformanceTest` | `timer` minimal (`std::chrono`), tracés gnuplot (courbe, histogramme) | mesure du temps |
| `2-BenchmarkFunction` | `BenchmarkCall`, `benchmark_timer`, **`pin_thread_to_core`** | mesure répétée, stabilité |
| `3-FunctionTimer` | `FunctionTimer`, `clock_timer` | chronométrage par fonction |
| `4-Parameters` | `gray_scott_param` : analyse des options CLI (`-n -e -r -c -k -f -u -v -t …`) | paramétrage |

Points clés : la **répétition** des mesures et l'**épinglage des threads sur les cœurs**
(`pin_thread_to_core`) éliminent le bruit de mesure et les migrations de threads.

---

## B. Optimisation séquentielle (cœur du cours)

### 5-DataLayout — l'impact de la disposition mémoire

Quatre implémentations équivalentes en résultat, mais d'efficacité très différente :

| Variante | Structure de données | Effet |
|---|---|---|
| `51-VeryNaiveApproach` | `vector<vector<Cell>>` — **AoS** imbriqué (lignes non contiguës) | référence, accès mémoire dispersés |
| `52-NaiveApproach` | `vector<Cell>` plat — **AoS** contigu | mémoire contiguë, mais champs entrelacés |
| `53-EfficientApproach` | **SoA** : `matU`, `matV`, `matOutU`, `matOutV` séparés + `__restrict__` + double tampon | accès réguliers, vectorisable |
| `54-SwapAxis` | identique à SoA mais boucle **colonne d'abord** | contre-exemple : un mauvais ordre d'accès détruit la performance |

Enseignements : **AoS → SoA** (Structure of Arrays) est le levier décisif ; le mot-clé
**`__restrict__`** garantit l'absence d'alias et autorise l'optimiseur ; l'**ordre de
parcours** doit suivre la contiguïté mémoire (ligne d'abord).

### 6-Vectorization — exploiter le SIMD

Construites sur la disposition efficace, ces variantes lèvent les obstacles à la
vectorisation automatique :

| Variante | Technique |
|---|---|
| `62-AutoVectorization` | disposition SoA + drapeaux de vectorisation |
| `63-AutoVectorizationLaplacian` | **Laplacien calculé en passe séparée** sur l'intérieur `[1, n-1[` (suppression des branches de bord → boucle vectorisable) |
| `64-AutoVecLaplacianLight` | version allégée du Laplacien |
| `65-AutoVectorization3x3` | stencil 3×3 explicite |
| `AlignedAllocator/` | allocateur **alignant la mémoire** sur la largeur SIMD (chargements vectoriels efficaces) |

Enseignement : une boucle se vectorise si elle est **régulière, sans branche et sans
dépendance** ; séparer le calcul du Laplacien des conditions de bord est la
transformation déterminante.

### 9-Simulation — la progression mesurée (`91 → 95`)

Mêmes variantes, intégrées au pipeline complet, avec les **drapeaux de compilation**
qui matérialisent l'optimisation :

| Cible | Approche | Drapeaux |
|---|---|---|
| `very_naive_gray_scott_O3` | AoS imbriqué | `-O3` |
| `naive_struct_gray_scott_O3` | AoS plat | `-O3` |
| `naive_gray_scott_O3` | SoA efficace | `-O3` |
| `autovec_gray_scott_O3` | SoA vectorisée | `-O3 -march=native -mtune=native -ftree-vectorize -funroll-loops` |
| `laplacian_gray_scott_O3_vectorize` | Laplacien vectorisé | idem |

Enseignement : à code algorithmique constant, le passage de `-O3` seul à
`-march=native -ftree-vectorize` produit un gain majeur. `10-FullHDSimulation` applique
la chaîne à une résolution Full-HD (scripts de génération et d'évaluation).

---

## C. Optimisation mémoire avancée (cache)

### 14-Blocking — pavage du cache (tiling spatial)

| Composant | Rôle |
|---|---|
| `141-BlockingLib` | indices de bloc (`BlockIdx`, `AxisIdx`), `copy_block`, tests dont **vectorizability** |
| `142-AutoVectorization` | Gray-Scott par blocs, vectorisé |
| `143-AutoVecCopy` | variante par blocs avec recopie |
| `scriptFindBestBlock.sh` | recherche empirique de la **taille de bloc optimale** |

Principe : découper la grille en blocs tenant dans le cache pour maximiser la
réutilisation des données chargées (pertinent en régime memory-bound).

### 15-AdvancedBlocking — pavage spatio-temporel (pyramides)

| Composant | Rôle |
|---|---|
| `151-PyramidLib` | `PyramidIdx`, `AntiPyramidIdx`, `PyramidIterator` |
| `152-Pyramid2Hdf5` | export du résultat pyramidal en HDF5 |
| `153-SimplePyramid` | implémentation directe |
| `scriptFindBestPyramid.sh` | recherche des meilleurs paramètres de pyramide |

Principe : une **pyramide** calcule plusieurs pas de temps au sein d'un même bloc de
cache. Comme le stencil réduit le domaine valide à chaque pas, la zone calculable
rétrécit en forme de pyramide — d'où un blocage **temporel** qui amortit davantage les
accès mémoire.

---

## D. Parallélisme multi-cœur (TBB)

Intel **Threading Building Blocks** répartit le calcul sur les cœurs. Chaque variante
combine parallélisme et une stratégie mémoire :

| Module | Stratégie |
|---|---|
| `GrayScottLoopTBB` | parallélisation des lignes (`parallel_for`) |
| `GrayScottBlockTBB` | parallélisation par blocs |
| `GrayScottPyramidTBB` | parallélisation des pyramides (spatio-temporel) |
| `GrayScottTBB` | variante de base |
| `HadamardProductTBB` | noyau d'introduction (produit terme à terme) |

Le **produit de Hadamard** (multiplication élément par élément) sert de noyau minimal
pour présenter chaque technologie avant son application à Gray-Scott. Performance
résultante : SIMD par cœur × nombre de cœurs.

---

## E. GPU

Cinq familles, du plus bas au plus haut niveau d'abstraction :

### CUDA (`GPU/CUDA`)
| Module | Apport |
|---|---|
| `CudaDeviceDiscovery` | interrogation du GPU, `ThreadGrid`, `StreamManager` |
| `CudaAllocator` | allocation mémoire device |
| `GrayScottCuda` | noyau 3×3 de base |
| `GrayScottCudaLaplacian` | Laplacien sur GPU |
| `GrayScottCudaLaplacianStream` | **streams** : recouvrement calcul / transferts |
| `GrayScottCudaLaplacianGraph` | **CUDA Graphs** : capture d'une séquence de noyaux pour réduire les allers-retours CPU↔GPU dans une boucle itérative |
| `GrayScottCudaLaplacianGraphWhile` | graphes **conditionnels** (`while`/`if` dans le graphe) |
| `GrayScottCudaLaplacianAsync` | exécution asynchrone |
| `GrayScottCudaPyramidLaplacian` | blocage temporel (pyramide) sur GPU |
| `HadamardProductCuda*` | noyaux d'introduction (sync, async, device) |

Le point fort de cette famille est l'optimisation du **coût de lancement** : en
itératif, lancer les noyaux un à un sature le lien CPU-GPU ; les **CUDA Graphs**
capturent et rejouent la séquence (`cudaStreamBeginCapture` … `cudaGraphLaunch`).

### Autres familles
| Famille | Approche |
|---|---|
| `NVCPP` | `nvc++` — offload GPU du C++ standard (`std::execution::par`) : `GrayScottNvcpp`, `…Laplacian`, `…Loop` |
| `THRUST` | bibliothèque d'algorithmes GPU haut niveau (itérateurs zip, lambdas, allocateurs) |
| `CUPP` | encapsulation C++ de CUDA |
| `TestCudaCapabilities` | détection des capacités matérielles du GPU |

---

## F. Entrées/sorties et visualisation

| Module | Rôle |
|---|---|
| `7-DataOutput` | `MatrixHdf5` : écriture/lecture de la pile d'images au format **HDF5** + test unitaire |
| `PerformanceHDF5` | mesure des performances d'E/S HDF5 (dataset complet vs par blocs) |
| `DataOutputCpuGpu` | écriture HDF5 depuis la mémoire **device** (GPU), via un allocateur dédié |
| `8-ImagePlotting` | conversion HDF5 → **PNG** (`phoenix_png`) |

Chaîne complète : `simulation → output.h5 → gray_scott_image → pics/*.png`.

---

## G. Calcul distribué / multi-périphérique

`29-DistributedComputing` introduit la **décomposition de domaine** (« MetaImage ») :
une grande image est découpée en **tuiles** traitées indépendamment, les pyramides
gérant le recouvrement temporel entre tuiles.

| Module | Rôle |
|---|---|
| `291-Pyramid` | bibliothèque générique : `GenericMetaImage`, `GenericTile`, `GenericPyramid`, `ProcessingManagerCpu`, `RessourceMemoryManagerCpu` |
| `292-PyramidCuda` | gestionnaires de traitement/mémoire **CUDA** |
| `293-GrayScottMetaImage` | application CPU (vectorisée) |
| `294-GrayScottMetaImageCuda` | application GPU |

L'architecture sépare **données** (`DeviceData`, générique CPU/GPU), **traitement**
(`ProcessingManager`) et **mémoire** (`RessourceMemoryManager`), ouvrant la voie au
multi-GPU et au multi-nœud.

---

## H. Industrialisation (`27-Deliverable`)

Recettes de conteneurisation **Apptainer** et **Docker** (environnements `base_compile`,
`compile_gpu`, `prod`) et script `gray_scott_compile_install` : packaging reproductible
de la solution optimisée pour le déploiement en production.

---

## Synthèse — l'arc d'optimisation

```
MESURE (timer, benchmark, pin-to-core)
   │
   ├─ DISPOSITION MÉMOIRE   AoS → SoA + __restrict__        (5)
   ├─ VECTORISATION         Laplacien sans branche + flags   (6, 9)
   ├─ BLOCKING              spatial (14) puis temporel (15)
   ├─ MULTI-CŒUR            TBB : loop / block / pyramid     (TBB)
   ├─ GPU                   CUDA (streams, graphs), nvc++, Thrust
   └─ DISTRIBUÉ             décomposition en tuiles (29)
        ↳ E/S HDF5 + visualisation PNG (7, 8) · industrialisation (27)
```

Chaque étape est isolée, mesurable, et conserve un résultat numérique identique —
illustration directe des principes du [Jour 1](day1.md).
