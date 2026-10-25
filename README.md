# TP3 – CI/CD avec GitHub Actions et Render

> **Enseignant :** Wahid Hamdi
> **Auteur :** Nawres Zarrouki

Pipeline CI/CD complet pour une petite application **Express / TypeScript** : vérification automatique avec **GitHub Actions**, déploiement sur **Render** uniquement si la CI réussit, protection de `main` par un **ruleset** et déploiements de prévisualisation pour les pull requests.

- **Dépôt GitHub :** <https://github.com/nawreszr/cicd-lab>
- **Production (Render) :** <https://VRAIE-URL.onrender.com>

## Sommaire

1. [Architecture du pipeline](#1-architecture-du-pipeline)
2. [Application et vérifications locales](#2-application-et-vérifications-locales)
3. [Premier déploiement sur Render](#3-premier-déploiement-sur-render)
4. [Intégration continue avec GitHub Actions](#4-intégration-continue-avec-github-actions)
5. [Déployer seulement si la CI réussit](#5-déployer-seulement-si-la-ci-réussit)
6. [Protection de la branche main](#6-protection-de-la-branche-main)
7. [Un changement normal dans le pipeline](#7-un-changement-normal-dans-le-pipeline)
8. [Démonstration d'un test en échec](#8-démonstration-dun-test-en-échec)
9. [Prévisualisation des pull requests](#9-prévisualisation-des-pull-requests)
10. [Résumé des captures](#10-résumé-des-captures)

---

## 1. Architecture du pipeline

```
 branche feature ──push──► Pull Request ──► GitHub Actions : « Verify »
                                              (typecheck + test + build)
                                                   │ ✔ réussi
                                                   ▼
                                       merge dans main (ruleset : check obligatoire)
                                                   │
                                                   ▼
                              Verify sur main ✔ ──► Render déploie (After CI Checks Pass)
```

| Élément | Rôle |
|---|---|
| `npm run verify` | Une seule commande : `typecheck` + `test` + `build` |
| GitHub Actions (`ci.yml`) | Exécute `verify` sur chaque PR vers `main` et chaque push sur `main` |
| Ruleset *Protect main* | Interdit de fusionner sans PR ni check `Verify` réussi |
| Render | Déploie `main`, uniquement après réussite de la CI |
| Pull Request Previews | Copie temporaire de l'application, avec sa propre URL |

---

## 2. Application et vérifications locales

Le point de départ est l'application du TP (Express + TypeScript, Node.js 24), avec un endpoint de santé `GET /api/health` qui retourne `{"status":"ok"}`.

```bash
git clone https://github.com/wahid007/cicd-lab-starter.git cicd-lab
cd cicd-lab
rm -rf .git
git init -b main

node --version        # doit commencer par v24.
npm ci
npm run dev           # http://localhost:3000 et /api/health
```

Scripts de `package.json`, avec la commande `verify` ajoutée :

```json
"scripts": {
  "dev": "tsx watch src/server.ts",
  "typecheck": "tsc --noEmit",
  "test": "node --import tsx --test test/app.test.ts",
  "build": "tsc",
  "start": "node dist/server.js",
  "verify": "npm run typecheck && npm test && npm run build"
}
```

```bash
npm run verify
```

Publication sur GitHub :

```bash
git add .
git commit -m "feat: add starter app with local verification"
git remote add origin https://github.com/nawreszr/cicd-lab.git
git push -u origin main
```

---

## 3. Premier déploiement sur Render

Service web Node créé depuis le dépôt GitHub, branche `main`, instance gratuite.

| Paramètre | Valeur |
|---|---|
| Build Command | `npm ci && npm run build` |
| Start Command | `npm start` |
| Health Check Path | `/api/health` |
| Auto-Deploy | On Commit (modifié plus tard, voir section 5) |

---

## 4. Intégration continue avec GitHub Actions

```bash
git switch -c feature/github-actions-ci
mkdir -p .github/workflows
```

Fichier `.github/workflows/ci.yml` :

```yaml
name: CI

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  contents: read

jobs:
  verify:
    name: Verify
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Check out repository
        uses: actions/checkout@v7

      - name: Set up Node.js
        uses: actions/setup-node@v7
        with:
          node-version: 24
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Verify application
        run: npm run verify
```

```bash
npm ci && npm run verify          # reproduire la CI en local
git add .
git commit -m "ci: add GitHub Actions verification workflow"
git push -u origin feature/github-actions-ci
```

Une pull request est ouverte vers `main`. Une fois `Verify` réussi, elle est fusionnée avec **Create a merge commit**, puis :

```bash
git switch main
git pull --ff-only
git branch -d feature/github-actions-ci
```

---

## 5. Déployer seulement si la CI réussit

Dans Render, le réglage **Auto-Deploy** passe de *On Commit* à **After CI Checks Pass**. Ainsi, un push sur `main` ne déploie que si la CI de ce commit est verte.

---

## 6. Protection de la branche main

*GitHub → Settings → Rules → Rulesets → New ruleset* : **Protect main**.

| Paramètre | Valeur |
|---|---|
| Enforcement | Active |
| Target | Default branch (`main`) |
| Require a pull request before merging | Activé |
| Required approvals | 0 (démo en solo) |
| Require status checks to pass | Activé |
| Required check | `Verify` (GitHub Actions) |
| Bypass list | Vide |

Le workflow *produit* les résultats ; le ruleset les *impose*.

---

## 7. Un changement normal dans le pipeline

```bash
git switch -c feature/update-homepage
```

Ajout dans le HTML de la page d'accueil (`src/app.ts`) :

```html
<p>Verified by GitHub Actions. Deployed on Render.</p>
```

```bash
npm run verify
git add .
git commit -m "feat: update homepage deployment message"
git push -u origin feature/update-homepage
```

Pull request → attente de `Verify` → merge → attente du check sur `main` et du déploiement Render.

```bash
git switch main
git pull --ff-only
git branch -d feature/update-homepage
```

---

## 8. Démonstration d'un test en échec

```bash
git switch -c feature/version-endpoint
```

Test ajouté **avant** la route, dans `test/app.test.ts` :

```ts
test('GET /api/version returns the current version', async () => {
  const response = await fetch(`${baseUrl}/api/version`);

  assert.equal(response.status, 200);
  assert.deepEqual(await response.json(), {
    version: process.env.APP_VERSION ?? 'development',
  });
});
```

Ce test échoue volontairement (HTTP 404 au lieu de 200).

```bash
npm test
git add .
git commit -m "test: demonstrate missing version endpoint"
git push -u origin feature/version-endpoint
```

La pull request ouverte est **bloquée** : le job `Verify` est en échec et le merge est impossible.

**Correction** : ajout de la route avant `export default app` dans `src/app.ts` :

```ts
app.get('/api/version', (_request, response) => {
  response.json({
    version: process.env.APP_VERSION ?? 'development',
  });
});
```

```bash
npm run verify
git add .
git commit -m "feat: add version endpoint"
git push
```

La pull request se met à jour toute seule et le nouveau `Verify` réussit. Après le merge et le déploiement, `/api/version` répond :

```json
{"version":"development"}
```

---

## 9. Prévisualisation des pull requests

Une **preview** est une copie temporaire de l'application, construite à partir d'une pull request, avec sa propre URL. La production continue de servir `main`, et Render supprime la preview quand la PR est fusionnée ou fermée.

Dans Render → *Previews* → **Pull Request Previews : Automatic**.

```bash
git switch -c feature/preview-demo
```

Ajout dans le HTML de `src/app.ts` :

```html
<p>Preview deployment</p>
```

```bash
npm run verify
git add .
git commit -m "feat: add preview deployment message"
git push -u origin feature/preview-demo
```

La pull request reste **ouverte** : la preview affiche le nouveau message, alors que la production ne l'affiche pas encore.

Après la revue, merge de la PR, puis :

```bash
git switch main
git pull --ff-only
git branch -d feature/preview-demo
```

La production affiche alors le message et Render supprime la preview.

---

## 10. Résumé des captures

| Capture | Contenu |
|---|---|
| `render-first-deploy.png` | Premier déploiement Render et réponse de `/api/health` |
| `actions-verify-pass.png` | Job `Verify` réussi |
| `render-auto-deploy.png` | Auto-Deploy : After CI Checks Pass |
| `ruleset-protect-main.png` | Ruleset *Protect main* avec le check `Verify` |
| `homepage-updated.png` | Page publique avec le nouveau message |
| `pr-failing-verify.png` | `Verify` en échec et merge bloqué |
| `api-version.png` | Réponse de `/api/version` |
| `preview-vs-production.png` | URL de preview et URL de production |

Le rapport (pipeline en mes propres mots, comparaison avec le TP2 et difficultés) est dans [`REPORT.md`](REPORT.md).
