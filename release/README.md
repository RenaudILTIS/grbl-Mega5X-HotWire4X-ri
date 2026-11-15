# release/ — firmwares compilés

Binaires **prêts à flasher** du firmware GRBL Mega‑5X personnalisé HotWire4X.
Ce dossier est **versionné** (contrairement à `build/`, qui est un cache jetable ignoré par git).

## Provenance

Ces `.hex` sont **compilés depuis les sources de ce dépôt** (`libraries/grbl/`), via l'IDE
Arduino : **Croquis ▸ Exporter les binaires compilés**, puis copie du fichier obtenu
(`grblUpload/build/arduino.avr.mega/grblUpload.ino.hex`) ici sous un nom parlant.

- **Carte cible :** Arduino Mega 2560 + shield RAMPS 1.4
- **Version firmware :** `1.2i` build `20260701` (`GRBL_VERSION` / `GRBL_VERSION_BUILD` dans `libraries/grbl/grbl.h`)
- **Variante fournie :** `.hex` **sans bootloader** → à flasher via **XLoader** ou **avrdude**
  (le bootloader Mega d'origine est conservé).

## Convention de nommage

```
grbl-Mega5X-HotWire4X-ri_v<version>-<build>.hex
```

Exemple : `grbl-Mega5X-HotWire4X-ri_v1.2i-20260701.hex`

## Publication

Chaque version est aussi publiée en **release GitHub**, sous le nom stable
`grbl-Mega5X-HotWire4X.hex` (sans numéro de version) : l'adresse
`…/releases/latest/download/grbl-Mega5X-HotWire4X.hex`, citée par le site hotwire4x.com,
suit ainsi toujours la dernière version. Le tag de la release reprend `v<version>-<build>`.

## Régénérer / mettre à jour

1. Compiler le croquis `grblUpload` dans l'IDE (carte *Arduino Mega or Mega 2560*, proc. *ATmega2560*).
2. **Croquis ▸ Exporter les binaires compilés**.
3. Copier `grblUpload/build/arduino.avr.mega/grblUpload.ino.hex` ici, renommé selon la convention
   ci‑dessus (mettre à jour `<version>`/`<build>` si `grbl.h` a changé).

> Rappel : à chaque changement de `N_AXIS` ou de disposition EEPROM, faire `$RST=*` après flash.
> Voir `firmware-grbl-mega-5x.md` (§5) pour les réglages `$` à appliquer.
