# Audit de sécurité — CampusFlow back-end

Périmètre : `BUT2_2026_SAE4.A-LayQuemenerCai-back/` (Flask 3, SQLite, Pydantic). Audit statique du code
source ; aucune exécution ni test d'intrusion. Les fichiers `.env`, `*.db`, `temporaire/` et `.venv`
n'ont pas été lus (aucun `.env` ni `.db` n'est présent dans le dépôt).

Niveaux : **Critique** > **Élevé** > **Moyen** > **Faible**.

## Synthèse

| # | Constat | Gravité |
|---|---------|---------|
| 1 | Aucune route de données n'est protégée : le jeton admin n'est jamais vérifié | Critique |
| 2 | Suppression massive, export et lecture des données personnelles ouverts à tous | Critique |
| 3 | `PUT /config/modif-password` sans jeton ; mot de passe en clair dans `.env` | Élevé |
| 4 | Connexion avec `{"password": null}` si `PASSWORD` n'est pas défini | Élevé |
| 5 | Pas de limitation de tentatives sur `/config/authentificate` | Élevé |
| 6 | Injection de formules dans les exports CSV | Moyen |
| 7 | Fichiers CSV partagés et persistants (fuite entre requêtes, RGPD) | Moyen |
| 8 | Messages d'exception renvoyés au client | Moyen |
| 9 | Validation d'entrée insuffisante (`bool("false")`, dates, e-mail, pagination) | Moyen |
| 10 | Jetons en mémoire, sans expiration, `logout` / `verif` qui plantent | Moyen |
| 11 | CORS dépendant d'une variable d'environnement non contrôlée | Moyen |
| 12 | Dépendances inutiles (ipython, jupyter, pymongo, requests…) | Moyen |
| 13 | Déni de service : filtrage et pagination faits en Python | Moyen |
| 14 | Connexions SQLite jamais fermées, code SQL cassé (modifs) | Faible |
| 15 | Données sensibles (RGPD) : handicap, mineurs, pas de rétention | Élevé |

---

## 1. Routes et endpoints exposés

Toutes les routes sont publiques. Seules `/config/authentificate`, `/config/verif-authentificate` et
`/config/logout` manipulent un jeton, **et aucune autre route ne le vérifie**.

| Méthode | Route | Effet | Protection actuelle |
|---------|-------|-------|---------------------|
| GET | `/` | Renvoie `CampusFlow` | aucune (sans risque) |
| GET | `/visiteurs` | Liste paginée des visiteurs (nom, e-mail, téléphone, naissance, handicap…) | aucune |
| GET | `/visiteurs/<id>` | Fiche complète d'un visiteur | aucune |
| POST | `/visiteurs` | Crée un visiteur | aucune (normal pour un formulaire public, mais sans anti-abus) |
| PUT | `/visiteurs/<id>` | Modifie un visiteur | aucune |
| DELETE | `/visiteurs` | **Supprime tous les visiteurs** | aucune |
| DELETE | `/visiteurs/<id>` | Supprime un visiteur | aucune |
| GET | `/visiteurs/export` | CSV de toutes les données personnelles | aucune |
| GET | `/visiteurs/export/email` | CSV de tous les e-mails | aucune |
| GET | `/visiteurs/stat*` | Statistiques (bac, handicap, immersion, réorientation) | aucune |
| GET | `/config/evenement`, `/config/formation` | Lecture de la configuration | aucune (acceptable) |
| POST/PUT/DELETE | `/config/evenement[/<id>]`, `/config/formation[/<id>]` | Écriture / suppression de la configuration | aucune |
| PUT | `/config/modif-password` | Change le mot de passe admin | connaître l'ancien mot de passe seulement |
| POST | `/config/authentificate` | Échange mot de passe → jeton | aucune limitation |
| POST | `/config/verif-authentificate`, `/config/logout` | Vérifie / supprime un jeton | — |

**Problèmes**

- **Contrôle d'accès absent (Critique).** N'importe qui atteignant l'API peut lire, modifier ou effacer
  toute la base, sans s'authentifier. L'authentification n'existe que côté front (masquage d'écrans),
  ce qui n'est pas une protection : l'API se contacte directement avec curl.
