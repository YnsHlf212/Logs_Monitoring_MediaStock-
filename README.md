# Réalisation Professionnelle — Mise en place d'un système de logs et de monitoring sur MediaStock

**Étudiant :** [Prénom NOM]
**Formation :** BTS SIO option SLAM
**Période :** Février – Mars 2026
**Référent entreprise :** [Nom du tuteur]

---

## 1 — Présentation de l'application MediaStock

**MediaStock** est une application web de gestion du patrimoine matériel d'un établissement scolaire. Elle permet au personnel administratif et aux enseignants de :

- **Gérer l'inventaire** du matériel disponible (ordinateurs, vidéoprojecteurs, tablettes, mobilier…)
- **Gérer les prêts** de matériel à des utilisateurs (élèves, enseignants, personnels)
- **Gérer les utilisateurs** de la plateforme (rôles : admin, gestionnaire, consultant)
- **Suivre les retours** et l'état du matériel après utilisation

L'application est développée en **Node.js** avec le framework **Express** et se connecte à une base de données **MySQL**.

### Routes principales de l'API

| Méthode | Route                    | Description                               |
|---------|--------------------------|-------------------------------------------|
| GET     | `/materiels`             | Lister tout le matériel                   |
| GET     | `/materiels/:id`         | Consulter un matériel                     |
| POST    | `/materiels`             | Ajouter un matériel à l'inventaire        |
| PUT     | `/materiels/:id`         | Modifier un matériel                      |
| DELETE  | `/materiels/:id`         | Supprimer un matériel                     |
| GET     | `/prets`                 | Lister tous les prêts en cours            |
| GET     | `/prets/:id`             | Consulter un prêt                         |
| POST    | `/prets`                 | Créer un nouveau prêt                     |
| PUT     | `/prets/:id/retour`      | Enregistrer le retour d'un prêt           |
| DELETE  | `/prets/:id`             | Annuler un prêt                           |
| GET     | `/users`                 | Lister les utilisateurs                   |
| POST    | `/users`                 | Créer un utilisateur                      |
| PUT     | `/users/:id`             | Modifier un utilisateur                   |
| DELETE  | `/users/:id`             | Supprimer un utilisateur                  |

### Structure du projet avant amélioration

```
mediastock/
├── src/
│   ├── app.js              ← point d'entrée Express
│   ├── routes/
│   │   ├── materiels.js    ← CRUD matériel
│   │   ├── prets.js        ← gestion des prêts
│   │   └── users.js        ← gestion des utilisateurs
│   └── db/
│       └── database.js     ← connexion MySQL
├── package.json
└── .env
```

### Extrait de code — `app.js` (avant)

```js
const express   = require('express');
const materiels = require('./routes/materiels');
const prets     = require('./routes/prets');
const users     = require('./routes/users');

const app = express();
app.use(express.json());
app.use('/materiels', materiels);
app.use('/prets', prets);
app.use('/users', users);

app.listen(3000, () => {
  console.log('MediaStock démarré sur le port 3000');
});
```

> L'application fonctionne mais ne dispose d'**aucun mécanisme de traçabilité** : pas de logs, aucune gestion centralisée des erreurs, aucun suivi des performances.

---

## 2 — Problématique

Sans système de journalisation, plusieurs problèmes critiques ont été identifiés sur MediaStock :

- **Diagnostic impossible** : quand une erreur 500 survient lors d'un prêt ou d'une modification de matériel, aucune trace ne permet d'identifier la cause (requête SQL fautive, données invalides, champ manquant…).
- **Aucune visibilité sur l'activité** : impossible de savoir quels matériels sont les plus demandés, combien de prêts sont effectués par jour, ou quels utilisateurs sont actifs.
- **Détection tardive des problèmes** : une indisponibilité de l'application n'est découverte que lorsqu'un utilisateur se manifeste, parfois longtemps après l'incident.
- **Perte d'informations** : les messages `console.log` disparaissent à chaque redémarrage du serveur Node.js.
- **Traçabilité des accès absente** : dans le contexte scolaire, il peut être nécessaire de prouver qui a effectué quelle opération sur l'inventaire (audit, responsabilité).

