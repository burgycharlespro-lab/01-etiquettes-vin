# Cours 1 — Lecteur d'étiquettes de vin

> **Module de cours produit par Claude Code (Anthropic), modèle Claude Fable 5.2. Créé le 25/11/2026**, à partir des sources officielles listées en fin de document. Le cours donne le chemin ; le code du projet est écrit par Charles Burgy.

| | |
|---|---|
| **Durée estimée** | 1 à 2 semaines, soit 8 à 12 séances de 1 à 2 h |
| **Niveau** | Départ. Python de base suffit |
| **Coût** | 0 €. Modèle local avec Ollama ; clé gratuite en option au module 7 |
| **Avant de commencer** | Guides 01, 02 et 03 lus, `verifier_installation.py` au vert |

## Le projet en une phrase

Une photo d'étiquette de vin devient une fiche structurée et vérifiée : domaine, appellation, cépage, millésime, mention grand cru, degré d'alcool.

## Ce que tu sauras faire à la fin

- Appeler l'API d'un modèle de langage en HTTP, sans bibliothèque spécialisée, puis avec la bibliothèque officielle.
- Envoyer une image à un modèle.
- Obliger un modèle à répondre dans un format précis, et vérifier qu'il l'a fait.
- Empêcher un modèle d'inventer ce qu'il ne voit pas.
- Gérer les erreurs, les lenteurs et les relances.
- Mesurer la qualité sur un jeu de 40 étiquettes, et la vitesse.

## L'arborescence visée

```
01-etiquettes-vin/
├── etiquettes/
│   ├── config.py        réglages lus dans .env
│   ├── llm.py           appel au modèle, relance, journal, cache
│   ├── schemas.py       la fiche (Pydantic)
│   ├── extraction.py    image -> fiche
│   └── cli.py           la commande
├── tests/
├── data/exemples/       quelques photos d'étiquettes
└── eval/
    ├── reference.csv    les 40 fiches remplies à la main
    ├── evaluer.py
    └── resultats/
```

---

## Séance 0 — Mise en place

**À faire**

1. Ouvrir un terminal dans `01-etiquettes-vin`.
2. Créer l'environnement Python du projet : `uv sync`. Un dossier `.venv` apparaît.
3. Copier `.env.example` en `.env` et y écrire le nom du modèle de vision choisi dans le guide 02.
4. Vérifier qu'Ollama répond : `ollama list` doit afficher ton modèle.
5. Créer le dépôt git local (guide 05, sections 1 et 2).
6. Écrire la première entrée de `JOURNAL.md`.

**Réussi quand** : `uv run python --version` répond, et `git status` ne montre pas `.env`.

---

## Module 1 — Le premier appel, à la main

**Objectif** : comprendre ce qu'est un appel d'API en l'écrivant sans aide.

**Notions**

- Une API HTTP : une adresse, une méthode (`POST`), un corps en JSON, une réponse en JSON, un code de statut (200 = réussi, 4xx = ta demande est mauvaise, 5xx = le serveur a un souci).
- Ollama est un serveur qui tourne sur ton PC, à l'adresse `http://localhost:11434`. Rien ne sort de ta machine.
- Une conversation est une liste de messages. Chaque message a un rôle : `system` (les consignes), `user` (la demande), `assistant` (la réponse).
- `stream: false` demande la réponse d'un seul bloc.

**Le squelette** (à taper toi-même, puis à modifier) :

```python
import httpx

reponse = httpx.post(
    "http://localhost:11434/api/chat",
    json={
        "model": "NOM_DU_MODELE",
        "messages": [
            {"role": "system", "content": "Tu es sommelier. Réponds en une phrase."},
            {"role": "user", "content": "Décris un riesling sec d'Alsace."},
        ],
        "stream": False,
    },
    timeout=300,
)
reponse.raise_for_status()
donnees = reponse.json()
print(donnees["message"]["content"])
```

**À faire**

