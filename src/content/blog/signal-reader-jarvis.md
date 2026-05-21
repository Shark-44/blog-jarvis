---
title: "Signal Reader — comment Jarvis prend le pouls de la maison"
description: "signal_reader.py est la première brique active du moteur GEMMA. Elle lit l'index.yaml, s'abonne à chaque entité, normalise les états bruts et produit un snapshot temps réel que le reste du système consomme. Voici comment elle fonctionne."
pubDate: 2026-05-18
tags: ["home-assistant", "appdaemon", "signal-reader", "gemma", "python", "architecture"]
---

Dans les articles précédents, j'ai posé les deux premières pierres : l'`index.yaml` qui inventorie la maison, et la philosophie GEMMA qui explique pourquoi on ne câble pas des capteurs directement sur des actionneurs. Il est temps de passer au code.

`signal_reader.py` est l'étape 2 du moteur. C'est elle qui transforme les entités Home Assistant en signaux exploitables.

## La place de signal_reader dans l'architecture

Avant d'ouvrir le fichier, un rappel du flux complet :

```
index.yaml
    ↓
signal_reader.py      → sensor.jarvis_signals   (signaux simples)
device_map_reader.py  → sensor.jarvis_vectors    (capteurs complexes)
    ↓
moteur_gemma.py       → input_text.gemma_etat_courant
    ↓
comportements.yaml    → actionneurs
```

`signal_reader.py` couvre les **devices simples** : une entité Home Assistant = une sortie normalisée True/False (ou une valeur brute pour les sensors). Pas de calcul, pas de logique dérivée — juste une lecture propre.

Les capteurs complexes comme le LD2450 (qui produit des coordonnées X/Y, des vitesses, des zones logicielles) sont traités par `device_map_reader.py`. C'est la séparation de responsabilités introduite lors de la session du 19 mai : chaque lecteur fait une seule chose, bien.

## Ce que fait signal_reader — en trois lignes

Au démarrage, elle :
1. Lit `index.yaml` pour savoir quelles entités surveiller
2. S'abonne aux changements d'état de chacune via AppDaemon
3. Écrit un **snapshot** dans `sensor.jarvis_signals` — une entité HA qui contient l'état de toute la maison en un seul attribut JSON

À chaque changement d'état, le snapshot est mis à jour et réécrit. `moteur_gemma.py` le lit pour calculer l'état humain courant.

## Pourquoi AppDaemon et pas des automatisations YAML ?

Une automatisation YAML réagit à un événement isolé. Elle ne sait pas ce que font les autres. `signal_reader.py` est une **application Python qui tourne en continu**, avec un état en mémoire (`self.snapshot`), capable de corréler n'importe quelle combinaison de signaux.

C'est la différence entre un panneau de relais et un automate programmable — la même analogie que dans le premier article, appliquée ici concrètement.

## Le code, section par section

### Initialisation

```python
def initialize(self):
    self.index_path = self.args.get("index_path", "/config/index.yaml")
    self.index    = self._load_index()
    self.snapshot = {}
    self.presence = self.get_app("presence")

    self._subscribe_all()
    self._subscribe_presence()
    self._take_snapshot()
```

`initialize()` est le point d'entrée AppDaemon — l'équivalent de `__init__` mais dans le cycle de vie du framework. Trois choses se passent dans l'ordre : chargement de l'index, abonnements aux entités, snapshot initial.

`self.get_app("presence")` est notable : on récupère ici une **brique voisine** AppDaemon, `presence.py`, pour déléguer la gestion des personnes. Chaque application AppDaemon peut appeler les méthodes publiques des autres — c'est le mécanisme d'injection de dépendances du système.

### L'itérateur d'entités

```python
def _iter_entities(self):
    pieces = self.index.get("pieces", {})
    for piece, contenu in pieces.items():
        for role in ("recepteurs", "actionneurs", "appareils"):
            for item in contenu.get(role, []):
                yield piece, role, item
```

Cette fonction traverse l'`index.yaml` et génère un tuple `(pièce, rôle, item)` pour chaque entité déclarée. C'est le seul endroit du code qui connaît la structure de l'index — tout ce qui suit consomme ces tuples sans se soucier de comment ils ont été produits.

### Les abonnements

```python
def _subscribe_all(self):
    for piece, role, item in self._iter_entities():
        entity = item.get("entity")
        if not entity:
            continue
        self.listen_state(
            self._on_state_change,
            entity,
            piece=piece,
            role=role,
            item=item,
        )
```

`listen_state` est la méthode AppDaemon qui s'abonne aux changements d'état d'une entité HA. Les kwargs supplémentaires (`piece`, `role`, `item`) sont passés directement au callback — c'est le moyen propre d'éviter les closures ou les variables globales.

À chaque changement d'état, `_on_state_change` est appelé avec le contexte complet :

```python
def _on_state_change(self, entity, attribute, old, new, kwargs):
    piece = kwargs["piece"]
    item  = kwargs["item"]
    self._update_snapshot(piece, item, new)
    self._write_snapshot()
```

