---
title: "JARVIS — architecture générale : du capteur à la décision"
description: "Vue d'ensemble de l'architecture JARVIS. Comment les capteurs, les moteurs et les fichiers de configuration s'articulent pour produire une maison qui comprend son contexte plutôt que d'obéir à des règles."
pubDate: 2026-06-01
tags: ["jarvis", "gemma", "architecture", "home-assistant", "appdaemon", "overview"]
---

Avant de plonger dans les articles techniques sur chaque brique du système, il est utile
d'avoir une vision globale de l'architecture. Comment les capteurs parlent aux moteurs.
Comment les moteurs consultent les fichiers de configuration. Comment la maison finit
par agir.

Cette architecture s'est construite progressivement, mais elle répond à un principe
unique : **la maison ne réagit pas à des événements. Elle comprend un contexte.**

---

## Vue d'ensemble

![architecture-jarvis](/blog-jarvis/images/architecture-jarvis.webp)

## Couche 1 — Les capteurs

Tout commence par l'observation. Les capteurs ne prennent aucune décision — ils
publient des faits bruts dans Home Assistant.

Un radar LD2450 détecte une position X/Y dans la chambre. Un PIR signale une présence
en cuisine. La météo indique qu'il pleut. Le GPS confirme que la maison est occupée.
La télévision est en lecture.

Aucun de ces signaux n'a de sens pris isolément. C'est leur combinaison qui raconte
une histoire.

---

## Couche 2 — L'inventaire : index.yaml

Avant de traiter quoi que ce soit, JARVIS a besoin de savoir ce qui existe dans la
maison.

L'`index.yaml` est le registre de tous les équipements, organisés par pièce et par
rôle — actionneurs (ce qu'on pilote) et récepteurs (ce qui observe). Chaque
équipement y est désigné par un **identifiant logique** indépendant de son entité
Home Assistant physique.

Changer une ampoule, remplacer un capteur — une seule ligne à mettre à jour dans
l'index. Le reste du système ne sait pas qu'il y a eu un changement matériel.

---

## Couche 3 — Les lecteurs

Deux briques lisent les capteurs et normalisent leurs données.

**`signal_reader.py`** parcourt l'`index.yaml` et s'abonne à toutes les entités
simples — binary sensors, media players, switches. Il produit un snapshot normalisé
dans `sensor.jarvis_signals` : chaque signal est True/False ou une valeur brute,
avec sa pièce et son horodatage.

**`device_map_reader.py`** traite les capteurs complexes — ceux qui produisent
plusieurs valeurs liées ou qui nécessitent des calculs. Le radar LD2450 (coordonnées
X/Y, zones logicielles), le capteur météo extérieur (conversion millivolts → lux,
hystérésis sur la pluie). Il publie dans `sensor.jarvis_vectors`.

Ces deux snapshots alimentent le moteur GEMMA. Lui ne sait pas d'où viennent les
signaux — il consomme une table plate.

---

## Couche 4 — Le moteur GEMMA

C'est le cerveau du système. Toutes les 30 secondes, `moteur_gemma.py` fusionne
les deux snapshots et répond à une question :

> **Que fait la personne en ce moment ?**

Pour y répondre, il consulte `poids.yaml` — la table de vérité du système. Ce
fichier décrit chaque état humain possible (`SOMMEIL`, `ACTIF_BUREAU`, `REPOS_TV`,
`NUTRITIF`, `ABSENT`...) avec les signaux qui le confirment ou l'infirment, et leur
poids relatif.

La formule est simple :

```
Score(état) = Σ ( signal × poids × contexte )
```

L'état avec le score le plus élevé au-dessus d'un seuil est retenu. Si aucun état
ne dépasse ce seuil, le moteur publie `INDÉTERMINÉ` et n'agit pas.

**En cas de doute, JARVIS ne force pas une décision.**

---

## Couche 5 — Le moteur Scoring

Une fois l'état humain connu, `moteur_scoring.py` répond à une autre question :

> **Comment la maison doit-elle se comporter dans ce contexte ?**

Il croise trois informations :

- L'état GEMMA courant (`ACTIF_BUREAU`, `SOMMEIL`, `REPOS_TV`...)
- La condition météo actuelle (`nuit`, `pluvieux`, `nuageux`, `dégagé`)
- La présence confirmée par GPS

Et il consulte `comportements.yaml` — une matrice qui associe à chaque combinaison
état × météo les valeurs cibles de chaque actionneur.

```yaml
ACTIF_BUREAU:
  nuageux:
    Lumiere_bureau:  200
    Volets_bureau:   false
  degage_chaud:
    Lumiere_bureau:  0      # lumière naturelle suffisante
    Volets_bureau:   true
  nuit:
    Lumiere_bureau:  150
```

Le moteur ne connaît pas les capteurs. Il ne sait pas d'où vient l'état GEMMA.
Il lit une cellule dans un tableau et actionne.

---

## Couche 6 — La maison agit

Les actionneurs reçoivent leurs instructions. Lumières, volets, VMC, chauffage —
chacun réagit à un état interprété, pas à un événement isolé.

La télévision n'allume pas directement la scène cinéma.
Le PIR ne commande pas directement la lumière.
Le GPS n'allume pas directement le chauffage.

Tout passe par l'interprétation du contexte.

---

## Ce qui arrive ensuite

Deux évolutions sont en cours de conception.

**Le chauffage** — `thermostat_manager.py` consommera les mêmes états GEMMA pour
piloter les radiateurs avec des consignes adaptées à l'activité, en apprenant
l'inertie thermique réelle de chaque pièce.

**Les notifications** — une couche supplémentaire au-dessus du moteur scoring,
capable de différer ou de prioriser les alertes selon le contexte. On ne dérange
pas une personne endormie pour une notification non urgente.

---

## Pourquoi cette architecture

Chaque couche a une responsabilité unique. Ajouter un capteur ne modifie pas les
moteurs. Ajuster un comportement ne touche pas aux lecteurs. Changer le matériel ne
casse pas la logique.

C'est le même principe qu'un automate industriel : les entrées/sorties physiques
sont déclarées dans une table de configuration. La logique ne les connaît pas
directement.

JARVIS n'est pas un ensemble d'automatisations. C'est un système de contrôle.
