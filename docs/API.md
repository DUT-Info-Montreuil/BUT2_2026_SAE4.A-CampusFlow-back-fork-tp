# API CampusFlow

Contrat entre le front et le back. Base : `http://127.0.0.1:5000` (Flask). CORS limité à `CORS_ORIGIN`.
Corps et réponses en JSON, sauf les exports (CSV). Les messages d'erreur sont des chaînes JSON.
Tenir ce fichier à jour à chaque modification de route.

## Visiteurs — `/visiteurs`

| Méthode | Route | Rôle | Codes |
|---|---|---|---|
| GET | `/visiteurs` | Liste paginée (vue courte) | 200 |
| GET | `/visiteurs/<id>` | Détail d'un visiteur (vue longue) | 200, 404 |
| POST | `/visiteurs` | Créer un visiteur | 201, 404 (erreur) |
| PUT | `/visiteurs/<id>` | Modifier un visiteur | 201, 400 |
| DELETE | `/visiteurs` | Supprimer tous les visiteurs | 204, 404 |
| DELETE | `/visiteurs/<id>` | Supprimer un visiteur | 204, 404 |
| GET | `/visiteurs/export` | Export CSV (mêmes filtres que la liste, sans pagination) | 200 |
| GET | `/visiteurs/export/email` | Export CSV des e-mails | 200 |
| GET | `/visiteurs/stat` | Stats : `{bac, reorientation, immersion, handicap}` | 200 |
| GET | `/visiteurs/stat/{bac,handicap,immersion,reorientation}` | Une seule stat (objet clé → effectif) | 200 |

**Filtres de `GET /visiteurs`** (query) : `nom`, `prenom`, `telephone`, `email`, `ville`, `bac`, `lycee`,
`formation_actuelle`, `formation_visee`, `reorientation|handicap|immersion` (`true`/autre), `page` (défaut 1),
`limit` (défaut 5). Réponse : `{"data": [...], "page": 1, "limit": 5}`.

**Vue courte** : `{id, nom, prenom, bac:{intitule, matiere1, matiere2, annee}, adresse:{ville, codePostal}}`.
**Vue longue** : vue courte + `date_naissance`, `lycee:{nom_lycee, codePostal}`, `options:{handicap,
reorientation, immersion}`, `email`, `telephone`, `formation_actuelle:{intitule, niveau_etudes}` (champs nuls omis).

**Corps de `POST /visiteurs`** (à plat) :
`nom`*, `prenom`*, `date_naissance`*, `email`, `telephone` (10 caractères ; email ou téléphone requis),
`bac_intitule`*, `bac_annee`*, `bac_matiere1`, `bac_matiere2`, `nom_lycee`*, `code_postal_lycee`*,
`adresse_ville`*, `adresse_codePostal`*, `handicap`, `reorientation`, `immersion` (booléens),
`formation_actuelle_intitule` + `formation_actuelle_niveau` (réorientation), `formationSouhaite` (id formation),
`evenementSouhaite` (id événement). (* = obligatoire)

**Corps de `PUT /visiteurs/<id>`** : notamment `bac_intitule`, `bac_annee`, `matiere1`, `matiere2`,
`adresse_ville`, `adresse_codePostal` ; voir `services/visiteurs_service.py::modif_visiteur` pour la liste exacte.

## Configuration — `/config`

| Méthode | Route | Corps | Codes |
|---|---|---|---|
| GET | `/config/evenement` | — → `[{id, intitule, lieu:{ville, codePostal}, date}]` | 200, 404 |
| POST | `/config/evenement` | `intitule`, `date`, `adresse_ville`, `adresse_codePostal` | 201, 400 |
| PUT | `/config/evenement/<id>` | idem POST | 201, 404 |
| DELETE | `/config/evenement` · `/config/evenement/<id>` | — | 204, 404 |
| GET | `/config/formation` | — → `[{id, intitule}]` | 200, 404 |
| POST | `/config/formation` | `intitule` | 201, 404 |
| PUT | `/config/formation/<id>` | `intitule` | 201, 404 |
| DELETE | `/config/formation` · `/config/formation/<id>` | — | 204, 404 |

## Authentification administrateur — `/config`

| Méthode | Route | Corps | Réponse |
|---|---|---|---|
| POST | `/config/authentificate` | `{password}` | 200 `{token}` / 400 |
| POST | `/config/verif-authentificate` | `{token}` | 200 / 400 |
| POST | `/config/logout` | `{token}` | 200 |
| PUT | `/config/modif-password` | `{oldPassword, newPassword, confPassword}` | 200 |

Le mot de passe est stocké dans `.env`. Les jetons sont gardés en mémoire côté serveur (perdus au redémarrage) ;
les routes ne contrôlent pas le jeton : le front doit le vérifier via `verif-authentificate`.
