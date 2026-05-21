---
title: "Moteur GEMMA — fusionner, scorer, décider"
description: "moteur_gemma.py est le cerveau du système. Il fusionne les deux snapshots, évalue chaque état défini dans poids.yaml, et publie l'état humain courant toutes les 30 secondes."
pubDate: 2026-05-20
tags: ["home-assistant", "appdaemon", "moteur-gemma", "poids", "scoring", "python", "architecture"]
---

On a maintenant deux sources de données :

- `sensor.jarvis_signals` — les entités simples normalisées par `signal_reader.py`
- `sensor.jarvis_vectors` — les sorties et sorties dérivées des capteurs complexes, produites par `device_map_reader.py`

`moteur_gemma.py` les fusionne, les pèse, et répond à une question : **que fait la personne en ce moment ?**

## Le cycle d'évaluation

Le moteur ne réagit pas aux événements — il tourne **toutes les 30 secondes** :

```python
self.run_every(self._evaluer, "now", 30)
```

Ce choix est délibéré. Un déclenchement par événement serait plus réactif, mais créerait des cascades : chaque changement de capteur déclencherait un recalcul, potentiellement plusieurs par seconde. À 30 secondes, le moteur lisse les transitoires. L'état humain ne change pas en quelques secondes — il change en quelques minutes.

## La fusion des snapshots

`_load_snapshot` reconstruit une table de signaux unique depuis les deux entités HA :

```python
def _load_snapshot(self):
    signals = {}

    # 1. Signaux directs (signal_reader.py)
    raw = self.get_state("sensor.jarvis_signals", attribute="all") or {}
    signals.update(raw.get("attributes", {}))

    # 2. Vectors (device_map_reader.py)
    raw = self.get_state("sensor.jarvis_vectors", attribute="all") or {}
    vectors = raw.get("attributes", {})

    for device_id, vecteur in vectors.items():
        piece = vecteur.get("piece")

        for cle, val in vecteur.get("sorties", {}).items():
            signals[f"{device_id}_{cle}"] = {"value": val, "piece": piece, "type": "sortie"}

        for cle, val in vecteur.get("sorties_derivees", {}).items():
            signals[f"{device_id}_{cle}"] = {"value": val, "piece": piece, "type": "sortie_derivee"}

        # vecteurs bruts → ignorés ici, réservés au futur moteur d'intention

    return signals
```

Après cette fusion, `signals` contient des entrées comme :

```python
{
  "Tele":                          {"value": "playing",  "piece": "salon", ...},
  "Presence_chambre_presence_lit": {"value": True,       "piece": "chambre", ...},
  "Meteo_locale_besoin_lumiere":   {"value": True,       "piece": "exterieur", ...},
  "maison_occupee":                {"value": True,       "piece": "global", ...},
}
```

`moteur_gemma.py` ne sait pas si un signal vient de `signal_reader` ou de `device_map_reader`. Il consomme une table plate. C'est volontairement opaque.

## poids.yaml — la table de vérité

Chaque état possible est décrit dans `poids.yaml`. Voici `ACTIF_BUREAU` :

```yaml
ACTIF_BUREAU:
  description: "Bureau — travail sur PC"
  contraintes:
    duree_min: 900

  signaux:
    PC_HP:                             { poids: 0.95, valeur_active: "on" }
    Presence_bureau_etat:              { poids: 0.8,  valeur_active: "actif" }
    Presence_bureau_presence_piece:    { poids: 0.7,  valeur_active: true }
    Presence_bureau_presence_cliclac:  { poids: -0.3, valeur_active: true }
    Tele:                              { poids: -0.4, valeur_active: "playing" }
```

Chaque signal a un `poids` et un critère d'activation (`valeur_active`, `seuil_max`, ou `seuil_min`). Les poids négatifs sont des **signaux d'exclusion** : si la télé joue, c'est probablement le salon, pas le bureau.

## La formule de scoring

