# Rapport TP3 : CI/CD avec GitHub Actions et Render

## 1. Le Pipeline en mes mots
Le pipeline CI/CD utilise GitHub Actions pour vérifier automatiquement le code (typecheck, tests, build) à chaque Pull Request. Une fois les tests validés (coche verte "Verify"), le code est fusionné dans la branche `main`. Render est configuré pour ne déployer la nouvelle version que si les checks CI de GitHub Actions sont passés avec succès (`After CI Checks Pass`).

## 2. Comparaison avec le TP2 (Coolify vs GitHub Actions + Render)
| Critère | Coolify (TP2) | GitHub Actions + Render (TP3) |
|---|---|---|
| **Hébergement** | Auto-hébergé (VM locale) | Managé (Cloud de Render) |
| **Déclencheur** | Webhook Forgejo → Coolify | Push/Merge GitHub → Actions → Render |
| **Gestion du serveur** | Totale (on gère Docker, Traefik) | Aucune (Render s'occupe de tout) |
| **CI (Tests)** | Non inclus par défaut | Natif via GitHub Actions |
| **Coût** | Gratuit (hors VM) | Freemium (instance Free) |

## 3. Difficultés rencontrées
- **Historique Git** : Conflit d'historique ("entirely different commit histories") lors de la création de la PR. Résolu en créant une branche propre basée sur `origin/main`.
- **Permissions du Token** : Refus de push du fichier `ci.yml` (scope `workflow` manquant). Résolu en générant un nouveau Token GitHub avec le scope `workflow`.
- **Script manquant** : Le pipeline a échoué car la commande `verify` n'existait pas dans le `package.json` poussé. Résolu en l'ajoutant et en poussant la correction.
