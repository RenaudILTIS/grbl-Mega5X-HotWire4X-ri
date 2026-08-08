# Prérequis techniques — Firmware de la machine

**HotWire4X** communique avec la machine de découpe au fil chaud via un firmware
**GRBL Mega‑5X** personnalisé pour cet usage. Ce firmware est un dérivé du fork
[`fra589/grbl-Mega-5X`](https://github.com/fra589/grbl-Mega-5X), recompilé avec une
configuration spécifique à HotWire4X.

## Firmware requis

| Élément | Spécification |
|---|---|
| Carte de commande | Arduino **Mega 2560** + shield **RAMPS 1.4** |
| Firmware | GRBL Mega‑5X (variante HotWire4X), rapporte **Grbl 1.2i** |
| Build minimum | **20260701** |
| Axes | **4 — XYZA** |
| Signal de chauffe | **PWM logique 0–5 V sur la broche D6, ~244 Hz** (pilote un étage de puissance externe) |

## Vérifier son firmware

Connectez la carte et envoyez la commande `$I`. La réponse doit contenir :

```
[BOARD:RAMPS 1.4 - HotWire4X PWM-D6 - R.ILTIS]
[VER:1.2i.20260701:]
[AXS:4:XYZA]
```

Si ces lignes n'apparaissent pas (firmware standard, mauvais nombre d'axes, chauffe sur D8…),
(re)flashez le firmware HotWire4X.

## Obtenir le firmware

Le firmware compilé (`.hex`, prêt à flasher via **XLoader** ou **avrdude**) est fourni sur
**[hotwire4x.com](https://hotwire4x.com)**.

Ce firmware est distribué sous licence **GPLv3**. Conformément à cette licence, le **code
source complet** correspondant est disponible **sur simple demande**.
