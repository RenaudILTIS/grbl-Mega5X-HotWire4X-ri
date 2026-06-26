# Firmware GRBL Mega 5X — HotWire4X

Notes de configuration et de compilation du firmware fil chaud 4 axes.

## Cible

| Élément | Valeur |
|---|---|
| Carte | Arduino Mega 2560 + shield RAMPS 1.4 |
| Firmware | grbl-Mega-5X (fork **fra589**), rapporte `Grbl 1.2i` |
| Source | dépôt canonique `fra589/grbl-Mega-5X` (compilé soi-même, **pas** le binaire RcKeith) |
| Machine | découpeuse fil chaud 4 axes, axes `XYZA` |
| Build | 20260521 |

Principe directeur : on compile depuis la source publique pour garder la maîtrise totale du `cpu_map.h` (broches, PWM, limites), comme sur la SKR Pico en grblHAL. La chauffe sort en **signal logique sur D6** (et non via le MOSFET D8) pour piloter un étage de puissance externe.

---

## 1. Ajustements firmware par fichier

### `config.h`

| Réglage | Valeur | Rôle |
|---|---|---|
| `N_AXIS` | `4` | 4 axes |
| `N_AXIS_LINEAR` | `4` | tous les axes linéaires (foam cutter) |
| `SPINDLE_PWM_ON_D8` | **commenté** | on n'utilise pas le MOSFET D8 |
| `SPINDLE_PWM_ON_D6` | **actif** | chauffe en PWM logique 0-5 V, 8 bits, sur Servo 2 (D6) |
| `SPINDLE_PWM_ON_D9` | commenté | — |
| `HOMING_INIT_LOCK` | **commenté** | démarrage en `Idle` et non en `Alarm` ; `$22` reste pleinement fonctionnel |