1. Ajouter la dépendance : `uv add httpx`.
2. Lancer le script, puis afficher `donnees` en entier et lire chaque champ.
3. Trouver dans la réponse le nombre de tokens lus, le nombre de tokens écrits et les durées. Dans quelle unité sont les durées ?
4. Changer la consigne `system` et observer l'effet.
5. Provoquer trois erreurs exprès et lire ce qui revient : nom de modèle faux, Ollama arrêté, délai `timeout=1`.

**Réussi quand** : tu peux expliquer chaque ligne du script et chaque champ de la réponse, et tu as vu les trois erreurs.

**Piège** : le premier appel est lent, parce que le modèle se charge en mémoire. Les suivants sont plus rapides tant qu'il reste chargé.

**Source** : documentation de l'API d'Ollama (voir fin de document).

---

## Module 2 — Envoyer une image

**Objectif** : faire décrire une photo d'étiquette.

**Notions**

- Une image voyage dans le JSON sous forme de texte, encodée en base64 (module `base64` de Python).
- Un modèle « multimodal » ou « de vision » accepte du texte et des images. Tous ne le font pas : vérifier dans le guide 02.
- Plus l'image est grande, plus l'appel est lent. On la réduit avant l'envoi.

**À faire**

1. Photographier 5 étiquettes, bien à plat, sans reflet. Les ranger dans `data/exemples/`.
2. `uv add pillow`, puis écrire une fonction qui réduit une image à 1 024 pixels de large au plus.
3. Écrire `decrire.py` : le chemin d'une image en argument, la description en sortie.
4. Mesurer la durée avec l'image d'origine, puis avec l'image réduite. Noter les deux dans le journal.

**Réussi quand** : `uv run python decrire.py data/exemples/etiquette-01.jpg` décrit la bonne étiquette.

**Piège** : une photo de téléphone tournée de 90° dérègle la lecture. Pillow sait remettre l'image droite (`ImageOps.exif_transpose`).

---

## Module 3 — Un seul endroit qui parle au modèle

**Objectif** : ranger l'appel dans `etiquettes/llm.py`, et passer à la bibliothèque officielle.

**Notions**

- Une bibliothèque cliente (ici le paquet `ollama`) fait le même appel HTTP que toi au module 1, avec moins de lignes. Tu sais maintenant ce qu'elle cache.
- Une seule fonction d'entrée pour tout le projet : si le fournisseur change, un seul fichier change (guide 03, section 3).
- Les réglages viennent de `.env`, lus dans `config.py`.

**À faire**

1. `uv add ollama python-dotenv`.
2. Écrire `config.py` : il lit `MODELE_VISION` et `OLLAMA_HOST` et s'arrête avec un message clair si l'un manque.
3. Écrire dans `llm.py` une fonction `demander(messages, images=None)` qui renvoie le texte de la réponse et les mesures (tokens, durée).
4. Réécrire `decrire.py` pour qu'il passe par `demander`.

**Réussi quand** : plus aucun fichier, à part `llm.py`, ne contient l'adresse d'Ollama ni le nom du modèle.

---

## Module 4 — Une sortie structurée

**Objectif** : obtenir une fiche que le code peut lire sans erreur, à chaque fois.

**Notions**

- Pydantic : on décrit la forme des données par une classe Python. Pydantic vérifie qu'un JSON la respecte et dit précisément ce qui ne va pas.
- Schéma JSON : la description de cette forme dans un format standard. Pydantic le produit (`model_json_schema()`).
- Sortie structurée : on donne le schéma au modèle (paramètre `format` chez Ollama) et il doit s'y tenir.
- Température : à 0, le modèle varie le moins possible. C'est le bon réglage pour extraire.

**À faire**

1. `uv add pydantic`.
2. Écrire dans `schemas.py` la classe `FicheVin` : domaine, appellation, cépage, millésime, grand cru (oui / non), degré d'alcool. Choisir le type de chaque champ.
3. Ajouter des règles de validation : millésime entre 1900 et l'année en cours, degré entre 5 et 20.
4. Écrire `extraction.py` : image en entrée, `FicheVin` en sortie.
5. Si la validation échoue, relancer une seule fois en donnant au modèle le message d'erreur de Pydantic. Compter les relances.
6. Écrire 4 tests pytest sur `FicheVin` : une fiche juste, un millésime impossible, un degré impossible, un champ manquant. Ces tests n'appellent pas le modèle.

