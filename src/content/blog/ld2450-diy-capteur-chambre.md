---
title: "Capteur de présence DIY : le LD2450 comme brique de contexte spatial pour Jarvis"
description: "Utilisation d'u capteur LD2450, parametrage pour l'intégrer"
pubDate: 2026-05-14
tags: [esphome, home-assistant, ld2450, radar, diy, domotique, jarvis, gemma]
draft: true
---

## Pourquoi ce capteur dans le projet Jarvis

Jarvis repose sur un moteur de scoring contextuel — GEMMA — qui infère en permanence
l'état probable de l'occupant (SOMMEIL, REPOS_TV, NUTRITIF, ACTIF_BUREAU, ENTRETIEN,
ABSENT…) à partir d'un ensemble de signaux : heure, météo, activité des appareils,
et position dans la pièce.

C'est ce dernier point qui justifie le LD2450. Un capteur de présence binaire classique
(PIR, mmWave simple) répond à la question "y a-t-il quelqu'un dans la pièce ?". Le
LD2450 répond à une question plus précise : **où se trouve cette personne dans la
pièce, et est-elle immobile ou en mouvement ?**

Cette granularité spatiale est indispensable à GEMMA pour distinguer des états qui
coexistent dans la même pièce :

- Une personne **dans la zone lit et immobile** → signal fort vers l'état **SOMMEIL**
- Une personne **dans la zone porte** → signal de transition, potentiellement **ABSENT**
  ou changement d'état imminent
- Une personne **active dans la pièce** mais hors zones définies → signal vers
  **REPOS** ou **ENTRETIEN** selon les autres signaux

Le LD2450 n'allume pas une lampe. Il informe GEMMA du contexte spatial pour que GEMMA
prenne la bonne décision sur l'ensemble des actionneurs de la pièce.

---

## Matériel

- ESP32 (board `esp32dev`)
- Module HLK-LD2450 (radar millimétrique 24 GHz, Hi-Link)
- Câblage UART : TX ESP32 (GPIO17) → RX LD2450, RX ESP32 (GPIO16) → TX LD2450
- Alimentation 5V

![2450-2](/blog-jarvis/images/2450-2.webp)

---

## Principe de fonctionnement du LD2450

Le LD2450 est un radar millimétrique capable de suivre simultanément **jusqu'à 3 cibles**
en mouvement ou immobiles. Pour chaque cible détectée, il transmet en continu via UART :

- La position **X** (axe horizontal, en mm, négatif à gauche / positif à droite)
- La position **Y** (axe de profondeur, en mm, toujours positif depuis le capteur)
- La **vitesse** (m/s)
- La **résolution** de mesure

Le capteur est positionné en hauteur face à la pièce. L'axe Y représente la profondeur
depuis le capteur, l'axe X la largeur.

![ld2450](/blog-jarvis/images/ld2450.webp)
---

## Étape 1 — YAML initial : tester le capteur

Avant de configurer les zones, l'objectif de cette première étape est de valider que
le capteur fonctionne, que les données X/Y remontent correctement dans Home Assistant,
et de comprendre les valeurs réelles mesurées dans la pièce.

Le composant natif `ld2450` d'ESPHome expose tous les sensors bruts du radar.