- **Suppression globale en une requête** (`DELETE /visiteurs`, `DELETE /config/evenement`,
  `DELETE /config/formation`) : perte totale de données, par erreur ou malveillance. Les suppressions
  en cascade (`ON DELETE CASCADE`) effacent aussi les choix de formation et la participation aux événements.
- **IDOR** : les identifiants sont des entiers séquentiels ; il suffit d'itérer `/visiteurs/1`, `/2`…
  pour aspirer tous les profils (même sans `GET /visiteurs`).
- Codes HTTP incohérents : `204` renvoyé avec un corps JSON (ignoré), `404` pour une erreur de saisie,
  `201` pour une modification. Cela gêne la supervision et masque les erreurs réelles.
- Pas d'en-têtes de sécurité, pas de HTTPS, pas de limitation de débit, pas de journal d'audit
  (qui a supprimé quoi ?).

**Remédiation** : décorateur `@admin_required` (jeton dans `Authorization: Bearer`) sur tout ce qui n'est
pas `POST /visiteurs`, `GET /config/*` et l'authentification ; supprimer ou confirmer explicitement les
suppressions globales ; ajouter Flask-Limiter ou un proxy (nginx) pour le débit.

---

## 2. Entrées utilisateur

Sources : corps JSON (`request.get_json()`), paramètres de requête (`request.args`), segments d'URL
(`<int:id>`, bien typés).

**Problèmes**

- **Pas de validation de type sur le JSON** : `data` peut être `None`, une liste, etc. Les accès
  `data["x"]` lèvent `KeyError`/`TypeError` → erreur 500 (voir §8). `request.get_json()["token"]`
  et `["password"]` dans `config_controller.py:108,116,125` ne sont pas protégés.
- **`bool(...)` sur des chaînes** (`visiteurs_mapper.py:150-152`, `visiteurs_service.py:66-69`) :
  `bool("false")` vaut `True`. Un client qui envoie `"handicap": "false"` enregistre `true` — corruption
  de données sensibles et de statistiques.
- **Champs peu contraints** (`CreerVisiteursDTO.py`) : `date_naissance` est un `str` libre (aucun
  format ni bornes), `email` sans contrôle de format, pas de longueur maximale sur `nom`, `prenom`,
  `ville`, `nom_lycee`…, `codePostal` / `annee` sans plage, téléphone : longueur 10 mais pas de chiffres
  (le message dit « 10 chiffres »). Un attaquant peut stocker des chaînes de plusieurs Mo.
- **`formationSouhaite` / `evenementSouhaite`** : lus avec `data.get(...)` puis `int(...)` dans le
  repository ; `None` ou texte → exception après l'`INSERT` du visiteur (insertion non validée, erreur
  renvoyée en clair). Les identifiants ne sont pas vérifiés côté métier (seule la clé étrangère protège).
- **Pagination** (`limit`, `page`) : `int` typé mais sans bornes. `limit=-1`, `page=0` ou `limit=10**9`
  produisent des tranches incohérentes ou une charge excessive (voir §13).
- **Filtres en chaîne libre** : utilisés uniquement en paramètres SQL ou en comparaison Python, donc
  pas d'injection SQL, mais `reorientation=xyz` est silencieusement interprété comme `false` sur
  `GET /visiteurs` et comme « pas de filtre » sur `/export` (comportements divergents).
- **XSS stocké** : les champs texte sont stockés et renvoyés tels quels en JSON. L'API elle-même n'est
  pas vulnérable (pas de HTML), mais le front doit échapper l'affichage ; aucune assainissement n'est
  fait ici.
- Injection de formules CSV : voir §6.

**Remédiation** : valider tout le corps via Pydantic (`VisiteurCreerDTO.model_validate(data)`) avec
`EmailStr`, `date`, `Field(max_length=…)`, `conint(ge=…, le=…)`, patron `^\d{10}$` ; bornes
`limit ∈ [1, 100]`, `page ≥ 1` ; convertir les booléens avec une fonction stricte.

---

## 3. Authentification et sessions

Fichiers : `Token.py`, `controllers/config_controller.py`, `services/config_service.py`.

**Problèmes**