```python
def _calculer_score(self, signals, config_etat, modif_contexte):
    score_total = 0.0
    poids_total = 0.0

    for signal_id, params in config_etat.get("signaux", {}).items():
        poids = params.get("poids", 0.5)
        poids_total += abs(poids)   # toujours positif — dénominateur juste

        valeur = self._lire_valeur(signals, signal_id)
        if valeur is None:
            continue                # signal absent → ne contribue pas, ne pénalise pas

        contribution = self._evaluer_signal(valeur, params) * poids
        score_total += contribution

    if poids_total == 0:
        return 0.0

    score_brut = score_total / poids_total
    return min(max(score_brut * modif_contexte, 0.0), 1.0)
```

Trois points importants :

**`abs(poids)` pour le dénominateur.** Un signal négatif de poids `-0.8` contribue `0.8` au dénominateur. Sans ça, un état avec beaucoup de signaux négatifs aurait un dénominateur artificiellement petit, donc des scores gonflés.

**Un signal absent ne pénalise pas.** Si le FP2 du salon n'est pas encore livré, il n'existe pas dans le snapshot. Le moteur l'ignore au lieu de noter l'état à 0. Le système fonctionne avec les capteurs disponibles.

**Clamp entre 0.0 et 1.0.** Le modificateur contextuel peut pousser un score au-delà de 1.0 — le clamp garantit que le seuil de décision garde un sens.

## L'évaluation d'un signal

```python
def _evaluer_signal(self, valeur, params):
    if "valeur_active" in params:
        attendu = params["valeur_active"]
        if isinstance(attendu, bool):
            return 1.0 if bool(valeur) == attendu else 0.0
        return 1.0 if str(valeur) == str(attendu) else 0.0

    if "seuil_max" in params:
        return 1.0 if float(valeur) <= float(params["seuil_max"]) else 0.0

    if "seuil_min" in params:
        return 1.0 if float(valeur) >= float(params["seuil_min"]) else 0.0

    return 0.0
```

Chaque signal retourne `1.0` ou `0.0` — actif ou non. La nuance est dans le poids, pas dans l'activation. C'est un choix d'architecture : des seuils binaires sont plus simples à déboguer que des activations continues.

## Le modificateur contextuel

L'heure amplifie ou atténue certains états :

```yaml
contexte:
  nuit:
    debut: 23
    fin: 6
    modificateur: 1.3

  matin:
    debut: 6
    fin: 9
    modificateur: 1.1
```

`SOMMEIL` avec un modificateur de `1.3` la nuit atteint le seuil de `0.75` avec moins de signaux confirmés. Un mouvement dans la chambre à 3h du matin est beaucoup plus probablement du sommeil agité que du travail.

Le modificateur gère les plages à chevauchement : si `debut > fin`, la tranche passe minuit (`23 → 6`).

## La décision

```python
scores_tries = sorted(scores.items(), key=lambda x: x[1], reverse=True)
meilleur_etat, meilleur_score = scores_tries[0]
nouvel_etat = meilleur_etat if meilleur_score >= seuil else "INDETERMINE"
```

L'état avec le score le plus élevé est retenu — à condition de dépasser le seuil global (`0.75` dans `poids.yaml`). En dessous du seuil, le moteur publie `INDETERMINE` plutôt que de deviner. `moteur_scoring.py` ignorera cet état et n'actionnera rien.

## La publication

Le résultat est écrit à deux endroits :

**`input_text.gemma_etat_courant`** — une entité HA texte. C'est le déclencheur de `moteur_scoring.py` : il écoute les changements de cette entité et réagit immédiatement.

**`gemma_state.json`** — un fichier disque avec le détail complet : tous les scores, le modificateur appliqué, l'heure. Utilisé pour le debug et comme source de vérité secondaire si l'entité HA est indisponible.

```json
{
  "generated_at": "2026-05-19T21:43:30",
  "etat_courant": "ACTIF_BUREAU",
  "score": 0.847,
  "heure": 21,
  "modificateur": 1.1,
  "tous_scores": {
    "ACTIF_BUREAU":   0.847,
    "LECTURE_BUREAU": 0.312,
    "REPOS_TV":       0.089,
    "SOMMEIL":        0.041,
    ...
  }
}
```

La liste `tous_scores` est précieuse pendant la mise au point : elle montre immédiatement quel état concurrence l'état dominant, et guide l'ajustement des poids.

---

Dans le prochain article : `moteur_scoring.py` — comment l'état GEMMA se traduit en actions concrètes sur les actionneurs, croisé avec la météo et la présence.
