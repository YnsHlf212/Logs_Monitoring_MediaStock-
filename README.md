# Réalisation Professionnelle — Mise en place d'un système de logs et de monitoring

**Étudiant :** [Prénom NOM]
**Formation :** BTS SIO option SLAM
**Entreprise :** TechSolutions SAS
**Période :** Février – Mars 2026
**Référent entreprise :** [Nom du tuteur]

---

## 1 — Contexte professionnel

**TechSolutions SAS** est une PME de 35 personnes spécialisée dans le développement de solutions web et mobiles pour des clients du secteur tertiaire. L'entreprise exploite en production une API REST de gestion d'utilisateurs, utilisée par plusieurs applications clientes internes.

Depuis sa mise en production, l'API fonctionne sans aucun système de journalisation. Lorsque des incidents surviennent — indisponibilité, erreurs 500, lenteurs — l'équipe de développement est dans l'incapacité de diagnostiquer rapidement la cause : elle ne dispose d'aucune trace des événements passés, des requêtes effectuées, ni des erreurs survenues.

À la suite d'un incident critique ayant provoqué une indisponibilité de l'API pendant deux heures sans qu'aucun développeur n'ait pu en identifier la cause, la direction technique a demandé la mise en place urgente d'un **système de logs et de monitoring** sur cette application.

**Ma mission**, en tant que développeur junior en alternance, est d'améliorer l'application existante en y intégrant un système complet de journalisation, de gestion des erreurs et de monitoring basique des performances.

---

## 2 — Présentation de l'application existante

L'application est une **API REST** développée en **Node.js** avec le framework **Express**. Elle expose des routes CRUD pour la gestion des utilisateurs et se connecte à une base de données **MySQL** via le module `mysql2`.

### Routes principales

| Méthode | Route          | Description                     |
|---------|----------------|---------------------------------|
| GET     | `/users`       | Récupérer la liste des utilisateurs |
| GET     | `/users/:id`   | Récupérer un utilisateur par ID |
| POST    | `/users`       | Créer un nouvel utilisateur     |
| PUT     | `/users/:id`   | Modifier un utilisateur         |
| DELETE  | `/users/:id`   | Supprimer un utilisateur        |

### Structure du projet avant amélioration

```
api-users/
├── src/
│   ├── app.js          ← point d'entrée Express
│   ├── routes/
│   │   └── users.js    ← routes CRUD
│   └── db/
│       └── database.js ← connexion MySQL
├── package.json
└── .env
```

### Extrait de code — `app.js` (avant)

```js
const express = require('express');
const usersRouter = require('./routes/users');

const app = express();
app.use(express.json());
app.use('/users', usersRouter);

app.listen(3000, () => {
  console.log('Serveur démarré sur le port 3000');
});
```

> L'application fonctionne mais ne dispose d'**aucun mécanisme de traçabilité** : pas de logs, pas de gestion centralisée des erreurs, pas de suivi des performances.

---

## 3 — Problématique

Sans système de journalisation, plusieurs problèmes critiques ont été identifiés :

- **Diagnostic impossible** : lors d'une erreur 500, aucune trace n'indique la ligne de code fautive, les paramètres de la requête, ni le contexte d'exécution.
- **Aucune visibilité sur l'activité** : impossible de savoir combien de requêtes sont reçues, quelles routes sont les plus sollicitées, quels utilisateurs sont actifs.
- **Détection tardive des incidents** : les pannes ne sont découvertes que lorsqu'un utilisateur se manifeste, parfois des heures après l'incident réel.
- **Perte d'informations** : chaque redémarrage du processus Node.js efface les messages `console.log` qui auraient pu aider au diagnostic.
- **Non-conformité** : certains contrats clients exigent une traçabilité des accès aux données (RGPD, audits internes).

---

## 4 — Objectifs de l'amélioration