- **Jeton jamais exigé** (cf. §1) : le mécanisme est inopérant pour protéger l'API.
- **Mot de passe unique, partagé, en clair** dans `.env` (`PASSWORD`), comparé avec `==`
  (`config_service.py:87`) : pas de hachage (bcrypt/argon2), comparaison non temporelle, pas de compte
  nominatif, pas de traçabilité. Le modèle `.env.exemple` contient `VotreMotDePasse`, qui risque de
  rester en production.
- **Connexion avec `null`** : si `PASSWORD` n'est pas défini, `os.getenv("PASSWORD")` renvoie `None` et
  `{"password": null}` donne un jeton valide (`None == None`). Même risque si la variable est absente
  suite à un `.env` mal chargé.
- **Aucune protection contre la force brute** sur `/config/authentificate` : tentatives illimitées,
  pas de délai, pas de verrouillage, pas de journalisation.
- **Jetons** : générés correctement (`secrets.token_hex(16)`, 128 bits) mais
  - stockés dans une liste mémoire de classe : perdus au redémarrage, non partagés entre processus /
    workers (gevent, gunicorn multi-workers) → connexions aléatoirement refusées ;
  - **sans expiration** ni durée de vie maximale ; un jeton volé reste valable jusqu'au redémarrage ;
  - recherche linéaire avec `in` (non temporelle, O(n)) et liste qui grossit sans limite à chaque
    connexion (**DoS mémoire** par connexions répétées) ;
  - `Token.sup_token` lève `ValueError` si le jeton est inconnu → 500 sur `/logout` ;
  - jeton transmis dans le **corps** JSON (`verif-authentificate`, `logout`) plutôt que dans
    `Authorization`, ce qui favorise les journaux / proxys qui le conservent.
- **Changement de mot de passe** (`modif_password`) : non protégé par jeton, pas de règle de robustesse,
  aucune réponse d'erreur (renvoie toujours `200` même si l'ancien mot de passe est faux → le front croit
  avoir réussi), `KeyError` → 500 si un champ manque, n'invalide pas les jetons existants. Le mot de passe
  est sinon accessible à tout attaquant qui a un jeton ou qui devine l'ancien mot de passe.
- Aucun cookie / session Flask (`SECRET_KEY` non utilisé) : pas de CSRF à gérer tant que le jeton n'est
  pas dans un cookie ; à reconsidérer si on passe aux cookies.

**Remédiation** : stocker un hash (`werkzeug.security.generate_password_hash`) ; jetons signés avec
expiration (`itsdangerous.URLSafeTimedSerializer`, déjà installé avec Flask, ou JWT) ; `hmac.compare_digest` ;
refuser le démarrage si `PASSWORD` est vide ; limiter les tentatives ; protéger `modif-password` par jeton.

---

## 4. Accès à la base de données

Fichiers : `database.py`, `init_db.py`, `repository/*.py`.

**Points positifs**
- Toutes les valeurs passent par des requêtes **paramétrées** (`?` ou `:nom`) : pas d'injection SQL
  détectée. Aucun nom de colonne ou de table n'est construit à partir d'une entrée utilisateur (les clés
  de filtre de `get_all` sont fixées dans le contrôleur).
- Clés étrangères activées (`PRAGMA foreign_keys = ON`).

**Problèmes**
- **Aucune fermeture de connexion** : `close_db` n'est jamais enregistrée (`app.teardown_appcontext`
  absent dans `app.py`). Chaque requête ouvre une connexion abandonnée au ramasse-miettes : fuite de
  descripteurs et verrous SQLite (`database is locked`) sous charge.
- **`get_db()` relit le `.env` à chaque appel** (`load_dotenv()` depuis `flask.cli`) et `sqlite3.connect`
  avec `DATABASE` non validé : si la variable vaut `None`, `connect(None)` lève une erreur ; si elle est
  falsifiée, l'application écrit dans un autre fichier (créé silencieusement).
- **Pas de chiffrement ni de permissions** : le `.db` contient des données personnelles en clair ;
  `.gitignore` ne contient pas `*.db` (risque de commit accidentel des données réelles).
