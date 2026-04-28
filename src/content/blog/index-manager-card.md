---
title: "Index Manager Card — l'inventaire qui change tout"
description: "Avant d'automatiser quoi que ce soit, il faut savoir ce qui existe. Voici comment j'ai structuré l'inventaire physique de ma maison avec AppDaemon et une carte Lovelace personnalisée."
pubDate: 2026-04-24
tags: ["home-assistant", "appdaemon", "index-manager-card", "yaml", "lovelace"]
---

Dans l'article précédent, j'expliquais pourquoi ma maison ne reçoit pas d'ordres. Avant d'arriver à cette logique, il y a une étape que personne ne mentionne jamais : **savoir ce qui existe**.

Pas dans l'interface Home Assistant. Dans un fichier structuré, stable, indépendant du matériel. Un inventaire.

## Le problème des entity_id bruts

Par défaut, Home Assistant identifie chaque appareil par un `entity_id` généré automatiquement : `light.hue_color_light_2`, `binary_sensor.0xa4c13864ce8b58f6_occupancy`, `switch.0xa4c13809f96ca9a9`...

Ces identifiants sont liés au matériel physique. Le jour où vous remplacez une ampoule Hue par une autre, l'`entity_id` change. Toutes vos automatisations qui la référencent sont à corriger. Un par un.

Ce n'est pas un problème quand on a dix entités. Ça devient ingérable à cinquante.

La solution : introduire une **couche d'abstraction**. On ne référence plus jamais une entité physique directement. On référence un nom logique — `Eclairage_bureau`, `Tele`, `Capteur_presence_cuisine` — et c'est un fichier central qui fait le lien entre ce nom et l'entité réelle.

C'est le rôle de l'`index.yaml`.

## L'index.yaml — l'inventaire physique de la maison

Ce fichier vit à la racine de Home Assistant, au même niveau que `configuration.yaml`. Il déclare chaque pièce, et pour chaque pièce, ses actionneurs (ce qu'on pilote) et ses récepteurs (ce qui observe).

```yaml
pieces:
  Salon:
    actionneurs:
      - id: Eclairage_Salon_tv
        entity: light.salon_tv
        type: light
      - id: Wled_Salon
        entity: light.wled
        type: light
      - id: Tele
        entity: media_player.65pus7304_12_2
        type: media_player
    recepteurs:
      - id: Capteur luminosité
        entity: sensor.0xa4c138ff0ecb6105_illuminance
        type: sensor
      - id: Ampli
        entity: binary_sensor.192_168_3_12
        type: binary_sensor

  Bureau:
    actionneurs:
      - id: Eclairage_bureau
        entity: light.hue_color_light_2
        type: light
      - id: Spot_bureau
        entity: light.hue_color_lamp_1
        type: light
    recepteurs:
      - id: Capteur_presence_bureau
        entity: binary_sensor.0xa4c13864ce8b58f6_occupancy
        type: binary_sensor
```

La structure est volontairement simple. Chaque entrée a trois champs : un `id` lisible par un humain, une `entity` physique Home Assistant, et un `type`. C'est tout.

**Ce fichier est un inventaire, pas une logique.** Il dit ce qui existe, pas comment ça se comporte. Cette séparation est fondamentale — la logique viendra plus tard, dans d'autres fichiers.

## AppDaemon — pourquoi il est indispensable ici

Pour lire cet `index.yaml` et exposer son contenu au reste du système, j'utilise **AppDaemon**. Mais avant d'aller plus loin, une précision importante selon votre installation.

### Les trois façons d'installer Home Assistant

**Home Assistant OS** — l'installation recommandée. Un système complet sur une carte SD ou un SSD. AppDaemon s'installe via les add-ons officiels, en quelques clics. Les fichiers sont accessibles via l'éditeur de fichiers intégré ou en SSH.

**Home Assistant Supervised** — c'est mon installation. HA tourne sur un Debian existant, avec le superviseur officiel. AppDaemon est disponible comme add-on, mais son comportement peut différer selon la version de Debian et la configuration réseau. Les chemins de fichiers sont identiques à HA OS, mais l'accès SSH et la gestion des add-ons demande un peu plus d'attention.

**Home Assistant Core** — installation Python pure, sans superviseur. AppDaemon n'est **pas** disponible comme add-on. Il faut l'installer séparément en tant que service Python (`pip install appdaemon`), le configurer manuellement et gérer son démarrage automatique soi-même. C'est faisable, mais plus technique.

> Si vous êtes sous **Debian Supervised** comme moi : AppDaemon s'installe normalement via les add-ons, mais vérifiez que le port `5050` est accessible localement — c'est celui qu'utilise l'API REST qu'on va voir juste après.

### Ce que fait AppDaemon dans ce projet

AppDaemon fait tourner `api_index.py`, une application Python qui :

1. Lit et parse l'`index.yaml` au démarrage
2. Expose une **API REST** sur le port `5050`
3. Répond aux requêtes du frontend avec la liste des pièces et entités

```
GET http://localhost:5050/api/appdaemon/index
→ retourne le contenu de l'index.yaml en JSON
```

C'est cette API que la carte Lovelace interroge pour afficher l'inventaire.

## La carte Lovelace — Index Manager Card

La [Index Manager Card](https://github.com/Shark-44/Index_Manager_Card) est une carte Lovelace personnalisée écrite en JavaScript (LitElement). Elle s'installe dans `/config/www/` et s'ajoute en ressource dans les paramètres du dashboard.

Elle permet de :

- **Visualiser** toutes les pièces et leurs entités organisées par rôle (actionneurs / récepteurs)
- **Gérer** l'inventaire directement depuis le dashboard, sans toucher au YAML à la main
- **Redémarrer AppDaemon** depuis l'interface via une carte complémentaire (`appdaemon_restart_card.js`)

### Installation

Copiez les fichiers dans Home Assistant :

```
config/www/
├── index_manager_card.js
└── appdaemon_restart_card.js

config/appdaemon/apps/
├── api_index.py
├── demo_bureau.py
└── apps.yaml

config/
└── index.yaml
```

Puis dans Home Assistant → Paramètres → Tableaux de bord → Ressources, ajoutez :

```
/local/index_manager_card.js?v=1
/local/appdaemon_restart_card.js?v=1
```

Et dans votre dashboard Lovelace :

```yaml
type: custom:index-manager-card
```

### Le principe d'abstraction en action

C'est ici que tout prend son sens. Dans tous les fichiers Python qui viendront ensuite — `signal_reader.py`, `moteur_gemma.py`, `moteur_scoring.py` — on ne verra jamais un `entity_id` brut. On verra uniquement des identifiants logiques comme `Eclairage_Salon_tv` ou `Capteur luminosité`.

Si demain je remplace mon capteur de luminosité Zigbee par un modèle différent, une seule ligne change dans `index.yaml`. Le reste du système ne sait même pas qu'il y a eu un changement matériel.

C'est exactement le principe d'un automate industriel : **les entrées/sorties physiques sont déclarées dans une table de configuration. La logique ne les connaît pas directement.**

## Ce que cet inventaire rend possible

L'`index.yaml` est le point de départ de toute l'architecture GEMMA. Sans lui, chaque brique du système devrait connaître les entités physiques — et chaque remplacement de matériel deviendrait une opération chirurgicale sur le code.

Avec lui, le système est découplé du matériel. On peut faire évoluer l'un sans toucher à l'autre.

---

Dans le prochain article, on entre dans le vif du sujet : `signal_reader.py`, la brique qui lit cet inventaire en temps réel et produit le snapshot de la maison à chaque instant.