| Objectif | Description |
|---|---|
| **Journalisation des événements** | Enregistrer toutes les actions significatives dans des fichiers de logs persistants |
| **Journalisation des requêtes HTTP** | Tracer chaque requête reçue (méthode, URL, statut, temps de réponse) |
| **Journalisation des erreurs** | Capturer et stocker toutes les erreurs dans un fichier dédié |
| **Analyse des logs** | Exploiter les logs via un script pour produire des statistiques |
| **Monitoring des performances** | Suivre le nombre de requêtes, les temps de réponse et les erreurs |

---

## 5 — Architecture avant et après amélioration

### Avant

```
Client HTTP
    │
    ▼
Application Express ──────► Base de données MySQL
    │
    ▼
console.log() [perdu au redémarrage]
```

### Après

```
Client HTTP
    │
    ▼
Application Express ──────► Base de données MySQL
    │
    ├──► Morgan → [app.log]  ◄─── Script Python d'analyse
    │                               (stats, rapports)
    ├──► Winston → [app.log]
    │
    └──► Winston → [error.log]
                        │
                        ▼
              Monitoring / Alertes
```

Cette évolution apporte une **persistance** des informations, une **séparation** des logs normaux et des erreurs, et la possibilité d'**analyser** les données en dehors de l'application.

---

## 6 — Choix des technologies

| Technologie | Rôle | Justification |
|---|---|---|
| **Node.js / Express** | Environnement d'exécution / framework web | Application existante, continuité technologique |
| **Winston** | Système de journalisation | Bibliothèque la plus utilisée dans l'écosystème Node.js. Gère les niveaux de logs, les formats, les transports (fichiers, console, etc.) |
| **Morgan** | Journalisation des requêtes HTTP | Middleware Express dédié au logging HTTP, s'intègre nativement avec Winston |
| **Python 3** | Analyse des logs | Langage polyvalent, bibliothèque standard suffisante. Utilisable indépendamment de l'application Node.js |

Winston et Morgan sont des standards de l'industrie pour les applications Express. Le script Python permet à une équipe DevOps ou QA d'analyser les logs sans connaître Node.js.

---

## 7 — Mise en place du système de logs (Winston)

### Installation

```bash
npm install winston morgan
```

### Configuration de Winston — `src/logger.js`

```js
const { createLogger, format, transports } = require('winston');
const { combine, timestamp, printf, colorize } = format;

// Format personnalisé des messages de log
const logFormat = printf(({ level, message, timestamp }) => {
  return `[${level.toUpperCase()}] ${timestamp} - ${message}`;
});

const logger = createLogger({
  level: 'info', // niveau minimum enregistré
  format: combine(
    timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
    logFormat
  ),
  transports: [
    // Tous les logs (info et au-dessus) → app.log
    new transports.File({
      filename: 'logs/app.log',
      level: 'info'
    }),
    // Uniquement les erreurs → error.log
    new transports.File({
      filename: 'logs/error.log',
      level: 'error'
    }),
    // Affichage coloré dans la console (développement)
    new transports.Console({
      format: combine(colorize(), timestamp({ format: 'HH:mm:ss' }), logFormat)
    })
  ]
});

module.exports = logger;
```

### Hiérarchie des niveaux de log (Winston)

```
error   ← problèmes critiques (DB down, crash)
warn    ← avertissements (token invalide, accès refusé)
info    ← informations normales (requête reçue, utilisateur créé)
debug   ← détails techniques (valeurs de variables, étapes internes)
```

---

## 8 — Journalisation des requêtes HTTP (Morgan)

Morgan est configuré pour utiliser Winston comme destination, garantissant que les logs HTTP sont écrits dans les mêmes fichiers que les autres événements.

### Configuration dans `app.js`