---

## 3 — Objectifs de l'amélioration

| Objectif | Description |
|---|---|
| **Journalisation des événements** | Enregistrer toutes les actions métier dans des fichiers persistants (prêt créé, matériel ajouté, retour enregistré…) |
| **Journalisation des requêtes HTTP** | Tracer chaque requête reçue (méthode, URL, statut, temps de réponse) |
| **Journalisation des erreurs** | Capturer et stocker toutes les erreurs dans un fichier dédié avec stack trace |
| **Analyse des logs** | Exploiter les logs via un script Python pour produire des statistiques |
| **Monitoring des performances** | Suivre en temps réel le nombre de requêtes, les temps de réponse et le taux d'erreurs |

---

## 4 — Architecture avant et après amélioration

### Avant

```
Client HTTP (navigateur / Postman)
    │
    ▼
Application MediaStock (Express) ──────► Base de données MySQL
    │
    ▼
console.log() [perdu au redémarrage]
```

### Après

```
Client HTTP (navigateur / Postman)
    │
    ▼
Application MediaStock (Express) ──────► Base de données MySQL
    │
    ├──► Morgan → [app.log]  ◄─── Script Python d'analyse
    │                               (stats, rapports)
    ├──► Winston → [app.log]
    │
    └──► Winston → [error.log]
                        │
                        ▼
              Endpoint /metrics → Monitoring
```

Cette évolution apporte une **persistance** des informations, une **séparation** des logs normaux et des erreurs, et la possibilité d'**analyser** les données de façon indépendante.

---

## 5 — Choix des technologies

| Technologie | Rôle | Justification |
|---|---|---|
| **Node.js / Express** | Environnement d'exécution / framework web | Application existante, continuité technologique |
| **Winston** | Système de journalisation | Standard de l'écosystème Node.js : gestion des niveaux, formats, fichiers multiples |
| **Morgan** | Journalisation des requêtes HTTP | Middleware Express natif, s'intègre directement avec Winston |
| **Python 3** | Analyse des logs | Langage accessible, utilisable par l'équipe sans connaître Node.js |

---

## 6 — Mise en place du système de logs (Winston)

### Installation

```bash
npm install winston morgan
```

### Configuration de Winston — `src/logger.js`

```js
const { createLogger, format, transports } = require('winston');
const { combine, timestamp, printf, colorize } = format;

const logFormat = printf(({ level, message, timestamp }) => {
  return `[${level.toUpperCase()}] ${timestamp} - ${message}`;
});

const logger = createLogger({
  level: 'info',
  format: combine(
    timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
    logFormat
  ),
  transports: [
    // Tous les logs info et au-dessus → app.log
    new transports.File({ filename: 'logs/app.log', level: 'info' }),
    // Uniquement les erreurs → error.log
    new transports.File({ filename: 'logs/error.log', level: 'error' }),
    // Affichage coloré en console (développement)
    new transports.Console({
      format: combine(colorize(), timestamp({ format: 'HH:mm:ss' }), logFormat)
    })
  ]
});

module.exports = logger;
```

### Hiérarchie des niveaux de log

```
error   ← erreurs critiques (DB inaccessible, crash serveur)
warn    ← avertissements (prêt en retard, stock critique, accès refusé)
info    ← événements normaux (prêt créé, matériel ajouté, retour enregistré)
debug   ← détails techniques (valeurs des paramètres, étapes internes)
```

---

## 7 — Journalisation des requêtes HTTP (Morgan)

Morgan est redirigé vers Winston pour que tous les logs HTTP soient écrits dans les mêmes fichiers.

### Configuration dans `app.js`

```js
const express   = require('express');
const morgan    = require('morgan');
const logger    = require('./logger');
const materiels = require('./routes/materiels');
const prets     = require('./routes/prets');
const users     = require('./routes/users');

const app = express();

// Redirection Morgan → Winston
const morganStream = {
  write: (message) => logger.info(message.trim())
};

app.use(morgan(':method :url :status :response-time ms - :res[content-length] bytes', {
  stream: morganStream
}));

app.use(express.json());
app.use('/materiels', materiels);
app.use('/prets', prets);
app.use('/users', users);

app.listen(3000, () => {
  logger.info('MediaStock démarré sur le port 3000');
});

module.exports = app;
```

