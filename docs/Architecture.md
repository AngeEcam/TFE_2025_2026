# Architecture technique

Choix d'architecture, protocoles, formats de trames et dimensionnement du système HMRA Monitor.
Pour la vue d'ensemble et les liens vers les dépôts de code, voir le [README du projet](../README.md).

## Sommaire

- [Principe directeur : local par local](#principe-directeur--local-par-local)
- [Couche capteurs — RS-485 / Modbus RTU](#couche-capteurs--rs-485--modbus-rtu)
- [Acquisition — nRF52832](#acquisition--nrf52832)
- [Tampon FIFO & continuité de service](#tampon-fifo--continuité-de-service)
- [Liaison BLE — Nordic UART Service](#liaison-ble--nordic-uart-service)
- [Passerelle — ESP32](#passerelle--esp32)
- [Backend — FastAPI + SQLite](#backend--fastapi--sqlite)
- [Formats de trames de bout en bout](#formats-de-trames-de-bout-en-bout)

---

## Principe directeur : local par local

JRI-MySirius porte loin (≈ 1 km, sub-GHz, répéteurs) pour couvrir un bâtiment depuis un point central. Sa documentation reconnaît plusieurs faiblesses : lien sensible aux murs, obstacles et personnes ; dégradation quand l'environnement électromagnétique se densifie ; **une liaison radio par capteur** (N capteurs = N liens fragiles).

À chaque cause d'instabilité répond un choix d'architecture :

| Cause d'instabilité JRI | Réponse de ce projet |
|---|---|
| Liaison RF longue à travers le bâtiment | **Saut BLE confiné au local** |
| 1 liaison radio par capteur | **Bus filaire mutualisé** (RS-485) → un seul émetteur radio par local |
| Lien dégradé / perte de données | **Saut court adaptatif + FIFO** → zéro perte si le lien tombe |
| Câblage lourd | Pas de câble supplémentaire entre module d'acquisition et passerelle |

---

## Couche capteurs — RS-485 / Modbus RTU

### Choix du protocole

| Couche physique | Protocole | Différentiel | Portée | Multipoint |
|---|---|---|---|---|
| **RS-485** | **Modbus RTU** | **Oui** | **1200 m** | **Oui** |
| CAN | CANopen | Oui | ~1000 m | Oui |
| 1-Wire | intégré | Non | ~30 m | Oui |
| SPI | intégré | Non | qq cm | Limité |
| I²C | intégré | Non | ~1 m | Oui |
| UART | intégré | Non | qq m | Non |

RS-485 et CAN partagent le même principe différentiel (robustesse au bruit). RS-485 / Modbus RTU est retenu pour sa portée, son multipoint et son écosystème de capteurs industriels.

### Capteurs

| Capteur | Type | Rôle | Adressage Modbus |
|---|---|---|---|
| **S-THP-01A** | Industriel — T° / Hum. / Pression | Frigos, incubateurs, locaux | esclave 1, **FC04** reg `0x0000` |
| **XY-MD04** | Économique — T° / Humidité | Comparaison low-cost | esclave 2, **FC04** reg `0x0001` |
| **PT100** (via **PTA8C04**) | Températures extrêmes | Congélateur (−80 °C), four (100 °C) | esclave 3, **FC03** reg `0x0000`, 2 canaux |

La préparation et l'adressage de ces capteurs se font avec le dépôt **[modbus-rs485-sensor-utils](https://github.com/AngeEcam/modbus-rs485-sensor-utils)**.

> **Ordre de détection critique** dans le scan du bus :
> 1. S-THP-01A **en premier** — répond aussi au FC03 ; testé après, il serait classé PT100 à tort.
> 2. XY-MD04 **en second** — FC04 sur `0x0001`, signature unique.
> 3. PT100 **en dernier** — FC03 uniquement.
>
> Aucune configuration préalable côté firmware : un capteur débranché disparaît au prochain scan, un capteur rebranché est redécouvert automatiquement.

---

## Acquisition — nRF52832

Dépôt : **[hmra-nrf52-firmware](https://github.com/AngeEcam/hmra-nrf52-firmware)** (C / Zephyr).

Comparatif des microcontrôleurs (ordres de grandeur datasheet, puce nue en fonctionnement actif) :

| Plateforme | Radio intégrée | Conso | Verdict |
|---|---|---|---|
| **nRF52832** | **BLE 5.0** | ~1–7 mA | ✅ **Retenu** |
| ESP32 | Wi-Fi / BT / Eth | ~30–240 mA | → utilisé en passerelle |
| STM32 | Non (externe) | ~1–10 mA | Pas de radio native |
| Arduino | Non | ~5–12 mA | Insuffisant |

- **Carte** : nRF52 DK (PCA10040), SoC nRF52832 (Cortex-M4F + BLE 5.0).
- **RTOS** : Zephyr.
- Un **thread capteurs** exécute le cycle d'acquisition toutes les `SENSOR_POLL_MS` = 60 000 ms ; un scan identifie chaque capteur par sa réponse Modbus ; chaque mesure (16 octets) est poussée dans le FIFO ; en fin de cycle, le contenu du FIFO est transmis en BLE.

Constantes clés (`src/main.c`) :

| Constante | Valeur | Rôle |
|---|---|---|
| `SCAN_SLAVE_MIN` / `SCAN_SLAVE_MAX` | 1 / 5 | plage d'adresses scannées |
| `SENSOR_POLL_MS` | 60 000 | intervalle d'acquisition |
| `NUS_MAX_RECORDS` | 10 | nb max de mesures par trame BLE |

---

## Tampon FIFO & continuité de service

### Structure d'une mesure — 16 octets, alignée sur 4

| Champ | Type | Description |
|---|---|---|
| `timestamp` | `uint32_t` | uptime en secondes depuis le boot |
| `temp` | `int16_t` | température × 10 (signé) |
| `humidity` | `uint16_t` | humidité × 10 |
| `pressure` | `uint16_t` | pression × 10 (0 si non mesurée) |
| `sensor_id` | `uint8_t` | identifiant capteur |
| `flags` | `uint8_t` | `VALID` / `ERROR` / `TIMEOUT` |
| `_pad[2]` | `uint8_t` | padding → 16 octets |

Valeurs stockées en **entiers** (dixièmes d'unité) pour éviter le flottant. Taille garantie à la compilation par `BUILD_ASSERT(sizeof(sensor_record_t) == 16, ...)`.

### Calcul de la couverture

| Paramètre | Valeur | Source |
|---|---|---|
| `FIFO_SIZE` | 1440 cases | `fifo_storage.h` |
| Taille d'un record | 16 octets | `BUILD_ASSERT` |
| Acquisition | 60 s | `main.c` · `SENSOR_POLL_MS` |

- Débit : **5 mesures / minute** (3 capteurs × 1 record + 1 module PT100 (2 sondes) × 1 record).
- Capacité RAM : 1440 × 16 = **23 Ko** (buffer circulaire).
- Couverture : 1440 ÷ 5 = **288 min ≈ 4 h 48**.

> **Politique circulaire** : au-delà de la capacité, la mesure la plus ancienne est écrasée et un compteur d'overflows est incrémenté (utile pour monitorer la qualité de la couverture réseau).
> **Thread-safety** : accès protégé par mutex Zephyr (`k_mutex`) — ne pas appeler depuis un ISR.

### Dimensionnement du mini-UPS (dimensionné, non testé)

Bilan de consommation ramené au rail 12 V :

| Composant | P (W) | I @ 12 V (mA) |
|---|---|---|
| S-THP-01A (×1) | 0,120 | 10,0 |
| XY-MD04 (×2) | 0,100 | 8,3 |
| Module PT100 | 0,216 | 18,0 |
| Module nRF52832 | 1,000 | 83,3 |
| Transceiver MAX485 | 0,050 | 4,2 |
| **Total système** | **1,486** | **123,8** |

Batterie **10 400 mAh / 12 V** → 10 400 ÷ 123,8 ≈ **84 h** théoriques. La conso du module nRF est une borne haute prudente (carte de dev, debugger inclus).

---

## Liaison BLE — Nordic UART Service

Le nRF expose un **Nordic UART Service (NUS)** ; l'ESP32 s'y connecte en central.

| UUID | Caractéristique |
|---|---|
| `6e400001-b5a3-f393-e0a9-e50e24dcca9e` | Service NUS |
| `6e400002-b5a3-f393-e0a9-e50e24dcca9e` | RX (central → nRF) |
| `6e400003-b5a3-f393-e0a9-e50e24dcca9e` | TX (nRF → central, notifications) |

Nom d'advertising BLE : **`HMRA_Monitor`**.

Toutes les mesures d'un cycle sont envoyées dans **un seul paquet** (élimine les pertes de notifications) :

```json
{"ts":2135,"d":[{"sid":1,"t":225,"h":411,"p":9948},
                {"sid":2,"t":263,"h":338,"p":0}]}
```

| Clé | Signification |
|---|---|
| `ts` | timestamp (uptime s) de la première mesure du lot |
| `sid` | identifiant capteur |
| `t` / `h` / `p` | température / humidité / pression × 10 (`0` = non mesurée) |

> Buffer JSON limité à 247 octets côté nRF (cohérent avec le MTU BLE) ; un dépassement est détecté et journalisé.

---

## Passerelle — ESP32

Dépôt : **[hmra-esp32-gateway](https://github.com/AngeEcam/hmra-esp32-gateway)** (C++ / Arduino).
Carte **WT32-ETH01** (LAN8720). Pipeline :

1. **Scan BLE** → connexion au device `HMRA_Monitor`.
2. Le callback de notification ne fait que **bufferiser** chaque ligne JSON (parse queue, 20 entrées) — pour ne rien bloquer dans le contexte BLE.
3. **Parsing** du JSON groupé, **éclatement par capteur** → un payload HTTP par mesure.
4. **HTTP POST** vers `/ingest`. En cas d'échec ou de réseau absent, la mesure part dans une **offline queue** (50 entrées), rejouée au retour du réseau.

---

## Backend — FastAPI + SQLite

Dépôt : **[hmra-monitoring-server](https://github.com/AngeEcam/hmra-monitoring-server)** (Python / HTML).

### Schéma SQLite

| Table | Rôle |
|---|---|
| `measurements` | mesures brutes (`received_at`, `ts`, `sid`, `temp_c`, `hum_rh`, `pres_hpa`, `flags`, `gw_ip`) |
| `alert_log` | historique des alarmes (déclenchement, acquittement, résolution auto) |
| `units` | enceintes surveillées (type, service d'utilisation, service responsable) |
| `sensor_config` | configuration par capteur (`mode`, `name`, `unit_id`, `deleted`) |
| `thresholds` | seuils min/max par capteur et paramètre |
| `users` | comptes (rôle, service, email, notification) |
| `meta` | clé/valeur divers |

### Modes de capteur

| Mode | Effet |
|---|---|
| `active` | enregistrement + seuils + alarmes |
| `maintenance` | enregistrement seul, pas d'alarme |
| `disabled` | mesure ignorée |

### Sécurité

| Mesure | Détail |
|---|---|
| **Authentification JWT** | OAuth2, jeton signé HS256, stocké en `sessionStorage` côté client |
| **Hachage bcrypt** | facteur de coût adaptatif |
| **Filtrage côté serveur** | cloisonnement par service appliqué sur chaque route |
| **Secrets externalisés** | `SECRET_KEY` et SMTP chargés depuis `.env` ; démarrage refusé sans clé |

> Référence complète des endpoints : voir le README du dépôt serveur.

---

## Formats de trames de bout en bout

```
Capteurs ──Modbus RTU──▶ nRF52832 ──BLE/NUS (JSON groupé)──▶ ESP32 ──HTTP POST──▶ FastAPI ──▶ SQLite ──▶ Dashboard
```

**Trame BLE (nRF → ESP32)** — groupée, entiers ×10 :
```json
{"ts":2135,"d":[{"sid":1,"t":225,"h":411,"p":9948}]}
```

**Trame HTTP (ESP32 → serveur)** — éclatée par capteur, en grandeurs physiques :
```json
{"ts":2135,"sid":1,"temp_c":22.5,"hum_rh":41.1,"pres_hpa":994.8,"flags":1,"gw_ip":"192.168.1.50"}
```

> L'ESP32 divise par 10 les entiers reçus et omet `hum_rh` / `pres_hpa` lorsqu'ils valent 0.