**Réussi quand** : sur tes 5 étiquettes, 100 % des sorties passent la validation, et les 4 tests passent.

**Piège** : un JSON valide n'est pas un JSON juste. La forme est bonne, le contenu peut être faux. C'est l'objet du module suivant et de l'évaluation.

---

## Module 5 — Ne rien inventer

**Objectif** : un champ absent de l'étiquette reste vide.

**Notions**

- Un modèle de langage complète volontiers ce qui manque. Sur une étiquette sans millésime, il en propose un plausible. Dans un vrai produit, c'est la faute la plus grave.
- L'ancrage (« grounding ») : la réponse ne contient que ce qui se trouve dans la source.
- Trois leviers : le schéma (champs facultatifs, valeur `None` permise), la consigne (dire quoi faire quand l'information manque), le contrôle après coup.

**À faire**

1. Rendre facultatifs les champs qui peuvent manquer.
2. Ajouter, pour chaque valeur, le texte exact lu sur l'étiquette qui la justifie (un champ `lu_sur_etiquette`).
3. Trouver ou photographier 3 étiquettes où il manque un champ (un vin sans millésime, une étiquette sans cépage).
4. Écrire la consigne `system` qui dit quoi faire quand l'information manque. Essayer deux formulations et comparer.
5. Restreindre le cépage à une liste fermée, construite à partir du cahier des charges de l'appellation (source en fin de document). Que doit faire le code si le modèle sort un cépage hors liste ?

**Réussi quand** : sur les 3 étiquettes incomplètes, les champs absents valent `None`.

---

## Module 6 — Tenir face aux erreurs

**Objectif** : le programme traite 40 images sans s'arrêter à la première difficulté.

**Notions**

- Délai maximum (`timeout`) : au-delà, on abandonne l'appel au lieu d'attendre sans fin.
- Relance espacée (« backoff exponentiel ») : après un échec on attend 1 s, puis 2 s, puis 4 s, avec un nombre d'essais limité.
- Toutes les erreurs ne se relancent pas. Un serveur occupé, oui. Un nom de modèle faux, non : relancer ne changera rien.
- Traitement par lot : une image qui échoue est notée en erreur et le lot continue.

**À faire**

1. Écrire la relance à la main, avec une boucle et `time.sleep`.
2. La réécrire avec la bibliothèque `tenacity` (`uv add tenacity`) et comparer les deux versions.
3. Écrire `cli.py` : un dossier d'images en entrée, un fichier CSV de fiches en sortie, et une ligne par image dans le terminal.
4. Tester la panne : arrêter Ollama au milieu d'un lot, le relancer, vérifier que le lot se termine.

**Réussi quand** : un lot de 10 images se termine même avec une panne provoquée, et le CSV dit quelles images ont échoué.

---

## Module 7 — Un deuxième fournisseur (option)

**Objectif** : la même fonction marche avec deux fournisseurs, et tu peux les comparer.

**Notions**

- Beaucoup de fournisseurs acceptent le même format de demande, dit « compatible OpenAI ». Ollama aussi. Un seul code peut donc parler à plusieurs services en changeant l'adresse et la clé.
- Une clé API identifie ton compte. Elle va dans `.env`, jamais dans le code.
- Un palier gratuit a des limites par minute et par jour. Au-delà, le serveur répond par l'erreur 429 : ta relance espacée du module 6 sert ici.
- Avec un service en ligne, l'image quitte ton PC. N'envoyer que des photos d'étiquettes, rien de personnel.

**À faire**

1. Choisir un fournisseur à clé gratuite dans le guide 02 (il doit accepter les images) et créer la clé.
2. Ajouter la clé et l'adresse dans `.env`, et leurs noms dans `.env.example`.
3. Faire évoluer `llm.py` : `demander(..., fournisseur="ollama")` ou `fournisseur="..."`.
4. Vérifier que `extraction.py` n'a pas changé d'une ligne.

**Sans clé** : faire le même module avec un deuxième modèle local. L'exercice d'architecture est le même.

**Réussi quand** : la même commande sort une fiche avec l'un ou l'autre fournisseur, choisi par une option.

---

## Module 8 — Journal, cache, mesures

**Objectif** : chaque appel laisse une trace, et rien n'est recalculé pour rien.

**À faire**

1. Écrire le journal des appels décrit dans le guide 03, section 5.
2. Écrire le cache décrit dans le guide 03, section 6.
3. Écrire `mesures.py` : il lit le journal et affiche, par modèle, le nombre d'appels, la durée moyenne, les tokens par seconde et le taux de relance.

**Réussi quand** : relancer deux fois le même lot ne fait aucun appel la deuxième fois, et le journal le montre.

---

## Module 9 — L'évaluation

**Objectif** : un chiffre qui dit ce que vaut ton lecteur d'étiquettes.

**À faire**

1. Lire le guide 04.
2. Réunir 40 photos d'étiquettes, dont une dizaine de cas difficiles : reflet, étiquette abîmée, texte en allemand, champ absent.
3. Remplir `eval/reference.csv` à la main, **avant** de lancer le modèle sur ces photos. Un champ absent se note `absent`.
4. Écrire `eval/evaluer.py` : il compare chaque fiche à la référence, champ par champ.
5. Décider comment comparer les textes : « Domaine Weinbach » et « domaine weinbach » sont-ils égaux ? Écrire la règle dans le README.
6. Sortir les scores, pour chaque modèle ou fournisseur :
   - part de champs justes, champ par champ ;
   - part de champs inventés (remplis alors que la référence dit `absent`) ;
   - durée moyenne par étiquette.
7. Regarder une par une les erreurs et les classer : photo, lecture, invention, règle de comparaison.

**Réussi quand** : le tableau des résultats est dans le README, avec la date, le modèle et la commande qui le refait.

---

## Module 10 — Finir le dépôt

**À faire**

1. Compléter le README (guide 05, section 7) : phrase d'accroche, exemple d'entrée et de sortie, tableau, limites.
2. Dessiner le schéma du trajet d'une image, en Mermaid (GitHub l'affiche directement).
3. Vérifier qu'un inconnu peut installer et lancer le projet avec les seules commandes du README.
4. Relire `JOURNAL.md` et en tirer 3 lignes « ce que j'ai appris » pour le README.