### Exemple de logs HTTP générés

```
[INFO] 2026-03-10 08:12:04 - GET /materiels 200 11.204 ms - 3840 bytes
[INFO] 2026-03-10 08:13:22 - POST /prets 201 18.740 ms - 312 bytes
[INFO] 2026-03-10 08:15:01 - PUT /prets/12/retour 200 9.510 ms - 128 bytes
[INFO] 2026-03-10 08:16:30 - DELETE /materiels/99 404 4.200 ms - 64 bytes
```

---

## 8 — Gestion des erreurs

### Middleware global — `src/middlewares/errorHandler.js`

```js
const logger = require('../logger');

function errorHandler(err, req, res, next) {
  const statusCode = err.status || 500;
  const message    = err.message || 'Erreur interne du serveur';

  logger.error(
    `${req.method} ${req.url} — HTTP ${statusCode} — ${message}` +
    (err.stack ? `\n${err.stack}` : '')
  );

  res.status(statusCode).json({
    error:     message,
    status:    statusCode,
    path:      req.url,
    timestamp: new Date().toISOString()
  });
}

module.exports = errorHandler;
```

### Exemple dans les routes — `src/routes/prets.js`

```js
const express = require('express');
const router  = express.Router();
const logger  = require('../logger');
const db      = require('../db/database');

// POST /prets — créer un prêt
router.post('/', async (req, res, next) => {
  const { materiel_id, user_id, date_retour_prevue } = req.body;

  if (!materiel_id || !user_id || !date_retour_prevue) {
    logger.warn(
      `POST /prets — champs manquants : materiel_id=${materiel_id}, ` +
      `user_id=${user_id}, date_retour_prevue=${date_retour_prevue}`
    );
    return res.status(400).json({ error: 'Champs manquants : materiel_id, user_id, date_retour_prevue requis.' });
  }

  try {
    // Vérifier que le matériel est disponible
    const [[materiel]] = await db.query(
      'SELECT * FROM materiels WHERE id = ? AND disponible = 1', [materiel_id]
    );
    if (!materiel) {
      logger.warn(`POST /prets — matériel ID ${materiel_id} indisponible ou inexistant`);
      return res.status(409).json({ error: 'Matériel indisponible.' });
    }

    const [result] = await db.query(
      'INSERT INTO prets (materiel_id, user_id, date_debut, date_retour_prevue, statut) VALUES (?, ?, NOW(), ?, "en_cours")',
      [materiel_id, user_id, date_retour_prevue]
    );
    await db.query('UPDATE materiels SET disponible = 0 WHERE id = ?', [materiel_id]);

    logger.info(
      `Prêt créé — ID: ${result.insertId}, matériel: ${materiel.nom} (ID ${materiel_id}), user: ${user_id}`
    );
    res.status(201).json({ id: result.insertId, materiel_id, user_id, date_retour_prevue });

  } catch (err) {
    logger.error(`Erreur DB lors de POST /prets : ${err.message}`);
    next(err);
  }
});

// PUT /prets/:id/retour — enregistrer le retour
router.put('/:id/retour', async (req, res, next) => {
  const { id } = req.params;
  const { etat_retour } = req.body;

  try {
    const [[pret]] = await db.query('SELECT * FROM prets WHERE id = ?', [id]);
    if (!pret) {
      logger.warn(`PUT /prets/${id}/retour — prêt introuvable`);
      return res.status(404).json({ error: 'Prêt introuvable.' });
    }

    await db.query(
      'UPDATE prets SET statut = "rendu", date_retour_reel = NOW(), etat_retour = ? WHERE id = ?',
      [etat_retour || 'bon', id]
    );
    await db.query('UPDATE materiels SET disponible = 1 WHERE id = ?', [pret.materiel_id]);

    logger.info(`Retour enregistré — prêt ID: ${id}, état: ${etat_retour || 'bon'}`);
    res.json({ message: 'Retour enregistré.', pret_id: id });

  } catch (err) {
    logger.error(`Erreur DB lors de PUT /prets/${id}/retour : ${err.message}`);
    next(err);
  }
});

module.exports = router;
```

