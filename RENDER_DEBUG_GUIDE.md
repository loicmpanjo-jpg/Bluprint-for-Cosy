# ============================================================
# COSY V0.1 — Guide de Débogage Render Blueprint
# ============================================================

## 🔴 Erreurs communes et solutions

### 1. "Blueprint sync failed"

**Cause :** Syntaxe YAML invalide ou champ requis manquant.

**Solution :**
```bash
# Vérifier la syntaxe YAML
npm install -g yaml-lint
yaml-lint render.yaml

# Ou utiliser Python
python3 -c "import yaml; yaml.safe_load(open('render.yaml'))"
```

### 2. "Service type 'keyvalue' not recognized"

**Cause :** Ancienne syntaxe avec section `keyvalue:` séparée.

**Solution :** `keyvalue` doit être dans `services:` avec `type: keyvalue`.

### 3. "Cannot reference service 'cosy-redis'"

**Cause :** `fromService` mal configuré pour Redis.

**Solution :** Utiliser `type: keyvalue` et `property: connectionString`.

```yaml
- key: REDIS_URL
  fromService:
    type: keyvalue
    name: cosy-redis
    property: connectionString
```

### 4. "generateValue not supported for JWT keys"

**Cause :** `generateValue: true` ne fonctionne pas pour les paires de clés.

**Solution :** Utiliser `sync: false` et générer les clés manuellement.

```bash
# Générer les clés JWT
openssl genrsa -out jwt-private.pem 2048
openssl rsa -in jwt-private.pem -pubout -out jwt-public.pem

# Copier le contenu dans le Dashboard Render
# cosy-api → Environment → Add Environment Variable
```

### 5. "Static site not serving"

**Cause :** `publishDir` au lieu de `staticPublishPath`.

**Solution :** Utiliser `staticPublishPath` pour les sites statiques.

### 6. "Database connection failed"

**Cause :** `fromDatabase` mal configuré ou DB pas encore créée.

**Solution :**
```yaml
- key: DATABASE_URL
  fromDatabase:
    name: cosy-db        # Doit matcher databases[].name
    property: connectionString
```

### 7. "Worker not starting"

**Cause :** Le worker n'a pas de `PORT` exposé (normal), mais Render attend un health check.

**Solution :** Pas de `healthCheckPath` pour les workers.

### 8. "Cron job not running"

**Cause :** Syntaxe cron invalide ou plan free limité.

**Solution :** Vérifier la syntaxe sur https://crontab.guru/

## 🟡 Warnings importants

### Free Tier PostgreSQL expire après 30 jours

**Solution :** Backup régulier ou upgrade vers `plan: basic-256mb` ($7/mois).

```bash
# Backup
pg_dump $DATABASE_URL > backup.sql

# Restore
psql $DATABASE_URL < backup.sql
```

### Free Tier Web Service spin down après 15 min

**Solution :** 
- UptimeRobot ping toutes les 10 minutes
- Ou upgrade vers `plan: starter` ($7/mois)

### Free Tier Redis data perdue au restart

**Solution :** Utiliser Redis uniquement pour cache/sessions, pas pour données persistantes.

## 🟢 Vérification post-déploiement

```bash
# 1. Health check API
curl https://cosy-api.onrender.com/health

# 2. Test OTP
curl -X POST https://cosy-api.onrender.com/api/v0/auth/otp/request   -H "Content-Type: application/json"   -d '{"phone":"+237612345678"}'

# 3. Test villes
curl https://cosy-api.onrender.com/api/v0/pulse/cities

# 4. Test DB connection
# Vérifier les logs dans le Dashboard Render
```

## 🔧 Structure du repo attendue par Render

```
cosy/
├── render.yaml              # ← À la RACINE
├── package.json             # ← À la RACINE (pour API + worker + cron)
├── server.js                # ← Point d'entrée API
├── worker.js                # ← Point d'entrée worker
├── cron.js                  # ← Point d'entrée cron
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

## 📋 Checklist pré-déploiement

- [ ] `render.yaml` à la racine du repo
- [ ] `package.json` avec scripts `start`, `worker`, `cron`
- [ ] Fichiers `server.js`, `worker.js`, `cron.js` existent
- [ ] Dossiers `frontend/` et `dashboard/` existent avec `index.html`
- [ ] Repo push sur GitHub/GitLab/Bitbucket
- [ ] Compte Render créé et connecté au repo

## 🚀 Étapes de déploiement manuel (si Blueprint échoue)

### Étape 1 : Créer la DB PostgreSQL
```
Dashboard Render → New → PostgreSQL
Name: cosy-db
Database: cosy
User: cosy_admin
Plan: Free
```

### Étape 2 : Créer le Redis Key Value
```
Dashboard Render → New → Key Value
Name: cosy-redis
Plan: Free
```

### Étape 3 : Créer l'API Web Service
```
Dashboard Render → New → Web Service
Name: cosy-api
Runtime: Node
Build: npm install
Start: npm start
Plan: Free
```

### Étape 4 : Configurer les variables d'environnement
```
cosy-api → Environment → Add:
- DATABASE_URL (from cosy-db connectionString)
- REDIS_URL (from cosy-redis connectionString)
- JWT_PRIVATE_KEY (sync: false, générer avec OpenSSL)
- JWT_PUBLIC_KEY (sync: false, générer avec OpenSSL)
- TWILIO_ACCOUNT_SID (sync: false)
- TWILIO_AUTH_TOKEN (sync: false)
```

### Étape 5 : Créer les sites statiques
```
Dashboard Render → New → Static Site
Name: cosy-app
Root: ./frontend
Build: echo "ok"
Publish: ./frontend

Name: cosy-dashboard
Root: ./dashboard
Build: echo "ok"
Publish: ./dashboard
```

### Étape 6 : Créer le worker
```
Dashboard Render → New → Background Worker
Name: cosy-worker
Runtime: Node
Build: npm install
Start: npm run worker
```

### Étape 7 : Créer le cron
```
Dashboard Render → New → Cron Job
Name: cosy-cron
Runtime: Node
Build: npm install
Start: npm run cron
Schedule: 0 0 * * *
```
