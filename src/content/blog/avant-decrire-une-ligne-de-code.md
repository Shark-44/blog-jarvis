---
title: "Avant d'écrire une ligne de code — les User Stories de GEMMA"
description: "Comment j'ai appliqué une méthode de développement produit à un projet de maison intelligente. Backlog, user stories, formule de scoring : la conception avant le code."
pubDate: 2026-04-27
tags: ["méthode", "user-stories", "agile", "gemma", "home-assistant", "conception"]
---

Il y a une habitude que j'ai prise pendant ma formation : ne jamais ouvrir un éditeur de code sans avoir d'abord compris ce qu'on cherche à résoudre. Pas techniquement — humainement.

Pour GEMMA, j'aurais pu commencer directement par AppDaemon et écrire des automatisations. J'ai fait autre chose. J'ai ouvert un document et j'ai écrit des user stories.

## Pourquoi des user stories pour une maison ?

Les user stories viennent du développement logiciel. Elles formalisent un besoin du point de vue de l'utilisateur, pas de la machine. Le format est simple : *En tant que... je veux... afin de...*

Ce format force à se poser la bonne question : **quel problème est-ce qu'on résout vraiment ?**

Une automatisation classique pense en événements : "la TV s'allume → j'active la scène cinéma". Une user story pense en besoin : "en tant qu'humain qui regarde la télévision, je veux que la luminosité s'adapte selon l'heure et la météo, afin d'éviter les reflets et la fatigue visuelle."

La différence semble subtile. Elle change tout. La première solution est fragile — elle casse dès que la réalité ne correspond pas au scénario. La deuxième est un objectif — le système trouve comment l'atteindre selon le contexte.

## La hiérarchie des états GEMMA

Avant d'écrire la moindre user story, j'ai modélisé les états possibles de la maison. Pas les états des appareils — les états de l'humain qui y vit.

| Niveau 0 | Niveau 1 | Niveau 2 | Niveau 3 |
|----------|----------|----------|----------|
| MAISON | ABSENT | Veille | Attend arrivée |
| | PRÉSENT | SOMMEIL | Endormissement / Profond / Besoin / Réveil |
| | | NUTRITIF | Préparation rapide / longue / Dégustation |
| | | ENTRETIEN | Ménage / Réparation |
| | | REPOS | TV / Musique / Jeu vidéo / Réception |

Cette hiérarchie est le squelette du système. Chaque état de niveau 3 est une situation de vie réelle, avec ses propres besoins de confort. C'est à partir de ces états que les user stories ont été écrites.

## Extraits du backlog — v1.0

Voici quelques user stories représentatives, organisées par thème.

### Présence

| ID | En tant que... | Je veux... | Afin de... |
|----|----------------|------------|------------|
| US-01 | Humain rentrant au foyer | que la maison détecte mon arrivée via plusieurs signaux combinés | éviter de dépendre uniquement du smartphone |
| US-03 | Humain ayant oublié son smartphone | que la maison détecte tout de même ma présence via d'autres capteurs | ne pas perturber le confort du foyer |

US-03 est particulièrement intéressante : elle force à ne jamais reposer sur un seul signal. Un système robuste se construit sur la redondance des sources.

### Sommeil

| ID | En tant que... | Je veux... | Afin de... |
|----|----------------|------------|------------|
| US-06 | Humain se levant la nuit pour un besoin naturel | que la maison allume une lumière douce et guidée | ne pas être agressé par une lumière forte et faciliter le retour au sommeil |
| US-07 | Humain ayant du mal à se rendormir | que la maison reconnaisse cet état intermédiaire entre sommeil et activité | adapter l'ambiance pour favoriser le retour au sommeil |

Ces deux user stories ne sont pas encore implémentées — elles dépendent du capteur de présence Aqara FP2 qui arrive prochainement. Mais elles existent dans le backlog. Le système est conçu pour les accueillir.

### Scalabilité — les user stories qui pensent long terme

| ID | En tant que... | Je veux... | Afin de... |
|----|----------------|------------|------------|
| US-22 | Humain changeant de maison | que mon profil de confort soit portable et réutilisable | ne pas reconfigurer entièrement le système |
| US-23 | Humain dont le mode de vie évolue | que le modèle s'adapte progressivement sans tout recasser | avoir un système vivant qui suit mon évolution |

US-22 et US-23 ne concernent pas une fonctionnalité. Elles concernent l'architecture. Elles ont directement influencé la décision de séparer l'inventaire physique (`index.yaml`) des comportements (`comportements.yaml`) et des poids (`poids.yaml`). Changer de maison = changer l'index. Le reste suit.

## La formule de scoring

Au cœur de GEMMA, il y a une formule simple :

```
État = Σ ( signal × poids × contexte )
```

| Terme | Définition | Exemple |
|-------|------------|---------|
| Signal | Valeur brute du capteur | TV allumée = 1 |
| Poids | Importance du signal | Ampli = 0.8 |
| Contexte | Modificateur situationnel | 23h = ×1.2 pour sommeil |
| Seuil | Déclenchement de l'action | > 0.8 = action automatique |

Cette formule permet au système de **peser l'évidence** plutôt que de chercher une certitude. Si plusieurs signaux pointent vers le même état avec des poids élevés, l'état est confirmé. Si aucun état ne dépasse le seuil, le système reste en `INDÉTERMINÉ` et n'agit pas.

C'est un principe emprunté à la théorie des systèmes de contrôle : en situation ambiguë, on ne force pas une sortie.

## Ce que ce backlog m'a appris

Écrire ces user stories avant de coder a eu trois effets concrets :

**Il a révélé des cas limites invisibles.** US-07 — l'humain qui a du mal à se rendormir — n'aurait jamais émergé en pensant uniquement aux capteurs disponibles. C'est en pensant au vécu de l'occupant qu'on découvre les vrais besoins.

**Il a structuré les priorités.** Le backlog distingue ce qui est implémenté aujourd'hui de ce qui attend le bon matériel ou le bon budget. Pas de dette technique cachée — juste un backlog honnête.

**Il a guidé les choix d'architecture.** Chaque décision technique — AppDaemon plutôt que YAML, séparation index/comportements, formule de scoring — trouve sa justification dans une user story. Le code n'est pas une fin en soi. Il répond à un besoin.

---

Le backlog complet, les états GEMMA détaillés et l'implémentation AppDaemon feront l'objet des prochains articles. Le système tourne déjà — mais il attend encore deux capteurs pour être complet. On en reparle dans quelques semaines.
