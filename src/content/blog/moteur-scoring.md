---
title: "Moteur Scoring — de l'état humain aux actionneurs"
description: "moteur_scoring.py est le traducteur final. Il prend l'état GEMMA, la météo, la présence, et applique la matrice comportements.yaml pour actionner les bons appareils au bon moment."
pubDate: 2026-05-20
tags: ["home-assistant", "appdaemon", "moteur-scoring", "comportements", "meteo", "python", "architecture"]
---

`moteur_gemma.py` a tranché : l'état courant est `ACTIF_BUREAU`. Mais qu'est-ce que ça veut dire concrètement pour la maison ? Quelle lumière, à quelle intensité, selon qu'il fait soleil ou qu'il pleut ?

C'est la question que `moteur_scoring.py` résout. Il est le seul fichier du système qui touche aux actionneurs.

## Une règle fondamentale

`moteur_scoring.py` **ne connaît pas les capteurs**. Il ne lit pas les signaux, ne consulte pas les snapshots des LD2450, ne sait pas si la présence vient du GPS ou d'un PIR.

Il consomme trois choses :
- l'état GEMMA (`input_text.gemma_etat_courant`)
- la météo (via HA `weather.meteofrance_maison` + `sun.sun`)
- la présence (`sensor.jarvis_presence`)

Et une configuration : `comportements.yaml`.

Cette séparation est importante. Ajouter un capteur ne modifie pas `moteur_scoring.py`. Changer un comportement ne modifie pas les lecteurs. Les deux couches évoluent indépendamment.

## Les déclencheurs

Le moteur écoute deux sources d'événements :

```python
# Changement d'état GEMMA → réaction immédiate
self.listen_state(self._on_gemma_change, "input_text.gemma_etat_courant")

# Changement météo ou position du soleil → réévaluation avec l'état courant
self.listen_state(self._on_meteo_change, "weather.meteofrance_maison")
self.listen_state(self._on_meteo_change, "sun.sun")
```

Deux déclencheurs séparés parce que les deux contextes changent à des rythmes différents. L'état GEMMA change toutes les 30 secondes au maximum. La météo peut basculer en milieu de journée sans que l'état humain change — mais le comportement doit changer quand même.

## La condition météo

```python
def _lire_condition_meteo(self):
    # 1. Nuit — priorité absolue
    if self.get_state(self.entity_sun) == "below_horizon":
        return "nuit"

    ciel = self.get_state(self.entity_meteo)
    temp = float(self.get_state(self.entity_meteo, attribute="temperature") or 15)

    if ciel in {"rainy", "pouring", "lightning", "lightning-rainy", "hail"}:
        return "pluvieux"

    if ciel in {"cloudy", "partlycloudy", "fog", "windy"}:
        return "nuageux"

    return "degage_chaud" if temp > 22 else "degage_froid"
```

Cinq conditions possibles : `nuit`, `pluvieux`, `nuageux`, `degage_chaud`, `degage_froid`. La nuit est prioritaire sur tout — au-dessus du seuil d'ensoleillement, c'est le ciel qui décide, puis la température.

La météo n'est pas là pour être précise à la minute près. Elle sert à moduler les comportements sur des durées de l'ordre de l'heure.

## La présence comme gardien

Avant tout calcul, une vérification :

```python
presence = self._lire_presence()
if not presence.get("maison_occupee", True):
    self.log("Maison vide — scoring suspendu.")
    return
```

Si la maison est vide, aucune action n'est déclenchée. La valeur par défaut de `maison_occupee` est `True` — si `sensor.jarvis_presence` est indisponible, le moteur préfère agir par excès de précaution plutôt que d'éteindre une maison potentiellement occupée.

C'est un principe de **fail-safe** : en cas de doute, ne pas éteindre.

## comportements.yaml — la matrice état × météo

```yaml
etats:
  ACTIF_BUREAU:
    nuageux:
      Lumiere_bureau:  200     # brightness 0-255
      Lumiere_couloir: false
      Volets_bureau:   false

    degage_chaud:
      Lumiere_bureau:  0       # lumière naturelle suffisante
      Lumiere_couloir: false
      Volets_bureau:   true    # volets ouverts

    pluvieux:
      Lumiere_bureau:  220
      Lumiere_couloir: false
      Volets_bureau:   false

    nuit:
      Lumiere_bureau:  150     # ambiance tamisée
      Lumiere_couloir: true
      Volets_bureau:   false
```

Pour chaque état GEMMA, une colonne par condition météo. Chaque ligne est un actionneur avec sa valeur cible :
- `bool` → switch on/off
- `int` → brightness lumière dimmable (0 = off)

Le moteur lit la cellule `etats[ACTIF_BUREAU][nuageux]` et actionne chaque entrée. C'est une simple lecture de dictionnaire.

## L'actionnement

```python
def _actionner(self, actionneur_id, valeur):
    entity = self._id_vers_entity(actionneur_id)
    if not entity:
        return

    if isinstance(valeur, bool):
        self.turn_on(entity) if valeur else self.turn_off(entity)

    elif isinstance(valeur, (int, float)):
        if valeur == 0:
            self.turn_off(entity)
        else:
            self.turn_on(entity, brightness=int(valeur))
```

Deux cas seulement. La complexité est dans `comportements.yaml`, pas dans le code. Ajouter un type d'actionneur (une scène, une température de chauffe) demande d'ajouter un cas ici et une entrée dans le YAML — rien d'autre.

## La résolution d'un actionneur

`comportements.yaml` utilise des identifiants comme `Lumiere_bureau`. Pour envoyer une commande HA, il faut l'entity_id réel (`light.bureau_principal`). La résolution se fait en deux étapes :

```python
def _id_vers_entity(self, actionneur_id):
    # Priorité 1 — sensor.jarvis_signals (déjà en mémoire, rapide)
    raw = self.get_state("sensor.jarvis_signals", attribute="all") or {}
    signals = raw.get("attributes", {})
    if actionneur_id in signals:
        return signals[actionneur_id].get("entity")

    # Priorité 2 — index.yaml (fallback disque)
    with open(self.index_path, "r") as f:
        index = yaml.safe_load(f)
    for piece, contenu in index.get("pieces", {}).items():
        for item in contenu.get("actionneurs", []):
            if item.get("id") == actionneur_id:
                return item.get("entity")

    return None
```

`sensor.jarvis_signals` est consulté en premier parce qu'il est déjà en mémoire HA — pas de lecture disque. `index.yaml` est le fallback pour les actionneurs qui ne sont pas dans les signaux (volets, chauffage, scènes).

## Ce que le moteur ne fait pas

`moteur_scoring.py` ne gère pas la progressivité (fondus de lumière, montée en température sur 10 minutes) — ces comportements relèveront de `comportements.yaml` enrichi et d'AppDaemon `self.call_service` avec des paramètres de transition.

Il ne gère pas non plus les séquences temporelles (allumer le couloir 30 secondes avant d'éteindre le bureau). Ces logiques viendront avec le moteur d'intention, qui anticipe les déplacements avant qu'ils se produisent.

---

Dans le prochain article : `presence.py` et `lieu_resolver.py` — les deux briques qui répondent aux questions "est-ce que quelqu'un est là ?" et "dans quelle pièce ?"
