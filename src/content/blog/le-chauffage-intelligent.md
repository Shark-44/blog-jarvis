---
title: "Le chauffage : quand JARVIS apprend l'inertie de ta maison"
description: "Après la lumière, JARVIS étend son cerveau au chauffage. Même philosophie : comprendre le contexte plutôt que réagir à des règles. Et cette fois, le système apprend."
pubDate: 2026-06-07
tags: ["jarvis", "gemma", "chauffage", "thermique", "home-assistant", "appdaemon", "anticipation"]
---

JARVIS pilote aujourd'hui l'éclairage de façon autonome. Il croise l'état humain détecté par GEMMA, la localisation dans la maison et les conditions météo pour décider quand et comment allumer une pièce.

L'étape suivante était évidente : le chauffage.

Pas pour automatiser à la place de l'humain. Mais pour rendre la maison capable d'anticiper ses besoins thermiques avec la même logique contextuelle.

## La même philosophie, un nouveau domaine

Ce qui fonctionne pour la lumière fonctionne pour la chaleur.

GEMMA produit des états : `SOMMEIL`, `ACTIF_BUREAU`, `REPOS_TV`, `ABSENT`...

Ces états ne décrivent pas seulement ce que tu fais. Ils décrivent ce dont tu as besoin.

Une pièce où quelqu'un dort n'a pas besoin de la même température qu'une pièce où quelqu'un travaille. Une cuisine en pleine activité génère de la chaleur. Un salon vide n'a aucune raison d'être chauffé à 21°C.

Le principe est donc simple :

```text
État GEMMA + pièce occupée → consigne thermique adaptée
```

Un nouveau composant, `thermostat_manager.py`, consomme exactement les mêmes sources que le moteur d'éclairage — les états GEMMA et la localisation spatiale — mais prend des décisions thermiques au lieu de décisions lumineuses.

## L'humain reste décisionnaire

Le chauffage, c'est de l'énergie. De l'argent. Une décision qui dépend de la saison, du budget, des habitudes.

JARVIS n'active pas le chauffage de sa propre initiative.

Un simple `input_boolean` dans Home Assistant joue le rôle de gardien. Tant qu'il est à `off`, le thermostat_manager se suspend complètement. L'humain décide d'activer le système. JARVIS décide ensuite *comment* il se comporte.

C'est une distinction fondamentale.

La domotique qui fait "tout toute seule" finit par agacer. La domotique qui amplifie intelligemment une intention humaine, elle, devient invisible — dans le bon sens du terme.

## Des consignes par état, pas par horaire

La plupart des thermostats programmables raisonnent en tranches horaires.

> De 6h à 8h : 20°C. De 8h à 18h : 17°C. De 18h à 22h : 21°C.

Ce modèle a un défaut : il ne sait rien de ce qui se passe réellement.

JARVIS raisonne différemment. La configuration associe une consigne à chaque état humain :

```yaml
consignes_par_etat:
  ABSENT:        15
  SOMMEIL:       17
  LECTURE_LIT:   19
  ACTIF_BUREAU:  20
  REPOS_TV:      21
  defaut:        19
```

Si tu t'endors plus tôt que prévu, la chambre passe à 17°C. Si tu travailles tard, le bureau reste à 20°C. Si tu n'es pas là, la maison tombe à 15°C sans attendre un horaire programmé.

La consigne suit l'activité. Pas l'inverse.

## Le problème de l'inertie

La chaleur ne se déplace pas instantanément.

Un radiateur électrique allumé maintenant ne produit pas une pièce confortable dans les trente secondes. Selon l'isolation, le volume de la pièce et le type de radiateur, il peut falloir 30 minutes, 45 minutes, parfois plus d'une heure.

C'est ce qu'on appelle l'inertie thermique.

Un thermostat naïf qui réagit quand la température est trop basse arrive toujours trop tard.

JARVIS aborde ce problème en deux phases.

**Phase 1 — configuration manuelle.** Chaque pièce dispose d'une inertie initiale déclarée dans `thermostat_config.yaml`. Chambre : 45 minutes. Bureau : 30 minutes. Salon : 60 minutes. Ce sont des estimations de départ, raisonnables pour commencer.

**Phase 2 — apprentissage automatique.** Le thermostat_manager mesure le temps réel entre l'activation d'un radiateur et l'atteinte de la consigne. Il stocke ces mesures dans un fichier de calibration :

```json
{ "Chambre": 42, "Bureau": 28, "Salon": 67 }
```

Au fil du temps, JARVIS apprend l'inertie réelle de ta maison. Il n'utilise plus des valeurs supposées, mais des valeurs mesurées.

C'est un détail qui change tout : le système s'améliore en fonctionnant.

## L'anticipation au réveil

L'exemple le plus concret de ce que rend possible l'inertie calibrée, c'est l'anticipation du réveil.

L'ESP32-S3 qui gère l'alarme publie son heure de déclenchement dans Home Assistant. Le thermostat_manager consulte cette information et calcule automatiquement à quelle heure lancer le chauffage :

```text
Réveil prévu  : 07:00
Inertie chambre : 42 min (mesurée)
→ Lancement radiateur : 06:18
→ Température de confort atteinte au réveil
```

Ce n'est pas une règle programmée à la main. C'est une décision calculée à partir de données réelles.

Chaque nuit, l'heure de lancement peut être différente selon l'heure d'alarme. Et plus la calibration s'affine, plus l'anticipation est précise.

## Ce que cela change concrètement

Comme pour l'éclairage, ce qui importe n'est pas le capteur ni l'actionneur.

La plaque de cuisson n'allume pas directement la hotte. Le PIR ne commande pas directement la VMC.

De la même façon, l'alarme n'allume pas directement le radiateur. L'état GEMMA ne commande pas directement la consigne.

Ces équipements réagissent à une interprétation.

La différence est fondamentale : la maison ne réagit plus à des événements isolés. Elle réagit à une compréhension de la situation.

## Ce qui arrive ensuite

Le thermostat_manager est en cours d'intégration.

Les fondations sont posées : la configuration par pièce, les consignes par état, la logique de calibration et l'anticipation via l'alarme sont définies.

Ce qui manque encore : l'arrêt automatique en été quand la température extérieure dépasse un seuil stable sur plusieurs jours. Et probablement quelques ajustements sur les consignes, parce que le confort thermique reste quelque chose de subjectif.

C'est aussi pour ça que l'humain garde la main.