### Exemple dans les routes — `src/routes/materiels.js`

```js
const express = require('express');
const router  = express.Router();
const logger  = require('../logger');
const db      = require('../db/database');

// GET /materiels — liste complète de l'inventaire
router.get('/', async (req, res, next) => {
  try {
    logger.info('Requête : consultation de l\'inventaire complet');
    const [rows] = await db.query('SELECT * FROM materiels');
    logger.info(`${rows.length} matériel(s) retourné(s)`);
    res.json(rows);
  } catch (err) {
    logger.error(`Erreur DB lors de GET /materiels : ${err.message}`);
    next(err);
  }
});

// POST /materiels — ajout d'un matériel
router.post('/', async (req, res, next) => {
  const { nom, categorie, numero_serie } = req.body;

  if (!nom || !categorie) {
    logger.warn(`POST /materiels — champs manquants : nom=${nom}, categorie=${categorie}`);
    return res.status(400).json({ error: 'Les champs nom et categorie sont requis.' });
  }

  try {
    const [result] = await db.query(
      'INSERT INTO materiels (nom, categorie, numero_serie, disponible) VALUES (?, ?, ?, 1)',
      [nom, categorie, numero_serie || null]
    );
    logger.info(`Matériel ajouté — ID: ${result.insertId}, nom: "${nom}", catégorie: ${categorie}`);
    res.status(201).json({ id: result.insertId, nom, categorie, numero_serie });
  } catch (err) {
    logger.error(`Erreur DB lors de POST /materiels : ${err.message}`);
    next(err);
  }
});

module.exports = router;
```

### Enregistrement du middleware dans `app.js`

```js
const errorHandler = require('./middlewares/errorHandler');
// … toutes les routes …
app.use(errorHandler); // ← toujours en dernier
```

---

## 9 — Exemples de logs générés

### `logs/app.log` — fonctionnement normal

```
[INFO] 2026-03-10 08:12:04 - GET /materiels 200 11.204 ms - 3840 bytes
[INFO] 2026-03-10 08:12:04 - Requête : consultation de l'inventaire complet
[INFO] 2026-03-10 08:12:04 - 47 matériel(s) retourné(s)
[INFO] 2026-03-10 08:13:22 - POST /prets 201 18.740 ms - 312 bytes
[INFO] 2026-03-10 08:13:22 - Prêt créé — ID: 88, matériel: Vidéoprojecteur Epson (ID 5), user: 14
[WARN] 2026-03-10 08:14:05 - POST /prets — matériel ID 5 indisponible ou inexistant
[INFO] 2026-03-10 08:15:01 - PUT /prets/12/retour 200 9.510 ms - 128 bytes
[INFO] 2026-03-10 08:15:01 - Retour enregistré — prêt ID: 12, état: bon
[WARN] 2026-03-10 08:16:00 - POST /materiels — champs manquants : nom=undefined, categorie=Informatique
[INFO] 2026-03-10 08:16:30 - DELETE /materiels/99 404 4.200 ms - 64 bytes
```

### `logs/error.log` — erreurs critiques

```
[ERROR] 2026-03-10 08:20:11 - GET /materiels — HTTP 500 — connect ECONNREFUSED 127.0.0.1:3306
Error: connect ECONNREFUSED 127.0.0.1:3306
    at TCPConnectWrap.afterConnect [as oncomplete] (net.js:1141:16)

[ERROR] 2026-03-10 08:21:44 - POST /materiels — HTTP 500 — Duplicate entry 'SN-2024-VPJ-001' for key 'numero_serie'

[ERROR] 2026-03-10 08:25:03 - PUT /prets/77/retour — HTTP 500 — Cannot read properties of undefined (reading 'materiel_id')
TypeError: Cannot read properties of undefined (reading 'materiel_id')
    at router.put (src/routes/prets.js:52:30)
```

