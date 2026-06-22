# Matériel — nomenclature, câblage & alimentation

Capteurs, bus RS-485, chaîne d'alimentation et secours du local pilote.
Vue d'ensemble : [README du projet](../README.md). Choix techniques : [ARCHITECTURE.md](ARCHITECTURE.md).

## Sommaire

- [Nomenclature (BOM)](#nomenclature-bom)
- [Capteurs](#capteurs)
- [Bus RS-485 — câblage 4 fils](#bus-rs-485--câblage-4-fils)
- [Raccordement d'un nœud (bornier Wago)](#raccordement-dun-nœud-bornier-wago)
- [Chaîne d'alimentation & secours](#chaîne-dalimentation--secours)
- [Microcontrôleurs](#microcontrôleurs)

---

## Nomenclature (BOM)

| Élément | Référence | Rôle |
|---|---|---|
| Capteur industriel | **S-THP-01A** | T° / Humidité / Pression |
| Capteur économique | **XY-MD04** | T° / Humidité (comparaison low-cost) |
| Sondes températures extrêmes | **PT100** × 2 | congélateur, four |
| Module d'acquisition PT100 | **PTA8C04** | conversion PT100 → Modbus (4 canaux) |
| Transceiver RS-485 | **MAX485** | interface bus différentiel ↔ UART du nRF |
| Carte d'acquisition | **nRF52 DK (PCA10040)** | maître Modbus + BLE |
| Passerelle | **WT32-ETH01** (ESP32 + LAN8720) | BLE → Ethernet |
| Connecteurs | **Wago 221** (3 entrées) | dérivation du bus en chaque nœud |
| Secours | **Mini-UPS 12 V**, batterie 10 400 mAh | autonomie sur coupure secteur |
| Convertisseur élévateur | **Boost 12 V → 24 V** | alimentation du bus capteurs |
| Convertisseur abaisseur | **Buck 12 V → 5 V** | alimentation nRF / ESP32 |
| Référence de validation | **Ebro EBI 310** | logger certifié |

---

## Capteurs

| Capteur | Mesures | Plage d'usage | Interface |
|---|---|---|---|
| **S-THP-01A** | T° / Hum. / Pression | enceintes courantes, locaux | Modbus RTU — esclave 1, FC04 reg `0x0000` |
| **XY-MD04** | T° / Hum. | comparaison économique | Modbus RTU — esclave 2, FC04 reg `0x0001` |
| **PT100** (via PTA8C04) | T° (2 canaux) | congélateur (−80 °C), four (100 °C) | Modbus RTU — esclave 3, FC03 reg `0x0000` |

Préparation et adressage (Slave ID unique) : dépôt **[modbus-rs485-sensor-utils](https://github.com/AngeEcam/modbus-rs485-sensor-utils)**.

---

## Bus RS-485 — câblage 4 fils

Le bus chaîne tous les capteurs en **daisy-chain**. Quatre fils traversent chaque nœud **sans coupure** :

| Fil | Couleur (schéma) | Rôle |
|---|---|---|
| **VCC** | rouge | alimentation +24 V |
| **GND** | noir | masse commune |
| **A+** | bleu | RS-485 données A+ |
| **B−** | vert | RS-485 données B− |

Caractéristiques RS-485 : signal **différentiel** (robuste au bruit EM du bâtiment), portée jusqu'à ~1200 m, **multipoint**. Deux **terminaisons** sont requises aux extrémités du bus.

---

## Raccordement d'un nœud (bornier Wago)

Chaque capteur se branche **en dérivation** sur le bus à l'aide de **connecteurs Wago 221 à 3 entrées**, à raison d'**un connecteur par signal** (4 connecteurs Wago par capteur : VCC, GND, A+, B−).

Pour chaque connecteur :

| Entrée | Raccordement |
|---|---|
| **Entrée 1** | câble venant du capteur précédent (n−1) |
| **Entrée 2** | dérivation vers le capteur courant (n) |
| **Entrée 3** | câble vers le capteur suivant (n+1) |

Le bus n'est jamais coupé : il « passe » par chaque nœud (entrée 1 → entrée 3) tandis que l'entrée 2 alimente/connecte le capteur local.

```
   bus (n-1) ──[E1] Wago [E3]── bus (n+1)
                    │ E2
                    ▼
                capteur (n)
```

> Schémas de référence du dépôt : `Schma_systme.png` (vue d'ensemble) et `Zoom_Bornier_Wago.png` (détail d'un nœud). Pensez à les ajouter au dossier `docs/` du dépôt pour qu'ils s'affichent ici.

---

## Chaîne d'alimentation & secours

```
Prise murale 230 V
        │
        ▼
  Mini-UPS 12 V (batterie 10 400 mAh)
        ├───────────────► Boost 12 V → 24 V ──► bus capteurs (VCC 24 V)
        │
        └───────────────► Buck 12 V → 5 V  ──► nRF52832 / ESP32
```

- Le **mini-UPS 12 V** maintient l'ensemble alimenté pendant une coupure secteur.
- Le **boost 12 → 24 V** fournit le rail des capteurs ; le **buck 12 → 5 V** alimente l'acquisition et la passerelle.
- **Autonomie estimée** : ≈ **84 h** théoriques pour une conso de ≈ 123,8 mA et une batterie 10 400 mAh (dimensionné, non testé — bilan détaillé dans [ARCHITECTURE.md](ARCHITECTURE.md)).

> Pendant la coupure, le **FIFO** du nRF continue d'empiler les mesures (≈ 4 h 48 de tampon) et l'**offline queue** de l'ESP32 met en attente les POST non aboutis. Au retour, tout est rejoué **sans perte**.

---

## Microcontrôleurs

| Carte | SoC | Rôle | Radio | Dépôt |
|---|---|---|---|---|
| nRF52 DK (PCA10040) | nRF52832 | acquisition, Modbus, FIFO | BLE 5.0 | [hmra-nrf52-firmware](https://github.com/AngeEcam/hmra-nrf52-firmware) |
| WT32-ETH01 | ESP32 + LAN8720 | passerelle BLE → HTTP | BLE + Ethernet | [hmra-esp32-gateway](https://github.com/AngeEcam/hmra-esp32-gateway) |

Le nRF dialogue avec le bus RS-485 via un transceiver **MAX485** sur son `uart0`. L'ESP32 se connecte au nRF en **BLE central** (NUS) puis relaie vers le serveur en **HTTP sur Ethernet**.
