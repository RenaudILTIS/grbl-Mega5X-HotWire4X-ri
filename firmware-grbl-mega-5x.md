# Firmware GRBL Mega 5X — HotWire4X

Notes de configuration, de compilation et de versionnement du firmware fil chaud 4 axes.

> Révision : intègre la fréquence PWM 244 Hz (D6), la polarité figée des boutons, le workflow Git (carnet de croquis pointé sur le dépôt) et les notes G93 / safety door.

## Cible

| Élément | Valeur |
|---|---|
| Carte | Arduino Mega 2560 + shield RAMPS 1.4 |
| Firmware | grbl-Mega-5X (fork **fra589**), rapporte `Grbl 1.2i` |
| Source | dépôt canonique `fra589/grbl-Mega-5X` (compilé soi-même, **pas** le binaire RcKeith) |
| Machine | découpeuse fil chaud 4 axes, axes `XYZA` |
| Build | 20260701 |

Principe directeur : on compile depuis la source publique pour garder la maîtrise totale du `cpu_map.h` (broches, PWM, limites), comme sur la SKR Pico en grblHAL. La chauffe sort en **signal logique sur D6, à ~244 Hz** (et non via le MOSFET D8) pour piloter un étage de puissance externe.

---

## 1. Ajustements firmware par fichier

### `config.h`

| Réglage | Valeur | Rôle |
|---|---|---|
| `N_AXIS` | `4` | 4 axes |
| `N_AXIS_LINEAR` | `4` | tous les axes linéaires (foam cutter) |
| `SPINDLE_PWM_ON_D8` | **commenté** | on n'utilise pas le MOSFET D8 |
| `SPINDLE_PWM_ON_D6` | **actif** | chauffe en PWM logique 0-5 V, 8 bits, sur Servo 2 (D6) — fréquence réglée dans `cpu_map.h` |
| `SPINDLE_PWM_ON_D9` | commenté | — |
| `HOMING_INIT_LOCK` | **commenté** | démarrage en `Idle` et non en `Alarm` ; `$22` reste pleinement fonctionnel |