```yaml
esphome:
  name: capteur-mvt-chambre
  friendly_name: Capteur mvt chambre

esp32:
  board: esp32dev
  framework:
    type: esp-idf

logger:
  baud_rate: 0

api:
  encryption:
    key: "VOTRE_CLE_API"

ota:

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "Capteur-Mvt-Chambre"
    password: "VOTRE_MOT_DE_PASSE"

captive_portal:

# LED bleue sur GPIO02
light:
  - platform: binary
    name: "Led bleu"
    output: light_output

output:
  - id: light_output
    platform: gpio
    pin: GPIO02

# LD2450 - UART
uart:
  id: uard1
  tx_pin: GPIO17
  rx_pin: GPIO16
  baud_rate: 256000
  parity: NONE
  stop_bits: 1

# LD2450 - Composant natif ESPHome
ld2450:
  id: ld2450_radar
  uart_id: uard1

binary_sensor:
  - platform: ld2450
    ld2450_id: ld2450_radar
    has_target:
      name: "Présence"
    has_moving_target:
      name: "Cible en mouvement"
    has_still_target:
      name: "Cible immobile"

sensor:
  - platform: ld2450
    ld2450_id: ld2450_radar
    target_count:
      name: "Nombre de cibles"
    target_1:
      x:
        name: "Cible 1 X"
      y:
        name: "Cible 1 Y"
      speed:
        name: "Cible 1 Vitesse"
      distance:
        name: "Cible 1 Distance"
    target_2:
      x:
        name: "Cible 2 X"
      y:
        name: "Cible 2 Y"
      speed:
        name: "Cible 2 Vitesse"
      distance:
        name: "Cible 2 Distance"
    target_3:
      x:
        name: "Cible 3 X"
      y:
        name: "Cible 3 Y"
      speed:
        name: "Cible 3 Vitesse"
      distance:
        name: "Cible 3 Distance"

switch:
  - platform: ld2450
    ld2450_id: ld2450_radar
    bluetooth:
      name: "Bluetooth LD2450"
    multi_target:
      name: "Suivi multi-cibles"

number:
  - platform: ld2450
    ld2450_id: ld2450_radar
    presence_timeout:
      name: "Timeout présence"
    zone_1:
      x1:
        name: "Zone 1 X1"
      y1:
        name: "Zone 1 Y1"
      x2:
        name: "Zone 1 X2"
      y2:
        name: "Zone 1 Y2"

button:
  - platform: ld2450
    ld2450_id: ld2450_radar
    factory_reset:
      name: "LD2450 Reset usine"
    restart:
      name: "LD2450 Redémarrer"

text_sensor:
  - platform: ld2450
    ld2450_id: ld2450_radar
    version:
      name: "LD2450 Firmware"
    mac_address:
      name: "LD2450 BT MAC"
    target_1:
      direction:
        name: "Cible 1 Direction"

select:
  - platform: ld2450
    ld2450_id: ld2450_radar
    baud_rate:
      name: "Baud rate"
    zone_type:
      name: "Type de zone"
```

### Ce que ce YAML apporte

Après le flash et l'intégration dans Home Assistant, on dispose de :

- Un binary sensor **Présence** global (true/false)
- Les coordonnées **X/Y en temps réel** de chaque cible dans HA
- Les sliders **Zone 1 X1/X2/Y1/Y2** pour définir le périmètre de détection
- Le sélecteur **Type de zone** pour passer en mode `détection` ou `filtrage`

### Calibration de la Zone 1 — périmètre de la pièce

Cette étape est essentielle : se positionner physiquement aux endroits clés de la
pièce (murs, emplacement du lit, de la porte) et relever les valeurs X/Y affichées
en temps réel dans HA. Ces coordonnées serviront de base pour définir Z2 et Z3.

| Paramètre | Valeur  | Signification                        |
|-----------|---------|--------------------------------------|
| X1        | +1850mm | Limite droite (mur droit)            |
| X2        | -1850mm | Limite gauche (mur gauche)           |
| Y1        | 0mm     | Origine (position du capteur)        |
| Y2        | 3400mm  | Limite de profondeur (mur du fond)   |

Mode sélectionné : **détection** — seules les cibles dans cette zone sont reportées.


---

## Étape 2 — Comprendre les zones et leur logique

### Les deux systèmes de zones à ne pas confondre

**Zones hardware (firmware LD2450)** — configurables via les sliders dans HA, traitées
directement par la puce radar. Elles filtrent la détection à un périmètre physique
mais ne produisent pas de binary sensor utilisable dans les automatisations ou par GEMMA.

**Zones logicielles (template ESPHome)** — des binary sensors calculés dans le firmware
de l'ESP32 en comparant les coordonnées X/Y de chaque cible à des limites définies dans
le YAML. Ce sont ces sensors qui remontent dans HA et alimentent GEMMA.

### Découpage adopté pour la chambre

