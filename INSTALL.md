# Gray Scott School 2026 — Environnement local

*🏠 [Accueil](README.md) · 🇬🇧 English: [INSTALL.en.md](INSTALL.en.md)*

Procédure de reproduction **en local** de l'environnement du cours HPC
*« Performance With Stencil »* (Gray Scott School 2026), **sans conteneur**.

Le cours s'appuie sur des bibliothèques **PHOENIX** (LAPP), **HDF5** et **TBB**.
Bonne nouvelle : tout est disponible via [**pixi**](https://pixi.sh) (gestionnaire
d'environnements basé sur conda), car les paquets PHOENIX sont publiés sur un canal
conda public (`prefix.dev/phoenix`). **Pas besoin d'apptainer ni d'accès in2p3.**

> ✅ Plateformes : **Linux x86-64** nativement. **Windows** via **WSL2** (voir plus bas).
> macOS : non supporté (les paquets sont `linux-64`) → utiliser une VM/serveur Linux.

---

## TL;DR (Linux)

```bash
# 1. Installer pixi
curl -fsSL https://pixi.sh/install.sh | bash && source ~/.bashrc

# 2. Récupérer le code du cours
git clone https://gitlab.in2p3.fr/CTA-LAPP/COURS/PerformanceWithStencil.git
cd PerformanceWithStencil          # le dossier contenant pixi.toml

# 3. Installer l'environnement (gcc, hdf5, tbb, phoenix…)
pixi install

# 4. Compiler
pixi run bash -c 'mkdir -p build && cd build && cmake .. $(phoenixcmake-config --cmake) -G Ninja && ninja -j$(nproc)'

# 5. Lancer un exemple
pixi shell
cd build
./9-Simulation/autovec_gray_scott_O3 -n 10 -e 200 -r 512 -c 512 -o output.h5
mkdir -p pics && ./8-ImagePlotting/gray_scott_image -i output.h5 -o pics/
```

> 📌 Le dossier de travail est **celui qui contient `pixi.toml`** (selon la version
> du dépôt : racine, `Examples/`, ou `day-2/`). Adapter le `cd` en conséquence.

---

## Prérequis

- **git**
- Connexion internet (les paquets se téléchargent depuis `prefix.dev`)
- ~2–3 Go d'espace disque (toolchain gcc + dépendances)
- Aucun droit `root` requis : pixi s'installe dans `~/.pixi`

---

## Linux (détaillé)

### 1. Installer pixi
```bash
curl -fsSL https://pixi.sh/install.sh | bash
source ~/.bashrc          # ou redémarre le terminal → la commande `pixi` est dispo
pixi --version            # vérifie (>= 0.70)
```

### 2. Récupérer le cours
```bash
git clone https://gitlab.in2p3.fr/CTA-LAPP/COURS/PerformanceWithStencil.git
cd PerformanceWithStencil
ls pixi.toml              # se placer dans le dossier contenant ce fichier
```

### 3. Installer l'environnement
```bash
pixi install
```
Cela télécharge et installe (versions figées par `pixi.lock`) :
`gcc/g++ 15`, `cmake 4`, `ninja`, `make`, **`hdf5 1.14`**, **`tbb-devel 2022`**,
et les libs **`phoenixcmake / phoenixcore / phoenixpng / phoenixoptionparser /
phoenixprogress / phoenixdatastream / phoenixtypestream`**.

Vérifier :
```bash
pixi list | grep -E 'hdf5|tbb|phoenix|gcc|cmake'
```

### 4. Compiler
```bash
pixi run bash -c '
  mkdir -p build && cd build
  cmake .. $(phoenixcmake-config --cmake) -G Ninja
  ninja -j$(nproc)
'
```

### 5. Lancer & vérifier
```bash
pixi shell                # entre dans l'environnement
cd build

# Ex 7 — test du format de données HDF5
ctest -R TestGrayScottDataFormat --output-on-failure

# Ex 9 — simulation -> fichier HDF5
./9-Simulation/autovec_gray_scott_O3 -n 10 -e 200 -r 512 -c 512 -o output.h5

# Ex 8 — HDF5 -> images PNG
mkdir -p pics
./8-ImagePlotting/gray_scott_image -i output.h5 -o pics/
ls pics/                  # série de .png produite (motifs de Turing)
```

Pour recompiler après une modification :
```bash
pixi run bash -c 'cd build && ninja -j$(nproc)'
```

---

## Windows (via WSL2)

Les paquets sont `linux-64` : sur Windows on passe par **WSL2** (Linux intégré à
Windows). C'est rapide et c'est la méthode recommandée.

### 1. Installer WSL2 (une fois)
Dans **PowerShell en administrateur** :
```powershell
wsl --install -d Ubuntu
```
Redémarrer si demandé, puis ouvrir **Ubuntu** depuis le menu Démarrer et créer un
utilisateur Linux.

> Si `wsl` existe déjà : `wsl --update` puis `wsl --install -d Ubuntu`.

### 2. Dans le terminal Ubuntu (WSL), suivre les étapes Linux
```bash
sudo apt update && sudo apt install -y git curl       # outils de base
curl -fsSL https://pixi.sh/install.sh | bash && source ~/.bashrc
git clone https://gitlab.in2p3.fr/CTA-LAPP/COURS/PerformanceWithStencil.git
cd PerformanceWithStencil
pixi install
pixi run bash -c 'mkdir -p build && cd build && cmake .. $(phoenixcmake-config --cmake) -G Ninja && ninja -j$(nproc)'
```
Puis lancer les exemples comme en **Linux** (section ci-dessus).

### Recommandations WSL
- Travailler dans le **système de fichiers Linux** (`~/` dans WSL), **pas** dans
  `/mnt/c/...` : la compilation y est nettement plus rapide.
- Visualisation des images PNG : depuis WSL, `explorer.exe .` ouvre le dossier courant
  dans l'explorateur Windows.
- VS Code : installer l'extension **WSL**, puis ouvrir le dossier avec `code .`.

---

## Voir / convertir les images

Les `.png` s'ouvrent directement. Pour en faire un GIF animé (si ImageMagick est
présent) :
```bash
convert -delay 20 pics/*.png animation.gif
```

---

## Dépannage

| Problème | Solution |
|---|---|
| `pixi: command not found` | `source ~/.bashrc` ou rouvre le terminal |
| `cmake`/`phoenixcmake-config` introuvable | lance les commandes via `pixi run …` ou `pixi shell` (pas le cmake système) |
| erreur de résolution / téléchargement | vérifie l'accès à `prefix.dev` ; relance `pixi install` |
| `pixi install` échoue sur Windows natif | utilise **WSL2** (les paquets sont `linux-64`) |
| binaire `*_gray_scott_O3` absent | vérifie que `ninja` s'est bien terminé ; `ls build/9-Simulation/` |
| `.so` introuvable à l'exécution | lance depuis `pixi shell` (l'env fixe `LD_LIBRARY_PATH`) |

---

## Pourquoi pixi (et pas le conteneur) ?

Le cours fournit aussi une image **apptainer**. Ici on utilise **pixi** parce que :
- c'est **multi-plateforme** et **sans root**,
- l'environnement est **reproductible à l'identique** (`pixi.lock` fige les versions),
- pas besoin de télécharger une grosse image conteneur ni d'un registre privé.

Les deux approches donnent le **même environnement** (mêmes versions de gcc, HDF5,
TBB, PHOENIX).

---

*Cours officiel : <https://cta-lapp.pages.in2p3.fr/COURS/PerformanceWithStencil/>*