- **Droits excessifs / pas de sauvegarde** : un seul fichier, pas de journal des suppressions.
- **Code SQL défectueux** (effets sur la sécurité : erreurs détaillées et fonctionnalité cassée) :
  - `modif_evenementRepository` : `UPDATE evenement` (table inexistante, c'est `evenements`) ; passe
    aussi `evenement.lieu` (un objet `AdresseDTO`) en paramètre ;
  - `modif_formationRepository` : `UPDATE formation` et virgule en trop avant `WHERE` ;
  - `update_visiteurRepository` : virgule en trop avant `WHERE`, et la signature attend 7 arguments
    alors que le service en passe 6 (`reorientation` manquant) → `TypeError` ;
  - `delete_*_allRepository` : remise à zéro de `sqlite_sequence` avec les noms `'evenement'` et
    `'formation'` au lieu de `'evenements'` et `'formations_iut'` (ne fait rien).
  Conséquence : les modifications (`PUT`) échouent toujours et renvoient le texte de l'exception au client
  (§8).
- **Incohérence transactionnelle** : dans `add_visiteurRepository`, l'`INSERT` du visiteur précède
  `int(formation_souhaitee_id)` ; l'erreur survient avant le `commit` mais sans `rollback` explicite.
- **Suppression** : `delete_*_by_id` fait `SELECT` puis `DELETE` sans transaction : condition de
  concurrence, et « succès » renvoyé même si la ligne a été effacée entre-temps.
- **Données en clair dans les stats** : les statistiques révèlent des petits effectifs (par ex. 1 visiteur
  en situation de handicap) — risque de ré-identification.

**Remédiation** : `app.teardown_appcontext(close_db)` ; chemin `DATABASE` validé au démarrage ; `*.db`
dans `.gitignore` ; corriger le SQL et écrire des tests ; `try/except` + `db.rollback()`.

---

## 5. Lecture et écriture de fichiers

Fichiers : `controllers/visiteurs_controller.py` (`/export`, `/export/email`),
`services/visiteurs_service.py` (`fichier_csv`, `fichier_csv_email`), `services/config_service.py`
(`set_key(".env", ...)`).

**Problèmes**
- **Pas de traversée de répertoire** : les chemins (`temporaire/visiteurs.csv`, `temporaire/email.csv`) sont
  constants, sans entrée utilisateur. Bon point.
- **Fichiers partagés entre requêtes** : le même fichier `temporaire/visiteurs.csv` est écrit puis lu par
  toutes les requêtes. Deux exports simultanés avec des filtres différents s'écrasent : un utilisateur
  peut recevoir l'export d'un autre (**fuite de données**, course critique). Utiliser un fichier temporaire
  unique ou générer le CSV en mémoire (`io.StringIO` + `Response`).
- **Données personnelles laissées sur disque** : les CSV ne sont jamais supprimés (`temporaire/` est
  ignoré par git, mais reste lisible sur le serveur) → conservation non maîtrisée, contraire au RGPD
  (minimisation, durée de conservation).
- **Chemins relatifs au répertoire courant** : `open()` dépend du `cwd`, `send_file()` de `app.root_path` ;
  lancer l'application depuis un autre dossier casse l'export ou crée `temporaire/` ailleurs.
  `os.mkdir` est testé puis créé sans protection (course `exists`/`mkdir`).
- **Encodage non précisé** (`open(..., 'w')`) : dépend de la plate-forme (échec possible avec des
  caractères accentués sous Windows).
- **Injection de formules CSV (Moyen, §6)**.
- **Écriture du `.env` à l'exécution** (`modif_password`) : `set_key(".env", ...)` écrit dans le répertoire
  courant (chemin relatif), sans verrou ni sauvegarde ; si le processus n'a pas les droits ou si le `cwd`
  est différent, le changement échoue silencieusement. L'écriture concurrente peut corrompre le fichier
  et l'application modifie son propre fichier de configuration secret.
- **Pas de limite de taille** sur l'export : tout le contenu de la table est chargé et écrit.

### 6. Injection CSV (détail)

Les champs `nom`, `prenom`, `ville`, `nom_lycee`, etc. proviennent d'un formulaire public
(`POST /visiteurs`) puis sont exportés tels quels. Une valeur telle que `=HYPERLINK("http://…","x")`
ou `=cmd|' /C calc'!A0` sera interprétée comme une formule à l'ouverture du CSV dans Excel/LibreOffice
par le personnel. **Remédiation** : préfixer par `'` toute cellule commençant par `= + - @ \t \r`.

---

## 7. Appels externes

Aucun appel sortant n'est réalisé par le code (pas de `requests`, `urllib`, `subprocess`, `os.system`,
`eval`, `pickle`, désérialisation non sûre, ni `render_template_string`). Il n'y a donc **pas de SSRF ni
d'injection de commandes** dans l'état actuel.

