# ============================================================
# COSY V0.1 — Guide de Déploiement Render (CORRIGÉ)
# ============================================================

## 🔴 Problème identifié

**Erreur:** `services[4].plan free not a valid plan for service type cron`

**Cause:** Les cron jobs et background workers NE supportent PAS le plan `free` sur Render.

**Documentation officielle:**
> `free` (not available for private services, background workers, or cron jobs)
> — Render Blueprint Spec citeweb_search:20#0

## 💰 Coût réel du déploiement

| Service | Plan minimum | Coût/mois |
|---------|-------------|-----------|
| API (web) | free | $0 |
| Frontend (static) | free | $0 |
| Dashboard (static) | free | $0 |
| PostgreSQL | free | $0 (expire après 30 jours) |
| Redis (keyvalue) | free | $0 |
| **Worker** | **starter** | **$7** |
| **Cron** | **starter** | **$1 minimum** |
| **TOTAL** | | **$8/mois minimum** |

> 💡 Les cron jobs sont facturés à la seconde d'exécution. Si le job tourne 1 min/jour = ~$0.05/mois.
> Mais Render facture un minimum de $1/mois par cron job. citeweb_search:20#6

## 🟢 Solution 1 — Blueprint complet (avec coût $8/mois)

Le `render.yaml` corrigé est prêt. Les worker et cron utilisent le plan par défaut (starter).

**Pour déployer:**
```bash
# 1. Placer render.yaml à la racine du repo
git add render.yaml
git commit -m "COSY V0.1 — Blueprint Render corrigé"
git push origin main

# 2. Render Dashboard → New → Blueprint
# 3. Connecter le repo → Deploy
```

## 🟡 Solution 2 — Blueprint sans worker ni cron (GRATUIT)

Si tu veux rester 100% gratuit, supprime worker et cron du Blueprint :

```yaml
services:
  - type: web
    name: cosy-api
    runtime: node
    plan: free
    ...

  - type: web
    name: cosy-app
    runtime: static
    ...

  - type: web
    name: cosy-dashboard
    runtime: static
    ...

  - type: keyvalue
    name: cosy-redis
    ...

databases:
  - name: cosy-db
    plan: free
    ...
```

**Conséquences:**
- ❌ Pas de file d'attente de jobs (upload, encoding, scoring)
- ❌ Pas de tâches planifiées (stats quotidiennes, distribution revenus)
- ✅ Tu peux exécuter les jobs manuellement via l'API
- ✅ Tu peux utiliser un cron externe gratuit (voir ci-dessous)

## 🟡 Solution 3 — Cron externe gratuit

Remplacer le cron Render par un service externe :

### Option A: cron-job.org (GRATUIT)
```
1. Créer compte sur https://cron-job.org
2. Ajouter un job : POST https://cosy-api.onrender.com/api/v0/cron/daily
3. Schedule : Every day at 00:00 UTC
4. Gratuit, illimité
```

### Option B: GitHub Actions (GRATUIT)
```yaml
# .github/workflows/cron.yml
name: Daily COSY Cron
on:
  schedule:
    - cron: '0 0 * * *'
jobs:
  cron:
    runs-on: ubuntu-latest
    steps:
      - run: curl -X POST https://cosy-api.onrender.com/api/v0/cron/daily
```

### Option C: UptimeRobot (GRATUIT + réveil instance)
```
1. Créer compte sur https://uptimerobot.com
2. Ajouter monitor : https://cosy-api.onrender.com/health
3. Intervalle : Every 10 minutes
4. Bénéfice secondaire : empêche le spin down du free tier
```

## 🟡 Solution 4 — Worker intégré dans l'API

Au lieu d'un worker séparé, exécuter les jobs dans le processus API :

```javascript
// server.js — Ajouter dans l'API
const Bull = require('bull');
const queue = new Bull('cosy-jobs', process.env.REDIS_URL);

// Process jobs in the same process
queue.process('upload', 5, async (job) => { /* ... */ });
queue.process('encode', 3, async (job) => { /* ... */ });
queue.process('score', 2, async (job) => { /* ... */ });

// Cron via setInterval (pas idéal mais gratuit)
setInterval(async () => {
  await runDailyStats();
}, 24 * 60 * 60 * 1000); // Toutes les 24h
```

**Inconvénient:** Les jobs ralentissent l'API si charge élevée.