```js
const express = require('express');
const morgan  = require('morgan');
const logger  = require('./logger');
const usersRouter = require('./routes/users');

const app = express();

// Stream Morgan → Winston
const morganStream = {
  write: (message) => logger.info(message.trim())
};

// Format Morgan : méthode URL statut temps-réponse taille-réponse
app.use(morgan(':method :url :status :response-time ms - :res[content-length] bytes', {
  stream: morganStream
}));

app.use(express.json());
app.use('/users', usersRouter);

app.listen(3000, () => {
  logger.info('Serveur démarré sur le port 3000');
});

module.exports = app;
```

### Exemple de log HTTP généré

```
[INFO] 2026-02-10 10:25:03 - GET /users 200 14.352 ms - 1024 bytes
[INFO] 2026-02-10 10:25:11 - POST /users 201 22.180 ms - 256 bytes
[INFO] 2026-02-10 10:26:05 - DELETE /users/42 404 5.100 ms - 48 bytes
```

---

## 9 — Gestion des erreurs

### Middleware de gestion globale des erreurs — `src/middlewares/errorHandler.js`

```js
const logger = require('../logger');

/**
 * Middleware Express de gestion globale des erreurs.
 * Doit être déclaré en DERNIER dans app.js (après toutes les routes).
 */
function errorHandler(err, req, res, next) {
  const statusCode = err.status || 500;
  const message    = err.message || 'Erreur interne du serveur';

  // Enregistrement dans error.log avec contexte complet
  logger.error(
    `${req.method} ${req.url} — HTTP ${statusCode} — ${message}` +
    (err.stack ? `\n${err.stack}` : '')
  );

  res.status(statusCode).json({
    error:   message,
    status:  statusCode,
    path:    req.url,
    timestamp: new Date().toISOString()
  });
}

module.exports = errorHandler;
```

### Utilisation dans les routes — `src/routes/users.js`

```js
const express = require('express');
const router  = express.Router();
const logger  = require('../logger');
const db      = require('../db/database');

// GET /users — liste de tous les utilisateurs
router.get('/', async (req, res, next) => {
  try {
    logger.info('Requête : récupération de la liste des utilisateurs');
    const [rows] = await db.query('SELECT * FROM users');
    logger.info(`${rows.length} utilisateur(s) retourné(s)`);
    res.json(rows);
  } catch (err) {
    logger.error(`Erreur DB lors de GET /users : ${err.message}`);
    next(err); // transmission au middleware global
  }
});

// POST /users — création d'un utilisateur
router.post('/', async (req, res, next) => {
  const { nom, email } = req.body;

  if (!nom || !email) {
    logger.warn(`POST /users — champs manquants : nom=${nom}, email=${email}`);
    return res.status(400).json({ error: 'Les champs nom et email sont requis.' });
  }

  try {
    const [result] = await db.query(
      'INSERT INTO users (nom, email) VALUES (?, ?)', [nom, email]
    );
    logger.info(`Utilisateur créé — ID: ${result.insertId}, email: ${email}`);
    res.status(201).json({ id: result.insertId, nom, email });
  } catch (err) {
    logger.error(`Erreur DB lors de POST /users : ${err.message}`);
    next(err);
  }
});

module.exports = router;
```

### Enregistrement dans `app.js` (après les routes)

```js
const errorHandler = require('./middlewares/errorHandler');
// … définition des routes …
app.use(errorHandler); // ← toujours en dernier
```

---

## 10 — Exemples de logs générés

### `logs/app.log` — fonctionnement normal

```
[INFO] 2026-02-10 10:25:03 - GET /users 200 14.352 ms - 1024 bytes
[INFO] 2026-02-10 10:25:03 - Requête : récupération de la liste des utilisateurs
[INFO] 2026-02-10 10:25:03 - 12 utilisateur(s) retourné(s)
[INFO] 2026-02-10 10:25:11 - POST /users 201 22.180 ms - 256 bytes
[INFO] 2026-02-10 10:25:11 - Utilisateur créé — ID: 47, email: alice@example.com
[WARN] 2026-02-10 10:25:45 - POST /users — champs manquants : nom=undefined, email=test@test.com
[INFO] 2026-02-10 10:26:05 - DELETE /users/42 404 5.100 ms - 48 bytes
```