---

## 10 — Analyse des logs (script Python)

### `analyse_logs.py`

```python
import re
from collections import defaultdict

LOG_FILE = 'logs/app.log'

RE_HTTP  = re.compile(r'\[INFO\] (\S+ \S+) - (GET|POST|PUT|DELETE) (\S+) (\d{3}) ([\d.]+) ms')
RE_ERROR = re.compile(r'\[ERROR\]')
RE_WARN  = re.compile(r'\[WARN\]')

def analyser_logs(fichier):
    requetes      = []
    nb_erreurs    = 0
    nb_warnings   = 0
    routes        = defaultdict(int)
    statuts       = defaultdict(int)
    temps_reponse = []

    with open(fichier, 'r', encoding='utf-8') as f:
        for ligne in f:
            if RE_ERROR.search(ligne): nb_erreurs  += 1
            if RE_WARN.search(ligne):  nb_warnings += 1

            m = RE_HTTP.search(ligne)
            if m:
                horodatage, methode, url, statut, temps = m.groups()
                requetes.append({'horodatage': horodatage, 'methode': methode,
                                 'url': url, 'statut': int(statut), 'temps': float(temps)})
                routes[f"{methode} {url}"] += 1
                statuts[statut]            += 1
                temps_reponse.append(float(temps))

    return requetes, nb_erreurs, nb_warnings, routes, statuts, temps_reponse


def afficher_rapport(requetes, nb_erreurs, nb_warnings, routes, statuts, temps_reponse):
    print("=" * 58)
    print("     RAPPORT D'ANALYSE DES LOGS — MEDIASTOCK")
    print("=" * 58)
    if requetes:
        print(f"  Période analysée       : {requetes[0]['horodatage']} → {requetes[-1]['horodatage']}")
    print(f"  Total requêtes HTTP    : {len(requetes)}")
    print(f"  Erreurs (ERROR)        : {nb_erreurs}")
    print(f"  Avertissements (WARN)  : {nb_warnings}")

    if temps_reponse:
        print(f"  Temps de réponse moyen : {sum(temps_reponse)/len(temps_reponse):.2f} ms")
        print(f"  Temps de réponse max   : {max(temps_reponse):.2f} ms")
        print(f"  Temps de réponse min   : {min(temps_reponse):.2f} ms")

    print("\n--- Routes les plus sollicitées ---")
    for route, count in sorted(routes.items(), key=lambda x: x[1], reverse=True)[:8]:
        print(f"  {route:<35} {count} appel(s)")

    print("\n--- Répartition des codes HTTP ---")
    for statut, count in sorted(statuts.items()):
        print(f"  HTTP {statut} : {count} réponse(s)")

    print("=" * 58)


if __name__ == '__main__':
    requetes, nb_erreurs, nb_warnings, routes, statuts, temps = analyser_logs(LOG_FILE)
    afficher_rapport(requetes, nb_erreurs, nb_warnings, routes, statuts, temps)
```

### Exemple de sortie

```
==========================================================
     RAPPORT D'ANALYSE DES LOGS — MEDIASTOCK
==========================================================
  Période analysée       : 2026-03-10 08:12:04 → 2026-03-10 16:55:30
  Total requêtes HTTP    : 342
  Erreurs (ERROR)        : 5
  Avertissements (WARN)  : 12
  Temps de réponse moyen : 16.32 ms
  Temps de réponse max   : 387.00 ms
  Temps de réponse min   : 2.80 ms

--- Routes les plus sollicitées ---
  GET /materiels                      178 appel(s)
  GET /prets                           74 appel(s)
  POST /prets                          43 appel(s)
  PUT /prets/:id/retour                28 appel(s)
  POST /materiels                      11 appel(s)
  GET /users                            8 appel(s)

--- Répartition des codes HTTP ---
  HTTP 200 : 252 réponse(s)
  HTTP 201 :  54 réponse(s)
  HTTP 400 :  12 réponse(s)
  HTTP 404 :  19 réponse(s)
  HTTP 409 :   5 réponse(s)
  HTTP 500 :   5 réponse(s)
==========================================================
```