**Z1 — Périmètre de la pièce** (zone hardware LD2450)
Filtre global appliqué directement par le firmware du radar. Tout ce qui est hors Z1
est ignoré avant même d'atteindre ESPHome. Isole la chambre du reste du logement.

**Z2 — Zone lit** (binary sensor logiciel → signal GEMMA)
Correspond à l'emprise physique du lit. Quand `Présence lit = true`, GEMMA reçoit
un signal contextuel fort indiquant que l'occupant est probablement couché. Combiné
à l'heure et à l'absence de mouvement, ce signal oriente le scoring vers l'état
**SOMMEIL**.

**Z3 — Zone porte** (binary sensor logiciel → signal GEMMA)
Correspond à la zone de passage devant la porte de sortie. Le pattern
`Présence porte = true` puis `false` est interprété par GEMMA comme un signal de
transition — potentiellement un départ vers l'état **ABSENT**.


### Coordonnées des zones

**Z1 — Pièce entière**

| Paramètre | Valeur   |
|-----------|----------|
| X min     | -1850 mm |
| X max     | +1850 mm |
| Y min     | 0 mm     |
| Y max     | 3400 mm  |

**Z2 — Lit (160cm × 200cm)**

| Paramètre | Valeur   |
|-----------|----------|
| X min     | -1850 mm |
| X max     | +150 mm  |
| Y min     | 600 mm   |
| Y max     | 2200 mm  |

**Z3 — Porte (~1m²)**

| Paramètre | Valeur   |
|-----------|----------|
| X min     | +850 mm  |
| X max     | +1850 mm |
| Y min     | 2400 mm  |
| Y max     | 3400 mm  |

### Pourquoi coder les zones en dur dans le YAML

Dans Jarvis, chaque capteur est lié à une pièce précise avec une géographie fixe.
Le lit ne se déplace pas, la porte non plus. Coder les coordonnées en dur présente
deux avantages : le YAML est auto-documenté, et HA ne reçoit que les signaux utiles
sans être pollué par des sliders de configuration. En contrepartie, une modification
nécessite un reflash — acceptable puisque la géographie de la pièce est stable.

---

## Étape 3 — Le timeout : stabiliser les signaux vers GEMMA

### Problème sans timeout

Une personne parfaitement immobile peut momentanément ne plus être détectée par le
radar. Sans timeout, `Présence lit` peut passer brièvement à `false` pendant le
sommeil, produire un signal contradictoire dans GEMMA et déclencher une transition
d'état incorrecte. Ce phénomène s'appelle un **rebond fantôme**.

Un moteur contextuel est aussi fiable que ses signaux d'entrée. Stabiliser les
signaux du LD2450 avant qu'ils atteignent GEMMA est donc critique.

### Solution : `delayed_off`

Le filtre `delayed_off` d'ESPHome retarde uniquement le passage à `false` :

- Passage à `true` → publication immédiate dans HA
- Passage à `false` → attente N ms avant publication
- Si retour à `true` pendant le délai → le `false` est annulé

Le signal ne passe à `false` dans GEMMA que si l'absence est confirmée pendant
toute la durée du timeout.

### Valeurs retenues

| Zone     | Timeout         | Justification                                                              |
|----------|-----------------|----------------------------------------------------------------------------|
| Z2 lit   | 60 000 ms (60s) | Absorbe les pertes de signal pendant le sommeil sans fausse transition GEMMA |
| Z3 porte | 5 000 ms (5s)   | Confirme la sortie sans bloquer la détection d'un retour immédiat          |

---

## Étape 4 — YAML final