## 📋 Structure du repo attendue

```
cosy/
├── render.yaml              # ← À la RACINE
├── package.json             # ← À la RACINE
├── server.js                # ← Point d'entrée API
├── worker.js                # ← Point d'entrée worker (si Solution 1)
├── cron.js                  # ← Point d'entrée cron (si Solution 1)
├── frontend/                # ← Dossier pour cosy-app
│   ├── index.html
│   └── cosy-app.html
├── dashboard/               # ← Dossier pour cosy-dashboard
│   ├── index.html
│   └── cosy_content_acquisition_dashboard.html
└── backend/                 # ← Code source API
    ├── routes/
    ├── models/
    └── migrations/
```

## 📋 package.json requis

```json
{
  "name": "cosy-api",
  "version": "0.1.0",
  "scripts": {
    "start": "node server.js",
    "worker": "node worker.js",
    "cron": "node cron.js",
    "migrate": "node migrations/run.js",
    "seed": "node migrations/seed.js"
  },
  "dependencies": {
    "fastify": "^4.x",
    "@fastify/cors": "^8.x",
    "@fastify/jwt": "^7.x",
    "pg": "^8.x",
    "redis": "^4.x",
    "bull": "^4.x",
    "twilio": "^4.x",
    "dotenv": "^16.x"
  }
}
```

## 🚀 Checklist de déploiement

### Étape 1 — Préparer le repo
- [ ] `render.yaml` à la racine
- [ ] `package.json` avec scripts `start`, `worker`, `cron`
- [ ] Fichiers `server.js`, `worker.js`, `cron.js` existent
- [ ] Dossiers `frontend/` et `dashboard/` avec `index.html`
- [ ] Repo push sur GitHub/GitLab/Bitbucket

### Étape 2 — Déployer via Blueprint
- [ ] Render Dashboard → New → Blueprint
- [ ] Connecter le repo
- [ ] Choisir branche `main`
- [ ] Vérifier les services détectés
- [ ] Cliquer **Deploy Blueprint**

### Étape 3 — Configurer les secrets
- [ ] cosy-api → Environment → Add variables
- [ ] JWT_PRIVATE_KEY (générer avec OpenSSL)
- [ ] JWT_PUBLIC_KEY (générer avec OpenSSL)
- [ ] TWILIO_ACCOUNT_SID
- [ ] TWILIO_AUTH_TOKEN
- [ ] TWILIO_PHONE_NUMBER

### Étape 4 — Migrations & Seed
```bash
# Via Render Dashboard → cosy-api → Shell (si plan payant)
# Ou via API:
curl -X POST https://cosy-api.onrender.com/api/v0/admin/migrate
curl -X POST https://cosy-api.onrender.com/api/v0/admin/seed
```

### Étape 5 — Vérification
```bash
# Health check
curl https://cosy-api.onrender.com/health

# Test OTP
curl -X POST https://cosy-api.onrender.com/api/v0/auth/otp/request   -H "Content-Type: application/json"   -d '{"phone":"+237612345678"}'

# Test villes
curl https://cosy-api.onrender.com/api/v0/pulse/cities
```

## 🚨 Points de vigilance

1. **Cold starts** : Le free tier spin down après 15min. Utiliser UptimeRobot pour ping toutes 10min.

2. **DB expiration** : La DB free expire après 30 jours. Backup régulier !

3. **Bandwidth** : 100GB/mois = ~5000 streams de 3min. Upgrade rapide nécessaire pour la prod.

4. **Worker/Cron** : Si tu choisis la Solution 1 ($8/mois), le worker et cron sont toujours actifs. Si tu choisis la Solution 2 (gratuit), tu dois gérer les jobs manuellement.

## 💡 Ma recommandation

**Pour le lancement V0 (juillet 2026) :**
- Utiliser la **Solution 2** (Blueprint gratuit sans worker/cron)
- Exécuter les jobs manuellement via API pendant la phase beta
- Ajouter un cron externe gratuit (cron-job.org) pour les stats
- Passer à la Solution 1 ($8/mois) quand tu as 100+ utilisateurs actifs

**Pour la production (janvier 2027) :**
- Upgrader vers `plan: starter` pour API ($7)
- Upgrader DB vers `plan: basic-256mb` ($7)
- Ajouter worker ($7) et cron ($1)
- **Total : ~$22/mois**

