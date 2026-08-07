# Flowmerce — Infrastructure

Repository d'**orchestration** de la plateforme Flowmerce.

Il ne contient aucun code applicatif : son unique rôle est de lancer, avec une seule
commande Docker Compose, les deux services qui composent la plateforme. Chaque
service reste dans son propre repository, avec son propre `Dockerfile` ; ce
repository ne fait que les assembler.

---

## 1. Rôle du repository

| Repository | Responsabilité | Contenu Docker |
| --- | --- | --- |
| `flowmerce-infrastructure` | Orchestration, réseau, variables d'environnement | `docker-compose.yml` |
| `flowmerce-web-app` | Application Next.js | `Dockerfile`, `.dockerignore` |
| `Flowmerce-ML` | API de prédiction (FastAPI) | `Dockerfile`, `.dockerignore` |

Les trois repositories restent **indépendants** — ceci n'est pas un monorepo.
Compose les référence par chemin relatif comme contextes de build.

---

## 2. Architecture Docker

```text
                          Docker Compose
                                │
                   ┌────────────┴────────────┐
                   │                         │
                   ▼                         ▼
             flowmerce-web              flowmerce-ml
               Next.js 16                 FastAPI
                 :3000                     :8000
                   │                         ▲
                   │   http://ml:8000        │
                   └─────────────────────────┘
                     réseau flowmerce_default

     hôte :3000 ──► web            hôte 127.0.0.1:8000 ──► ml
                     │
                     └──► PostgreSQL externe (hors périmètre Compose)
```

Arborescence attendue sur le disque — les chemins relatifs du
`docker-compose.yml` en dépendent :

```text
flowmerce/
├── flowmerce-infrastructure/
│   ├── docker-compose.yml
│   ├── .env.example
│   ├── .gitignore
│   └── README.md
├── flowmerce-web-app/
│   ├── Dockerfile
│   └── .dockerignore
└── Flowmerce-ML/
    ├── Dockerfile
    └── .dockerignore
```

---

## 3. Les deux services

### `web` — application Next.js

- **Contexte de build** : `../flowmerce-web-app`
- **Image** : `flowmerce-web:local` (~333 Mo)
- **Port** : `3000`, publié sur toutes les interfaces
- **Build** : multi-stage `deps → builder → runner`, sortie Next.js
  `standalone`, exécution en utilisateur non-root (`nextjs`)
- **Healthcheck** : requête HTTP sur `/`, toutes les 30 s

Le mode `standalone` est activé par la variable de build `DOCKER_BUILD=true`,
lue par `next.config.ts`. Les builds Vercel et Capacitor du repository web ne
sont donc pas affectés.

### `ml` — API Machine Learning

- **Contexte de build** : `../Flowmerce-ML`
- **Image** : `flowmerce-ml:local` (~778 Mo)
- **Port** : `8000`, publié sur `127.0.0.1` uniquement
- **Point d'entrée** : `uvicorn api.server:app`
- **Build** : multi-stage `builder → runtime` sur `python:3.11-slim`,
  exécution en utilisateur non-root (`appuser`)
- **Healthcheck** : endpoint `/health` existant de l'API, toutes les 30 s

Les artefacts ML (modèle, encodeur, scaler…) sont téléchargés depuis Hugging
Face au démarrage — `HF_REPO_ID` est donc **obligatoire**. Une copie locale est
également embarquée dans l'image (`models/`).

Le service `web` n'attend pas seulement que `ml` soit démarré, mais qu'il soit
**sain** (`depends_on: condition: service_healthy`) : le chargement des
artefacts prend quelques dizaines de secondes au premier lancement.

---

## 4. Prérequis

- Docker Engine **24+** et le plugin Docker Compose **v2** (`docker compose`, sans tiret)
- Les trois repositories clonés **côte à côte**, comme dans l'arborescence ci-dessus
- Une base **PostgreSQL accessible** (Supabase, Neon, …) — voir ci-dessous
- Un accès réseau à Hugging Face pour les artefacts ML

Vérification :

```bash
docker --version
docker compose version
```

> **PostgreSQL est hors périmètre de ce Compose.** Aucun service de base de
> données n'est déclaré : l'application se connecte à une instance externe via
> `DATABASE_URL` / `DIRECT_URL`.

---

## 5. Configurer `.env`

```bash
cp .env.example .env
```

Puis renseigner les valeurs dans `.env`. Toutes les variables non commentées y
sont **obligatoires** : `web` valide son environnement au démarrage (zod) et
refuse de démarrer si une valeur manque ou est mal formée.

Générer les secrets qui n'existent pas encore :

```bash
openssl rand -base64 32   # NEXTAUTH_SECRET, AUTH_SECRET
openssl rand -hex 32      # CRON_SECRET
```

Points d'attention :

- **`ML_INTERNAL_SECRET`** est saisi **une seule fois**. Compose l'injecte dans
  `web` sous ce nom et dans `ml` sous le nom `INTERNAL_API_KEY` : les deux côtés
  ne peuvent pas se désynchroniser.
- **`NEXT_PUBLIC_BASE_URL`** est *inlinée dans le bundle client au moment du
  build*. Après l'avoir changée, il faut reconstruire : `docker compose build web`.
- **`AUTH_TRUST_HOST` et `AUTH_URL`** sont spécifiques à l'exécution en
  conteneur — voir la section 13 ci-dessous. Les laisser aux valeurs par défaut
  convient en local.