```yaml
esphome:
  name: capteur-mvt-chambre
  friendly_name: Capteur mvt chambre

esp32:
  board: esp32dev
  framework:
    type: esp-idf

logger:
  baud_rate: 0

api:
  encryption:
    key: "VOTRE_CLE_API"

ota:

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "Capteur-Mvt-Chambre"
    password: "VOTRE_MOT_DE_PASSE"

captive_portal:

light:
  - platform: binary
    name: "Led bleu"
    output: light_output

output:
  - id: light_output
    platform: gpio
    pin: GPIO02

uart:
  id: uart_ld2450
  tx_pin: GPIO17
  rx_pin: GPIO16
  baud_rate: 256000
  parity: NONE
  stop_bits: 1

ld2450:
  id: ld2450_radar
  uart_id: uart_ld2450

# -----------------------------------------------------------------------------
# ZONES - Définition des coordonnées (en mm)
#
# Z1 - Pièce entière (filtre hardware, configuré dans HA lors de l'étape 1)
#   X : -1850 à +1850 / Y : 0 à 3400
#
# Z2 - Lit (1.60m x 2.00m) → signal GEMMA état SOMMEIL
#   X : -1850 à +150 / Y : 600 à 2200
#
# Z3 - Porte (~1m²) → signal GEMMA transition ABSENT
#   X : +850 à +1850 / Y : 2400 à 3400
# -----------------------------------------------------------------------------
globals:
  - id: z2_x_min
    type: float
    restore_value: no
    initial_value: '-1850'
  - id: z2_x_max
    type: float
    restore_value: no
    initial_value: '150'
  - id: z2_y_min
    type: float
    restore_value: no
    initial_value: '600'
  - id: z2_y_max
    type: float
    restore_value: no
    initial_value: '2200'
  - id: z3_x_min
    type: float
    restore_value: no
    initial_value: '850'
  - id: z3_x_max
    type: float
    restore_value: no
    initial_value: '1850'
  - id: z3_y_min
    type: float
    restore_value: no
    initial_value: '2400'
  - id: z3_y_max
    type: float
    restore_value: no
    initial_value: '3400'

binary_sensor:
  - platform: ld2450
    ld2450_id: ld2450_radar
    has_target:
      name: "Présence"
    has_moving_target:
      name: "Cible en mouvement"
    has_still_target:
      name: "Cible immobile"

  # Z2 - Présence lit → signal GEMMA état SOMMEIL
  # Timeout 60s : absorbe les pertes de signal pendant le sommeil
  - platform: template
    name: "Présence lit"
    device_class: occupancy
    filters:
      - delayed_off: 60000ms
    lambda: |-
      auto targets = id(ld2450_radar).get_targets();
      for (auto* target : targets) {
        if (!target->is_present()) continue;
        float x = target->get_x();
        float y = target->get_y();
        if (x >= id(z2_x_min) && x <= id(z2_x_max) &&
            y >= id(z2_y_min) && y <= id(z2_y_max)) {
          return true;
        }
      }
      return false;

  # Z3 - Présence porte → signal GEMMA transition ABSENT
  # Timeout 5s : confirmation rapide de la sortie
  - platform: template
    name: "Présence porte"
    device_class: occupancy
    filters:
      - delayed_off: 5000ms
    lambda: |-
      auto targets = id(ld2450_radar).get_targets();
      for (auto* target : targets) {
        if (!target->is_present()) continue;
        float x = target->get_x();
        float y = target->get_y();
        if (x >= id(z3_x_min) && x <= id(z3_x_max) &&
            y >= id(z3_y_min) && y <= id(z3_y_max)) {
          return true;
        }
      }
      return false;

sensor:
  - platform: ld2450
    ld2450_id: ld2450_radar
    target_count:
      name: "Nombre de cibles"
    still_target_count:
      name: "Cibles immobiles"
    moving_target_count:
      name: "Cibles en mouvement"
    target_1:
      x:
        name: "Cible 1 X"
      y:
        name: "Cible 1 Y"
      speed:
        name: "Cible 1 Vitesse"
      angle:
        name: "Cible 1 Angle"
      distance:
        name: "Cible 1 Distance"
      resolution:
        name: "Cible 1 Résolution"
    target_2:
      x:
        name: "Cible 2 X"
      y:
        name: "Cible 2 Y"
      speed:
        name: "Cible 2 Vitesse"
      angle:
        name: "Cible 2 Angle"
      distance:
        name: "Cible 2 Distance"
      resolution:
        name: "Cible 2 Résolution"
    target_3:
      x:
        name: "Cible 3 X"
      y:
        name: "Cible 3 Y"
      speed:
        name: "Cible 3 Vitesse"
      angle:
        name: "Cible 3 Angle"
      distance:
        name: "Cible 3 Distance"
      resolution:
        name: "Cible 3 Résolution"

switch:
  - platform: ld2450
    ld2450_id: ld2450_radar
    bluetooth:
      name: "Bluetooth LD2450"
    multi_target:
      name: "Suivi multi-cibles"

number:
  - platform: ld2450
    ld2450_id: ld2450_radar
    presence_timeout:
      name: "Timeout présence"
    zone_1:
      x1:
        name: "Zone 1 X1"
      y1:
        name: "Zone 1 Y1"
      x2:
        name: "Zone 1 X2"
      y2:
        name: "Zone 1 Y2"
    zone_2:
      x1:
        name: "Zone 2 X1"
      y1:
        name: "Zone 2 Y1"
      x2:
        name: "Zone 2 X2"
      y2:
        name: "Zone 2 Y2"
    zone_3:
      x1:
        name: "Zone 3 X1"
      y1:
        name: "Zone 3 Y1"
      x2:
        name: "Zone 3 X2"
      y2:
        name: "Zone 3 Y2"

button:
  - platform: ld2450
    ld2450_id: ld2450_radar
    factory_reset:
      name: "LD2450 Reset usine"
    restart:
      name: "LD2450 Redémarrer"

text_sensor:
  - platform: ld2450
    ld2450_id: ld2450_radar
    version:
      name: "LD2450 Firmware"
    mac_address:
      name: "LD2450 BT MAC"
    target_1:
      direction:
        name: "Cible 1 Direction"
    target_2:
      direction:
        name: "Cible 2 Direction"
    target_3:
      direction:
        name: "Cible 3 Direction"

select:
  - platform: ld2450
    ld2450_id: ld2450_radar
    baud_rate:
      name: "Baud rate"
    zone_type:
      name: "Type de zone"
```

