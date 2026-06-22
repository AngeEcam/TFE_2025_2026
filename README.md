# TFE_ECAM_2025_2026 — Monitoring environnemental d'un local pilote hospitalier
 
> **Développement et intégration d'un système de monitoring environnemental pour un local pilote hospitalier**
> Ange Simpalingabo — Master en Sciences de l'Ingénieur Industriel, orientation électronique
> Haute École ICHEC–ECAM–ISFSC · Année académique 2025–2026 · Service Biomédical de l'HMRA
 
Ce dépôt est le point d'entrée du projet. Il raconte tout : des idées de conception jusqu'au câblage, et renvoie vers les dépôts de code de chaque brique du système.
 
---
 
## Sommaire
 
- [Le projet en une page](#le-projet-en-une-page)
- [Les dépôts du projet](#les-dépôts-du-projet)
- [Contexte & problématique](#contexte--problématique)
- [Idées de conception](#idées-de-conception)
- [Architecture du système](#architecture-du-système)
- [Mise en œuvre — ordre de montage](#mise-en-œuvre--ordre-de-montage)
- [Documentation détaillée](#documentation-détaillée)
- [Résultats de validation](#résultats-de-validation)
- [Limites & perspectives](#limites--perspectives)
---
 
## Le projet en une page
 
L'objectif : remplacer une solution commerciale fermée (JRI-MySirius) par une chaîne d'acquisition maîtrisable en interne, qui collecte en continu température, humidité et pression sur 5 points de mesure, sans aucune perte de données même en cas de coupure réseau ou électrique.
 
L'idée directrice est un découplage entre acquisition et transmission :
 
- l'acquisition locale est assurée par un module Nordic nRF52 qui dialogue avec les capteurs sur un bus RS-485 (Modbus RTU) -> immunité au bruit électromagnétique, fiabilité sur de longues distances 
- la transmission vers l'infrastructure hospitalière passe par une passerelle ESP32 en Ethernet, reliée au nRF en BLE (ou UART), ce qui isole la couche capteurs de la couche réseau.
| | |
|---|---|
| **Points de mesure** | 5 capteurs max par local |
| **Bus capteurs** | RS-485 daisy-chain, Modbus RTU |
| **Intervalle d'acquisition** | 60 s |
| **Stockage local (FIFO)** | 1440 mesures × 16 octets ≈ 23 Ko RAM → ≈ 4 h 48 de couverture réseau |
| **Liaison montante** | BLE (Nordic UART Service), confinée au local |
| **Autonomie secours** | mini-UPS 12 V → ≈ 84 h théoriques (dimensionné, non testé) |
| **Backend** | FastAPI + SQLite + dashboard temps réel |
| **Référence de validation** | logger certifié Ebro EBI 310 |
 
---
 
## Les dépôts du projet
 
Le système est découpé en briques indépendantes, chacune dans son propre dépôt :
 
| Dépôt | Rôle | Techno |
|---|---|---|
| **[modbus-rs485-sensor-utils](https://github.com/AngeEcam/modbus-rs485-sensor-utils)** | Outils Python pour tester et configurer chaque capteur (scan, lecture, attribution du Slave ID) — **à utiliser en premier** | Python |
| **[hmra-nrf52-firmware](https://github.com/AngeEcam/hmra-nrf52-firmware)** | Firmware d'acquisition : maître Modbus RTU, découverte auto des capteurs, tampon FIFO, transmission BLE | C / Zephyr |
| **[hmra-esp32-gateway](https://github.com/AngeEcam/hmra-esp32-gateway)** | Passerelle : client BLE central, parsing JSON, relais HTTP vers le serveur via Ethernet | C++ / Arduino |
| **[hmra-monitoring-server](https://github.com/AngeEcam/hmra-monitoring-server)** | Backend FastAPI + SQLite, authentification, seuils, alarmes email, dashboard temps réel | Python / HTML |
 
```
modbus-rs485-sensor-utils   ──►  hmra-nrf52-firmware  ──BLE──►  hmra-esp32-gateway  ──HTTP──►  hmra-monitoring-server
   (préparer les capteurs)        (acquisition + FIFO)            (passerelle Ethernet)          (serveur + dashboard)
```
 
---
 
## Contexte & problématique
 
Le Service Biomédical de l'HMRA surveille ses enceintes critiques (réfrigérateurs, congélateurs, incubateurs, fours) avec le système commercial JRI-MySirius. Trois limites motivent ce projet :
 
- **Communication difficile avec le fournisseur** et faible réactivité sur les incidents ;
- **Dépendance à une solution externe** : système fermé qui interdit toute intervention ou évolution par le service ;
- **Instabilité du signal radio**, aggravée par la densification du bruit électromagnétique dans le bâtiment.
> **Pourquoi du BLE alors que le reproche porte sur la radio ?**
> Le problème de JRI n'est pas la radio en soi, mais le *réseau de radios* : JRI porte loin (≈ 1 km, sub-GHz, répéteurs) avec une liaison RF par capteur, à travers tout le bâtiment — sensible aux murs, obstacles et personnes. Ici, la réponse est locale, par local : un bus filaire mutualisé remonte tous les capteurs vers un module unique, et un seul saut BLE court reste confiné à la pièce.
 
---
 
## Idées de conception
 
À chaque cause d'instabilité de JRI répond un choix d'architecture :
 
| Cause d'instabilité JRI | Réponse de ce projet |
|---|---|
| Liaison RF longue à travers le bâtiment | **Saut BLE confiné au local** — ne sort pas de la pièce |
| 1 liaison radio par capteur | **Bus filaire mutualisé** (RS-485) → un seul émetteur radio par local |
| Lien dégradé / perte de données | **Saut court adaptatif + tampon FIFO** → zéro perte si le lien tombe |
| Câblage lourd | Pas de câble supplémentaire entre module d'acquisition et passerelle |
 
Les choix techniques structurants (protocole RS-485/Modbus, plateforme nRF52, sélection des capteurs, dimensionnement du FIFO et du secours) sont justifiés en détail dans **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**.
 
---
 
## Architecture du système
 
```
┌──────────────────────────────────────────────────────────────────────────┐
│  COUCHE CAPTEURS (filaire, isolée)            COUCHE RÉSEAU                 │
│                                                                            │
│   [S-THP-01A]──┐                                                           │
│   [XY-MD04]────┤  RS-485 daisy-chain    ┌───────────┐  BLE   ┌──────────┐  │
│   [PT100 ×2]───┼───── Modbus RTU ───────│ nRF52832  │═══════▶│  ESP32   │  │
│   (PTA8C04)    │                        │ Maître    │  (NUS) │ WT32-    │  │
│                                         │ + FIFO    │        │ ETH01    │  │
│                                         └───────────┘        └────┬─────┘  │
└───────────────────────────────────────────────────────────────── │ ───────┘
                                                          HTTP POST  │ Ethernet
                                                                     ▼
                                              ┌────────────────────────────┐
                                              │  Serveur — FastAPI + SQLite │
                                              │  /ingest · JWT · seuils ·   │
                                              │  alarmes · email · dashboard│
                                              └────────────────────────────┘
```
 
**Flux de données :**
 
1. Le **nRF52832** interroge les capteurs en Modbus RTU toutes les 60 s (découverte automatique des capteurs présents).
2. Chaque mesure est poussée dans un **FIFO circulaire** en RAM (tampon anti-coupure).
3. Le FIFO est vidé vers l'**ESP32** via BLE (Nordic UART Service) sous forme d'un **JSON groupé** : `{"ts":2135,"d":[{"sid":1,"t":225,"h":411,"p":9948}, ...]}`.
4. L'**ESP32** éclate le JSON par capteur et le relaie en **HTTP POST** vers le serveur (file d'attente locale si le serveur est injoignable).
5. Le **serveur** enregistre la mesure (SQLite), évalue les seuils, déclenche les alarmes et notifie par email.
6. Le **dashboard** affiche les données et les alarmes en temps réel.
Schéma matériel complet (câblage 4 fils, bornier Wago, alimentation) : **[docs/HARDWARE.md](docs/HARDWARE.md)**.
 
---
 
## Mise en œuvre — ordre de montage
 
Suivre les étapes dans l'ordre : on prépare les capteurs, on câble, on flashe, puis on lance le serveur.
 
### Étape 1 — Préparer chaque capteur individuellement
 
Avant de connecter plusieurs capteurs sur le même bus, vérifier que chacun fonctionne seul, puis lui attribuer une adresse Slave ID unique (1–247) pour éviter les conflits.
 
Dépôt **[modbus-rs485-sensor-utils](https://github.com/AngeEcam/modbus-rs485-sensor-utils)** : scripts prêts à l'emploi par capteur (dossier `sensors/`).
Exemple : `set_slave_ID_sthp01a.py` pour adresser un S-THP-01A.
 
> Adressage retenu dans ce projet : **S-THP-01A → 1**, **XY-MD04 → 2**, **module PT100 (PTA8C04) → 3**.
 
### Étape 2 — Câbler le bus et l'alimentation
 
Raccorder les capteurs en **daisy-chain** sur le bus RS-485 4 fils (VCC / GND / A+ / B−) à l'aide de connecteurs Wago, et mettre en place la chaîne d'alimentation 230 V → mini-UPS → boost/buck.
 
Procédure complète, schémas et nomenclature : **[docs/HARDWARE.md](docs/HARDWARE.md)**.
 
### Étape 3 — Flasher le module d'acquisition (nRF52)
 
Compiler et flasher le firmware Zephyr sur le nRF52 DK.
 
Dépôt **[hmra-nrf52-firmware](https://github.com/AngeEcam/hmra-nrf52-firmware)** (`west build` / `west flash`).
Le firmware découvre seul les capteurs présents — aucune configuration supplémentaire.
 
### Étape 4 — Flasher la passerelle (ESP32)
 
Renseigner l'URL du serveur dans le firmware, puis téléverser via l'Arduino IDE.
 
Dépôt **[hmra-esp32-gateway](https://github.com/AngeEcam/hmra-esp32-gateway)** (dossier `esp32_gateway/`).
 
### Étape 5 — Lancer le serveur et ouvrir le dashboard
 
Installer les dépendances, créer le fichier `.env`, démarrer le serveur.
 
Dépôt **[hmra-monitoring-server](https://github.com/AngeEcam/hmra-monitoring-server)** :
```bash
pip install -r requirements.txt
# créer un .env avec au minimum SECRET_KEY
python main.py        # http://<serveur>:8080/dashboard
```
 
### Étape 6 — Vérifier la chaîne de bout en bout
 
Les mesures doivent apparaître sur le dashboard toutes les 60 s. Couper brièvement le réseau pour vérifier que rien n'est perdu (FIFO côté nRF + file hors-ligne côté ESP32, rejouées au retour).
 
---
 
## Documentation détaillée
 
| Document | Contenu |
|---|---|
| **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** | Choix d'architecture, protocoles, FIFO, formats de trames BLE/HTTP, sécurité, dimensionnement |
| **[docs/HARDWARE.md](docs/HARDWARE.md)** | Nomenclature, câblage RS-485 4 fils, bornier Wago, chaîne d'alimentation et secours |
| **[docs/VALIDATION.md](docs/VALIDATION.md)** | Protocole de validation vs Ebro EBI 310, résultats congélateur & local technique |
 
---
 
## Résultats de validation
 
Le système a été confronté au logger certifié **Ebro EBI 310**, dans la même enceinte, sur le même cycle de 60 s, dans deux environnements thermiques contrastés (congélateur ≈ −18 °C ; local technique ≈ 24 °C). Le capteur **S-THP-01A** s'est révélé le plus juste (≈ 0,32 °C d'erreur propre en sondes co-localisées). Les écarts sont principalement **systématiques**, donc corrigeables par un simple **étalonnage par offset**.
 
Détails et graphiques : **[docs/VALIDATION.md](docs/VALIDATION.md)** (les analyses et figures sont produites par `analyse_capteurs.py` dans le dépôt serveur).
 
---
 
## Limites & perspectives
 
- **HTTP simple** — flux non chiffré, acceptable sur le LAN isolé du pilote ; **HTTPS requis** en infrastructure partagée.
- **Dashboard monolithique** — tout le JS dans un seul HTML : à découper pour un déploiement à plus grande échelle.
- **Adresse Modbus sur 1 octet** (247 max/bus) — verrou pour le multi-passerelles ; piste : identifiant à deux niveaux `(passerelle, esclave)`.
- **Topologie en bus (daisy-chain)** — câblage en ligne avec deux terminaisons ; piste : répartiteur actif (hub) pour ramifier en branches indépendantes.
- **FIFO en RAM** — effacé à la perte d'alimentation ; piste : buffer en **flash non-volatile** pour persister au-delà de l'UPS et viser la cible de 24 h.
---
 
## Auteur
 
**Ange Simpalingabo** — TFE 2025–2026, Haute École ICHEC–ECAM–ISFSC, en collaboration avec le Service Biomédical de l'HMRA.
