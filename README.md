# Examen final — Sécurité des données — OWASP Juice Shop

Audit de sécurité, remédiation et automatisation CI/CD réalisés sur OWASP Juice Shop
dans le cadre de l'examen final du module Sécurité des données (L3 SIMAC, UN-CHK).

**Auteur :** Babacar Gueye

## Structure du dépôt

```
juice-shop-exam/
├── README.md                    → ce fichier
├── Jenkinsfile                  → pipeline CI/CD (Checkout → Build → SAST → SCA → Report)
├── reports/                     → rapports générés par le pipeline
│   ├── bearer-report.html       → rapport SAST + secrets (Bearer CLI), lisible dans un navigateur
│   ├── bearer-report.json       → même rapport, format JSON
│   └── npm-audit-report.json    → résultat de npm audit dans le pipeline (voir limite ci-dessous)
├── screenshots/examen/           → captures d'écran des étapes validées (sous-dossier dédié,
│                                    distinct des captures promotionnelles d'origine du projet)
└── [code source de Juice Shop]  → application auditée (bkimminich/juice-shop)
```

## Comment exécuter le projet

L'application est déployée via Docker, dans une VM Kali Linux (ou tout environnement Linux avec Docker installé) :

```bash
docker pull bkimminich/juice-shop
docker run -d -p 127.0.0.1:3000:3000 --name juice-shop bkimminich/juice-shop
```

L'application est ensuite accessible sur : http://localhost:3000

## Comment lancer les analyses de sécurité

### Analyse SAST + détection de secrets (Bearer CLI)

```bash
bearer scan . --scanner=sast,secrets --format html --output reports/bearer-report.html --exit-code 0
bearer scan . --scanner=sast,secrets --format json --output reports/bearer-report.json --exit-code 0
```

### Analyse des dépendances (SCA — npm audit)

```bash
npm install --package-lock-only --ignore-scripts
npm audit
```

> **Remarque :** le fichier `.npmrc` fourni avec Juice Shop contient `package-lock=false`,
> ce qui empêche `npm audit` de fonctionner dans un environnement d'exécution propre
> (erreur `ENOLOCK`) tant qu'un `package-lock.json` n'a pas été généré manuellement au
> préalable, comme ci-dessus. C'est pour cette raison que `npm audit` a été exécuté
> manuellement en complément du pipeline plutôt que de dépendre uniquement de son
> résultat automatisé (voir section « Limites » ci-dessous et la Partie 7.6 du rapport).

## Pipeline Jenkins

Le pipeline est défini dans le [`Jenkinsfile`](./Jenkinsfile) à la racine du dépôt, selon
l'architecture suivante :

```
1. Checkout                    → clone du dépôt GitHub (branche main)
2. Build / Preparation         → vérification de l'environnement Node.js
3. Security Analysis           → scan Bearer CLI (SAST + secrets)
4. Additional Security Check   → npm audit (SCA)
5. Report Generation           → archivage des rapports comme artefacts Jenkins
```

Le pipeline est déclenché automatiquement à chaque `git push` sur `main`, via un webhook
GitHub pointant vers l'instance Jenkins locale (exposée via un tunnel ngrok pendant les tests).

### Credential requis

Le pipeline utilise un credential Jenkins de type « Username with password » (Personal
Access Token GitHub avec le scope `repo`), enregistré sous l'ID `identifiants-github`.

## Outils utilisés

| Outil | Rôle | Étape du pipeline |
|---|---|---|
| Docker | Conteneurisation de l'application cible et de Jenkins | Environnement |
| Bearer CLI | SAST (analyse statique) + détection de secrets codés en dur | Security Analysis |
| npm audit | SCA (analyse de composition logicielle / dépendances) | Additional Security Check |
| Jenkins | Orchestration du pipeline CI/CD | — |
| ngrok | Exposition temporaire de Jenkins pour recevoir le webhook GitHub | — |
| GitHub | Hébergement du code source, déclenchement du pipeline | — |

## Limites connues

- **Notification automatique non implémentée** : le pipeline s'arrête à l'archivage des
  rapports (`Report Generation`). Aucune étape d'envoi automatique par email n'a été
  ajoutée (contrairement à une architecture de référence utilisant le plugin *Extended
  E-mail Notification* avec SMTP Gmail) — choix assumé par manque de temps, documenté
  dans le rapport (Partie 7.6).
- **npm audit dans le pipeline** : échoue avec l'erreur `ENOLOCK` du fait du `.npmrc` du
  projet (`package-lock=false`). L'analyse SCA complète a donc été menée manuellement en
  parallèle (résultat : 46 vulnérabilités, détaillées dans le rapport).

## Vulnérabilités couvertes par cet audit

Sept vulnérabilités ont été identifiées, exploitées et documentées avec preuve à l'appui
dans le rapport (`Rapport_Examen_Securite_Donnees.docx`) : injection SQL (CWE-89), fuite
du hash de mot de passe via l'API (CWE-200), exposition du dossier `/ftp/` (CWE-200/548),
XSS DOM-based (CWE-79), secret RSA codé en dur (CWE-798), mauvaise configuration des
en-têtes de sécurité (CWE-16), et dépendances tierces vulnérables (CWE-1104 et divers).

**Décision finale de déploiement : Reject Deployment** — voir le rapport complet pour le
détail de l'analyse d'impact, des remédiations et de l'analyse critique.