---

## Résultat dans Home Assistant

| Entité | Type | Rôle dans Jarvis |
|--------|------|-----------------|
| `binary_sensor.presence` | Binary sensor | Présence globale Z1 |
| `binary_sensor.presence_lit` | Binary sensor | Signal GEMMA → état SOMMEIL |
| `binary_sensor.presence_porte` | Binary sensor | Signal GEMMA → transition ABSENT |
| `sensor.cible_N_x` / `_y` | Sensor | Coordonnées brutes pour debug |

> **Image à ajouter** : capture des entités dans HA

---

## Intégration dans GEMMA

Les binary sensors produits par ce capteur sont des **signaux d'entrée** du moteur
de scoring GEMMA, au même titre que l'heure ou la météo. Ils n'agissent pas
directement sur les actionneurs — ils influencent le calcul du score de chaque état.

| Signal                              | État GEMMA favorisé | Condition associée                           |
|-------------------------------------|---------------------|----------------------------------------------|
| `Présence lit = true`               | **SOMMEIL**         | Combiné à heure tardive, absence de mouvement |
| `Présence lit = false`              | Sortie de SOMMEIL   | Transition vers REPOS ou ACTIF_BUREAU        |
| `Présence porte = true → false`     | **ABSENT**          | Combiné à absence de présence globale        |

Le `signal_reader.py` de Jarvis lit ces entités HA et les transmet au moteur de
scoring. Le `lieu_resolver.py` utilise la combinaison des zones actives pour
résoudre la position probable de l'occupant dans le logement.

> Pour comprendre la logique de pondération des signaux, se référer aux fichiers
> `poids_v0.x.yaml` et à la documentation `moteur_scoring` du projet Jarvis.

---

## Références

- [Composant LD2450 ESPHome](https://esphome.io/components/sensor/ld2450.html)
- [GitHub ArminasTV/esphome](https://github.com/ArminasTV/esphome) — inspiration
  pour la gestion des zones logicielles
- [GitHub Shark-44/projet_jarvis](https://github.com/Shark-44/projet_jarvis) — projet Jarvis
- Datasheet HLK-LD2450 — Hi-Link