**Points de vigilance**
- **Dépendances inutiles** dans `requirements.txt` : `ipython`, `jupyter_*`, `nbconvert`, `tornado`,
  `pymongo`, `requests`, `tldextract`, `beautifulsoup4`, `bleach`, `PySocks`, etc. Elles étendent la
  surface d'attaque (CVE à suivre), alourdissent le déploiement et ne sont pas importées. `gevent` est
  listé mais l'application est lancée avec le serveur de développement Werkzeug. Le paquet `dotenv==0.9.9`
  est un alias de `python-dotenv` (doublon).
- **Aucune analyse des vulnérabilités** (pas de `pip-audit`, Dependabot) ; versions figées, donc à
  maintenir.
- **Dépendance de la chaîne du front** : le navigateur appelle l'API en `http://localhost:5173` →
  fonctionne en HTTP clair ; en production, tout transit (jeton, mot de passe, données personnelles) serait
  lisible sans TLS.
- Serveur : `python app.py` utilise le serveur de développement Flask. Un `FLASK_DEBUG=1` oublié en
  production ouvre le débogueur Werkzeug, donc l'exécution de code à distance. Utiliser gunicorn /
  gevent derrière un reverse proxy TLS.

---

## 8. Gestion des erreurs et fuite d'information

- Les contrôleurs renvoient `str(exception)` au client (`visiteurs_controller.py:81,109`,
  `config_controller.py:23,51,69,97`) : messages SQLite (noms de tables, requêtes), erreurs Pydantic
  avec la valeur saisie, noms d'arguments Python, etc. **Reconnaissance facilitée** pour un attaquant.
  Préfixe `Problème?` et codes `404`/`400` mal choisis.
- Les routes d'authentification et `DELETE` n'ont pas de `try/except` : une erreur provoque une 500 avec
  la page d'erreur Flask (pas de trace en production, mais comportement non maîtrisé).
- Aucun gestionnaire d'erreurs global (`@app.errorhandler`), aucun journal applicatif : impossible de
  détecter une attaque ou d'auditer les suppressions.
- La version du serveur est révélée dans les en-têtes (`Server: Werkzeug/…`).

**Remédiation** : message générique côté client, détails dans les logs ; gestionnaire d'erreurs JSON ;
journaliser les connexions (réussies et échouées) et les suppressions.

---

## 9. Secrets et configuration

Variables : `PASSWORD`, `DATABASE`, `CORS_ORIGIN` (chargées par `python-dotenv`).

**Problèmes**
- **Secret applicatif en clair** et modifié à chaud par l'application (voir §3, §5). Pas de gestionnaire
  de secrets, pas de rotation, même mot de passe pour tous les administrateurs.
- **Valeur par défaut faible** dans `.env.exemple` (`VotreMotDePasse`) ; aucune vérification au
  démarrage que le mot de passe a été changé ni qu'il est non vide.
- **Pas de contrôle des variables manquantes** : `DATABASE` ou `CORS_ORIGIN` absents provoquent des erreurs
  tardives (à la première requête) ou un comportement par défaut permissif (cf. CORS ci-dessous).
- **CORS** (`app.py:12-13`) : `CORS(app, origins=os.getenv("CORS_ORIGIN"))`. Si la variable est absente
  (`None`), le comportement de flask-cors retombe sur sa valeur par défaut, potentiellement `*`
  (toutes origines) — à vérifier et à interdire explicitement. `CORS_ORIGIN` accepte une seule origine ;
  en production, n'autoriser que le domaine HTTPS du front. CORS ne protège **pas** contre les clients
  non navigateur (curl) : il ne remplace pas l'authentification.
