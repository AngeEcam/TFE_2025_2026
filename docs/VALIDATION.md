# Validation — protocole & résultats

Comment le système a été éprouvé face à une référence certifiée, et ce que disent les mesures.

## Sommaire

- [Protocole](#protocole)
- [Résultats — congélateur](#résultats--congélateur)
- [Résultats — local technique](#résultats--local-technique)
- [Interprétation](#interprétation)
- [Reproduire l'analyse](#reproduire-lanalyse)

---

## Protocole

Le système est comparé au logger certifié **Ebro EBI 310**, placé dans la **même enceinte**, sur le **même cycle de 60 s**, dans deux environnements thermiques contrastés :

| Environnement | Température | Intérêt |
|---|---|---|
| **Congélateur** | ≈ −18 °C | régime le plus contraignant pour les capteurs numériques |
| **Local technique** | ≈ 24 °C | conditions stables, proches de l'étalonnage usine |

Deux configurations de placement permettent de séparer deux sources d'erreur :

| Config | Placement | Ce qu'elle isole |
|---|---|---|
| **C-1** | sondes réparties dans l'enceinte | effet de **position** (gradients thermiques) |
| **C-2** | sondes **co-localisées** avec la référence | **erreur propre** du capteur |

---

## Résultats — congélateur

Biais mesuré vs Ebro EBI 310, en valeur absolue (°C). La config C-2 isole l'erreur propre du capteur.

| Capteur | C-1 · réparti (effet position) | C-2 · co-localisé (erreur propre) |
|---|---|---|
| **S-THP-01A** | 1,81 | **0,32** |
| **XY-MD04** | 0,09 | 1,89 |
| **PT100** | 1,37 | 1,89 |

➡️ Le **S-THP-01A** est le plus juste sur son erreur propre : **0,32 °C** en co-localisé.

---

## Résultats — local technique

En conditions stables (≈ 24 °C), les écarts se resserrent ; l'effet d'**inertie thermique** des sondes et de leur boîtier devient le facteur dominant des transitoires. Le S-THP-01A confirme son comportement de référence du projet.

> Les graphiques détaillés (écarts au congélateur, température, humidité) sont versionnés dans le dépôt serveur :
> [`graph_ecarts_frigo.png`](https://github.com/AngeEcam/hmra-monitoring-server/blob/main/graph_ecarts_frigo.png),
> [`graph_temperature.png`](https://github.com/AngeEcam/hmra-monitoring-server/blob/main/graph_temperature.png),
> [`graph_humidite.png`](https://github.com/AngeEcam/hmra-monitoring-server/blob/main/graph_humidite.png).

---

## Interprétation

- Les écarts sont **principalement systématiques** (biais quasi constant), donc **corrigeables par un simple étalonnage par offset** plutôt que par un changement de capteur.
- Le **S-THP-01A** est retenu comme capteur de référence du système ; le **XY-MD04** reste pertinent comme solution économique, en gardant à l'esprit son erreur propre plus élevée à froid.
- La chaîne complète a été validée sur l'environnement le plus exigeant : **Modbus RTU → nRF52832 → BLE → ESP32 → SQLite**, sans perte de données.

---

## Reproduire l'analyse

L'analyse comparative et la génération des graphiques sont assurées par le script
[`analyse_capteurs.py`](https://github.com/AngeEcam/hmra-monitoring-server/blob/main/analyse_capteurs.py)
du dépôt **[hmra-monitoring-server](https://github.com/AngeEcam/hmra-monitoring-server)**.

Le script exploite les mesures stockées en base (table `measurements`) confrontées aux relevés de l'Ebro EBI 310, et produit les figures `graph_*.png`.
