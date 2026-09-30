# Back-end (Flask)

Règles générales : voir `../CLAUDE.md`. Ici, uniquement ce qui est propre au back.

- Périmètre : ce dossier seulement. Le contrat avec le front est `../docs/API.md`.
- Respecter le flux controller → service → repository ; le SQL reste dans `repository/`.
- Nouvelle entrée/sortie JSON : ajouter le DTO Pydantic dans `dtos/` et le mapper dans `mappers/`.
- Nouvelle route ou changement de format : mettre à jour `../docs/API.md`.
- Ne pas lire ni commiter : `.env`, `*.db`, `temporaire/`, `__pycache__/`, `.venv/`.
- Nouvelle dépendance : l'ajouter à `requirements.txt` et en informer explicitement.
- Lancement : `flask --app app run` (nécessite un `.env`, modèle : `.env.exemple`).
