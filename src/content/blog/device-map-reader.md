---
title: "Device Map Reader — quand un capteur produit plus qu'un on/off"
description: "signal_reader.py lit les entités simples. device_map_reader.py traite ce que signal_reader ne peut pas faire : les seuils, l'hystérésis, les valeurs dérivées, et les vecteurs bruts en attente du moteur d'intention."
pubDate: 2026-05-19
tags: ["home-assistant", "appdaemon", "device-map-reader", "gemma", "python", "architecture"]
---

Dans l'article précédent, `signal_reader.py` lisait l'`index.yaml` et produisait un snapshot de toutes les entités simples — un on/off, un float, une string. Propre, direct, sans transformation.

Mais certains capteurs ne se résument pas à une seule valeur. Le LD2450 produit des coordonnées X/Y, des vitesses, des directions, et des zones logicielles. Le capteur extérieur sort deux tensions en millivolts qu'il faut convertir en "il pleut" ou "besoin de lumière". La télévision a un état `playing/paused/idle` qu'on veut découper en deux signaux binaires distincts.

C'est le rôle de `device_map_reader.py`.

## La séparation de responsabilités

La session du 19 mai a posé une règle simple :

- `signal_reader.py` → devices **simples** : une entité = une sortie normalisée → `sensor.jarvis_signals`
- `device_map_reader.py` → devices **complexes** : plusieurs entités, calculs, seuils → `sensor.jarvis_vectors`

`moteur_gemma.py` consomme les deux. Cette séparation évite qu'un seul fichier devienne un monstre illisible dès que le nombre de capteurs augmente.

## La structure device_map.yaml

C'est le fichier de configuration que `device_map_reader.py` lit. Chaque device y est décrit en cinq sections :

| Section | Rôle | Consommateur |
|---|---|---|
| `sorties` | Lecture directe on/off d'un binary_sensor ESPHome | `moteur_gemma.py` |
| `sorties_derivees` | Binaires calculés depuis mesures + seuils | `moteur_gemma.py` |
| `vecteurs` | Valeurs brutes float/str (position, vitesse...) | Futur moteur d'intention |
| `mesures` | Entrées brutes pour le calcul des sorties dérivées | `device_map_reader.py` |
| `seuils` | Paramètres de conversion et d'hystérésis | `device_map_reader.py` |

Un exemple concret avec le capteur extérieur :

```yaml
exterieur:
  recepteurs:
    - device: CapteurExterieur
      id: Meteo_locale
      mesures:
        lumiere_mv: sensor.capteur_ext_lumiere
        pluie_mv:   sensor.capteur_ext_pluie
      seuils:
        lux_conversion:  0.02276   # mV → lux
        lux_seuil:       50        # seuil besoin lumière
        lux_hysterese:   10        # zone morte ±10 lux autour de 50
        pluie_seuil_mv:  200       # au-dessus = il pleut
      sorties_derivees:
        pleut:          bool
        besoin_lumiere: bool
```

Le YAML déclare **quoi calculer** et **avec quels paramètres**. Le code Python décide **comment**.

## Le handler générique

Contrairement à `signal_reader.py` qui a un callback par type d'entité, `device_map_reader.py` n'en a qu'un seul — `_on_change` — qui gère les trois sections en séquence :

```python
def _on_change(self, entity, attribute, old, new, kwargs):
    if new == old:
        return

    device = kwargs.get("device")

    snapshot_device = {
        "sorties":          {},
        "sorties_derivees": {},
        "vecteurs":         {},
    }

    # 1. Sorties directes
    for cle, entity_id in device.get("sorties", {}).items():
        val = self.get_state(entity_id)
        snapshot_device["sorties"][cle] = self._to_bool(val)

    # 2. Sorties dérivées
    mesures = {cle: self.get_state(eid)
               for cle, eid in device.get("mesures", {}).items()}
    for cle_sortie in device.get("sorties_derivees", {}):
        snapshot_device["sorties_derivees"][cle_sortie] = \
            self._calculer_sortie_derivee(device_id, cle_sortie, mesures, seuils)

    # 3. Vecteurs bruts
    for cle, entity_id in device.get("vecteurs", {}).items():
        snapshot_device["vecteurs"][cle] = self._lire_vecteur(cle, entity_id, seuils)

    self._write_snapshot(device_id, snapshot_device)
```

À chaque changement d'état d'une entité appartenant à un device, tout le device est recalculé. C'est volontairement conservateur : recalculer les trois sorties coûte quelques millisecondes, mais garantit que le snapshot est toujours cohérent.

## Le calcul des sorties dérivées

C'est le cœur du fichier. Quatre cas sont implémentés aujourd'hui.

### Télévision — comparaison string