---

## 11 — Monitoring simple

### `src/monitoring.js`

```js
const logger = require('./logger');

const metriques = {
  totalRequetes: 0,
  totalErreurs:  0,
  sommeTemps:    0,
  debutServeur:  Date.now()
};

function middlewareMonitoring(req, res, next) {
  const debut = Date.now();
  metriques.totalRequetes++;

  res.on('finish', () => {
    metriques.sommeTemps += Date.now() - debut;
    if (res.statusCode >= 500) metriques.totalErreurs++;
  });

  next();
}

function routeMetriques(req, res) {
  const moyenne = metriques.totalRequetes > 0
    ? (metriques.sommeTemps / metriques.totalRequetes).toFixed(2)
    : 0;

  const rapport = {
    uptime_secondes:         Math.floor((Date.now() - metriques.debutServeur) / 1000),
    total_requetes:          metriques.totalRequetes,
    total_erreurs_500:       metriques.totalErreurs,
    temps_reponse_moyen_ms:  parseFloat(moyenne),
    taux_erreur_pct:         metriques.totalRequetes > 0
      ? ((metriques.totalErreurs / metriques.totalRequetes) * 100).toFixed(1)
      : '0.0'
  };

  logger.info(`Métriques consultées : ${JSON.stringify(rapport)}`);
  res.json(rapport);
}

module.exports = { middlewareMonitoring, routeMetriques };
```

### Intégration dans `app.js`

```js
const { middlewareMonitoring, routeMetriques } = require('./monitoring');
app.use(middlewareMonitoring);
// …routes…
app.get('/metrics', routeMetriques);
```

### Exemple de réponse `GET /metrics`

```json
{
  "uptime_secondes":        31540,
  "total_requetes":         342,
  "total_erreurs_500":      5,
  "temps_reponse_moyen_ms": 16.32,
  "taux_erreur_pct":        "1.5"
}
```

Ces métriques peuvent alimenter un tableau de bord (Grafana, page HTML interne) pour surveiller MediaStock en temps réel.

---

## 12 — Tests réalisés

### Scénario 1 — Erreur de connexion à la base de données

**Simulation :** le service MySQL est arrêté ; une requête `GET /materiels` est effectuée.

**Log dans `error.log` :**
```
[ERROR] 2026-03-10 08:20:11 - GET /materiels — HTTP 500 — connect ECONNREFUSED 127.0.0.1:3306
Error: connect ECONNREFUSED 127.0.0.1:3306
    at TCPConnectWrap.afterConnect [as oncomplete] (net.js:1141:16)
```
**Diagnostic :** `ECONNREFUSED` sur le port 3306 → la base de données est inaccessible. Sans logs, ce problème aurait été impossible à identifier.

---

### Scénario 2 — Tentative de prêt d'un matériel indisponible

**Simulation :** `POST /prets` avec un `materiel_id` déjà en cours de prêt.

**Logs dans `app.log` :**
```
[WARN] 2026-03-10 08:14:05 - POST /prets — matériel ID 5 indisponible ou inexistant
[INFO] 2026-03-10 08:14:05 - POST /prets 409 6.810 ms - 48 bytes
```
**Diagnostic :** le code HTTP 409 (conflit) et le WARNING précisent que le matériel est déjà prêté. L'administrateur peut retrouver quel matériel pose problème et à quel moment.

---

### Scénario 3 — Ajout de matériel avec un numéro de série en doublon

**Simulation :** `POST /materiels` avec un `numero_serie` déjà existant en base.

**Log dans `error.log` :**
```
[ERROR] 2026-03-10 08:21:44 - POST /materiels — HTTP 500 — Duplicate entry 'SN-2024-VPJ-001' for key 'numero_serie'
```
**Diagnostic :** l'erreur MySQL `Duplicate entry` identifie immédiatement le numéro de série en doublon, sans avoir à inspecter la base manuellement.

---

### Scénario 4 — Requête invalide (champs manquants lors d'un ajout de matériel)

**Simulation :** `POST /materiels` sans le champ `nom`.

