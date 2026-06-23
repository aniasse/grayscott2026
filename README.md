# Gray Scott School 2026 — Notes & Synthèses

Mes notes de la **Gray Scott School 2026** (CINERI / LAPP) : une école d'été sur
le **calcul haute performance (HPC)** en C++, construite autour d'un fil rouge —
optimiser une simulation du modèle de **Gray-Scott** (réaction-diffusion).

L'idée de ce dépôt : **partager**, en clair et en court, ce qu'on y apprend — les
concepts, le code, et comment refaire tourner l'environnement chez soi.

<p align="center">
  <img src="docs/img/gs_evolution_3.png" width="280" alt="Motif de Turing généré par Gray-Scott"/>
  <br/>
  <em>Un motif de Turing produit par la simulation Gray-Scott.</em>
</p>

## 📚 Sommaire

| | Contenu |
|---|---|
| 🧠 **[Jour 1 — Les fondations](docs/day1.md)** | CPU, vectorisation, parallélisme, memory/compute-bound, concurrence, pureté, compilation, tests |
| ⚡ **[Jour 2 — Optimiser un stencil](docs/day2.md)** | Gray-Scott, data layout, vectorisation, blocking, TBB, I/O HDF5, images, MAQAO |
| 🛠️ **[Installation](INSTALL.md)** ([🇬🇧 EN](INSTALL.en.md)) | Reproduire l'environnement en local (Linux & Windows/WSL2), sans conteneur |

## 🎯 En une phrase

> Le HPC, ce n'est pas « écrire du code rapide » par magie : c'est **mesurer**,
> comprendre **où** ça coince (le calcul ? la mémoire ?), et exploiter le matériel
> à fond — **vectorisation** dans chaque cœur, **parallélisme** entre les cœurs,
> et **GPU/multi-nœuds** au-delà.

## 🔗 Liens

- Cours officiel : <https://cta-lapp.pages.in2p3.fr/COURS/PerformanceWithStencil/>
- Modèle de Gray-Scott : réaction-diffusion produisant des « motifs de Turing »

---

*Notes prises pendant l'école — corrections et contributions bienvenues.*
