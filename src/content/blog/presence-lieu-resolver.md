---
title: "Présence et localisation — est-ce que quelqu'un est là, et où ?"
description: "presence.py et lieu_resolver.py répondent à deux questions distinctes : la maison est-elle occupée (GPS/réseau), et quelle pièce est active (capteurs physiques). Deux sources différentes, deux granularités différentes."
pubDate: 2026-05-20
tags: ["home-assistant", "appdaemon", "presence", "lieu-resolver", "python", "architecture"]
---

Deux questions traversent tout le système GEMMA :

1. **Est-ce que quelqu'un est dans la maison ?** — réponse binaire, basée sur le GPS et le réseau
2. **Dans quelle pièce ?** — réponse graduée, basée sur les capteurs physiques

Ce sont deux problèmes différents. `presence.py` répond à la première, `lieu_resolver.py` à la seconde.

---

## presence.py — la couche GPS/réseau

`presence.py` est le plus simple des deux. Il écoute les `device_tracker` des personnes déclarées dans `PersonRepository`, détecte les arrivées et les départs, et publie le résultat dans `sensor.jarvis_presence`.

### Abonnements

```python
for nom in self._get_all_persons():
    tracker = self.persons.get_tracker(nom)
    self.listen_state(self.on_arrive, tracker, new="home")
    self.listen_state(
        self.on_part, tracker,
        new="not_home", old="home",
        duration=30           # 30 secondes avant de confirmer le départ
    )
```

Le délai de 30 secondes sur le départ évite les faux positifs GPS — un signal `not_home` fugace au moment où le téléphone change de cellule ne déclenche pas l'extinction de la maison.

### Arrivée et départ

```python
def on_arrive(self, entity, attribute, old, new, kwargs):
    nom = self._get_nom_by_tracker(entity)
    self.log(f"Maison OCCUPÉE — {nom} est arrivé")
    self._write_snapshot_presence()

def on_part(self, entity, attribute, old, new, kwargs):
    nom = self._get_nom_by_tracker(entity)
    self._write_snapshot_presence()

    if len(self.persons.get_all_home()) == 0:
        self.log("Maison VIDE — délégation à eteindre_maison")
        self.eteindre.dernier_parti(nom)
```

Quand le dernier occupant part, `presence.py` délègue à `eteindre_maison` — une brique séparée qui gère la séquence d'extinction (pas encore documentée). `presence.py` ne sait pas ce qui doit être éteint. Il constate juste que la maison est vide.

### Le snapshot presence

```python
def _write_snapshot_presence(self):
    home = self.persons.get_all_home()
    self.set_state(
        "sensor.jarvis_presence",
        state="ok",
        attributes={
            "maison_occupee": len(home) > 0,
            "personnes_home": home,
            "derniere_maj":   datetime.now().isoformat(),
        }
    )
```

L'entité `sensor.jarvis_presence` expose trois attributs. `moteur_scoring.py` lit `maison_occupee` comme premier garde-fou avant tout actionnement. `personnes_home` est disponible pour les comportements personnalisés par personne — pas encore utilisé, mais la structure est prête.

### Ce que presence.py ne fait pas

Il ne sait pas dans quelle pièce se trouve quelqu'un. Il ne lit pas les LD2450, les FP2, ou les PIR. Il répond uniquement à "est-ce que quelqu'un est là ?" à l'échelle de la maison entière. La localisation par pièce, c'est `lieu_resolver.py`.

---

## lieu_resolver.py — la déduction spatiale

`lieu_resolver.py` croise deux sources : l'`index.yaml` (ce qui existe par pièce) et le snapshot de `signal_reader.py` (ce qui est actif en ce moment). Il produit une liste de pièces occupées avec un **score de certitude**.

### Le principe

Chaque type de capteur a un poids différent dans la certitude de localisation :

```python
self.poids_type = {
    "binary_sensor": 0.8,   # PIR — détection directe
    "sensor":        0.4,   # lux, temp — signal indirect
    "media_player":  0.7,   # TV active → humain au salon
    "switch":        0.3,   # switch allumé — signal faible
}
```

Un PIR qui détecte quelqu'un dans le bureau est un signal fort. Une lumière allumée dans la cuisine est un signal faible — quelqu'un peut avoir oublié de l'éteindre. La certitude est la somme pondérée des signaux actifs rapportée au total pondéré possible.

### L'évaluation pièce par pièce

