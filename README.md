# RamScope

[![Licence: MIT](https://img.shields.io/badge/licence-MIT-blue.svg)](LICENSE)
[![CI](https://github.com/Roger364/q-memory-manager/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/Roger364/q-memory-manager/actions/workflows/ci.yml)

RamScope inventorie la mémoire, les services et le démarrage automatique d'un PC Windows 10/11, explique ce qui occupe la RAM, et propose des réglages réversibles. Il ne supprime aucun fichier et ne modifie aucun réglage sans confirmation explicite.

Un seul exécutable, aucune installation, aucune dépendance à l'exécution. L'interface est une page web servie sur la boucle locale et ouverte dans une fenêtre d'application.

## Compiler

Prérequis : Rust stable, cible Windows.

```text
cd ramscope
cargo build --release
```

Le binaire est produit dans `ramscope\target\release\ramscope.exe`.

## Lancer

```text
ramscope.exe                 # collecte réelle ; lecture seule si le processus n'est pas déjà élevé
ramscope.exe --admin         # collecte réelle, élévation UAC au lancement
ramscope.exe --no-admin      # aucune élévation demandée ; ne rétrograde pas une session déjà élevée
ramscope.exe --demo          # données simulées, aucun accès à la machine
```

| Option | Effet |
|---|---|
| `--port N` | port d'écoute (8765 par défaut, essaie les 11 suivants s'il est occupé) |
| `--host ADRESSE` | adresse d'écoute : `127.0.0.1` (défaut) ou `localhost` ; toute autre valeur est refusée au démarrage. `localhost` se lie à la résolution locale (souvent `::1`) : l'alias `127.0.0.1` n'est pas garanti |
| `--no-browser` | ne pas ouvrir la fenêtre automatiquement |
| `--admin` | demander l'élévation UAC au lancement, hors `--demo` |
| `--no-admin` | interdit toute demande d'élévation, y compris depuis l'interface ; ne rétrograde pas une session déjà élevée |
| `--demo` | données de démonstration, aucun accès réel |
| `--help` | aide |

Sans option, RamScope ne demande pas l'UAC. S'il n'est pas déjà élevé, il démarre en lecture seule : l'inventaire reste consultable, et pour appliquer un changement il faut cliquer sur « Passer en administrateur » et accepter l'UAC, ou relancer avec `--admin`. Depuis un terminal déjà administrateur, la session reste élevée. `--no-admin` désactive toute demande d'élévation : le bouton est masqué et `/api/elevate` répond `403` ; il ne rétrograde pas non plus une session déjà élevée. Un refus UAC lors de `--admin` laisse la session courante en lecture seule.

## Lire les chiffres

- **Mémoire physique** : total, disponible et charge, via `GlobalMemoryStatusEx`. « Utilisée » vaut total moins disponible.
- **Working set** : mémoire résidente d'un processus, la valeur que montre le Gestionnaire des tâches.
- **Mémoire privée** : mémoire non partagée, celle qu'un processus rend réellement en se fermant. C'est la base des gains « tout de suite ».
- **Commit, noyau, cache système** : compteurs système, via `GetPerformanceInfo`.
- **Mémoire compressée** : working set du processus « Memory Compression ». C'est une approximation du compteur système, pas le compteur lui-même.
- **RAM d'un service** : part du processus hôte (`RAM de l'hôte ÷ nombre de services hébergés`). Plusieurs services d'un même `svchost.exe` partagent la même mémoire : ces parts sont approximatives et ne s'additionnent pas.
- **Gains** : chaque gain porte sa nature — mesuré, estimé ou non chiffrable — et sa portée. Un gain « tout de suite » correspond à une fermeture ou à un arrêt ; un gain « au prochain démarrage » correspond à un service passé en manuel ou à un démarrage désactivé. Sans mesure, le gain est « non chiffrable », jamais inventé. Un zéro mesuré reste affiché « 0 Mo mesuré » : il n'est jamais confondu avec l'absence de mesure.
- **Hôtes partagés** : la part d'un service dans un `svchost.exe` n'est pas additive. Un hôte ne compte que si tous ses services figurent dans le lot ; son gain vaut alors la RAM de l'hôte, une seule fois. Sinon, aucun gain n'est annoncé pour cet hôte.
- **Arrêt immédiat** : cocher « Arrêter aussi les services maintenant » ne transforme pas un gain différé en gain immédiat. La mémoire disponible est mesurée avant et après l'application, avec une attente bornée ; le delta est affiché comme mesuré.

## Niveaux

| Niveau | Sens |
|---|---|
| Vital | Composant dont l'arrêt casse Windows ou la session. |
| Système | Composant Windows normal, à laisser tel quel. |
| À garder | Utile pour ton usage (pilote, sécurité, usage quotidien). |
| Selon usage | Le bon réglage dépend de ce que tu fais. |
| À alléger | Démarrage automatique ou agent d'arrière-plan superflu. |
| Superflu | Inutile dans la quasi-totalité des cas. |
| Non répertorié | RamScope ne sait pas : aucune action proposée. |

Un cadenas signale un élément **verrouillé** : le serveur puis le moteur refusent toute action, quel que soit le point d'entrée, et pas seulement dans l'interface. Le verrou est distinct du niveau : un service verrouillé mais non vital garde son niveau de la table (par exemple « À garder ») et n'est plus étiqueté « Vital ». Les règles portent une provenance de famille, pas une revue individuelle. Le panneau de détails et le rapport exporté affichent cette source et la mention de revue (`non datée` tant qu'aucune revue de fond n'a eu lieu) : voir `SAFETY.md`.

## Actions

- Type de démarrage d'un service : automatique, automatique différé, manuel, désactivé.
- Entrée de démarrage : activée ou désactivée via le mécanisme `StartupApproved`, le même que le Gestionnaire des tâches.
- Tâche planifiée : activée ou désactivée.
- Arrêt immédiat d'un service.
- Terminaison d'un processus : irréversible, sans restauration possible. Le PID est rouvert avant l'action ; si le chemin d'image ou l'heure de création diffèrent de l'analyse, la terminaison est refusée.

Chaque lot appliqué qui contient au moins une action réussie et réversible crée un point de restauration JSON ; un lot qui ne contient que des terminaisons n'en crée aucun. Le rapport exporté sépare le gain immédiat et le gain au prochain démarrage, avec la nature de chacun et la provenance de la règle appliquée. La restauration rejoue les actions inverses ; elle ne recrée pas un processus terminé.

## Interface de décision

- La sélection de masse « Sélectionner les éléments réversibles à gain différé » ne coche que des services, démarrages et tâches classés « superflu » ou « à alléger » ; elle ne sélectionne jamais un processus.
- « Terminer » reste une action directe : bouton rouge, étiquette « irréversible », jamais inclus dans un lot. La confirmation nomme le processus et rappelle que le travail non enregistré sera perdu.
- Le bouton « Détails » ouvre, pour chaque ligne, la source détectée, l'état actuel, l'effet tout de suite, l'effet au prochain démarrage, la réversibilité et la justification.
- La barre de sélection sépare le gain tout de suite et le gain au prochain démarrage ; chaque valeur porte son badge mesuré, estimé ou non chiffrable.

## Fichiers écrits

Sous Windows, dans `%LOCALAPPDATA%\RamScope` :

- `rollbacks\<id>.json` — points de restauration ;
- `rapports\ramscope-<date>.json` et `.html` — exports (créés à la demande) ;
- `erreur.log` — écrit uniquement si l'interface n'a pas pu démarrer.

Hors Windows, `ramscope-data\` dans le répertoire courant, pour le développement uniquement.

## Sécurité

- Écoute uniquement sur la boucle locale (`127.0.0.1` ou `localhost`) ; toute autre adresse est refusée au démarrage.
- L'en-tête `Host` est validé contre les adresses de la session : une requête portant un autre hôte, y compris par rebinding DNS, reçoit `403`, pages comprises.
- Toute route `/api/` exige un jeton de session généré par le CSPRNG du système (`BCryptGenRandom`, 32 octets) ; toute mutation exige un `Origin` présent et comparé exactement au `Host`, sans comparaison par suffixe (`403` si absent ou invalide). Un `Sec-Fetch-Site` présent et différent de `same-origin` est aussi refusé.
- Le jeton reste présent dans la page servie, mais uniquement pour un `Host` autorisé. Il protège la session navigateur : il n'authentifie pas un processus local, qui peut lire la page et récupérer le jeton. Voir la décision D9-1 dans `SAFETY.md`.
- Le corps d'une mutation est limité à **1 Mio** : `Content-Length` est contrôlé avant lecture, puis la lecture est bornée par `take` ; un dépassement reçoit `413`, y compris sans `Content-Length` ou en chunked.
- `/api/apply` recollecte le snapshot et refuse une action sans état observé, avec un état divergent, ou qui n'est pas exactement celle proposée par l'analyse courante. Le démarrage d'un service est réservé à la restauration interne.
- Le lancement par défaut ne demande pas l'UAC ; `--admin` est le seul mode qui l'élève au démarrage. `--no-admin` interdit `/api/elevate` côté serveur, et l'interface masque le bouton d'élévation.
- La politique de contenu interdit désormais les scripts inline (`script-src 'self'`) : le jeton est lu dans une meta et le bouton d'élévation est lié dans `app.js`. Les styles inline restent nécessaires aux largeurs et couleurs calculées ; `style-src 'unsafe-inline'` est conservé pour cette raison.

## Ce qui n'a pas été testé

- C2 a été vérifié en démonstration par des requêtes HTTP et dans Chromium (refus d'un état nul/divergent, refus d'une action hors snapshot), puis par des actions réelles dans un Windows Sandbox jetable. Le service `dmwappushservice` est passé de Manuel à Désactivé, la tâche `StartupAppTask` de Ready à Disabled, et une entrée de démarrage de test a vu sa valeur native `StartupApproved` passer de l'état activé (valeur absente) à `3` (désactivée), puis à `2` (activée) ; `Win32_Service.StartMode`, `Get-ScheduledTask` et le registre ont confirmé les changements, et les restaurations ont ramené les objets à leur état initial. L'entrée de démarrage synthétique a ensuite été supprimée. Le test de tâche a aussi révélé puis corrigé la lecture d'état via le repli `schtasks` quand `System32\Tasks` est inaccessible. Limites : un service, une tâche et une entrée de démarrage synthétique, compte Sandbox administrateur, pas de consentement UAC standard ni d'action processus réelle.
- La collecte réelle a été exécutée le 2026-11-14 en lecture seule (`--no-admin`). La mémoire totale, les 289 services et les 201 tâches planifiées correspondent aux API Windows (`Get-Service`, `Get-ScheduledTask`) ; les 7 entrées de démarrage sont plus complètes que les 6 de `Win32_StartupCommand`, car RamScope lit aussi le registre 32 bits. Le chemin administrateur qui lit directement les XML des tâches n'a pas été comparé au repli `schtasks`, et le repli ajoute environ 0,6 à 0,8 s à la collecte en lecture seule.
- Le repli des tâches utilise `%SystemRoot%\System32\schtasks.exe` par chemin absolu ; les ACL partielles sont fusionnées par chemin et dédupliquées.
- Les tests unitaires couvrent la validation des actions, l'identité simulée d'un processus, l'agrégation des gains par hôte, la nature des gains dans le rapport, l'audit des 601 règles (unicité, casse, modes, sources, verrous), le faux positif `php`/`hp` et l'absence d'action des heuristiques sans source. Les refus d'identité réelle (`win.rs`) restent non exercés sur un processus vivant ; le delta de mémoire mesuré a été vérifié en démonstration, où la mémoire simulée ne varie pas. La revue visuelle de l'étape 5 a ouvert les panneaux de détails et les modales dans Chromium ; elle n'a pas fait de parcours clavier avec de vraies frappes ni rouvert les fichiers exportés. Les tests `decode_une_tache_xml`, `lit_l_etat_natif_d_une_tache_pour_le_rollback`, `le_repli_couvre_une_acl_partielle`, `fusionne_les_taches_par_chemin_sans_doublon` et `decode_la_sortie_de_schtasks` couvrent le parseur, la décision de repli, la déduplication et le wrapper, mais pas l'injection d'une ACL Windows réelle ni une sortie réelle de `schtasks` en échec. Le test réel en Sandbox a couvert le repli de lecture d'état. Le durcissement C3/C4 est couvert par `le_serveur_refuse_origine_absente_et_corps_geant_avant_mutation` (vrai gestionnaire HTTP et corps chunked), `une_mutation_sans_origine_est_refusee`, `une_origine_cross_site_est_refusee` et `un_corps_trop_volumineux_est_refuse_avant_lecture`, avec vérification HTTP réelle des `403` et `413` ; C5 par les tests directs `le_moteur_refuse_un_service_protege_sans_plan`, `le_moteur_refuse_de_demarrer_un_service_protege_sans_plan` et `le_moteur_refuse_un_etat_divergent_sans_plan`. Le détail des commandes et des codes retour figure dans `PLAN.md`, sections 13 à 17. C1/C14 sont couverts par `l_audit_accepte_une_regle_datee_et_refuse_une_date_invalide`, `le_snapshot_expose_la_provenance_des_regles`, `le_rapport_expose_la_provenance_des_regles`, `un_zero_mesure_reste_mesure_dans_le_rapport`, `un_zero_mesure_survit_au_merge` et le test Node `un zéro mesuré reste une mesure, un zéro estimé est ignoré` ; le rendu n'a pas été rouvert dans un navigateur. C9–C13 sont couverts par `extract_exe_supporte_une_casse_unicode_sans_paniquer`, `le_dernier_port_est_essaye_et_la_plage_ne_deborde_pas`, `l_arret_cooperatif_attend_les_quatre_threads` et `echappe_l_apostrophe_comme_l_interface` ; la correction des deux « Irrréversible » est vérifiée par le test d'interface. Le test d'arrêt utilise quatre vrais travailleurs mais ne lance pas la commande `serve` complète ; aucun socket n'est ouvert sur le port 65 535. C8 extrait les décisions pures de l'interface dans `web/core.js`, testé par `node --test` : quatre tests Node couvrent les lots, les gains (dont le zéro mesuré), les hôtes partagés et la fermeture de modale. `cargo test` exige donc Node.js ; la CI l'installe déjà. Le câblage DOM et le rendu visuel ne sont pas couverts par ces tests.
- D9-2a est couvert par des tests unitaires : défaut sans UAC, `--admin`, `--no-admin` strict, conflit des deux options, relance `--elevated` sans boucle et propagation de `can_elevate` au snapshot. Le consentement UAC et son annulation ont été testés le 2026-11-15 dans un Windows Sandbox avec un **compte standard** : acceptation réelle (`admin:true`, `can_elevate:false`) et annulation réelle (`admin:false`, `can_elevate:true`), avec `/api/quit` 200 dans les deux cas. Le Sandbox étant livré avec UAC désactivé, `EnableLUA=1` a été appliqué et le Sandbox redémarré ; le bureau sécurisé a été désactivé uniquement pour automatiser le clic. La route `POST /api/elevate` du bouton a aussi été testée sur compte standard : elle a déclenché une UAC acceptée et l'instance élevée est restée sur le port 8898 (`admin:true`, `can_elevate:false`, `/api/quit` 200). Ce test a révélé puis fait corriger un conflit de port lors de la relance élevée. Le clic DOM et le rechargement ont été exécutés avec Chrome for Testing portable : clic sur « Passer en administrateur », UAC acceptée, puis page rechargée affichant `Administrateur` sans bouton d'élévation. Edge n'a pas pu être installé dans le Sandbox (MSI 1603/1722) ; le test DOM utilise donc Chromium, pas Edge. Le compte standard a aussi validé `--no-admin` (`can_elevate:false`, `/api/elevate` 403, aucun `consent.exe`) et `--host localhost` (page et API via `localhost:8899`) ; l'alias `127.0.0.1` n'était pas joignable sur cette liaison. La note contradictoire du snapshot `--no-admin` a été corrigée et couverte par `le_snapshot_strict_ne_propose_pas_de_relance_admin`. Les preuves détaillées sont conservées dans l'archive locale `sandbox-test-2026-11-15.zip`, non versionnée.

## Open source

RamScope est publié sous licence **MIT** par Bisse Studio. Le code peut être utilisé, modifié et redistribué selon les termes de [`LICENSE`](LICENSE). La licence ne concède ni le nom « RamScope » ni la marque Bisse Studio.

Les contributions sont bienvenues ; lisez [`CONTRIBUTING.md`](CONTRIBUTING.md) avant d'ouvrir une pull request. Pour signaler une vulnérabilité, utilisez la procédure privée décrite dans [`SECURITY.md`](SECURITY.md). Les composants tiers et leurs textes de licence sont reproduits dans [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md), généré avec `cargo-about`.

Voir `SAFETY.md` pour les limites d'usage et `PLAN.md` pour le plan de travail et les défauts connus.

## Code signing policy

RamScope sera signé avec un certificat fourni par SignPath Foundation (mise en place en cours).

- **Free code signing provided by SignPath.io, certificate by SignPath Foundation.**
- **Équipe et rôles** : projet solo maintenu par Roger Briguet (Bisse Studio). Le mainteneur occupe tous les rôles (commit, review, approbation des releases).
- **Confidentialité** : ce programme ne transfère aucune information vers d'autres systèmes réseau, sauf à la demande explicite de l'utilisateur.
- **Désinstallation** : aucune installation requise — supprimer le fichier `.exe` suffit. RamScope ne modifie le registre ni aucun fichier système.
- **Processus de release** : chaque release est construite automatiquement par GitHub Actions (workflow `.github/workflows/release.yml`), puis la demande de signature est approuvée manuellement dans le tableau de bord SignPath avant publication.
- **Vérifiabilité** : le workflow inclut les tests, clippy, fmt et la vérification des checksums. Les builds ne sont pas présentés comme reproductibles : aucune mesure ne l'étaye à ce jour (règle 4 du workflow).
