# RamScope — dossier de passation pour Hermes

- **Destinataire :** Hermes, responsable du site et de la majeure partie du développement
- **Mise à jour :** 21 novembre 2026
- **Dernier commit pris en compte :** `778acd2` (`docs: align signing status with reality (SignPath request pending)`), branche `main`.
- **Dépôt :** <https://github.com/Roger364/q-memory-manager> (public)
- **Site :** <https://bissestudio.ch> — code du site séparé de ce dépôt
- **Version :** 1.0.0
- **Licence :** MIT, copyright Bisse Studio

Ce document est un point d'entrée. Les références faisant autorité restent le code, les tests, [`SAFETY.md`](../SAFETY.md) et [`PLAN-LOT9.md`](../PLAN-LOT9.md). En cas de contradiction, le comportement réel du code et les garde-fous de sécurité priment ; il faut alors corriger la documentation.

## 1. Le projet en bref

RamScope est un outil Windows 10/11 qui inventorie la mémoire, les services, les tâches planifiées et les entrées de démarrage, puis explique ce qui occupe la RAM et propose des réglages réversibles. Il ne supprime aucun fichier et ne modifie rien sans confirmation explicite.

- Un seul exécutable Rust, sans installation ni dépendance à l'exécution.
- Interface web servie uniquement sur la boucle locale, ouverte dans une fenêtre d'application.
- Actions réversibles pour les services, démarrages et tâches ; la terminaison d'un processus est explicitement irréversible.
- Positionnement : **observation et décision**, pas nettoyeur automatique ni « accélérateur Windows ».
- Public visé : utilisateurs avancés, support informatique, développeurs, personnes qui veulent comprendre la RAM plutôt que cliquer sur un bouton magique.

### État au 21 novembre 2026

- Référence actuelle : `778acd2` (`docs: align signing status with reality (SignPath request pending)`), branche `main`, arbre propre.
- CI GitHub verte sur `778acd2` (runs du 16 et du 20/11).
- Suite locale : **83 tests Rust + 4 tests Node**, tous verts ; clippy et fmt propres.
- La page Open Source `/open-source/` est publiée (cf. handoff `STATE.md`, phase 6) ; ce document ne la décrit plus comme absente.
- Branche `main`, dépôt public, GitHub détecte la licence MIT.
- CI GitHub verte : run `43682552060`, toutes les étapes réussies.
- Suite locale : **83 tests Rust + 4 tests Node**, tous verts.
- Le dossier de preuves a été supprimé ; l'archive locale `sandbox-test-2026-11-15.zip` est conservée (117 entrées, 16 Mo, SHA-256 `60547ca05d73470a8298078e2f47ba9c2e26260f1b3ea4e310e8d014fcd64dd9`).
- Le Sandbox C6 a été fermé après la suppression.
- Le fichier `review - workbuddy.md` n'est plus dans l'arbre courant, mais reste dans l'historique Git public.

## 2. Rôles et frontières de décision

| Sujet | Qui décide | Règle pratique |
|---|---|---|
| Vision produit, priorités | Ro | Hermes peut proposer et arbitrer le technique, mais les changements de positionnement se valident avec Ro. |
| Site, contenu, SEO, publication | Hermes | Le code du site n'est pas dans ce dépôt ; aucune credential de site ne doit être ajoutée ici. |
| Code, tests, CI, documentation technique | Hermes | Les changements ordinaires peuvent être commités et poussés après vérification. |
| Licence, copyright, marque | Ro | Ne pas changer `LICENSE`, le nom du projet ou la mention Bisse Studio sans accord explicite. |
| Réécriture d'historique, force-push, suppression de données | Ro | Aucune réécriture sans autorisation ; préférer un commit correctif. |
| Vulnérabilité | Ro + canal privé | Ne pas ouvrir d'issue publique ; utiliser `contact@bissestudio.ch`. |

« Ro » est un surnom. La notice publique reste `Copyright (c) 2026 Bisse Studio` ; ne pas la remplacer par le surnom ni inventer un nom civil.

## 3. Carte du dépôt