### `logs/error.log` — erreurs critiques

```
[ERROR] 2026-02-10 10:26:30 - GET /users — HTTP 500 — connect ECONNREFUSED 127.0.0.1:3306
Error: connect ECONNREFUSED 127.0.0.1:3306
    at TCPConnectWrap.afterConnect [as oncomplete] (net.js:1141:16)

[ERROR] 2026-02-10 10:27:15 - POST /users — HTTP 500 — Duplicate entry 'alice@example.com' for key 'email'
```

---

## 11 — Analyse des logs (script Python)

### `analyse_logs.py`

```python
import re
from collections import defaultdict
from datetime import datetime

LOG_FILE = 'logs/app.log'

# Expressions régulières pour extraire les informations des logs HTTP (Morgan)
RE_HTTP   = re.compile(r'\[INFO\] (\S+ \S+) - (GET|POST|PUT|DELETE) (\S+) (\d{3}) ([\d.]+) ms')
RE_ERROR  = re.compile(r'\[ERROR\]')
RE_WARN   = re.compile(r'\[WARN\]')

def analyser_logs(fichier):
    requetes        = []
    nb_erreurs      = 0
    nb_warnings     = 0
    routes          = defaultdict(int)
    statuts         = defaultdict(int)
    temps_reponse   = []

    with open(fichier, 'r', encoding='utf-8') as f:
        for ligne in f:
            # Comptage des erreurs et avertissements
            if RE_ERROR.search(ligne):
                nb_erreurs += 1
            if RE_WARN.search(ligne):
                nb_warnings += 1

            # Extraction des requêtes HTTP
            m = RE_HTTP.search(ligne)
            if m:
                horodatage, methode, url, statut, temps = m.groups()
                requetes.append({
                    'horodatage': horodatage,
                    'methode':    methode,
                    'url':        url,
                    'statut':     int(statut),
                    'temps':      float(temps)
                })
                routes[f"{methode} {url}"] += 1
                statuts[statut]            += 1
                temps_reponse.append(float(temps))

    return requetes, nb_erreurs, nb_warnings, routes, statuts, temps_reponse


def afficher_rapport(requetes, nb_erreurs, nb_warnings, routes, statuts, temps_reponse):
    print("=" * 55)
    print("       RAPPORT D'ANALYSE DES LOGS — API USERS")
    print("=" * 55)
    print(f"  Période analysée       : {requetes[0]['horodatage']} → {requetes[-1]['horodatage']}"
          if requetes else "  Aucune requête HTTP trouvée.")
    print(f"  Total requêtes HTTP    : {len(requetes)}")
    print(f"  Erreurs (ERROR)        : {nb_erreurs}")
    print(f"  Avertissements (WARN)  : {nb_warnings}")

    if temps_reponse:
        print(f"  Temps de réponse moyen : {sum(temps_reponse)/len(temps_reponse):.2f} ms")
        print(f"  Temps de réponse max   : {max(temps_reponse):.2f} ms")
        print(f"  Temps de réponse min   : {min(temps_reponse):.2f} ms")

    print("\n--- Routes les plus sollicitées ---")
    for route, count in sorted(routes.items(), key=lambda x: x[1], reverse=True)[:5]:
        print(f"  {route:<30} {count} appel(s)")

    print("\n--- Répartition des codes HTTP ---")
    for statut, count in sorted(statuts.items()):
        print(f"  HTTP {statut} : {count} réponse(s)")

    print("=" * 55)


if __name__ == '__main__':
    requetes, nb_erreurs, nb_warnings, routes, statuts, temps = analyser_logs(LOG_FILE)
    afficher_rapport(requetes, nb_erreurs, nb_warnings, routes, statuts, temps)
```

