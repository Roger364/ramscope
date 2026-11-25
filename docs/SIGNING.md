# Signature de code — RamScope

## État actuel

- **Pas de signature** : le `.exe` n'est pas signé (v1.0.0). SmartScreen affiche un avertissement au premier lancement.
- **Signature en cours** : demande soumise à SignPath Foundation (novembre 2026). Le workflow `release.yml` est prêt à signer dès que SignPath approuve le projet et que les variables/secrets GitHub sont configurés.
- **Release existante** : v1.0.0 publiée sur GitHub (binaire non signé + SHA-256).
- La page site renvoie vers GitHub + instructions de compilation et de vérification du SHA-256.

## Piste choisie : SignPath.io

SignPath.io est gratuit pour les projets open source. Il signe les binaires via un workflow GitHub Actions.

### Prérequis

1. **Compte SignPath.io** : créer un compte sur https://signpath.io (gratuit pour open source).
2. **Organisation** : créer une organisation SignPath (ex. `bisse-studio`).
3. **Projet** : créer un projet `ramscope` dans SignPath.
4. **Politique de signature** : créer une politique `release-signing` de type `release`.
5. **Configuration d'artefact** : créer `windows-exe` (type `File`, extension `.exe`).
6. **Certificat** : SignPath fournit un certificat OV gratuit pour les projets open source.

### Configuration GitHub

Dans le dépôt `Roger364/ramscope` :

1. **Variables** (Settings → Variables → Actions) :
   - `SIGNPATH_ORGANIZATION_ID` = ID de l'organisation SignPath

2. **Secrets** (Settings → Secrets → Actions) :
   - `SIGNPATH_API_TOKEN` = jeton API SignPath (généré dans SignPath → API tokens)

### Workflow

Le fichier `.github/workflows/release.yml` est déjà en place. Il :
1. Build le `.exe` en release.
2. Lance les tests + clippy + fmt.
3. Soumet le binaire à SignPath pour signature (si `SIGNPATH_ORGANIZATION_ID` est défini).
4. Crée une release GitHub draft avec le binaire (signé ou non) + SHA-256.

### Déclenchement

- **Tag** : `git tag v1.0.1 && git push origin v1.0.1`
- **Manuel** : GitHub → Actions → Release → Run workflow → entrer la version.

### Fallback (sans SignPath)

Si `SIGNPATH_ORGANIZATION_ID` n'est pas défini, le workflow produit un binaire non signé et crée la release quand même. Le site ne proposera pas de téléchargement direct tant que le binaire n'est pas signé.

### Mise à jour du site

Une fois un binaire signé disponible :
1. Mettre à jour `site-v2/content/site.ts` → `downloadUrl` dans `openSourceTools[0]`.
2. Mettre à jour la page Open source pour afficher le bouton de téléchargement.
3. Mettre à jour `ramscope.md` dans le vault.

## Dossier de candidature (prêt à soumettre)

État vérifié le 2026-11-21 : dépôt public `Roger364/ramscope`, licence MIT, CI verte (runs `43681932387`, `43663058587` et Release `44925940966`), `release.yml` déjà câblé sur `signpath/github-action-submit-signing-request@v1` (variable `SIGNPATH_ORGANIZATION_ID`, secret `SIGNPATH_API_TOKEN`). Le compte et l'organisation SignPath restent à créer par le mainteneur ; aucun secret n'est versionné.

Texte à coller dans le formulaire :

```text
Project: RamScope
Repository: https://github.com/Roger364/ramscope (public, MIT)
Maintainer: Roger364
Platform: Windows 10/11 x64, single portable .exe, no installer
What it does: inventories RAM, services and startup items, explains memory usage,
and applies reversible tweaks only after explicit user confirmation. The UI is
served on loopback (127.0.0.1); no telemetry, no network access, no bundled
third-party binaries, no auto-update.
Why signing: the .exe is distributed directly from GitHub Releases and triggers
SmartScreen when unsigned.
Build: Rust stable, `cargo build --release` in ramscope/ (static CRT); the release
workflow submits ramscope.exe to SignPath with
signpath/github-action-submit-signing-request@v1.
Artifact: ramscope.exe (Windows x64), release signing policy.
```

Avant le dépôt : publier la page de présentation du site et la note de version pour que le projet soit identifiable publiquement (dépôt, CI et release v1.0.0 le sont déjà).

## Alternatives évaluées

| Option | Coût | SmartScreen | Verdict |
|---|---|---|---|
| SignPath.io | Gratuit (open source) | Évité après réputation | **Choisi** |
| Microsoft Store (MSIX) | ~19 $ one-time | Évité | Packaging MSIX incompatible avec UAC/%LOCALAPPDATA% |
| winget | Gratuit | S'applique quand même | Canal secondaire possible |
| Certificat OV/EV payant | 200-500 $/an | EV évite immédiatement | Trop cher pour l'instant |