| Chemin | Rôle |
|---|---|
| `ramscope/src/main.rs` | CLI, modes d'élévation, démarrage du serveur et de la fenêtre. |
| `ramscope/src/model.rs` | Types sérialisés : snapshot, actions, gains, provenance, rapports. |
| `ramscope/src/rules.rs` | 601 règles, niveaux, sources, listes de services et processus protégés. |
| `ramscope/src/win.rs` | Collecte Win32, application des actions, restauration, élévation. |
| `ramscope/src/plan.rs` | Logique pure de validation et de calcul des lots, sans FFI. |
| `ramscope/src/server.rs` | Serveur HTTP local, routes, jeton, `Host`, `Origin`, CSP, fichiers statiques. |
| `ramscope/src/report.rs` | Rapport JSON/HTML et échappement. |
| `ramscope/src/demo.rs` | Données simulées ; aucun accès réel à la machine. |
| `ramscope/src/web/` | Interface : `index.html`, `app.js`, `core.js`, `style.css`, tests Node. |
| `ramscope/src/ffi.rs`, `platform.rs`, `util.rs` | Déclarations Win32, abstraction de plateforme, utilitaires. |
| `PLAN.md` | Journal historique des lots, décisions et défauts connus. |
| `PLAN-LOT9.md` | Plan de travail courant et état des constats C1–C15. |
| `SAFETY.md` | Limites d'usage, modèle de menace, actions réversibles ou non. |
| `README.md` | Documentation utilisateur et options CLI. |
| `CONTRIBUTING.md` | DCO, tests, règles de contribution. |
| `SECURITY.md` | Canal privé et périmètre de sécurité. |
| `LICENSE` | MIT, copyright Bisse Studio. |
| `THIRD_PARTY_NOTICES.md` | Textes intégraux des dépendances, généré par `cargo-about`. |
| `.github/workflows/ci.yml` | Vérification Windows, Rust, Node, notices et fichiers non suivis. |
| `docs/evidence/SHA256SUMS` | Sommes des preuves versionnées vérifiées par la CI. |

Les preuves historiques ne sont plus dans l'arbre courant ; elles restent dans l'archive ZIP locale, ignorée par Git.

## 4. Construire, tester, lancer

Prérequis : Windows 10/11, Rust stable avec la cible Windows, Node.js 24 pour les tests, Git Bash pour les commandes de la CI. `cargo-about` n'est nécessaire que pour régénérer les notices.

```bash
cd ramscope
cargo fmt --check
cargo test
cargo clippy --all-targets -- -D warnings
cargo build --release
```

Le binaire est produit dans `ramscope/target/release/ramscope.exe`.

Lancement :

```bash
ramscope.exe --demo          # données simulées, aucun accès réel
ramscope.exe --no-admin      # lecture seule, aucune élévation possible
ramscope.exe                 # collecte réelle, lecture seule si non élevé
ramscope.exe --admin         # élévation UAC au lancement
```

Options utiles : `--port N`, `--host 127.0.0.1|localhost`, `--no-browser`, `--help`.

Pour les tests Node seuls :

```bash
cd ramscope
node --test src/web/core.test.mjs
```

`cargo test` lance aussi ces tests Node : Node doit être installé, sinon la suite Rust échoue volontairement.

### Régénérer les notices tierces

Après toute modification de `Cargo.toml` ou `Cargo.lock` :

```bash
cd ramscope
cargo about generate about.hbs -o ../THIRD_PARTY_NOTICES.md

# Variante avec l'installation locale du dépôt :
target/cargo-tools/bin/cargo-about.exe generate about.hbs -o ../THIRD_PARTY_NOTICES.md
```

Vérifier le diff et committer le fichier régénéré avec la mise à jour des dépendances. La configuration `about.toml` accepte Apache-2.0, MIT, Unicode-3.0 et Unlicense ; les textes intégraux doivent rester présents.

## 5. Architecture et flux

```text
main.rs
  └─ choix du mode (demo / lecture seule / admin / strict)
      └─ collecte du snapshot (demo.rs ou win.rs)
          └─ règles et conseils (rules.rs)
              └─ serveur local (server.rs)
                  └─ interface web (web/)
                      └─ /api/apply ou /api/restore
                          └─ plan::validate + garde moteur (win.rs)
                              └─ point de restauration JSON
```

Points importants :