## Le livrable

- [ ] Le dépôt sur GitHub, licence MIT, sans clé ni fichier `.env`
- [ ] La commande qui transforme un dossier de photos en CSV de fiches
- [ ] Les tests pytest qui passent sans modèle
- [ ] Le jeu d'évaluation (`reference.csv`) et quelques photos d'exemple
- [ ] Le tableau qualité et vitesse, par modèle
- [ ] La liste de ce qui échoue encore

## Pour aller plus loin

- Lire aussi une note de dégustation écrite à la main.
- Ajouter le dos de la bouteille (contre-étiquette) et fusionner les deux fiches.
- Comparer trois tailles d'image et tracer qualité contre durée.

## Ce que le projet 2 réutilise

`config.py`, `llm.py` (appel, relance, journal, cache) et la méthode d'évaluation.

## Sources officielles

- API d'Ollama : https://docs.ollama.com/api/introduction
- Ollama, images : https://docs.ollama.com/capabilities/vision
- Ollama, sorties structurées : https://docs.ollama.com/capabilities/structured-outputs
- Bibliothèque Python d'Ollama : https://github.com/ollama/ollama-python
- HTTPX : https://www.python-httpx.org/
- Pydantic : https://pydantic.dev/docs/validation/latest/get-started/
- Pillow : https://pillow.readthedocs.io/en/stable/
- Tenacity : https://tenacity.readthedocs.io/en/latest/
- pytest : https://docs.pytest.org/en/stable/
- Codes de statut HTTP (MDN) : https://developer.mozilla.org/fr/docs/Web/HTTP/Reference/Status
- Cahiers des charges des appellations (INAO) : https://www.inao.gouv.fr/rechercher-un-produit
