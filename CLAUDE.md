# CampusFlow — back-end

API REST de gestion des visiteurs des événements de l'IUT (portes ouvertes, salons…) : collecte des
visiteurs (bac, lycée, formations visées, options), configuration des événements/formations,
exports et statistiques. Projet étudiant BUT2 SAÉ 4.A (Lay, Quemener, Cai). Le front tourne sur
`http://localhost:5173` (CORS).

## Stack
- Python 3, Flask 3 + flask-cors, gevent
- SQLite (module `sqlite3`, SQL écrit à la main, pas d'ORM)
- Pydantic v2 pour les DTO, python-dotenv pour la config

## Structure
Code dans `BUT2_2026_SAE4.A-LayQuemenerCai-back/` ; `DocumentsARendre/` : journal technique et RGPD (PDF).
- `app.py` — point d'entrée Flask, CORS, enregistrement des blueprints, appel de `init_db()`
- `init_db.py` — création des tables (`CREATE TABLE IF NOT EXISTS`) : visiteurs, evenements,
  formations_iut, choix_formations_visees, evenements_participes
- `database.py` — `get_db()` / `close_db()` (SQLite dans `flask.g`, clés étrangères activées)
- `Token.py` — jetons admin en mémoire (perdus au redémarrage)
- `controllers/` — routes (`/visiteurs`, `/config`) : `visiteurs_controller`, `config_controller`
- `services/` — logique métier ; `repository/` — requêtes SQL ; `mappers/` — BDD ↔ DTO/JSON front
- `dtos/` — modèles Pydantic (`Creer*DTO` entrée, `Get*DTO` sortie)
Flux : controller → service → repository (+ mapper/DTO).

## Installation
```bash
cd BUT2_2026_SAE4.A-LayQuemenerCai-back
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.exemple .env   # puis renseigner PASSWORD, DATABASE (chemin du .db), CORS_ORIGIN
```

## Lancement
```bash
flask --app app run    # ou : python app.py (http://127.0.0.1:5000) ; tables créées par init_db()
```

## Tests
Aucun test automatisé. Vérification manuelle des routes (curl, Postman) ; `GET /` renvoie `CampusFlow`.

## Conventions observées
- Noms en français (visiteurs, evenement, formation, lycée…) ; fichiers en `snake_case`
  (`visiteurs_service.py`), sauf DTO en PascalCase (`CreerVisiteursDTO.py`) et `Token.py`.
- Colonnes SQL en `snake_case` ; JSON envoyé au front en camelCase partiel (`codePostal`, `niveau_etudes`).
- Chaque fonction est précédée d'un docstring-bloc en français placé avant le `def`.

## Règles de travail
- Ne pas commiter les fichiers secrets (`.env`, etc.).
- Pour chaque nouvelle fonctionnalité majeure, créer une nouvelle branche.
- N'importer aucune nouvelle librairie sans informer de son ajout.
- Travailler dans un seul périmètre à la fois (front ou back). Ne lire ni les dépendances installées
  (`.venv`, `node_modules`), ni les fichiers de verrouillage, ni les données, logs ou fichiers générés
  (`*.db`, `temporaire/`, `__pycache__/`).
- Front et back communiquent via `docs/API.md` : s'y référer plutôt que lire l'autre partie du projet.
  Toute modification d'une route met ce fichier à jour.
- Ce fichier reste général ; les règles propres à chaque partie sont dans le `CLAUDE.md` de son dossier.

## Points d'attention
- `__pycache__/` est versionné alors qu'il devrait être ignoré ; le `.gitignore` de la racine
  préfixe mal le chemin de `.venv`.