### Exemple de sortie

```
=======================================================
       RAPPORT D'ANALYSE DES LOGS — API USERS
=======================================================
  Période analysée       : 2026-02-10 10:25:03 → 2026-02-10 14:42:11
  Total requêtes HTTP    : 218
  Erreurs (ERROR)        : 4
  Avertissements (WARN)  : 9
  Temps de réponse moyen : 18.74 ms
  Temps de réponse max   : 412.00 ms
  Temps de réponse min   : 3.20 ms

--- Routes les plus sollicitées ---
  GET /users                     143 appel(s)
  POST /users                     41 appel(s)
  PUT /users/:id                  22 appel(s)
  DELETE /users/:id               12 appel(s)

--- Répartition des codes HTTP ---
  HTTP 200 : 143 réponse(s)
  HTTP 201 :  41 réponse(s)
  HTTP 400 :   9 réponse(s)
  HTTP 404 :  21 réponse(s)
  HTTP 500 :   4 réponse(s)
=======================================================
```

---

## 12 — Monitoring simple

Un module de monitoring est intégré directement dans l'application pour exposer des métriques en temps réel.

### `src/monitoring.js`

```js
const logger = require('./logger');

const metriques = {
  totalRequetes:    0,
  totalErreurs:     0,
  sommeTemps:       0,
  debutServeur:     Date.now()
};

// Middleware : incrémente les compteurs à chaque requête
function middlewareMonitoring(req, res, next) {
  const debut = Date.now();
  metriques.totalRequetes++;

  res.on('finish', () => {
    const duree = Date.now() - debut;
    metriques.sommeTemps += duree;
    if (res.statusCode >= 500) {
      metriques.totalErreurs++;
    }
  });

  next();
}

// Route exposant les métriques au format JSON
function routeMetriques(req, res) {
  const uptimeMs  = Date.now() - metriques.debutServeur;
  const moyenne   = metriques.totalRequetes > 0
    ? (metriques.sommeTemps / metriques.totalRequetes).toFixed(2)
    : 0;

  const rapport = {
    uptime_secondes:          Math.floor(uptimeMs / 1000),
    total_requetes:           metriques.totalRequetes,
    total_erreurs_500:        metriques.totalErreurs,
    temps_reponse_moyen_ms:   parseFloat(moyenne),
    taux_erreur_pct:          metriques.totalRequetes > 0
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
// …routes existantes…
app.get('/metrics', routeMetriques);
```

### Exemple de réponse `GET /metrics`

```json
{
  "uptime_secondes":         3842,
  "total_requetes":          218,
  "total_erreurs_500":       4,
  "temps_reponse_moyen_ms":  18.74,
  "taux_erreur_pct":         "1.8"
}
```

Ces données peuvent alimenter un tableau de bord (Grafana, Kibana, ou même une page HTML simple) pour offrir une vue en temps réel de l'état de l'API.

---

## 13 — Tests réalisés

### Scénario 1 — Erreur de connexion à la base de données

**Simulation :** le service MySQL est arrêté, une requête `GET /users` est effectuée.

**Log généré dans `error.log` :**
```
[ERROR] 2026-02-10 10:26:30 - GET /users — HTTP 500 — connect ECONNREFUSED 127.0.0.1:3306
Error: connect ECONNREFUSED 127.0.0.1:3306
    at TCPConnectWrap.afterConnect [as oncomplete] (net.js:1141:16)
```

**Diagnostic permis :** l'erreur `ECONNREFUSED` sur le port 3306 indique immédiatement que la base de données est inaccessible. Avant la mise en place des logs, ce problème aurait été invisible.

---

### Scénario 2 — Requête invalide (champs manquants)

**Simulation :** `POST /users` envoyé sans le champ `email`.

**Log généré dans `app.log` :**
```
[WARN] 2026-02-10 10:25:45 - POST /users — champs manquants : nom=Alice, email=undefined
[INFO] 2026-02-10 10:25:45 - POST /users 400 3.210 ms - 62 bytes
```