- `plan.rs` reste pur et testable sur toutes les plateformes ; les appels FFI restent dans `win.rs`/`ffi.rs`.
- Les actions d'un lot sont revérifiées contre un snapshot recollecté : le client ne fait pas autorité.
- Les services et processus protégés sont refusés dans `plan.rs` **et** dans le moteur `win.rs`.
- Les entrées de démarrage et les tâches n'ont pas de liste de protection indépendante : validation amont + relecture de l'état avant écriture. Ne pas présenter cela comme « deux verrous ».
- Un lot avec au moins une action réussie et réversible écrit un point de restauration JSON. Une terminaison de processus ne peut pas être restaurée.
- Les données locales sont sous `%LOCALAPPDATA%\RamScope` : `rollbacks\`, `rapports\`, `erreur.log`.
- En développement hors Windows : `ramscope-data/` dans le répertoire courant.

## 6. Interface HTTP et invariants de sécurité

Routes principales :

| Méthode | Route | Usage |
|---|---|---|
| GET | `/api/snapshot` | Snapshot courant ; `force=1` pour recollecter. |
| GET | `/api/rollbacks` | Liste des points de restauration. |
| GET | `/api/health` | État `ok`, `demo`, `admin`, `can_elevate`. |
| POST | `/api/apply` | Applique un lot validé. |
| POST | `/api/restore` | Rejoue les actions inverses d'un point de restauration. |
| POST | `/api/elevate` | Demande l'élévation UAC ; `403` en mode strict. |
| POST | `/api/export` | Écrit les rapports JSON et HTML. |
| POST | `/api/open-dir` | Ouvre le dossier des rapports. |
| POST | `/api/quit` | Arrêt coopératif du serveur. |

Invariants à ne jamais affaiblir :

- Écoute uniquement sur `127.0.0.1` ou `localhost` ; toute autre adresse est refusée au démarrage.
- En-tête `Host` validé exactement, pages comprises ; protection contre le rebinding DNS.
- Toute route `/api/` exige `X-RamScope-Token`, jeton CSPRNG de 32 octets.
- Toute mutation exige un `Origin` présent et exact ; `Sec-Fetch-Site` différent de `same-origin` refusé.
- Corps de mutation limité à 1 Mio, vérifié avant et pendant la lecture ; `413` au-delà.
- Validation serveur des actions et recollecte de l'état avant écriture.
- Élévation UAC explicite ; `--no-admin` rend `/api/elevate` inaccessible et masque le bouton.
- CSP : `script-src 'self'`, pas de script inline. `style-src 'unsafe-inline'` reste justifié par les styles calculés.
- Un processus local hostile du même compte reste hors du modèle de menace de la v1 : il peut lire la page et le jeton. Ne pas promettre une protection contre un malware local.

## 7. Tests, CI et limites connues

La CI exécute, sur `windows-latest` :

1. `cargo fmt --check`
2. `cargo test` (Rust + Node)
3. `cargo clippy --all-targets -- -D warnings`
4. `cargo build --release`
5. `node --check` sur `core.js` et `app.js`
6. `sha256sum -c docs/evidence/SHA256SUMS`
7. Refus des fichiers générés ou non suivis (`git diff --exit-code` + `git ls-files --others --exclude-standard`)

Le workflow a seulement `contents: read` : aucune publication automatique.

Dependabot est configuré pour ouvrir chaque semaine des PR Cargo (dossier `ramscope/`) et GitHub Actions. Toute mise à jour de dépendance doit passer la CI et être accompagnée de la régénération des notices.

Limites à annoncer honnêtement :

- Les refus d'identité réelle d'un processus vivant ne sont pas exercés par les tests unitaires.
- Le delta de mémoire mesuré a été vérifié en démonstration, pas sur une variation réelle.
- Le câblage DOM, les parcours clavier complets et le rendu visuel ne sont pas couverts par la suite Node.
- Edge n'a pas pu être installé dans le Sandbox de test ; le clic DOM a été validé sous Chrome for Testing (même moteur Chromium).
- Les 601 règles ont une provenance de famille, pas une revue individuelle datée.
- Il n'y a ni installeur, ni signature de code, ni publication automatique dans ce dépôt.
- D9-2b (helper élevé one-shot) est une conception, pas une fonctionnalité livrée : une instance lancée en administrateur le reste jusqu'à sa fermeture.

Ne pas transformer une limite en argument marketing. En cas de doute, écrire « non testé » plutôt que supposer.

## 8. Licence, contributions et distribution

- Licence **MIT** ; le texte est dans `LICENSE`.
- Copyright public : `Copyright (c) 2026 Bisse Studio`.
- La licence ne concède ni le nom « RamScope » ni la marque Bisse Studio.
- `ramscope/Cargo.toml` porte `license = "MIT"`, `repository` et `publish = false` : pas de publication sur crates.io.
- Les contributions utilisent le **DCO** (`Signed-off-by`), sans CLA pour l'instant.
- Toute contribution est distribuée sous MIT.
- Ne pas ajouter de secret, de jeton, de rapport contenant des données personnelles ni de preuve Sandbox dans le dépôt.
- Une diffusion commerciale, une double licence ou un changement de titulaire doit être validé par Ro et, si nécessaire, par un conseil juridique.

## 9. Site et présence publique

Page Open Source publiée au 21 novembre 2026 (`/open-source/` en ligne, cf. handoff `STATE.md`, phase 6) ; ce document ne la présente plus comme absente. Le site `bissestudio.ch` est géré séparément. Je n'ai pas audité son code, son hébergement ni ses credentials.

- présenter RamScope en une phrase : inventaire mémoire/services/démarrage, explications et réglages réversibles ;
- lien vers le dépôt GitHub et vers `LICENSE` ;
- mention « Windows 10/11 », « MIT », « Bisse Studio » ;
- lien vers les instructions de compilation plutôt que promettre un installeur signé qui n'existe pas ;
- lien vers `SECURITY.md` ou le contact privé pour les vulnérabilités ;
- éviter « nettoyeur », « accélérateur garanti », « supprime les fichiers inutiles », « 100 % sûr » ;
- ne pas suggérer une affiliation Microsoft.

La page légale existante indique « Bisse Studio — Ro, activité individuelle établie en Valais, Suisse ». « Ro » est un surnom ; ne pas le présenter comme un nom civil complet. Les mentions légales, la TVA et les obligations de consommation doivent être validées séparément avant une diffusion commerciale.

## 10. Travail ouvert et priorités

| Sujet | État | Suite proposée |
|---|---|---|
| Page Open Source | Publiée (21/11) | Page courte, honnête et liée au dépôt, en ligne sous `/open-source/`. |
| D9-2b — helper élevé one-shot | Conception seulement | Ne pas l'annoncer comme livré ; cadrer le canal, l'authentification, la durée de vie et les tests avant implémentation. |
| `review - workbuddy.md` dans l'historique | Ouvert | Ne pas réécrire l'historique sans Ro ; un `force-push` invaliderait les hashes cités dans les plans. |
| Test navigateur/Edge | Partiel | Refaire un parcours manuel des actions principales sous Edge ou Chromium, et consigner les limites. |
| Installeur et signature de code | Absents | À traiter avant une distribution grand public ; ne pas promettre un binaire signé. |
| Revue juridique commerciale | À faire | Mentions légales, consommation, marque et titularité ; hors périmètre du code. |
| Archive Sandbox | Conservée localement | Ne pas supprimer le ZIP ; il contient les 117 preuves et son hash est documenté. |

Ordre pragmatique : page Open Source → vérification manuelle navigateur → préparation d'une release propre → D9-2b → signature/installeur.

## 11. Méthode de travail

- Un commit par sujet, message court et explicite, en français comme l'historique existant.
- Pas de `force-push`, pas de réécriture d'historique, pas de suppression de données sans accord.
- Avant de pousser : exécuter les contrôles locaux, vérifier `git diff`, et s'assurer qu'aucun fichier non suivi ne serait publié.
- Toute correction de comportement doit avoir un **test de réversion nommé** qui échoue si le défaut revient.
- Toute modification d'interface doit être vérifiée dans un navigateur, avec les limites de test écrites.
- Toute modification de dépendance impose la régénération de `THIRD_PARTY_NOTICES.md`.
- Ne jamais contourner `Host`, le jeton, `Origin`, la validation serveur ou les verrous moteur pour faire passer un test.
- Les tests d'écriture réelle restent limités à une VM ou un Windows Sandbox jetable ; la machine quotidienne reste en lecture seule (`--no-admin`).

### Définition de « terminé »

1. Le comportement demandé est implémenté.
2. Un test de réversion nommé couvre le changement, ou la limite est explicitement documentée.
3. `cargo fmt --check`, `cargo test`, Clippy et le build release passent.
4. La documentation utilisateur ou sécurité est mise à jour si nécessaire.
5. La CI est verte et le dépôt reste propre.

## 12. Premiers pas recommandés pour Hermes

1. Cloner le dépôt et lancer `ramscope.exe --demo` pour voir l'interface sans toucher à la machine.
2. Lire `README.md`, `SAFETY.md` puis `PLAN-LOT9.md` (sections 9.9 à 9.12 et 10).
3. Rejouer les commandes de la section 4 et vérifier les 83 + 4 tests.
4. Publier la page Open Source sur `bissestudio.ch`.
5. Ouvrir une issue pour toute amélioration non triviale avant de coder.
6. Pour toute question de licence, de marque ou d'historique public, revenir vers Ro.

## 13. Contacts et ressources

- Dépôt : <https://github.com/Roger364/q-memory-manager>
- CI : <https://github.com/Roger364/q-memory-manager/actions>
- Site : <https://bissestudio.ch>
- Contact général et sécurité : `contact@bissestudio.ch`
- Vulnérabilités : canal privé décrit dans [`SECURITY.md`](../SECURITY.md), pas d'issue publique.
- Licence : [`LICENSE`](../LICENSE)
- Notices tierces : [`THIRD_PARTY_NOTICES.md`](../THIRD_PARTY_NOTICES.md)

Ce dossier décrit l'état du projet, pas un engagement contractuel. Les décisions juridiques et commerciales restent à Ro.