Deux lignes. Le callback ne fait qu'une chose : mettre à jour le snapshot et l'écrire.

### La normalisation

C'est ici que les états bruts HA deviennent des valeurs exploitables par GEMMA :

```python
def _normalize(self, kind, entity, raw):
    if raw in (None, "unavailable", "unknown"):
        return None

    if kind in ("binary_sensor", "switch"):
        return raw == "on"

    if kind == "light":
        return raw == "on"

    if kind == "media_player":
        return raw  # "playing", "paused", "idle", "off"

    if kind == "sensor":
        try:
            return float(raw)
        except (ValueError, TypeError):
            return raw

    return raw
```

Quelques points :

- `binary_sensor` et `switch` → `True`/`False`. Le moteur de scoring n'a jamais à comparer des strings `"on"` / `"off"`.
- `media_player` → la string est conservée (`"playing"`, `"paused"`...). `moteur_gemma.py` compare directement ces valeurs dans `poids.yaml` via le champ `valeur_active`.
- `sensor` lux → `float` brut. La conversion en lux utile avec hystérésis est gérée par `device_map_reader.py` pour les capteurs extérieurs, et `get_lux()` expose le capteur intérieur directement.
- `None` si l'entité est indisponible — le moteur ignore les signaux `None` au scoring.

### Le snapshot

Chaque signal stocké dans `self.snapshot` a la même structure :

```python
self.snapshot[signal_id] = {
    "entity":     "binary_sensor.0xa4c138...",
    "piece":      "Salon",
    "type":       "binary_sensor",
    "raw":        "on",
    "value":      True,
    "updated_at": "2026-05-19T21:43:02",
}
```

`raw` conserve la valeur brute HA pour le débogage. `value` est la valeur normalisée que GEMMA consomme. `updated_at` permet de détecter les signaux qui ne se sont pas mis à jour depuis longtemps — utile pour la robustesse du scoring.

### L'écriture dans HA

```python
def _write_snapshot(self):
    self.set_state(
        "sensor.jarvis_signals",
        state="ok",
        attributes=self.snapshot,
    )
```

Le snapshot est écrit dans les **attributs** d'une entité virtuelle HA — pas dans un fichier disque. Cela rend le snapshot visible directement dans l'interface HA (Développeurs → États), observable avec des templates, et consommable par n'importe quelle autre application AppDaemon via `get_state("sensor.jarvis_signals", attribute="all")`.

### La présence humaine

La présence est un cas particulier : elle n'est pas dans `index.yaml` (les personnes ne sont pas des entités physiques de la maison), mais elle est indispensable au scoring.

`signal_reader.py` délègue à la brique `presence.py` via `self.presence.persons`. Le résultat est injecté dans le snapshot sous deux formes :

- `maison_occupee` → signal global booléen
- `presence_{nom}` → signal individuel par personne déclarée

```python
self.snapshot["maison_occupee"] = {
    "entity":  "presence",
    "piece":   "global",
    "type":    "presence",
    "value":   len(presents) > 0,
    ...
}
```

`moteur_gemma.py` peut ainsi discriminer l'état `ABSENT` (maison vide) avant même de regarder les capteurs physiques.

## L'API publique

Trois méthodes exposées aux briques voisines :

```python
def get_snapshot(self):
    """Retourne le snapshot complet en mémoire."""
    return self.snapshot.copy()

def get_signal(self, signal_id):
    """Retourne la valeur normalisée d'un signal par son id."""
    entry = self.snapshot.get(signal_id)
    return entry["value"] if entry else None

def get_lux(self):
    """Retourne la luminosité ambiante en lux."""
    entry = self.snapshot.get("Capteur luminosité")
    if entry and entry["value"] is not None:
        return float(entry["value"])
    return None
```

`moteur_gemma.py` appelle `get_snapshot()` pour construire sa table de signaux. `get_lux()` est un point d'entrée dédié pour la régulation de l'éclairage — un exemple d'API explicite plutôt que de laisser les consommateurs fouiller dans le snapshot.

## Ce que signal_reader ne fait pas

C'est aussi important à dire. `signal_reader.py` ne :

- **calcule pas** de valeurs dérivées (seuils, hystérésis, états calculés) — c'est `device_map_reader.py`
- **gère pas** les timeouts ou la stabilisation temporelle — c'est ESPHome pour les LD2450
- **prend pas** de décision sur les actionneurs — c'est `moteur_gemma.py`
- **expose pas** les vecteurs bruts (X/Y, vitesse, direction) — ils passent par `device_map_reader.py`

Cette délimitation stricte est ce qui rend le système lisible et maintenable. Chaque fichier a une responsabilité, une seule.

---

Dans le prochain article : `device_map_reader.py` — comment les capteurs complexes comme le LD2450 et le capteur extérieur sont traités pour produire des signaux binaires exploitables, avec la gestion des seuils et de l'hystérésis.
