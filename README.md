# grbl-Mega5X-HotWire4X-ri

Firmware **GRBL Mega‑5X** personnalisé pour une utilisation en **fil chaud** (hot wire)
avec le logiciel **HotWire4X**.

- **Cible matérielle :** Arduino **Mega 2560** + Shield **RAMPS 1.4**
- **Carte (FQBN) :** `arduino:avr:mega:cpu=atmega2560` → dans l'IDE : *Arduino Mega or Mega 2560*, processeur *ATmega2560*
- **Base :** [grbl-Mega-5X](https://github.com/fra589/grbl-Mega-5X) de fra589

---

## ⚙️ Ce repo EST le carnet de croquis (sketchbook) de l'IDE Arduino

L'IDE Arduino ne compile pas un dossier quelconque : il compile le croquis ouvert et
résout `#include <grbl.h>` vers la bibliothèque présente dans **son carnet de croquis**.
Pour que l'IDE compile **directement ce repo**, le *Sketchbook location* de l'IDE pointe ici :

> **File ▸ Preferences ▸ Sketchbook location** = `C:\Github\grbl-Mega5X-HotWire4X-ri`
>
> (stocké dans `C:\Users\<user>\.arduinoIDE\arduino-cli.yaml`, champ `directories.user`)

Avec ce réglage, l'IDE trouve tout seul la bibliothèque `grbl` dans `libraries/`, et il n'y
a **qu'une seule** copie en jeu — c'est important (voir l'avertissement plus bas).

### Structure (imposée par le format « carnet Arduino »)

```
grbl-Mega5X-HotWire4X-ri/        ← carnet de croquis (Sketchbook location de l'IDE)
├── grblUpload/                  ← le croquis à ouvrir dans l'IDE
│   ├── grblUpload.ino           ←   se contente de #include <grbl.h>
│   └── license.txt
├── libraries/
│   └── grbl/                    ← LA bibliothèque (tout le code du firmware + config.h)
│       ├── config.h             ←   configuration des fonctionnalités (à éditer ici)
│       ├── cpu_map.h            ←   affectation des broches (RAMPS 1.4)
│       └── ...                  ←   *.c / *.h
├── firmware-grbl-mega-5x.md     ← notes/doc firmware
└── build/                       ← artefacts de compilation (ignoré par git)
```

---

## 🔨 Compiler / téléverser

1. Ouvrir le croquis : **File ▸ Sketchbook ▸ grblUpload**
2. Carte : **Tools ▸ Board ▸ Arduino AVR ▸ Arduino Mega or Mega 2560**, processeur **ATmega2560**
3. Port : **Tools ▸ Port ▸** (le COM de la carte)
4. **Verify** (Ctrl+R) pour compiler, **Upload** (Ctrl+U) pour flasher.

La personnalisation se fait dans `libraries/grbl/config.h` (et `cpu_map.h` pour les broches).

### Récupérer le `.hex`

- **Verify/Upload** : le `.hex` part dans un cache temporaire d'Arduino
  (`%LOCALAPPDATA%\arduino\sketches\<hash>\`), écrasé à chaque build.
- **Croquis ▸ Exporter les binaires compilés** : écrit dans `grblUpload/build/arduino.avr.mega/`
  (à côté du `.ino`). Ce `build/` est ignoré par git.

---

## ⚠️ Une seule bibliothèque `grbl` à la fois

Avoir **deux** bibliothèques `grbl` visibles par l'IDE provoque des erreurs de lien
« *multiple definition* » (le firmware est compilé et lié deux fois). Tant que le
*Sketchbook location* pointe sur ce repo, seule `libraries/grbl` d'ici est utilisée.
Ne pas réintroduire une autre copie de `grbl` dans un dossier `libraries/` du carnet.

> Historique : ce repo provient d'une copie de l'ancien carnet
> `…\OneDrive\Documents\Arduino`. Le fichier `eeprom.h`, requis par `grbl.h`, manquait
> dans la copie initiale et a été restauré.