- **`.env` est ignoré par Git.** Ne committez jamais de secret réel ; seul
  `.env.example` est versionné.

---

## 6. Construire les images

```bash
docker compose build
```

Un seul service :

```bash
docker compose build web
docker compose build ml
```

Reconstruction complète, en ignorant le cache :

```bash
docker compose build --no-cache
```

---

## 7. Démarrer les services

Au premier plan (logs affichés, `Ctrl+C` pour arrêter) :

```bash
docker compose up
```

En arrière-plan :

```bash
docker compose up -d
```

Construire puis démarrer en une seule commande :

```bash
docker compose up -d --build
```

Une fois démarré :

| Service | URL |
| --- | --- |
| Application Next.js | <http://localhost:3000> |
| API ML | <http://localhost:8000> |
| Documentation OpenAPI de l'API ML | <http://localhost:8000/docs> |

---

## 8. Arrêter les services

```bash
docker compose down
```

En supprimant aussi les volumes — **efface les réclamations enregistrées par
`/save_claim`** :

```bash
docker compose down -v
```

Arrêter sans supprimer les conteneurs :

```bash
docker compose stop
```

---

## 9. Consulter les logs

```bash
docker compose logs -f          # les deux services, en continu
docker compose logs -f web      # application Next.js seulement
docker compose logs -f ml       # API ML seulement
docker compose logs --tail=100  # les 100 dernières lignes
```

---

## 10. Reconstruire les images

Après une modification du code d'un des deux repositories :

```bash
docker compose build web && docker compose up -d web
```

Reconstruction complète de la stack :

```bash
docker compose down
docker compose build --no-cache
docker compose up -d
```

---

## 11. Vérifier que les services fonctionnent

État et santé des conteneurs — les deux doivent afficher `(healthy)` :

```bash
docker compose ps
```

```text
NAME            SERVICE   STATUS
flowmerce-ml    ml        Up (healthy)
flowmerce-web   web       Up (healthy)
```

Application Next.js :

```bash
curl -i http://localhost:3000/
```

API ML :

```bash
curl http://localhost:8000/health
```

Health check applicatif complet — vérifie la base de données **et** l'API ML :

```bash
curl "http://localhost:3000/api/health/?deep=1"
```

```json
{
  "status": "ok",
  "database": "connected",
  "checks": {
    "database": { "status": "ok" },
    "ml_api":   { "status": "ok" }
  }
}
```

---

## 12. Communication entre les conteneurs

Compose crée un réseau `bridge` par défaut (`flowmerce_default`) auquel les deux
services sont attachés. Docker y fournit une **résolution DNS interne par nom de
service** : depuis `web`, le nom d'hôte `ml` pointe vers le conteneur de l'API.

```text
web ──► http://ml:8000/predict ──► ml
        en-tête X-Internal-Key
```

C'est pourquoi la configuration utilise :

```bash
ML_API_URL=http://ml:8000
```

et **jamais** `http://localhost:8000` : à l'intérieur du conteneur `web`,
`localhost` désigne le conteneur `web` lui-même, pas l'hôte ni le conteneur
`ml`. `localhost:8000` depuis votre machine reste valide — c'est le port publié
sur la loopback de l'hôte, un chemin différent de la communication interne.

Les appels de `web` vers `ml` portent l'en-tête `X-Internal-Key`, que l'API ML
compare à son `INTERNAL_API_KEY` ; une clé absente ou incorrecte renvoie `403`.

Vérifier la communication interne de bout en bout :

```bash
docker compose exec web node -e "fetch('http://ml:8000/health').then(r=>r.text()).then(console.log)"
```

Le port `8000` n'a pas besoin d'être publié pour que `web` joigne `ml` : il ne
l'est, sur la loopback, que pour votre confort en développement local.

---

## 13. Authentification en conteneur

Deux variables du service `web` n'existent que parce que l'application tourne
en conteneur. Elles sont préremplies dans `.env.example` et ne demandent aucune
modification du code applicatif.

### `AUTH_TRUST_HOST=true`

En `NODE_ENV=production`, Auth.js v5 rejette toute requête dont l'hôte n'est pas
déclaré de confiance :

```text
[auth][error] UntrustedHost: Host must be trusted. URL was: http://localhost:3000/api/auth/session
```

Auth.js reconnaît Vercel automatiquement (`VERCEL=1`) et fait confiance par
défaut hors production — d'où l'absence du problème en développement et sur
Vercel. Dans un conteneur, ni l'un ni l'autre : il faut le lui dire.

### `AUTH_URL`

Le serveur Next.js `standalone` doit écouter sur `HOSTNAME=0.0.0.0` pour être
joignable depuis l'extérieur du conteneur. Sans origine canonique, Auth.js
déduit ses redirections de ce hostname et renvoie l'utilisateur vers
`http://0.0.0.0:3000/…`, une adresse qu'aucun navigateur ne peut ouvrir.

`AUTH_URL` fixe l'origine réelle du site. Laissée vide, elle reprend
`NEXT_PUBLIC_BASE_URL`. En production, la renseigner avec le domaine public.

### Vérifier

```bash
docker compose logs web | grep -i untrustedhost   # doit ne rien renvoyer
curl -sL -o /dev/null -w "%{http_code}\n" http://localhost:3000/api/auth/session
```

Les redirections d'Auth.js doivent pointer vers votre domaine, jamais vers
`0.0.0.0` :

```bash
curl -s -D - -o /dev/null http://localhost:3000/api/auth/signin/ | grep -i location
```