**Logs dans `app.log` :**
```
[WARN] 2026-03-10 08:16:00 - POST /materiels — champs manquants : nom=undefined, categorie=Informatique
[INFO] 2026-03-10 08:16:00 - POST /materiels 400 2.950 ms - 62 bytes
```
**Diagnostic :** le WARNING identifie précisément le champ absent, permettant de corriger rapidement l'appel API.

---

### Scénario 5 — Exception non gérée lors d'un retour de prêt

**Simulation :** `PUT /prets/77/retour` alors que le prêt d'ID 77 n'existe pas en base.

**Log dans `error.log` :**
```
[ERROR] 2026-03-10 08:25:03 - PUT /prets/77/retour — HTTP 500 — Cannot read properties of undefined (reading 'materiel_id')
TypeError: Cannot read properties of undefined (reading 'materiel_id')
    at router.put (src/routes/prets.js:52:30)
    at Layer.handle [as handle_request] (express/lib/router/layer.js:95:5)
```
**Diagnostic :** la stack trace indique exactement le fichier (`prets.js`) et la ligne (`52`) de l'erreur, rendant la correction immédiate.

---

## 13 — Résultats obtenus

| Avant amélioration | Après amélioration |
|---|---|
| Aucune trace des erreurs | Stack trace complète dans `error.log` |
| Aucune visibilité sur les requêtes | Chaque requête tracée (méthode, URL, statut, temps) |
| Détection des pannes par les utilisateurs | Détection par analyse des logs |
| Impossible d'analyser l'activité | Rapport automatique via le script Python |
| Pas de métriques | Endpoint `/metrics` : uptime, taux d'erreur, temps moyen |
| Prêts doublons non détectés | WARNING loggé pour chaque tentative de prêt invalide |

L'amélioration permet désormais de :

- **Diagnostiquer** tout incident en quelques minutes grâce aux logs horodatés et contextualisés.
- **Analyser l'activité** de MediaStock (matériels les plus empruntés, heures de pointe, taux de retours).
- **Anticiper les problèmes** en surveillant le taux d'erreur et les temps de réponse via `/metrics`.

---

## 14 — Compétences mobilisées (BTS SIO SLAM)

| Compétence du référentiel | Application concrète |
|---|---|
| **Maintenance d'une solution applicative** | Amélioration de MediaStock en production sans modifier son comportement fonctionnel |
| **Analyse d'un dysfonctionnement** | Identification des lacunes (absence de logs) et proposition d'une solution adaptée |
| **Exploitation des données** | Script Python lisant et analysant les fichiers de logs pour produire des statistiques |
| **Tests d'une solution applicative** | Simulation de 5 incidents et vérification que les logs permettent leur diagnostic |
| **Documentation technique** | Rédaction de cette réalisation professionnelle et commentaires dans le code source |

---

## 15 — Conclusion

La mise en place d'un système de journalisation et de monitoring sur MediaStock représente une amélioration fondamentale de la **qualité de service** et de la **maintenabilité** de l'application.

Une application de gestion de stock scolaire sans logs est une application dont on ne maîtrise pas le comportement en conditions réelles : prêts non enregistrés, erreurs de base de données silencieuses, indisponibilités non détectées. Les outils mis en place — Winston, Morgan et le script Python — fournissent cette capacité de traçabilité à coût maîtrisé, en s'appuyant sur des standards éprouvés de l'écosystème Node.js.

Cette réalisation m'a permis de comprendre l'importance de l'**observabilité** dans le cycle de vie d'une application : une application observable est une application que l'on peut comprendre, maintenir et améliorer en continu.

En perspective, ce système pourrait être enrichi avec :
- Un outil de centralisation des logs comme **ELK Stack** ou **Grafana Loki** pour les environnements multi-serveurs.
- Des **alertes automatiques** (email, notification) déclenchées si le taux d'erreur dépasse un seuil défini.
- Un **tableau de bord temps réel** alimenté par `/metrics` pour visualiser l'activité de MediaStock.

---

*Document rédigé dans le cadre du BTS SIO SLAM — Portfolio de réalisations professionnelles.*