**Diagnostic permis :** le WARNING identifie précisément le champ manquant et le contexte de la requête invalide.

---

### Scénario 3 — Erreur serveur générique (exception non gérée)

**Simulation :** une erreur JavaScript inattendue est levée dans un contrôleur.

**Log généré dans `error.log` :**
```
[ERROR] 2026-02-10 11:14:02 - GET /users/99 — HTTP 500 — Cannot read properties of undefined (reading 'id')
TypeError: Cannot read properties of undefined (reading 'id')
    at router.get (src/routes/users.js:34:25)
    at Layer.handle [as handle_request] (express/lib/router/layer.js:95:5)
```

**Diagnostic permis :** la stack trace complète indique le fichier (`users.js`) et la ligne exacte (`34`) de l'erreur, rendant le débogage immédiat.

---

## 14 — Résultats obtenus

| Avant amélioration | Après amélioration |
|---|---|
| Aucune trace des erreurs | Erreurs complètes avec stack trace dans `error.log` |
| Aucune visibilité sur les requêtes | Chaque requête tracée avec méthode, URL, statut, temps |
| Détection des pannes par les utilisateurs | Détection immédiate par analyse des logs |
| Impossible d'analyser l'activité | Statistiques générées automatiquement par le script Python |
| Pas de métriques de performance | Endpoint `/metrics` exposant uptime, taux d'erreur, temps moyen |

L'amélioration permet désormais de :

- **Diagnostiquer** tout incident en moins de 5 minutes grâce aux logs horodatés et contextualisés.
- **Suivre l'activité** de l'API sur une journée complète via le script d'analyse.
- **Anticiper les problèmes** en surveillant le taux d'erreur et les temps de réponse.

---

## 15 — Compétences mobilisées (BTS SIO SLAM)

| Compétence du référentiel | Application concrète |
|---|---|
| **Maintenance d'une solution applicative** | Amélioration d'une API existante en production sans modification de son comportement fonctionnel |
| **Analyse d'un dysfonctionnement** | Identification des lacunes de l'application (absence de logs) et proposition d'une solution adaptée |
| **Exploitation des données** | Création d'un script Python pour lire, parser et analyser les fichiers de logs |
| **Tests d'une solution applicative** | Simulation d'incidents et vérification que les logs permettent bien de les diagnostiquer |
| **Documentation technique** | Rédaction de cette réalisation professionnelle et commentaires dans le code |

---

## 16 — Conclusion

La mise en place d'un système de journalisation et de monitoring sur cette API REST représente une amélioration fondamentale de la **qualité de service** et de la **maintenabilité** de l'application.

Dans un contexte professionnel, une application sans logs est une application dont on ne maîtrise pas le comportement en production. Les incidents sont inévitables ; ce qui différencie une équipe efficace, c'est sa **capacité à les diagnostiquer et à les résoudre rapidement**. Winston, Morgan et le script Python mis en œuvre fournissent cette capacité à coût maîtrisé, en s'appuyant sur des outils éprouvés et intégrés à l'écosystème Node.js.

Cette réalisation m'a permis de comprendre l'importance de l'**observabilité** dans le cycle de vie d'une application : une application observable est une application que l'on peut comprendre, maintenir et améliorer en continu.

En perspective, ce système pourrait être enrichi avec :
- Un outil de centralisation des logs comme **ELK Stack** (Elasticsearch, Logstash, Kibana) ou **Grafana Loki** pour les environnements multi-serveurs.
- Des **alertes automatiques** (email, Slack) déclenchées lorsque le taux d'erreur dépasse un seuil.
- Un **tableau de bord temps réel** alimenté par les métriques de l'endpoint `/metrics`.

---

*Document rédigé dans le cadre du BTS SIO SLAM — Portfolio de réalisations professionnelles.*