- **Pas de `SECRET_KEY`**, pas de `SESSION_COOKIE_*`, pas de limite de taille de requête
  (`MAX_CONTENT_LENGTH`) : un corps JSON de plusieurs centaines de Mo est accepté (DoS).
- **Hygiène du dépôt** : `.env` est bien ignoré (`.gitignore`), mais `*.db` ne l'est pas, le chemin
  `.venv` du `.gitignore` est mal préfixé (déjà signalé) et `__pycache__/` est versionné. Un historique
  git contenant un ancien `.env` ou une base doit être vérifié (`git log --all -- .env '*.db'`) ;
  lancer un scan de secrets.
- **Initialisation au chargement** : `init_db()` est exécuté à l'import de `app.py` (effet de bord,
  crée une base vide si `DATABASE` est erroné).

---

## 10. Protection des données personnelles (RGPD)

- Données collectées : identité, e-mail, téléphone, date de naissance, adresse, lycée, scolarité, et
  **information sur le handicap** (donnée de santé, catégorie particulière — art. 9). Les visiteurs
  peuvent être **mineurs**.
- Ces données sont lisibles sans authentification (§1), exportables en masse (§1, §5) et conservées
  sans durée limitée ; pas de consentement enregistré, pas de droit à l'effacement ni d'anonymisation
  automatique, pas de journal d'accès.
- Un seul fichier SQLite non chiffré, sans sauvegarde chiffrée.
- Les statistiques sur petits effectifs peuvent ré-identifier (§4).

**Remédiation** : minimiser (le handicap est-il nécessaire ?), durée de conservation, purge
automatique, consentement explicite, chiffrement au repos, contrôle d'accès strict, registre des
traitements (voir le document RGPD de `DocumentsARendre/`).

---

## 11. Disponibilité (déni de service)

- `get_all` (`visiteurs_service.py:8-26`) charge **toute** la table puis filtre et pagine en Python :
  coût O(n) par requête, sans `LIMIT`/`OFFSET` SQL. La combinaison `formation_visee` est de plus
  ignorée si aucun autre filtre n'est fourni (`visiteurs_controller.py:59-62`), donc le filtre est
  contourné silencieusement.
- `POST /visiteurs` public, sans CAPTCHA, limite de débit ni quota : remplissage de la base par un
  script.
- `DELETE /visiteurs` / `POST /authentificate` répétés : voir §1 et §3.
- Corps de requête et chaînes sans limite de taille (§2, §9).
- Liste de jetons illimitée (§3).
- SQLite sans fermeture de connexions (§4) : verrous sous charge concurrente.

**Remédiation** : pagination et filtres en SQL (`WHERE … LIMIT ? OFFSET ?`), `MAX_CONTENT_LENGTH`,
limitation de débit par IP, bornes sur `limit`, anti-spam sur le formulaire public.

---

## Plan de correction priorisé

1. **Protéger toutes les routes d'écriture, de suppression, d'export et de lecture des données
   personnelles** par un jeton vérifié (§1, §3).
2. **Sécuriser l'authentification** : hash du mot de passe, refus du démarrage sans mot de passe, limite de
   tentatives, jetons signés avec expiration, protéger `modif-password` (§3).
3. **Ne plus renvoyer les exceptions** et ajouter journalisation et gestionnaires d'erreurs (§8).
4. **Valider strictement les entrées** via Pydantic, corriger `bool("false")`, borner la pagination
   (§2).
5. **Exports** : générer en mémoire, neutraliser les formules, ne rien laisser sur disque (§5, §6).
6. **Base de données** : fermer les connexions, corriger les requêtes cassées, paginer en SQL, ajouter `*.db`
   au `.gitignore` (§4, §11).
7. **Configuration** : CORS explicite, `MAX_CONTENT_LENGTH`, serveur de production derrière TLS, nettoyer
   `requirements.txt`, lancer `pip-audit` (§7, §9).
8. **Conformité RGPD** : minimisation, rétention, consentement, chiffrement (§10).

> Toute modification de route devra aussi mettre à jour `docs/API.md` (règle du projet).
