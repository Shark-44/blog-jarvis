---
title: "JARVIS — Spécifications : Le Centre de Notifications Contextuel"
description: "User Story et critères d'acceptation du moteur de notifications de JARVIS. Gestion des urgences, synchronisation multi-appareils (ACK/Clear) et gestion des transitions First In / Last Out."
pubDate: 2026-06-05
tags: ["jarvis", "notifications", "architecture", "user-story", "design-pattern"]
---

Après avoir stabilisé les couches d'acquisition (GEMMA) et de décision (SCORING), la brique **Notifications** (initialement prévue comme "Feature future") nécessite sa propre modélisation. 

Fidèle à notre philosophie industrielle ("PLC thinking"), ce centre de notifications ne se contente pas d'envoyer des messages : **il distribue l'information selon l'état de la maison et synchronise la réaction des humains.**

---

## 1. L'Histoire Utilisateur (User Story)

**En tant qu'** habitant de la maison (ou compagnon),  
**Je veux que** JARVIS gère les alertes et les notifications de manière contextuelle (selon ma présence, mon absence, la météo et le niveau de danger),  
**Afin de** recevoir uniquement les informations pertinentes au bon moment, de ne pas être harcelé inutilement, et d'être alerté immédiatement en cas de risque critique pour le bâtiment.

---

## 2. La Matrice des Degrés d'Urgence

Pour éviter la fatigue informationnelle, les notifications sont classées en 3 catégories de distribution :

| Niveau | Logique de distribution | Exemple type |
| :--- | :--- | :--- |
| **`info`** | Uniquement si l'humain ciblé (ou un habitant) est **présent**. Pas de harcèlement en cas d'absence. | Cycle de machine à laver terminé, rappel litière. |
| **`important`** | Trié selon le contexte. Envoyé aux personnes présentes. Peut être masqué ou différé si l'information est redondante. | Fenêtre restée ouverte alors que le vent se lève. |
| **`critique`** | **Priorité absolue.** Transmis à tous les compagnons (présents ou absents). Outrepasse le mode "Ne pas déranger". | Détection de fuite d'eau, fumée cuisine, intrusion. |

---

## 3. Critères d'Acceptation & Scénarios Tests

### Scénario 1 : L'alerte de confort au premier arrivant (*First In*)
* **Étant donné que** la machine à laver termine son cycle alors que la maison est **vide**,
* **Quand** le premier habitant franchit la porte de la maison (la présence globale passe à `occupee`),
* **Alors** le Centre de Notifications extrait la tâche en attente et lui envoie l'alerte : *"Bienvenue ! La machine est terminée, pensez à la vider"*.
* *Résultat :* Aucun téléphone n'a été pollué à l'extérieur pendant la journée de travail.

### Scénario 2 : L'alerte de sécurité au départ (*Last Out*)
* **Étant donné qu'** une fenêtre est restée ouverte dans le salon,
* **Quand** le dernier habitant quitte la maison (la présence globale passe à `vide`),
* **Alors** JARVIS intercepte la transition et envoie immédiatement une notification `important` **uniquement** au dernier partant : *"Attention, vous partez mais la fenêtre du salon est ouverte"*.

### Scénario 3 : L'extinction d'une règle (Masquage Météo)
* **Étant donné que** le dernier partant a déjà été notifié de l'oubli de la fenêtre (*Scénario 2*),
* **Quand** le vent se lève de manière violente dehors pendant son absence (facteur météo actif),
* **Alors** le Moteur Scoring ignore le comportement d'alerte météo (la règle s'efface). Le Centre de Notifications ne renvoie pas de message pour éviter le harcèlement, l'humain étant déjà conscient de l'état de la fenêtre.

### Scénario 4 : L'urgence absolue (Fuite d'eau ou Fumée)
* **Étant donné qu'** un capteur technique de sécurité (eau/feu) se déclenche,
* **Quand** l'événement survient (peu importe la météo, et que la maison soit vide ou pleine),
* **Alors** JARVIS applique une priorité absolue : il court-circuite l'analyse de contexte et diffuse immédiatement une alerte `critique` à **tous** les compagnons.

### Scénario 5 : L'acquittement global (*Le premier qui répond*)
* **Étant donné que** l'alerte critique du *Scénario 4* a été diffusée sur tous les smartphones de la famille,
* **Quand** l'habitant A clique sur l'action reçue (*"Je m'en occupe"*),
* **Alors** JARVIS valide l'interception (ACK), fige le processus de relance, et envoie un ordre d'effacement synchrone (`clear_notification`) avec le même `tag` aux smartphones des habitants B et C pour supprimer la notification de leurs écrans.

---

## 4. Architecture de Données (Extension des configurations)

Pour soutenir cette User Story sans toucher aux moteurs logiques, les fichiers de configuration s'étendent de la manière suivante :

### `poids.yaml` (Rôle de GEMMA : Catégorisation des risques)
Le moteur GEMMA qualifie l'état de l'environnement matériel et le comportement humain sous forme de scores catégorisés.
```yaml
controle_alarme:
  technique_eau:
    - entity: "binary_sensor.capteur_eau_buanderie"
      poids: 100
  fenetres_perimetriques:
    - entity: "binary_sensor.ouverture_fenetre_salon"
      type: "ouverture"
      poids: 100
```
comportements.yaml (***Rôle de SCORING***: Choix de la sortie)


Le moteur Scoring associe l'état sémantique de GEMMA aux facteurs de contexte externes (Météo, Présence) pour désigner l'actionneur. Le Centre de Notifications devient un actionneur logiciel.

```yaml
urgence_fuite_eau:
  condition_score: "controle_alarme.technique_eau > 0"
  comportements:
    - action: "notifier"
      alerte_id: "fuite_eau_buanderie" # Transmis au Notification Center (Urgence Critique)

anomalie_ouvrants:
  condition_score: "controle_alarme.fenetres_perimetriques > 0"
  comportements:
    - contexte_presence: "absent"
      action: "notifier"
      alerte_id: "oubli_fenetre_depart" # Géré au moment du Last Out
```

### Conclusion

Grâce à cette séparation stricte, le matériel manquant n'est pas un frein. Les capteurs physiques qui viendront plus tard (verrous de fenêtres, pinces ampèremétriques pour la machine) s'inséreront simplement dans le tableau poids.yaml.

La logique de distribution, d'urgence et d'effacement synchrone développée dans cette User Story est immuable, garantissant un système pérenne et un traitement digne d'un véritable système de contrôle industriel.