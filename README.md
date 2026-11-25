# Lecteur d'étiquettes de vin

> **État : projet en cours.** Les résultats sont remplis au fil des mesures. Une case « à mesurer » n'a pas encore de chiffre.

Une photo d'étiquette de vin devient une fiche structurée et vérifiée : domaine, appellation, cépage, millésime, mention grand cru, degré d'alcool. Tout tourne en local, avec un modèle ouvert.

## Ce que le projet montre

- Appel d'une API de modèle en HTTP, puis avec la bibliothèque officielle
- Envoi d'images à un modèle de vision
- Sorties structurées, validées avec Pydantic
- Champs absents laissés vides : le modèle n'invente pas
- Relances, délais, journal des appels, cache
- Évaluation sur 40 étiquettes annotées à la main

## Résultats mesurés

| Modèle | Champs justes | Champs inventés | Durée par étiquette |
|---|---|---|---|
| à mesurer | | | |

- Date de la mesure : à compléter
- Modèle : à compléter
- Nombre de cas : à compléter
- Commande pour refaire la mesure : à compléter

## Installation

Il faut Python 3.12, [uv](https://docs.astral.sh/uv/) et [Ollama](https://ollama.com/).

```
uv sync
copy .env.example .env
```

Puis écrire dans `.env` le nom du modèle utilisé, téléchargé avec `ollama pull`.

## Utilisation

À compléter : la commande principale et un exemple d'entrée et de sortie.

## Comment ça marche

À compléter : un schéma (Mermaid) et cinq lignes d'explication.

## Limites connues

À compléter : ce qui échoue encore, avec un ou deux exemples.

## Coût

0 €. Le projet tourne en local avec des modèles ouverts. Aucun service payant n'est nécessaire.

## Crédits

- **Code, données et mesures** : Charles Burgy.
- **Cours suivi ([COURS.md](COURS.md)) et structure de départ** : module produit par Claude Code (Anthropic), modèle Claude Fable 5.2, créé le 25/11/2026.
- **Sources officielles** : listées à la fin de [COURS.md](COURS.md).

## Licence

[MIT](LICENSE). Projet d'apprentissage, sans but commercial.