```python
if cle_sortie == "en_lecture":
    raw = mesures.get("etat_raw")
    return raw == seuils.get("valeur_active")   # "playing"

if cle_sortie == "en_pause":
    raw = mesures.get("etat_raw")
    return raw == seuils.get("valeur_pause")    # "paused"
```

La TV expose un état `media_player` avec des valeurs `playing/paused/idle/off`. `signal_reader.py` conservait cette string brute. Ici, on la découpe en deux signaux binaires indépendants que `poids.yaml` peut peser séparément — regarder activement vs mettre sur pause sont deux signaux de contexte différents.

### Capteur extérieur — seuil numérique simple

```python
if cle_sortie == "pleut":
    raw = mesures.get("pluie_mv")
    return float(raw) > float(seuils.get("pluie_seuil_mv", 200))
```

La sonde pluie sort une tension en millivolts. Au-dessus de 200 mV → il pleut. En dessous → non. Simple.

### Capteur extérieur — seuil avec hystérésis

C'est le cas le plus intéressant. Un seuil dur sur la luminosité créerait des oscillations : si la lumière naturelle fluctue autour de 50 lux, `besoin_lumiere` passerait True/False plusieurs fois par minute, provoquant des allumages et extinctions répétés.

L'hystérésis résout ça en créant une **zone morte** autour du seuil :

```python
if cle_sortie == "besoin_lumiere":
    lux = float(raw) * float(seuils.get("lux_conversion", 0.02276))
    seuil     = float(seuils.get("lux_seuil", 50))
    hysterese = float(seuils.get("lux_hysterese", 10))
    cache_key = f"{device_id}_besoin_lumiere"
    etat_actuel = self._hysterese_cache.get(cache_key)

    if etat_actuel is None:
        nouvel_etat = lux < seuil                    # premier calcul
    elif etat_actuel:
        nouvel_etat = lux < (seuil + hysterese)      # était True → repasse False si > 60 lux
    else:
        nouvel_etat = lux < (seuil - hysterese)      # était False → repasse True si < 40 lux

    self._hysterese_cache[cache_key] = nouvel_etat
    return nouvel_etat
```

Le seuil est à 50 lux, l'hystérésis à 10 :
- Si `besoin_lumiere` est `True` (lumières allumées), il faut dépasser **60 lux** pour l'éteindre
- Si `besoin_lumiere` est `False`, il faut descendre sous **40 lux** pour l'allumer

Le signal ne oscille plus — il attend une transition franche. C'est un principe classique de l'automatisme industriel : on ne change d'état que sur confirmation.

## Les vecteurs bruts

Les vecteurs sont lus mais **non interprétés**. La méthode `_lire_vecteur` fait uniquement du typage :

```python
def _lire_vecteur(self, cle, entity_id, seuils):
    raw = self.get_state(entity_id)

    if "vitesse" in cle:
        v = float(raw)
        seuil_silence = float(seuils.get("vitesse_silence", 0))
        return 0.0 if v < seuil_silence else round(v, 2)

    if any(k in cle for k in ("_x", "_y", "distance", "angle")):
        return round(float(raw), 3)

    if "nb_cibles" in cle:
        return int(float(raw))

    if "direction" in cle:
        return str(raw)
    ...
```

Un filtre de bruit minimal sur les vitesses (en dessous d'un seuil = 0.0), des arrondis sur les coordonnées, des entiers pour le comptage de cibles. Le reste est stocké tel quel.

Ces vecteurs attendent le moteur d'intention — ils seront consommés quand ce fichier existera.

## La structure du snapshot

`sensor.jarvis_vectors` reçoit un dictionnaire organisé par device :

```json
{
  "Presence_chambre": {
    "piece": "chambre",
    "timestamp": "2026-05-19T21:43:02",
    "sorties": {
      "presence_z1": true,
      "presence_z2": true,
      "passage_z3":  false
    },
    "sorties_derivees": {},
    "vecteurs": {
      "nb_cibles":       1,
      "cible_1_vitesse": 0.0,
      "cible_1_direction": "stationnaire"
    }
  },
  "Meteo_locale": {
    "piece": "exterieur",
    "sorties": {},
    "sorties_derivees": {
      "pleut":          false,
      "besoin_lumiere": true
    },
    "vecteurs": {}
  }
}
```

`moteur_gemma.py` injecte les `sorties` et `sorties_derivees` dans sa table de signaux sous la forme `{device_id}_{cle}` — `Presence_chambre_presence_z2`, `Meteo_locale_besoin_lumiere` — et ignore explicitement les vecteurs.

---

Dans le prochain article : `moteur_gemma.py` — comment ces deux snapshots sont fusionnés, scorés, et comment l'état humain courant en sort.