Cycle de homing (bloc `#if N_AXIS == 4`) — un seul cycle regroupant les 4 axes, pour un homing simultané avec **arrêt indépendant par axe** (le premier qui touche son contact s'arrête, les autres continuent) :

```c
#if N_AXIS == 4 // 4 axis : homing
  #define HOMING_CYCLE_0 ((1<<AXIS_1)|(1<<AXIS_2)|(1<<AXIS_3)|(1<<AXIS_4)) // X, Y, Z, A ensemble, arrêt indépendant par axe
```

> Important : ne laisser **qu'un seul** cycle. L'ancien `HOMING_CYCLE_0 (1<<AXIS_3)` (Z first) et `HOMING_CYCLE_1 ((1<<AXIS_1)|(1<<AXIS_2))` (X,Y) sont supprimés. Le « Z first to clear workspace » est une logique fraisage sans objet sur un fil chaud. L'arrêt indépendant par axe suppose **un contact par axe sur sa propre broche** — c'est le cas (D3/D14/D18/D49).

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

> **Polarité des boutons figée à la compilation.** Contrairement aux fins de course (inversables par `$5`), les entrées de contrôle n'ont **aucun paramètre `$`** d'inversion. Leur polarité est fixée par `INVERT_CONTROL_PIN_MASK` dans `config.h` (par défaut désactivé → boutons **NO**, actifs au niveau bas, pull-up interne). Aucune lettre de `[OPT:]` ne l'expose. Vérification du comportement réel via le champ `Pn:` du report `?` (lettres `R`=Reset, `H`=Feed hold, `S`=Cycle start, `D`=Door) : au repos, **aucune** de ces lettres ne doit apparaître ; un test guidé bouton par bouton confirme la transition appui/relâche.

#### Fréquence PWM de chauffe (bloc D6)

Le prescaler du Timer4 est passé de 1/8 à **1/256** pour obtenir **244 Hz** :

```c
// Bloc SPINDLE_PWM_ON_D6 (repère : TCCRA à UN SEUL bit WGM (1<<WGM41), TOP=255)
#define SPINDLE_TCCRB_INIT_MASK ((1<<WGM42) | (1<<WGM43) | (1<<CS42)) // CS42 = 1/256
#define SPINDLE_OCRA_TOP_VALUE  0xFF  // TOP=255, 8 bits (résolution inchangée)
```

Calcul : `f = F_CPU / (prescaler × (TOP+1)) = 16e6 / (256 × 256) = 244 Hz`.

Choix justifié : compatible découpe fil chaud polystyrène (la fréquence est indifférente thermiquement, le fil intègre tout ce qui dépasse quelques Hz) **et** optocoupleurs habituels type PC817 (période ~4 ms, fronts propres). La **résolution reste à 255 pas** (changer le prescaler ne touche pas au TOP).

Autres fréquences possibles (même résolution, on ne change que le prescaler dans `SPINDLE_TCCRB_INIT_MASK`) :

| Bits CS | Prescaler | Fréquence |
|---|---|---|
| `CS40` | 1/1 | 62,5 kHz |
| `CS41` | 1/8 | 7,8 kHz |
| `CS41`+`CS40` | 1/64 | 977 Hz |
| **`CS42`** | **1/256** | **244 Hz** ← retenu |
| `CS42`+`CS40` | 1/1024 | 61 Hz |

> ⚠️ **Deux pièges identifiés.** (1) Le firmware contient trois blocs spindle mutuellement exclusifs (`_D8`/`_D6`/`_D9`) ; seul le bloc **D6** est compilé ici. Le distinguer du D8 par `TCCRA_INIT_MASK` : **un seul** bit WGM `(1<<WGM41)` = D6 (8 bits) ; **deux** bits `((1<<WGM40)|(1<<WGM41))` = D8 (16 bits). (2) Le commentaire d'origine « about 1.9 kHz at 1/8 prescaler » sur le bloc D6 est un copier-coller **erroné** du bloc D8 (où TOP=1024 → 1,9 kHz). Sur D6, TOP=255, donc 1/8 donnerait 7,8 kHz, pas 1,9 kHz. Commentaire corrigé dans le fichier.

Vérification : `M3 S500` puis oscilloscope sur D6 → ~244 Hz.

### `defaults.h`

Fonctions de sécurité **désactivées par défaut** (valeurs de première mise sous tension, upstream à `1`) :

| Réglage | Valeur | Rôle |
|---|---|---|
| `DEFAULT_HOMING_ENABLE` | `0` | pas de homing au boot |
| `DEFAULT_SOFT_LIMIT_ENABLE` | `0` | soft limits off |
| `DEFAULT_HARD_LIMIT_ENABLE` | `0` | hard limits off |

Sur cette découpeuse fil chaud amateur, la mise à l'origine se fait **à la main** (déplacement des chariots) : pas de séquence de homing par défaut. Les soft limits n'ayant de sens qu'après un homing (position absolue connue), les trois sont désactivés ensemble par cohérence. Ce ne sont que des **défauts** de première mise sous tension : ils restent activables à chaud via `$22` (homing), `$20` (soft limits) et `$21` (hard limits) — cf. §5 — sans recompiler, et sont modifiables depuis HotWire4X.

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
| PWM chauffe fil | header Servo 2 | D6 (0-5 V logique, **244 Hz**) |

Contacts câblés entre Signal et masse (pull-ups internes). Mode NC retenu (`$5=1`) : fil coupé = alarme.

---

## 3. Notes firmware diverses

**Modes d'avance G93 / G94.** Aucune option de compilation : ce sont des modes du parseur g-code, **toujours disponibles**. Au boot (et après reset) le firmware est en **G94** (unités/min) ; il bascule en **G93** (inverse time) dès qu'une ligne `G93` est reçue. En G93, chaque `G1/G2/G3` doit porter un mot `F`. C'est le sender (HotWire4X) qui décide ; rien à régler dans le firmware.

**Safety door (A12, non câblé).** Entrée d'interverrouillage de capot : à l'ouverture, Grbl passe en état `DOOR`, décélère, **coupe broche et coolant**, et reprend sur Cycle start une fois refermé. Comportement volontairement **inverse** du feed hold (qui, lui, laisse la chauffe active pendant la pause). Inutile sur une machine fil chaud ouverte → laissée non câblée (pull-up interne, jamais déclenchée).

---

## 4. Méthode de compilation (Arduino IDE)

1. Installer **l'Arduino IDE** (inclut avr-gcc + avrdude + support Mega 2560).
2. Récupérer les sources : `fra589/grbl-Mega-5X` → `<> Code` → *Download ZIP*, puis dézipper. Le code est dans le sous-dossier **`grbl/`**.
3. Appliquer les modifications ci-dessus dans `grbl/config.h`, `grbl/cpu_map.h`, `grbl/report.c`.
4. Rendre la bibliothèque visible par l'IDE — pour le développement versionné, voir la **section 6** (carnet de croquis pointé directement sur le dépôt). À défaut, copier le dossier `grbl` dans `…/Arduino/libraries/`, ou l'importer zippé (zipper le **dossier** `grbl`, pas son contenu en vrac).
5. Ouvrir l'exemple `Fichier → Exemples → grbl → GrblUpload` (ne pas modifier le `.ino`).
6. `Outils → Type de carte → Arduino Mega or Mega 2560`, processeur `ATmega2560`, puis sélectionner le port.
7. **Téléverser**.

### Récupérer le `.hex`

- Propre : `Croquis → Exporter les binaires compilés` → fichier dans `…/grblUpload/build/arduino.avr.mega/grblUpload.ino.hex` (version **sans** bootloader, à flasher via XLoader/avrdude).
- Fiable : activer le mode verbose (`Préférences`), relancer, et lire le chemin `.hex` dans la ligne avrdude (`-Uflash:w:<chemin>:i`).

Renommable en `firmware-mega2560.hex` pour distribution/flash via XLoader.

---

## 5. Réglages runtime après flash (paramètres `$`, pas des fichiers)

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

**Distribution / cohérence EEPROM.** Le flash ne touche pas l'EEPROM ; `$RST=*` est nécessaire après tout changement de `N_AXIS` (réalignement des blocs par axe). Pour un firmware distribué : incrémenter `SETTINGS_VERSION` (`settings.h`) à chaque changement de disposition EEPROM force l'auto-restauration au 1er boot ; soigner les `DEFAULT_*` (`defaults.h`). Côté HotWire4X : à la connexion, comparer un sous-ensemble **critique** de `$$` (géométrie + sécurité : `$100-$103`, `$110-$113`, `$120-$123`, `$130-$133`, `$5/$20/$21/$22/$23`) au profil attendu ; en cas d'écart, **désactiver la découpe** et imposer la validation manuelle via l'écran de paramétrage (pas d'écrasement à l'aveugle). Distinguer « mismatch lisible » (→ écran de diff) de « EEPROM illisible / version incohérente » (→ `$RST=*` puis push profil). Relire `$$` après écriture avant de réactiver la découpe.

---

## 6. Environnement de développement et versionnement

Objectif : un **seul exemplaire physique** des sources, servant à la fois de dépôt Git et de carnet de croquis Arduino, hors OneDrive.

**Organisation retenue.** Le dépôt vit dans `C:\Github\grbl-Mega5X-HotWire4X-ri` (hors OneDrive → `.git` à l'abri de la synchro, qui peut corrompre un `.git`). On y fait pointer **directement le carnet de croquis** de l'IDE Arduino, **sans jonction** : `Fichier → Préférences → Emplacement du carnet de croquis` → `C:\Github\grbl-Mega5X-HotWire4X-ri`. L'IDE cherche alors ses bibliothèques dans le sous-dossier `libraries\` de ce dépôt.

Arborescence requise (le carnet de croquis impose ce niveau `libraries\<nom>\`) :

```
C:\Github\grbl-Mega5X-HotWire4X-ri\   ← carnet de croquis ET dépôt Git
└── libraries\
    └── grbl\
        ├── config.h, cpu_map.h, grbl.h, report.c, …
        └── examples\
            └── GrblUpload\
                └── GrblUpload.ino
```

Si `Fichier → Exemples → grbl → GrblUpload` apparaît et compile, la structure est bonne. Le seul piège serait de poser les fichiers `grbl` à la racine du dépôt sans le niveau `libraries\grbl\`.

**Limite à connaître.** Le carnet de croquis est global à l'IDE : cette approche est valable tant qu'il n'y a **qu'un seul projet** de bibliothèque. Pour gérer plusieurs dépôts en parallèle plus tard, il faudra soit les faire cohabiter dans ce `libraries\`, soit basculer vers une jonction (`mklink /J`) par projet.

Édition à l'IDE Arduino **ou** via VS Code ouvert sur `C:\Github\grbl-Mega5X-HotWire4X-ri` (même contenu). L'IDE compile/flashe ; GitHub Desktop versionne le même dossier.

`.gitignore` suggéré — ici Git versionne **tout le carnet de croquis**, pas seulement `grbl`, donc on filtre aussi les éventuels dossiers créés par l'IDE :
```gitignore
build/
*.elf
*.o
.vscode/
# hardware/   # si l'IDE crée des sous-dossiers parasites
# *.hex       # décommenter si on ne veut pas versionner de binaire ;
#             # sinon garder release/firmware-mega2560.hex pour distribution
```

**Licence (GPLv3).** Ce dépôt est un dérivé de grbl-Mega-5X : conserver les en-têtes de licence et l'attribution à fra589, et lister les modifications dans le `README` (PWM D6 244 Hz, limites D49, `[BOARD:]`, homing 4 axes). Distribuer le binaire impose de rendre la source disponible — un dépôt public satisfait cette obligation.

---

## 7. Identification du firmware (`$I`)

Réponse attendue :

```
[BOARD:RAMPS 1.4 - HotWire4X PWM-D6 - R.ILTIS]
[VER:1.2i.20260701:]
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
