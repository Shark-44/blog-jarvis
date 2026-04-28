---
title: "Ma maison ne reçoit pas d'ordres"
description: "Pourquoi j'ai choisi de construire une maison qui observe et déduit, plutôt qu'une maison qui obéit. La genèse du projet GEMMA."
pubDate: 2026-04-25
tags: ["vision", "home-assistant", "appdaemon", "gemma", "domotique"]
---

La plupart des maisons intelligentes fonctionnent comme des télécommandes glorifiées. Tu dis "allume la lumière", elle allume la lumière. Tu dis "éteins le chauffage", elle éteint le chauffage. C'est pratique. Mais ce n'est pas intelligent.

Ce que je voulais construire, c'est différent.

## Une maison qui observe

Imagine rentrer chez toi après une longue journée. Tu ne veux pas dire à ta maison ce que tu ressens — tu veux qu'elle le comprenne. Que les lumières s'adaptent sans que tu y penses. Que l'ambiance soit déjà là, au bon moment, sans interaction.

C'est le principe fondateur de ce projet : **la maison ne gère pas des événements, elle atteint des objectifs de confort.**

La différence est énorme. Une maison événementielle réagit : "la TV s'est allumée → j'active la scène cinéma". Une maison orientée objectifs observe : "la TV joue, la luminosité est basse, il est 21h30 → l'occupant est probablement en mode détente → j'ajuste l'éclairage en conséquence."

## Pourquoi pas des automatisations YAML ?

C'est la première question légitime. Home Assistant permet déjà de créer des automatisations puissantes en YAML. Alors pourquoi aller plus loin ?

Les automatisations YAML sont excellentes pour des déclencheurs simples et prévisibles. Mais elles ont une limite structurelle : elles pensent en événements isolés, pas en état global. Chaque règle ignore ce que font les autres. On finit rapidement avec des dizaines d'automatisations qui se contredisent, se court-circuitent, et deviennent impossibles à maintenir.

C'est là qu'**AppDaemon** entre en jeu. Là où Home Assistant gère des entités et des événements, AppDaemon permet d'écrire de vraies applications Python qui tournent en continu, avec un état en mémoire, de la logique complexe, des boucles, des calculs. C'est la différence entre un panneau de relais et un automate programmable.

Et c'est exactement cette analogie qui a changé ma façon de penser.

## La culture automate industriel

En automatisation industrielle, on ne câble pas des capteurs directement sur des actionneurs. On passe par un **automate programmable** — un système qui lit tous les signaux d'entrée, calcule un état global, et décide des sorties en fonction d'une logique programmée.

Cette culture, je l'ai appliquée à ma maison.

Plutôt que de connecter "TV allumée → lumières tamisées", j'ai construit un cycle :

1. **Lire** tous les signaux disponibles (capteurs, médias, présence, luminosité)
2. **Calculer** un état humain global (que fait l'occupant en ce moment ?)
3. **Décider** des actions en fonction de cet état et du contexte (heure, météo)
4. **Agir** sur les actionneurs

C'est un automate. Pas de la domotique au sens classique. Un système de contrôle.

Ce changement de paradigme est la clé de tout ce qui suit.

## Le point de départ : Index Manager Card

Avant même de parler de logique ou d'intelligence, il y a une question fondamentale : **comment organiser ses entités Home Assistant de façon durable ?**

Par défaut, Home Assistant travaille avec des `entity_id` bruts. `light.salon_lamp`, `switch.bureau_prise`, etc. Ces identifiants sont liés au matériel physique. Le jour où tu changes une ampoule connectée pour une autre marque, l'`entity_id` change, et toutes tes automatisations qui la référencent sont à mettre à jour.

C'est le problème que j'ai résolu en premier avec **[Index Manager Card](https://github.com/Shark-44/Index_Manager_Card)**.

L'idée est simple : introduire une couche d'abstraction entre le matériel et la logique. Au lieu de référencer `light.salon_lamp`, on référence `Salon → Lampe`. Le mapping entre ce nom logique et l'entité physique est géré dans un fichier `index.yaml` centralisé, piloté par AppDaemon via une API REST et une carte Lovelace personnalisée.

```yaml
# index.yaml — extrait
Salon:
  Lampe:
    entity: light.salon_lamp
    type: actuator
  Tele:
    entity: media_player.65pus7304_12_2
    type: receiver
```

Si demain je remplace la lampe, je mets à jour une seule ligne dans `index.yaml`. Aucune automatisation ne change. Aucune logique ne casse.

Ce n'est pas qu'une commodité technique — c'est un principe architectural. **L'inventaire physique est séparé des comportements.** C'est cette séparation qui rend tout le reste possible.

*Un article dédié à l'Index Manager Card est en préparation, avec le détail de l'installation et de la carte Lovelace.*

## Le projet GEMMA — une adaptation, pas une révolution

GEMMA est né de cette base. Une fois que l'inventaire est propre et stable, on peut construire dessus.

GEMMA reconnaît plusieurs états humains : sommeil, repos devant la TV, écoute musicale, travail au bureau, activité en cuisine... Pour chaque état, combiné à la météo du moment, il sait comment l'environnement doit se comporter.

Ce qui me plaît dans cette architecture, c'est qu'elle est **adaptable sans toucher au code**. Ajouter un capteur ? On l'inscrit dans l'inventaire. Ajuster un comportement ? On modifie un fichier de configuration. Intégrer un nouveau type de détection ? La structure l'accueille.

Et si aucun état n'est suffisamment clair ? GEMMA ne devine pas. Il reste en `INDÉTERMINÉ` et n'agit pas. Mieux vaut ne rien faire que faire faux.

C'est aussi un principe d'automate industriel : en cas d'ambiguïté, on ne force pas une sortie. On attend un signal clair.

## Ce que ce blog va raconter

Ce n'est pas un blog de tutoriels "comment allumer une lumière avec une scène". C'est le journal de construction d'un système de contrôle pour une maison — avec l'architecture, les fichiers réels, les décisions techniques et les erreurs.

Dans les prochains articles : le détail de l'Index Manager Card, puis l'architecture complète de GEMMA — comment les signaux sont lus, comment l'état humain est calculé, et comment les actionneurs sont pilotés. Avec les fichiers, commentés, dans leur contexte.

Parce qu'une maison intelligente, ça se construit. Et ça se partage.
