---
title: "La cuisine : quand un simple PIR devient un modèle d'activité"
description: "Pourquoi la cuisine est l'exemple parfait pour comprendre la philosophie de GEMMA. Observer, interpréter et enrichir progressivement les états grâce à de nouveaux capteurs."
pubDate: 2026-05-31
tags: ["gemma", "cuisine", "scoring", "capteurs", "home-assistant", "contexte"]
---

Quand on découvre GEMMA pour la première fois, on pourrait penser que le système a besoin d'une maison remplie de capteurs pour fonctionner. La réalité est beaucoup plus simple.

La cuisine est probablement le meilleur exemple pour comprendre la philosophie du projet.

Aujourd'hui, ma cuisine n'est quasiment pas instrumentée. Je ne dispose ni de mesure de consommation sur la plaque de cuisson, ni de suivi du four, ni de compteur d'eau sur l'évier. Pourtant, GEMMA est déjà capable de raisonner sur cet espace.

Pourquoi ? Parce que le but n'est pas de tout mesurer. Le but est de construire progressivement une compréhension de l'activité humaine.

## Observer avec ce que l'on possède

Actuellement, l'état `NUTRITIF` repose principalement sur un détecteur de mouvement PIR.

Ce capteur ne sait pas si je prépare un repas, si je prends un café ou si je traverse simplement la pièce.

Il fournit une observation brute :

> Quelqu'un est présent dans la cuisine.

À partir de là, GEMMA utilise le temps comme information complémentaire.

Une présence de quelques secondes n'a pas la même signification qu'une présence continue pendant plusieurs minutes.

Ce n'est pas une vérité absolue.

C'est simplement la meilleure estimation possible avec les informations disponibles aujourd'hui.

## Accepter l'incertitude

L'un des principes du projet est de ne jamais prétendre savoir ce qu'on ne sait pas.

Un PIR ne permet pas de conclure avec certitude qu'un repas est en préparation.

Il permet seulement d'augmenter le poids de cette hypothèse.

C'est précisément pour cette raison que GEMMA repose sur un système de scoring :

```text
État = Σ ( signal × poids × contexte )
```

L'objectif n'est pas de trouver une certitude.

L'objectif est de peser les indices disponibles.

## Demain : enrichir le modèle

La cuisine est également l'endroit où l'évolution du système est la plus facile à illustrer.

Imaginons l'ajout progressif de nouveaux capteurs.

### Plaque de cuisson

Un module ESPHome pourrait mesurer la consommation électrique de la plaque.

Dans ce cas, le moteur disposerait d'une nouvelle observation :

> Une source de chaleur est utilisée.

Cette information aurait naturellement un poids plus important qu'un simple mouvement.

### Four

Le four apporte un second indice indépendant.

La combinaison :

- présence en cuisine
- plaque active
- four actif

renforce fortement l'hypothèse d'une préparation de repas.

### Micro-ondes

Le micro-ondes pourrait au contraire orienter vers une préparation rapide.

Tous les signaux n'ont pas la même valeur.

Tous les signaux ne racontent pas la même histoire.

## L'eau raconte aussi une histoire

La consommation d'eau est particulièrement intéressante.

Un débit observé à l'évier peut correspondre à plusieurs situations :

- préparation d'un repas ;
- lavage de légumes ;
- vaisselle ;
- ménage.

Pris seul, ce signal est ambigu.

Combiné à d'autres observations, il devient beaucoup plus utile.

C'est toute la philosophie de GEMMA :

> Un capteur isolé décrit un événement.
>
> Plusieurs capteurs décrivent une activité.

## Quand l'état devient plus important que le capteur

La conséquence la plus intéressante n'est pas l'amélioration de la détection.

C'est ce qu'elle rend possible ensuite.

Si GEMMA conclut qu'une activité de cuisine est en cours, alors la maison peut adapter son comportement.

Par exemple :

- ajuster automatiquement l'éclairage ;
- modifier la vitesse de la VMC ;
- activer une hotte domotisée ;
- différer certaines notifications non urgentes ;
- préparer d'autres automatismes liés au confort.

La plaque de cuisson n'allume pas directement la hotte.

Le PIR ne commande pas directement la VMC.

Ces équipements réagissent à un état interprété par le système.

La différence est fondamentale.

## Une architecture pensée pour évoluer

L'intérêt de cette approche est qu'elle ne dépend pas du matériel présent aujourd'hui.

Le modèle fonctionne avec peu de capteurs.

L'ajout de nouveaux équipements ne remet pas en cause l'architecture.

Il enrichit simplement la qualité de l'observation.

Le comportement associé à l'état `NUTRITIF` reste identique.

Seule la confiance accordée à cet état progresse.

## Ce que la cuisine m'apprend sur GEMMA

La cuisine illustre parfaitement la philosophie du projet.

GEMMA n'est pas un ensemble d'automatisations.

Ce n'est pas non plus une télécommande géante pour objets connectés.

C'est un moteur décisionnel dont le rôle est d'observer l'environnement, d'interpréter l'activité humaine et d'adapter la maison à cette compréhension.

Aujourd'hui, un simple PIR permet une première estimation.

Demain, la plaque de cuisson, le four, le micro-ondes, l'évier ou le lave-vaisselle viendront renforcer cette compréhension.

L'objectif n'est pas d'ajouter des capteurs pour ajouter des capteurs.

L'objectif est de rendre la maison progressivement plus pertinente dans sa manière d'accompagner ses occupants.