# Gray Scott School 2026 — Notes et synthèses

Synthèses du cours de **calcul haute performance (HPC)** de la Gray Scott School 2026
(CINERI / LAPP). Le cours porte sur l'**optimisation de performance en C++**, illustrée
par la simulation du modèle de **Gray-Scott** (réaction-diffusion).

Ce dépôt rassemble les concepts fondamentaux, les méthodes d'optimisation, et la
procédure de reproduction de l'environnement de développement.

<p align="center">
  <img src="docs/img/gs_evolution_3.png" width="280" alt="Motif de Turing généré par Gray-Scott"/>
  <br/>
  <em>Figure — Motif de Turing produit par la simulation Gray-Scott.</em>
</p>

## Sommaire

| Document | Contenu |
|---|---|
| **[Jour 1 — Fondations](docs/day1.md)** | CPU, vectorisation, parallélisme, memory/compute-bound, concurrence, pureté, compilation, tests |
| **[Jour 2 — Optimisation d'un stencil](docs/day2.md)** | Gray-Scott, data layout, vectorisation, blocking, TBB, E/S HDF5, visualisation, MAQAO |
| **[Installation](INSTALL.md)** ([EN](INSTALL.en.md)) | Reproduction de l'environnement en local (Linux et Windows/WSL2), sans conteneur |

## Objet

L'optimisation HPC repose sur trois principes : **mesurer** la performance,
**identifier** le facteur limitant (calcul ou mémoire), puis **exploiter** le matériel —
vectorisation au sein d'un cœur, parallélisme entre cœurs, accélération GPU et
multi-nœuds au-delà d'une machine.

## Références

- Cours officiel : <https://cta-lapp.pages.in2p3.fr/COURS/PerformanceWithStencil/>
- Modèle de Gray-Scott : système de réaction-diffusion produisant des motifs de Turing.

---

*Document de synthèse. Contributions et corrections bienvenues.*