```python
def _evaluer_piece(self, piece, contenu, snapshot):
    tous_signaux = contenu.get("recepteurs", []) + contenu.get("appareils", [])
    score_total = 0.0
    poids_total = 0.0

    for item in tous_signaux:
        signal_id = item.get("id")
        type_item = item.get("type", "binary_sensor")
        poids = self.poids_type.get(type_item, 0.5)
        poids_total += poids

        entree = snapshot.get(signal_id)
        if not entree:
            continue

        if self._est_actif(entree.get("value"), type_item):
            score_total += poids

    if poids_total == 0:
        return None

    certitude = round(score_total / poids_total, 2)
    if certitude < 0.3:
        return None             # sous le seuil → pièce ignorée

    return {"piece": piece, "certitude": certitude, "signaux_actifs": [...]}
```

Le seuil à `0.3` est conservateur. Une pièce avec uniquement une lumière oubliée allumée ne sera pas considérée comme occupée.

### La logique d'activation par type

```python
def _est_actif(self, valeur, type_item):
    if type_item == "binary_sensor":
        return valeur is True               # PIR, contact → True = actif

    if type_item == "media_player":
        return valeur in ("playing", "paused")  # TV ou ampli en cours = humain

    if type_item == "sensor":
        if isinstance(valeur, (int, float)):
            return valeur > 0
        return str(valeur).lower() not in ("off", "idle", "0", "unknown")

    if type_item == "switch":
        return valeur is True
```

Le `media_player` mérite une attention particulière : `paused` compte comme occupation. Quelqu'un qui met la télé sur pause est toujours dans la pièce — il reviendra.

### Le résultat

```python
def _publier(self, contextes):
    sortie = {
        "generated_at":    datetime.now().isoformat(timespec="seconds"),
        "contextes_actifs": contextes,
    }
    with open(self.lieu_state_path, "w") as f:
        json.dump(sortie, f, ensure_ascii=False, indent=2)
```

Exemple de `lieu_state.json` :

```json
{
  "generated_at": "2026-05-19T21:44:15",
  "contextes_actifs": [
    {
      "piece":          "Bureau",
      "certitude":      0.92,
      "signaux_actifs": ["PC_HP", "Presence_bureau_etat"]
    },
    {
      "piece":          "Salon",
      "certitude":      0.41,
      "signaux_actifs": ["Tele"]
    }
  ]
}
```

Les pièces sont triées par certitude décroissante. La pièce en tête est la plus probablement occupée. Le moteur d'intention consommera cette liste — avec les vecteurs LD2450, il pourra raffiner encore : pas juste "quelqu'un est au bureau", mais "quelqu'un quitte le bureau en direction de la porte".

### L'API publique

```python
def get_contextes(self):
    return list(self.contextes_precedents.values())

def get_pieces_actives(self):
    return list(self.contextes_precedents.keys())

def est_piece_active(self, piece):
    return piece in self.contextes_precedents
```

Ces trois méthodes sont appelables depuis n'importe quelle autre brique AppDaemon. `moteur_scoring.py` peut ainsi décider d'actionner uniquement les lumières des pièces réellement occupées — pas de la maison entière.

---

## La relation entre les deux briques

`presence.py` et `lieu_resolver.py` sont complémentaires mais indépendants :

| | `presence.py` | `lieu_resolver.py` |
|---|---|---|
| Source | GPS / réseau (device_tracker) | Capteurs physiques (PIR, LD2450, TV) |
| Granularité | Maison entière | Par pièce |
| Latence | Secondes (GPS) | 15 secondes (cycle) |
| Certitude | Haute (GPS est fiable) | Graduée (0 à 1) |
| Consommateur | `moteur_scoring.py` (garde-fou) | `moteur_scoring.py`, futur moteur d'intention |

`presence.py` est le gardien : si la maison est vide selon le GPS, `moteur_scoring.py` ne déclenche rien, même si `lieu_resolver.py` détecte une activité (un capteur qui disjoncte, une TV restée allumée).

`lieu_resolver.py` affine : une fois la présence confirmée, il permet d'adapter les comportements pièce par pièce plutôt que de traiter la maison comme un bloc monolithique.

---

La prochaine étape — le moteur d'intention — s'appuiera sur ces deux briques, et y ajoutera la dimension temporelle : pas seulement "où est la personne" mais "où va-t-elle dans les prochaines secondes". Ce sera possible dès la livraison des capteurs Aqara FP2, qui fourniront les coordonnées X/Y des flux inter-pièces que les LD2450 ne couvrent pas.