Cycle de homing (bloc `#if N_AXIS == 4`) — un seul cycle regroupant les 4 axes, pour un homing simultané avec **arrêt indépendant par axe** (le premier qui touche son contact s'arrête, les autres continuent) :

```c
#if N_AXIS == 4 // 4 axis : homing
  #define HOMING_CYCLE_0 ((1<<AXIS_1)|(1<<AXIS_2)|(1<<AXIS_3)|(1<<AXIS_4)) // X, Y, Z, A ensemble, arrêt indépendant par axe
```

> Important : ne laisser **qu'un seul** cycle. L'ancien `HOMING_CYCLE_0 (1<<AXIS_3)` (Z first) et `HOMING_CYCLE_1 ((1<<AXIS_1)|(1<<AXIS_2))` (X,Y) sont supprimés. Le « Z first to clear workspace » est une logique fraisage sans objet sur un fil chaud.

### `cpu_map.h`

Hard limits activés :

```c
#define ENABLE_RAMPS_HW_LIMITS   // décommenté
```

Fins de course — **un seul contact par axe**, en NC, avec `min` et `max` pointant sur la **même broche** par axe (aucune broche flottante → NC-safe) :

| Axe | Broche | Port/bit AVR | Connecteur RAMPS |
|---|---|---|---|
| 1 — X | D3 | port E, bit 5 | header endstop X |
| 2 — Y | D14 | port J, bit 1 | header endstop Y |
| 3 — Z | D18 | port D, bit 3 | header endstop Z |
| 4 — A | **D49** | port L, bit 0 | **AUX-3** (convention RcKeith) |

> Le passage de l'axe A sur AUX-3 (D49) sépare le câblage des fins de course (AUX-3) de celui des boutons (AUX-2) et reste compatible avec les montages RcKeith existants. Valide tant que `N_AXIS = 4` : D49/D53 sont sinon les broches de l'axe 6, non compilées ici.

Boutons (inchangés) — port K, connecteur **AUX-2** :

| Bouton | Fonction grbl | Broche |
|---|---|---|
| E-Stop | Reset (soft / abort) | A9 |
| Pause | Feed hold | A10 |
| Resume | Cycle start | A11 |
| (non câblé) | Safety door | A12 |

Sortie chauffe (PWM) : **D6** = signal Servo 2 (0-5 V logique).

### `report.c`

Ligne d'identification ajoutée en tête de `report_build_info()` (juste après le bloc `SORT_REPORT_BY_AXIS_NAME`, avant le `[VER:]`) :

```c
printPgmString(PSTR("[BOARD:RAMPS 1.4 - HotWire4X PWM-D6 - R.ILTIS]\r\n"));
```

Survit à `$RST=*` (contrairement à la chaîne `$I=`). Donne une signature `[BOARD:…HotWire4X…]` commune aux deux firmwares de l'écosystème (SKR Pico grblHAL + RAMPS Mega-5X).

---

## 2. Brochage récapitulatif

| Fonction | Emplacement RAMPS | Broche(s) |
|---|---|---|
| Driver axe 1 (X) | socket X | step A0 / dir A1 / en D38 |
| Driver axe 2 (Y) | socket Y | step A6 / dir A7 / en A2 |
| Driver axe 3 (Z) | socket Z | step D46 / dir D48 / en A8 |
| Driver axe 4 (A) | socket **E0** | step D26 / dir D28 / en D24 |
| Fin de course X | header endstop | D3 |
| Fin de course Y | header endstop | D14 |
| Fin de course Z | header endstop | D18 |
| Fin de course A | **AUX-3** | D49 |
| Bouton E-Stop | AUX-2 | A9 |
| Bouton Pause | AUX-2 | A10 |
| Bouton Resume | AUX-2 | A11 |
| PWM chauffe fil | header Servo 2 | D6 (0-5 V logique) |

Contacts câblés entre Signal et masse (pull-ups internes). Mode NC retenu (`$5=1`) : fil coupé = alarme.

---

## 3. Méthode de compilation (Arduino IDE)

1. Installer **l'Arduino IDE** (inclut avr-gcc + avrdude + support Mega 2560).
2. Récupérer les sources : `fra589/grbl-Mega-5X` → `<> Code` → *Download ZIP*, puis dézipper. Le code est dans le sous-dossier **`grbl/`**.
3. Appliquer les modifications ci-dessus dans `grbl/config.h`, `grbl/cpu_map.h`, `grbl/report.c`.
4. Importer la bibliothèque — deux options :
   - **Recommandé pour itérer** : copier directement le dossier `grbl` dans `Documents/Arduino/libraries/` (→ `libraries/grbl/`), éditer sur place, recompiler. Pas de re-zippage à chaque essai.
   - **Add .ZIP Library** : zipper le **dossier `grbl`** lui-même (pas son contenu en vrac) en `grbl.zip`, puis `Croquis → Inclure une bibliothèque → Ajouter la bibliothèque .ZIP`. (Supprimer toute ancienne bibliothèque grbl au préalable.)
5. Ouvrir l'exemple `Fichier → Exemples → grbl → GrblUpload` (ne pas modifier le `.ino`).
6. `Outils → Type de carte → Arduino Mega or Mega 2560`, processeur `ATmega2560`, puis sélectionner le port.
7. **Téléverser**.

### Récupérer le `.hex`

- Propre : `Croquis → Exporter les binaires compilés` → fichier dans `…/grblUpload/build/arduino.avr.mega/grblUpload.ino.hex` (version **sans** bootloader, à flasher via XLoader/avrdude).
- Fiable : activer le mode verbose (`Préférences`), relancer, et lire le chemin `.hex` dans la ligne avrdude (`-Uflash:w:<chemin>:i`).

Renommable en `firmware-mega2560.hex` pour distribution/flash via XLoader.

---

## 4. Réglages runtime après flash (paramètres `$`, pas des fichiers)

Obligatoire d'abord (changement de `N_AXIS`) :

```
$RST=*
```

Puis :

| Paramètre | Valeur | Rôle |
|---|---|---|
| `$5` | `1` | inversion limites → NC |
| `$21` | `1` | hard limits activés |
| `$22` | `0` ou `1` | homing (activé à la demande) |
| `$23` | (masque) | direction de homing par axe |
| `$24` / `$25` | — | feed / seek rate homing |
| `$26` | — | debounce |
| `$27` | — | pull-off |
| `$130`–`$133` | — | courses max X / Y / Z / A |

---

## 5. Identification du firmware (`$I`)

Réponse attendue :

```
[BOARD:RAMPS 1.4 - HotWire4X PWM-D6 - R.ILTIS]
[VER:1.2i.20260521:]
[AXS:4:XYZA]
[OPT:VNMGHL,35,255,xx]
ok
```

Notes sur `[OPT:]` :
- `V` variable spindle, `N` line numbers, `M` mist M7, `G` **safety door support** (toujours émis).
- `H` = `HOMING_SINGLE_AXIS_COMMANDS` activé.
- `L` apparaît car `HOMING_INIT_LOCK` est désactivé (`#ifndef HOMING_INIT_LOCK`) → confirmation que le boot n'est plus verrouillé.

Détection côté HotWire4X :
- `[VER:]` présent **et** absence de `[FIRMWARE:GRBLHAL]` / `[NEWOPT:]` → classé `MEGA5X`. (Le `[OPT:]` classique est neutre ; seul `[NEWOPT:]` basculerait en grblHAL — à ne jamais émettre.)
- Marqueur positif plus robuste : présence de `[AXS:` (propre au fork fra589, absent du grbl gnea).
- Identité produit : token `HOTWIRE4X` dans la ligne `[BOARD:]`.
