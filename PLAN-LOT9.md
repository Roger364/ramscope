# Plan d'améliorations — RamScope, lot 9

**Statut :** plan révisé le 2026-11-14 après évaluation de la revue Hermes-Beam. C7 et C6 sont implémentés localement avec leurs tests ; C6 est validé dans Windows Sandbox pour la propagation des options. C2 est implémenté, poussé et validé le 2026-11-15 par des actions réelles de service, de tâche planifiée et d'entrée de démarrage, chacune suivie d'une restauration dans Windows Sandbox ; le correctif de repli des tâches issu de ces tests est inclus. D9-2a est implémenté dans 9.11 et couvert par 68 tests unitaires au moment du lot, 75 après C3/C4/C5, puis 82 après C1/C14, 86 après C9–C13, puis 83 après C8 (quatre tests de chaînes remplacés par quatre tests Node et un test Rust) ; le test UAC sur un compte standard a été exécuté le 2026-11-15 dans un Windows Sandbox de test (acceptation et annulation réelles). C1, C14, C9–C13, C8 et C15 sont traités ; C3/C4/C5 sont corrigés. La licence MIT est appliquée, publiée et la CI GitHub est verte.
**Base :** revue WorkBuddy (audit `d08e412`) et revue Hermes-Beam (audit `3673120`) ; plan révisé sur `main` au commit `761343e`, après la fusion de la mise à jour Dependabot.
**Objectif :** rendre RamScope distribuable sans refonte, en corrigeant d'abord la frontière de confiance, puis les écarts entre le code et la documentation, puis la qualité des tests.

> Les constats viennent d'une lecture de code et de contrôles rejoués, pas d'exploits exécutés. Le plan distingue donc ce qui doit être corrigé de ce qui doit d'abord être décidé.

## 1. Évaluation de la revue Hermes-Beam

### 1.1 Ce que la revue apporte

La revue est crédible et utile. Elle audite `3673120`, cite des lignes précises, rejoue les contrôles et confirme les constats de la revue WorkBuddy sans reprendre son ancien diagnostic de remote. Elle apporte trois éléments qui manquaient ou qui étaient trop implicites dans le premier plan :

- **C1** : `Rule.source` et `Rule.reviewed` sont effectivement du code mort, et l'audit de `rules.rs` impose `reviewed == "non datée"`. Une vraie date de revue ferait donc échouer un test censé garantir la qualité des règles.
- **C8** : plusieurs tests de `server.rs` vérifient la présence de chaînes dans les sources (`include_str!`) plutôt que le comportement. Ils peuvent casser sur un refactor inoffensif et passer sur une régression qui conserve la chaîne.
- **C15** : aucun fichier de licence n'était présent. La décision a été prise le 2026-11-15 : MIT.

Elle précise aussi deux points déjà présents dans le plan : le jeton de session n'authentifie pas un processus local, et le champ `from` fourni par le client peut être absent, ce qui saute la revérification d'état.

### 1.2 Ce que je corrige dans son classement

Je ne reprends pas son ordre tel quel :

- **C1 est une dette de vérité des données, pas une élévation de privilèges.** Il reste un critère de sortie avant distribution, mais il ne doit pas retarder C7 et C2, qui sont des risques d'exécution élevée et de frontière de confiance.
- **C2 est un risque local, pas une faille distante.** Le plan doit rendre explicite le modèle de menace et éviter de promettre une authentification que le jeton actuel ne fournit pas.
- **C15 était une décision, pas un correctif de code.** La décision MIT a été prise par Ro le 2026-11-15 ; elle est appliquée et publiée.
- **C8 est un chantier de qualité de tests**, séparé des correctifs de sécurité. Il ne doit pas être noyé dans la CI déjà livrée.

### 1.3 Verdict par constat

| Constat | Verdict | Destination | Effet sur le plan |
|---|---|---|---|
| C1 — provenance morte et audit figé | Corrigé le 2026-11-15 | 9.6 | Source et date de revue exposées ; audit souple avec test daté |
| C2 — API locale élevée et état fourni par le client | Corrigé le 2026-11-15 | 9.3 | Précision sur `from`, `was_enabled`, le jeton et le canal local |
| C3 — origine optionnelle | Corrigé le 2026-11-15 | 9.4 | `Origin` obligatoire, tests nommés |
| C4 — corps non borné | Corrigé le 2026-11-15 | 9.4 | 1 Mio avant lecture puis lecture bornée |
| C5 — verrous non indépendants | Corrigé le 2026-11-15 | 9.5 | Garde moteur services/processus ; limites startup/tâche documentées |
| C6 — options CLI perdues à l'élévation | Corrigé le 2026-11-14 | 9.2 | Options conservées, test d'arguments |
| C7 — chemins système via l'environnement | Corrigé le 2026-11-14 | 9.1 | Chemins système absolus |
| C8 — tests sur le texte source | Corrigé le 2026-11-15 | 9.9 | Décisions pures testées par Node ; gardes de source limitées |
| C9 — `extract_exe` et Unicode | Confirmé, déjà connu | 9.8 | Inchangé |
| C10 — arrêt brutal avec threads actifs | Confirmé, déjà connu | 9.8 | Inchangé |
| C11 — plage de ports vide | Confirmé, déjà connu | 9.8 | Inchangé |
| C12 — faute « Irrréversible » | Confirmé, déjà connu | 9.8 | Inchangé |
| C13 — apostrophe non échappée | Confirmé, déjà connu | 9.8 | Inchangé |
| C14 — zéro mesuré confondu avec absence | Corrigé le 2026-11-15 | 9.7 | Affichage piloté par la nature ; zéro mesuré conservé |
| C15 — licence absente | Décidé le 2026-11-15 : MIT | 9.10 | Fichiers de licence et de communauté ajoutés |

Aucun constat n'est rejeté. La revue ne remplace toutefois pas un test d'action réelle : ses constats de sécurité restent des lectures de code avec préconditions explicitées.

## 2. Principes

- Ne jamais appliquer d'action réelle à la machine de tous les jours ; les tests d'écriture se font dans une VM ou un Windows Sandbox jetable.
- Un correctif sans test de réversion nommé n'est pas terminé.
- Ne pas annoncer une garantie de sécurité qui n'a pas été testée.
- Conserver les garde-fous existants : `Host` exact, jeton, origine stricte, validation serveur et verrous moteur.
- Un commit par sujet, avec la commande et le code retour consignés dans `PLAN.md`.
- Corriger d'abord les risques concrets et peu risqués ; les décisions d'architecture viennent après.
- Ne pas marquer un constat « corrigé » sans test qui échoue si la correction est annulée.

## 3. Décisions à prendre avant de coder

### D9-1 — Les processus locaux sont-ils dans le modèle de menace ?

**Décision (2026-11-15) : non fiables.** Un processus local du même compte peut lire la page, récupérer le jeton et forger un `Origin` : le jeton HTTP ne l'authentifie pas. RamScope ne doit donc pas prétendre protéger une session élevée contre un programme malveillant déjà exécuté sous ce compte.

Conséquences retenues pour la v1 :

- C2 est appliqué au niveau HTTP : jeton CSPRNG, état observé obligatoire, action limitée à celles proposées par le snapshot courant, restauration interne séparée.
- Le risque résiduel « processus local hostile » est **documenté et accepté pour la v1**, pas nié. Il ne doit pas être présenté comme un point corrigé.
- Le named pipe/DACL n'est **pas** un prérequis de la v1 : une DACL ouverte au navigateur du même utilisateur ne bloquerait pas forcément un processus du même compte. Cette refonte, ou un helper élevé avec consentement par lot, fera l'objet d'un lot séparé si le besoin est confirmé.
- La réduction de la durée de la session administrateur et l'élévation explicite restent recommandées avant une distribution publique (voir D9-2), mais ne sont pas mélangées à C2.

### D9-2 — Conserver l'élévation UAC par défaut ?

**Décision validée le 2026-11-15 (option A) et implémentée :** ne plus élever au lancement par défaut. Le lancement normal devient une session de lecture seule s'il n'est pas déjà élevé ; l'utilisateur demande explicitement l'élévation depuis l'interface après avoir consulté l'inventaire. Un nouveau `--admin` permet l'élévation au lancement pour ceux qui la veulent dès le départ, et `--no-admin` devient un mode strict qui interdit toute élévation ultérieure.

Cette étape réduit la durée d'exposition avant l'UAC, pas encore la durée de la session élevée. Un helper d'application one-shot, qui ne garderait les droits administrateur que quelques secondes, reste un lot séparé (D9-2b) : il touche au canal local et à l'authentification, et ne doit pas être mélangé à C2/D9-1. Le détail du lot est en **9.11** ; le test UAC sur compte standard a été exécuté le 2026-11-15.

### D9-3 — API HTTP ou canal local dédié ?

Le remplacement complet de HTTP par un named pipe avec DACL est structurel et coûteux. Le plan propose une étape minimale d'abord, puis une décision séparée sur le canal. Ne pas lancer cette refonte sans avoir mesuré le besoin réel.

### D9-4 — Portée du lot

Le lot 9 ne doit pas contenir de refonte graphique, de nouvelle base de règles, de signature de code ni de télémétrie.

### D9-5 — Licence et distribution

**Décision (2026-11-15) :** RamScope est publié sous MIT. Bisse Studio conserve ses autres applications et prestations hors du périmètre de cette licence. Le dépôt peut devenir public et le binaire être distribué avec les notices tierces ; le push et la CI restent à finaliser.

### D9-6 — Portée des tests comportementaux

C8 ne signifie pas automatiser tout le navigateur. Le plan privilégie des fonctions pures testables (sélection, formulation, calcul de gains) et ne réserve les tests DOM ou navigateur qu'aux parcours qui ne peuvent pas être testés autrement.

## 4. Vue d'ensemble

| Ordre | Chantier | Constats | Effort indicatif |
|---:|---|---|---:|
| 1 | Chemins système et données non influençables | C7 | 0,5 j |
| 2 | Options CLI conservées à l'élévation | C6 | 0,25 j |
| 3 | Frontière de confiance de l'API élevée | C2 | 1,5–2 j |
| 4 | Origine obligatoire et corps borné | C3, C4 | 0,5 j |
| 5 | Verrous réellement indépendants | C5 | 0,5 j |
| 6 | Provenance réelle des règles | C1 | 0,5 j |
| 7 | Zéro mesuré respecté | C14 | 0,25 j |
| 8 | Robustesse et finitions | C9–C13 | 0,5 j |
| 9 | Tests comportementaux | C8 | 1 j |
| 10 | Industrialisation et licence | C15 + état CI | 0,1 j |
| 11 | Réduction de l'élévation UAC | D9-2a | 0,5–0,75 j |

Total indicatif : **7 à 8 jours**, hors tests réels en VM et hors décision sur le canal local et le helper one-shot.

## 5. Chantiers détaillés

### 9.1 — Chemins système et données : C7

**État :** implémenté le 2026-11-14, tests ajoutés.

**Objectif.** Empêcher un environnement forgé de faire exécuter un binaire élevé ou de rediriger les points de restauration.

**Périmètre.** `win.rs` : `system32_path`, `data_dir`, éventuellement `elevate` ; `server.rs` pour `explorer.exe` ; tests de `win.rs`.

**Changements.**
- Résoudre `System32` avec `GetSystemDirectoryW`, jamais avec `SystemRoot` ou `windir`.
- Résoudre `%LOCALAPPDATA%` avec `SHGetKnownFolderPath(FOLDERID_LocalAppData)`.
- Conserver un helper unique et testable pour les chemins d'outils (`schtasks.exe`, `explorer.exe`, `Tasks`).
- Supprimer le repli silencieux vers `C:\Windows` et `"."` sur Windows : un échec de résolution doit produire une erreur explicite ou une note de collecte, pas un chemin choisi par l'environnement.
- Vérifier que le processus élevé ne dépend d'aucune variable d'environnement héritée pour localiser un outil système.

**Tests.**
- Test nommé : `systemroot_forge_ne_change_pas_le_chemin_systeme`.
- Test nommé : `localappdata_forge_ne_change_pas_le_dossier_donnees`.
- Test du helper pour `schtasks.exe` et `explorer.exe`.
- Vérification FFI ou manuelle : le chemin retourné est identique avec un `SystemRoot` normal et forgé.

**Critère de sortie.**
- `grep` : plus de `std::env::var("SystemRoot")` ni de `std::env::var("LOCALAPPDATA")` dans les chemins d'autorité.
- Les nouveaux tests passent avec un environnement forgé.
- Aucun changement de comportement sur une collecte réelle en lecture seule.

**Réversion.** Le test avec `SystemRoot` forgé échoue si le code revient à la variable d'environnement.

### 9.2 — Élévation : C6

**État :** implémenté le 2026-11-14, tests ajoutés. Validation Sandbox effectuée le 2026-11-15 ; binaire release autonome vérifié.

**Objectif.** Les options choisies ne doivent pas disparaître au moment où RamScope demande l'élévation.

**Périmètre.** `main.rs` : construction des arguments ; `win.rs` : `elevate` ; tests d'arguments.

**Changements.**
- Transmettre à l'instance élevée la ligne de commande complète, pas seulement `--elevated`.
- Éviter la duplication de `--elevated` et toute boucle d'élévation.
- Échapper correctement les arguments contenant des espaces.
- Tester `--port`, `--host localhost`, `--no-browser` et la combinaison avec `--elevated`.
- Consigner le port et l'hôte effectifs dans un log de démarrage ou les exposer dans `/api/health` pour rendre le test vérifiable.

**Tests.**
- Test unitaire de la fonction de reconstruction des arguments : la liste est préservée et `--elevated` n'apparaît qu'une fois.
- Test avec un faux `ShellExecuteW` : les paramètres reçus contiennent le port et l'hôte.
- Test réel en VM : `--port 8917 --host localhost --no-browser` avec élévation, vérifier l'écoute sur `localhost:8917` et l'absence d'ouverture de fenêtre.

**Validation Windows Sandbox (2026-11-15, binaire CRT statique).**
- Commande initiale : `--port 8917 --host 127.0.0.1 --no-browser --no-admin --demo`.
- Processus élevé observé, PID `9060` :
  `"C:\Users\WDAGUtilityAccount\AppData\Local\Temp\ramscope-c6.exe" --port 8917 --host 127.0.0.1 --no-browser --demo --elevated`.
- `POST /api/elevate` : HTTP `200` ; ancien PID `4792` ; processus élevé unique.
- Contrôles : port conservé, hôte `127.0.0.1` conservé, `--no-browser` conservé, `--no-admin` retiré, `--elevated` unique : **tous vrais**.
- Le serveur a répondu sur `127.0.0.1:8917` ; le mode `--demo` évite toute action sur le système invité.
- Le test a été exécuté avec le binaire release de `920 576` octets et **sans `vcruntime140.dll`** copiée dans l'invité ; le script échoue désormais si cette DLL est présente.

**Limites du test.** Le compte du Sandbox est administrateur : ce test valide la reconstruction et la transmission des arguments par `ShellExecuteW`, pas le consentement UAC d'un compte standard. Le test n'a pas été rejoué avec `--host localhost` ; l'hôte effectif consigné est `127.0.0.1`. Le binaire release importait auparavant `VCRUNTIME140.dll` ; `ramscope/.cargo/config.toml` force désormais la CRT statique (`-C target-feature=+crt-static`), et les imports `VCRUNTIME140.dll`, `VCRUNTIME140_1.dll`, `MSVCP140.dll` et les DLL UCRT dynamiques vérifiées sont absents du binaire reconstruit.

**Critère de sortie.** Le chemin par défaut n'ignore plus aucune option documentée ; le binaire release ne dépend plus de la CRT VC++ dynamique.

**Réversion.** Le test d'arguments échoue si la reconstruction ne conserve pas le port ; retirer `ramscope/.cargo/config.toml` réintroduit l'import `VCRUNTIME140.dll`.

### 9.3 — Frontière de confiance de l'API élevée : C2

**Objectif.** Le serveur ne doit pas croire l'état fourni par le client et ne doit accepter que les actions réellement proposées par l'analyse courante. Un processus local hostile reste hors du modèle de menace de la v1 : le jeton protège la session navigateur, pas l'identité d'un programme local (D9-1).

**Périmètre.** `server.rs`, `plan.rs`, `win.rs`, `util.rs`, `model.rs`, `SAFETY.md`, `README.md`, tests HTTP et tests unitaires.

**Changements minimaux, dans cet ordre.**
1. **Jeton** : générer le jeton avec `BCryptGenRandom` (Windows) plutôt qu'avec `new_id()` horloge/PID/xorshift.
2. **Validation d'état côté serveur** : pour `Service`, `Startup` et `Task`, exiger `from`/`was_enabled` sur toute action ordinaire, puis les comparer au snapshot serveur fraîchement collecté. Une action sans état observé est refusée, sauf les actions internes de restauration qui ne passent pas par cette route. Une action doit aussi être exactement celle proposée par le snapshot (`advice.action`), et non une cible arbitraire du même type.
3. **Restauration** : distinguer explicitement le chemin interne de restauration du chemin public `/api/apply` ; ne pas accepter `from: None` comme contournement sur une action utilisateur. Le démarrage d'un service (`StartService`) est réservé à ce chemin interne.
4. **Documentation du modèle** : acter D9-1 dans `SAFETY.md` et le README : processus locaux non fiables, jeton limité à la session navigateur, risque résiduel accepté pour la v1. Ne pas annoncer un contrôle d'intégrité du client ni un canal authentifié qui n'existe pas encore.
5. **Décision structurelle séparée** : le named pipe/DACL ou un helper élevé avec consentement par lot reste un lot dédié. Ne pas mélanger cette refonte avec les correctifs ci-dessus.

**Tests.**
- `refuse_un_from_null_sur_une_action_utilisateur` : `from: null` → refus.
- `refuse_un_etat_divergent_du_snapshot` : état différent de l'état serveur → refus.
- `accepte_les_actions_et_l_etat_du_snapshot` : les trois types d'action proposés passent avec l'état courant.
- `refuse_une_action_non_proposee_meme_avec_le_bon_etat` : cible réelle mais action hors snapshot → refus.
- `refuse_le_demarrage_d_un_service_sur_la_route_utilisateur` : `StartService` réservé à la restauration.
- `accepte_un_arret_immediat_propose_et_refuse_un_arret_arbitraire` : l'arrêt immédiat suit la même règle de proposition.
- `accepte_une_terminaison_proposee_par_le_snapshot` : la terminaison doit correspondre à l'identité et à l'action proposées.
- `une_restauration_interne_rejoue_un_undo_sans_etat_observe` : les inverses `from: None` restent applicables par le moteur.
- `lit_l_etat_natif_d_une_tache_pour_le_rollback` : le parseur d'état accepte `Enabled=false`, `true` et l'absence de balise (activée par défaut).
- `le_jeton_de_session_est_long_hexadecimal_et_imprevisible` : 32 octets CSPRNG, 64 caractères, deux jetons différents.
- Les tests existants de `Host`, d'origine et de jeton restent verts ; un jeton absent ou invalide est toujours refusé.

**Critère de sortie.**
- Le scénario « processus local non élevé lit le jeton puis applique une action » est explicitement documenté comme hors modèle de menace de la v1 (D9-1), avec le risque résiduel nommé.
- Aucune action de service/startup/tâche ne passe sans état observé, et aucune action hors snapshot n'est acceptée par `/api/apply`.
- `SAFETY.md` et le README ne prétendent plus que le jeton authentifie l'appelant.

**Réversion.** `refuse_un_from_null_sur_une_action_utilisateur` et `refuse_une_action_non_proposee_meme_avec_le_bon_etat` échouent si la validation d'état ou la limitation au snapshot est retirée.

**Vérification HTTP en démonstration (2026-11-15).** Le binaire release a été lancé avec `--demo --no-browser --port 0` (port effectif 63242), puis un client Python a lu le jeton dans la page et envoyé les requêtes avec `Host`, `Origin` et `X-RamScope-Token` corrects :

- `from: null` → `409`, « État constaté du service manquant » ;
- `from` divergent → `409`, « L'état du service a changé depuis l'analyse » ;
- état courant conservé mais autre `start` → `409`, « Action non proposée par l'analyse courante » ;
- action exacte du snapshot (service `dmwappushservice`, Manuel → Désactivé) → `200`, point de restauration créé ;
- restauration interne de ce point (`from: None`) → `200`, service revenu à Manuel.

Les tests unitaires couvrent la fonction pure et le moteur ; ce test HTTP couvre le branchement réel de la route. Depuis ce test, deux tests unitaires supplémentaires couvrent l'arrêt immédiat et la terminaison proposés ; le test de la tâche planifiée en Sandbox a ensuite motivé un test de lecture d'état natif (57 tests au total).

**Vérification interface (2026-11-15).** L'interface a été ouverte dans Chromium sur le binaire release en mode démonstration. Le service `dmwappushservice` (Manuel → Désactivé) a été sélectionné, confirmé et appliqué : « Application : 1/1 réussi », point de restauration `dfb8a171e56522c1` créé. Le même point a ensuite été restauré depuis l'onglet Restauration : « Restauration : 1/1 réussi », service revenu à Manuel. Le journal navigateur ne contient aucune erreur console et les appels `/api/apply` et `/api/restore` ont répondu `200`. Aucune action n'a été appliquée sur une machine réelle lors de ce test d'interface.

**Vérification réelle en Windows Sandbox (2026-11-15).** Le binaire release a été exécuté **sans `--demo`**, avec `--elevated`, dans le Sandbox C6 déjà ouvert (le second Sandbox a été refusé : une seule instance à la fois).

- **Service.** `dmwappushservice` a été choisi parmi les actions proposées : `manual` avant, `disabled` après `POST /api/apply` (`200`), confirmé par `Win32_Service.StartMode=Disabled`. Le point `526d88c293ac7808` a ensuite été restauré par `POST /api/restore` (`200`) : snapshot et `Win32_Service.StartMode` revenus à `manual`/`Manual`.
- **Tâche planifiée.** Le premier essai a révélé un défaut réel : la tâche `\Microsoft\Windows\Application Experience\StartupAppTask` était proposée depuis le repli `schtasks`, mais `current_task_enabled` ne lisait que `System32\Tasks`, donc l'application répondait « tâche introuvable ». La lecture d'état utilise désormais le même repli `schtasks /query /tn <chemin> /xml`. Après reconstruction, la tâche est passée de `Ready` à `Disabled` (`POST /api/apply`, `200`), puis de `Disabled` à `Ready` après restauration (`POST /api/restore`, `200`, point `145b6b0780efdd93`). L'état natif a été vérifié avec `Get-ScheduledTask` avant, après application et après restauration.
- **Entrée de démarrage.** Aucune entrée proposée n'était présente dans l'image du Sandbox ; le test a créé une entrée HKCU `RamScopeC2StartupTest` (commande `spotify.exe`) pour exercer le mécanisme réel `StartupApproved`. L'état natif avant était activé avec une valeur d'approbation absente ; après `POST /api/apply` (`200`), la valeur est passée à `3` (désactivée), puis à `2` (activée) après `POST /api/restore` (`200`, point `ae9d473adf0a7a32`). L'entrée de test et sa valeur d'approbation ont été supprimées après vérification ; le Run original n'a pas été modifié.

Le test service vérifie aussi l'absence de copie de `vcruntime140.dll`. Le Sandbox est resté ouvert. Limites : un seul service, une seule tâche et une entrée de démarrage synthétique, compte Sandbox administrateur, pas de test du consentement UAC standard ni d'action processus réelle.

### 9.4 — Origine et taille des requêtes : C3, C4

**Objectif.** Réduire la surface de mutation et empêcher un déni de service local par corps géant.

**Périmètre.** `server.rs`, `README.md`, `SAFETY.md`, tests HTTP.

**Changements.**
- Exiger `Origin` sur toute mutation (`POST`), avec la même comparaison exacte que pour l'API.
- Ajouter `Sec-Fetch-Site: same-origin` en défense en profondeur si le client navigateur le fournit ; ne pas en faire la seule authentification.
- Rejeter `Content-Length` supérieur à une limite documentée (par exemple 1 Mio) avant lecture.
- Borner aussi la lecture réelle (`take`) pour couvrir les corps sans `Content-Length` ou en chunked.
- Choisir et documenter le code retour : `400` ou `413`, mais rester cohérent et testé.
- Mettre à jour les exemples `curl` de `PLAN.md` et le README pour inclure `Origin`.

**Tests.**
- `POST /api/apply` sans `Origin` → `403`.
- `POST /api/apply` avec `Origin` inconnue → `403`.
- `POST` avec `Content-Length` > 1 Mio → `413` ou `400` sans lecture complète.
- Test de corps chunked surdimensionné.
- Les tests existants d'application et de restauration envoient l'origine correcte.

**Critère de sortie.** Aucun `POST` sans origine valide ne peut atteindre une mutation ; aucun corps arbitrairement grand n'est lu.

**Réversion.** Le test sans `Origin` échoue si le contrôle redevient optionnel.

**État (2026-11-15).** Implémenté dans `server.rs` : `Origin` obligatoire, `Sec-Fetch-Site` cross-site refusé, limite `MAX_BODY_BYTES = 1 Mio`, contrôle de `Content-Length` avant lecture et `take` pour les corps sans longueur/chunked. Tests nommés : `le_serveur_refuse_origine_absente_et_corps_geant_avant_mutation` (vrai `handle`, serveur local), `une_mutation_sans_origine_est_refusee`, `une_origine_cross_site_est_refusee`, `un_corps_trop_volumineux_est_refuse_avant_lecture`. Vérification HTTP réelle en démonstration : sans origine 403, origine inconnue 403, `Content-Length` 1 048 577 → 413, corps chunked de même taille → 413, origine correcte + `{}` → 400 « aucune action demandée ». Détail dans `PLAN.md` section 13.

### 9.5 — Verrous réellement indépendants : C5

**Objectif.** Soit les deux niveaux de refus sont réellement indépendants, soit la documentation cesse de le prétendre.

**Périmètre.** `win.rs`, `server.rs`, `SAFETY.md`, `PLAN.md`, tests.

**Changements.**
- Ajouter dans `apply_service`, `apply_startup` et `apply_task` une vérification directe des listes protégées, indépendante de `plan::validate`.
- Vérifier de nouveau l'état courant juste avant l'écriture, même si le serveur a déjà validé.
- Si cette duplication n'est pas souhaitée, corriger la formulation « deux verrous » en « validation aux deux points d'entrée », avec un renvoi à `plan::validate`.
- Documenter précisément ce que chaque couche garantit.

**Tests.**
- Test qui appelle directement `win::apply` avec un service protégé, sans passer par `server`, et attend un refus.
- Test qui appelle directement `win::apply` avec un état `from` devenu faux et attend un refus.
- Test de non-régression pour l'application normale d'un service non protégé.

**Critère de sortie.** Un défaut dans `plan::validate` ne suffit plus à contourner tous les refus sur les services, ou la documentation ne parle plus de verrous indépendants.

**Réversion.** Le test d'appel direct à `win::apply` échoue si le contrôle moteur est retiré.

**État (2026-11-15).** `engine_guard` est appelé au début de `win::apply` pour les services et processus protégés, avant d'utiliser la décision de `plan::validate` ; `control_service` vérifie aussi le démarrage. Tests directs sans serveur : `le_moteur_refuse_un_service_protege_sans_plan`, `le_moteur_refuse_de_demarrer_un_service_protege_sans_plan`, `le_moteur_refuse_un_etat_divergent_sans_plan`. Les entrées de démarrage et les tâches n'ont pas de liste globale distincte : la relecture d'état avant écriture reste leur garde moteur, et `SAFETY.md` précise que « deux verrous » ne vaut que pour les services et processus. Le test de non-régression vérifie seulement que `engine_guard` laisse passer un service ordinaire ; il n'exécute pas de modification réelle.

### 9.6 — Provenance réelle des règles : C1

**Objectif.** La provenance affichée doit être réelle, et l'audit ne doit plus figer la dette « non datée » en invariant impossible à faire évoluer.

**Périmètre.** `rules.rs`, `model.rs`, `report.rs`, `web/app.js`, tests.

**Changements.**
- Exposer `source` et `reviewed` dans `Advice` et dans le rapport exporté.
- Afficher ces champs dans le panneau de détails et dans l'export.
- Remplacer l'assertion `reviewed == REVIEWED` par : « `non datée` ou date ISO valide ».
- Ajouter un test avec une règle datée pour prouver que l'audit accepte une vraie revue.
- Ne pas ajouter une fausse date de revue aux 601 règles : la provenance de famille doit rester honnête.
- Supprimer `#[allow(dead_code)]` sur les champs lorsqu'ils sont effectivement consommés.

**Tests.**
- Une règle avec `reviewed = "non datée"` passe l'audit.
- Une règle avec une date ISO valide passe l'audit.
- Une date invalide échoue avec un message explicite.
- L'API et le rapport contiennent la source et la date de revue.

**Critère de sortie.** Le rapport et l'interface montrent la source et la date de revue ; l'audit accepte les deux états sans imposer la dette.

**État (2026-11-15).** `Advice.source` et `Advice.reviewed` sont sérialisés dans l'API, repris dans une colonne **Provenance** du rapport HTML et affichés dans le panneau de détails. Les quatre moteurs de conseil copient la provenance de la règle trouvée ; les conseils de repli sans règle nommée laissent les deux champs vides, et l'interface ne les invente pas. L'audit accepte désormais `non datée` ou une date `AAAA-MM-JJ` valide, refuse les dates calendaires impossibles avec un message explicite, et un test avec une règle réellement datée prouve que l'audit ne fige plus la dette. Aucune date n'a été ajoutée aux 601 règles.

**Tests.** `l_audit_accepte_une_regle_datee_et_refuse_une_date_invalide`, `le_snapshot_expose_la_provenance_des_regles`, `le_rapport_expose_la_provenance_des_regles` ; `conseille_la_telemetrie` vérifie aussi la provenance du conseil. Le rendu du panneau a été vérifié par la présence du champ dans `app.js`, pas rouvert dans un navigateur pour ce lot.

**Réversion.** Le test avec une règle datée échoue si l'assertion rigide est rétablie.

### 9.7 — Zéro mesuré respecté : C14

**Objectif.** Un zéro mesuré ne doit jamais être présenté comme une absence de mesure.

**Périmètre.** `model.rs`, `report.rs`, `web/app.js`, tests.

**Changements.**
- Faire dépendre l'affichage d'un gain de sa `nature` avant sa valeur.
- `Gain::measured(0)` doit rester `Measured` dans `merge`, le rapport et l'interface.
- Ne pas transformer `estimated(0)` en mesure : il reste indisponible, comme aujourd'hui.
- Tester `Gain::measured(0)` dans le rapport et le rendu JS.

**Critère de sortie.** Un gain mesuré nul affiche « 0 mesuré », jamais « non chiffrable ».

**État (2026-11-15).** Le rapport décide d'abord de la nature : un `Gain::measured(0)` affiche « 0 Mo (mesuré) », tandis qu'un `Gain::estimated(0)` reste indisponible et affiche « non chiffrable ». L'interface suit la même règle avec `gainKnown`, et l'agrégation conserve une nature mesurée quand tous les gains additionnés le sont. `Gain::merge` initialise un total vide avec un zéro mesuré sans le confondre avec un gain indisponible.

**Tests.** `un_zero_mesure_reste_mesure_dans_le_rapport`, `un_zero_mesure_survit_au_merge`, `un_zero_estime_reste_indisponible` et, depuis C8, le test Node `un zéro mesuré reste une mesure, un zéro estimé est ignoré`. Ce dernier teste la décision de rendu sans navigateur ; le rendu visuel n'a pas été rouvert.

**Réversion.** Le test `measured(0)` échoue si le rendu redevient basé uniquement sur `bytes == 0`.

### 9.8 — Robustesse et finitions : C9–C13

**Objectif.** Supprimer des petits défauts qui peuvent produire un panic ou une expérience incohérente.

**Périmètre.** `util.rs`, `main.rs`, `server.rs`, `web/app.js`, `report.rs`.

**Changements.**
- C9 : `extract_exe` ne doit pas utiliser un index calculé sur `to_lowercase()` pour découper la chaîne originale ; utiliser `char_indices` ou une recherche ASCII insensible à la casse.
- C10 : remplacer `std::process::exit(0)` par un arrêt coopératif : signaler l'arrêt, attendre les threads, puis quitter.
- C11 : gérer proprement la plage de ports proche de 65 535, avec un message explicite si aucun port n'est disponible.
- C12 : corriger « Irrréversible » en « Irréversible ».
- C13 : harmoniser `esc()` et l'échappement JS pour l'apostrophe.
- Ajouter un test de ligne de commande Unicode dans `extract_exe` et un test de dernier port.

**Critère de sortie.**
- Aucun panic possible sur une ligne de commande Unicode dans `extract_exe`.
- Arrêt sans écriture de rollback tronquée.
- Textes utilisateur sans faute visible.

**État (2026-11-15).** `extract_exe` recherche `.exe` sur une copie ASCII minuscule, dont les indices restent ceux de la chaîne d'origine ; le test avec `İ` échoue avec l'ancien `to_lowercase()` (découpe décalée). La boucle de ports utilise douze candidats plafonnés à 65 535 et affiche une plage explicite quand tous sont occupés. L'arrêt passe par `spawn_workers`/`stop_workers` : le drapeau est posé, les quatre travailleurs sont réveillés avec `unblock`, puis rejoints avant de rendre la main ; `process::exit` a disparu du serveur. Les deux occurrences « Irrréversible » sont corrigées, et l'échappement HTML du rapport traite désormais l'apostrophe comme l'interface.

**Tests.** `extract_exe_supporte_une_casse_unicode_sans_paniquer`, `le_dernier_port_est_essaye_et_la_plage_ne_deborde_pas`, `l_arret_cooperatif_attend_les_quatre_threads`, `echappe_l_apostrophe_comme_l_interface` ; le test d'interface vérifie aussi l'absence de « Irrréversible » et la présence des échappements `&quot;`/`&#39;`. Limite : le test d'arrêt utilise quatre vrais travailleurs de test, mais ne relance pas la commande `serve` complète ; aucun socket n'est ouvert sur le port 65 535.

### 9.9 — Tests comportementaux : C8

**Objectif.** Réduire les tests qui vérifient des chaînes de source et augmenter la couverture des décisions réelles.

**Périmètre.** `server.rs`, `web/app.js`, éventuellement extraction de fonctions pures.

**Changements.**
- Recenser les tests `include_str!` de `server.rs` et conserver uniquement ceux qui protègent une contrainte de sécurité ou de formulation impossible à tester autrement.
- Extraire progressivement les décisions pures : `batchable`, calcul des gains, formulation utilisateur, fermeture de modale.
- Ajouter des tests unitaires sur ces fonctions pures.
- Pour les parcours DOM qui ne peuvent pas être testés sans navigateur, documenter la limite au lieu de prétendre les couvrir avec `contains`.
- Ne pas introduire une infra E2E lourde dans ce lot sans décision explicite.

**Critère de sortie.** Les tests de comportement nommés couvrent les décisions critiques ; les tests de chaînes restants sont justifiés et limités.

**État (2026-11-15).** Les décisions pures de l'interface sont extraites dans `web/core.js` : `selectable`, `batchable`, `gainBytes`, `gainNature`, `gainKnown`, `addGain`, `aggregateGains` et `modalShouldClose`. `app.js` consomme ce module, et `core.test.mjs` exécute quatre tests Node nommés sur les gains, les lots, les hôtes et la modale. Les quatre tests `include_str!` remplacés (`un_zero_mesure...`, `l_interface_ne_convertit...`, `une_terminaison...`, `le_fond_de_modale...`) ont été supprimés. Restent des tests de source justifiés : CSP, jeton, porte d'élévation et formulations bannies ; ils protègent des contraintes de sécurité ou de texte qui ne peuvent pas être exercées sans navigateur. `cargo test` appelle Node : Node.js est requis pour la suite de tests.

**Limite.** Le câblage DOM des événements, le rendu visuel et les parcours de clic ne sont pas couverts par ces tests purs ; ils restent à vérifier dans un navigateur.

### 9.10 — Industrialisation et licence : C15

**Objectif.** Terminer ce qui est déjà livré et formaliser la décision de licence.

**Décision (2026-11-15).** RamScope est publié sous **MIT**. Bisse Studio conserve ses autres applications et prestations hors du périmètre de cette licence ; la licence ne concède pas la marque. Le choix privilégie l'adoption, la contribution et la visibilité, conformément au positionnement du studio.

**Déjà livré.**
- CI Windows : formatage, 83 tests Rust et 4 tests Node, clippy, build release, JS, sommes SHA-256, refus des fichiers non suivis.
- Dependabot pour Cargo et GitHub Actions.
- Remote `Roger364/q-memory-manager`.

**Application locale.**
- `LICENSE` MIT à la racine, copyright Bisse Studio.
- `license = "MIT"` et `repository` dans `ramscope/Cargo.toml`.
- Badge et section **Open source** dans `README.md`.
- `THIRD_PARTY_NOTICES.md` : textes intégraux générés par `cargo-about` pour les 17 crates permissives ; `about.toml` et `about.hbs` permettent de le régénérer.
- `SECURITY.md` : canal privé, périmètre, versions et réduction du risque.
- `CONTRIBUTING.md` : DCO, licence MIT des contributions, commandes de test et règles de sécurité.

**État final (2026-11-15).** Push effectué (`e178d68`), dépôt public sur `https://github.com/Roger364/q-memory-manager`, licences détectées par GitHub (`MIT`). Les revues Hermes ont été supprimées ; les preuves de test ont été archivées dans `sandbox-test-2026-11-15.zip`, qui reste local et ignoré par Git, et ne sont pas publiées. La CI GitHub `43681932387` a réussi toutes ses étapes.

**Critère de sortie.** La décision de licence est consignée et appliquée ; la CI reste verte ; le dépôt distant est synchronisé après accord de Ro.

### 9.11 — Réduction de l'élévation UAC : D9-2a

**État :** implémenté le 2026-11-15, tests unitaires au vert (68 tests au moment du lot, 75 après C3/C4/C5). Le consentement UAC sur un compte standard a été testé le 2026-11-15 : acceptation et annulation réelles, avec `admin:true` après acceptation et `admin:false` / `can_elevate:true` après annulation. La route `POST /api/elevate` du bouton a aussi été testée ; elle a révélé puis fait corriger le conflit de port de la relance élevée. Le clic DOM et le rechargement de la page ont été validés avec Chrome for Testing portable : Edge n'a pas pu être installé dans le Sandbox (MSI 1603/1722).

**Décision validée le 2026-11-15 (option A).** Le lancement sans option devient une session de **lecture seule s'il n'est pas déjà élevé** ; l'élévation n'est demandée qu'après une action explicite de l'utilisateur.

**Alternatives examinées.**
- **A. Lecture seule par défaut + `--admin` (recommandée).** Réduit l'exposition initiale, garde un parcours simple pour appliquer après revue, et laisse un mode strict pour l'audit.
- **B. Statu quo.** Élévation automatique au lancement, avec `--no-admin` strict. Changement minimal, mais ne réduit pas la durée d'exposition et ne répond pas à D9-2.
- **C. Lecture seule sans `--admin`.** Plus simple, mais casse les scripts et les utilisateurs qui veulent une session élevée dès le départ ; rejetée pour cette raison.

| Lancement | UAC au démarrage | Élévation depuis l'interface | Usage |
|---|---|---|---|
| `ramscope.exe` | non | oui | parcours normal : inventaire, puis « Passer en administrateur » |
| `ramscope.exe --admin` | oui, sauf si déjà élevé | déjà élevé | utilisateur qui veut appliquer tout de suite |
| `ramscope.exe --no-admin` | non | non | lecture stricte, audit, aucune relance élevée |
| `ramscope.exe --demo` | non | non | démonstration |
| `--elevated` | interne | déjà élevé | relance de l'instance élevée, ne jamais l'exposer comme option utilisateur |

**Constat avant D9-2a.**
- `main.rs` élevait automatiquement dès le lancement si `--demo`, `--no-admin` et `--elevated` étaient absents et que le processus n'était pas déjà administrateur.
- `--no-admin` empêchait seulement l'élévation initiale : l'interface affichait encore « Passer en administrateur » et `/api/elevate` pouvait relancer une instance élevée.
- Une fois élevée, l'instance reste administrateur jusqu'à sa fermeture ; C2 ne change pas cette durée.
- `--elevated` et `--no-admin` sont retirés des arguments de la relance, puis `--elevated` est ajouté.

**Changements implémentés.**
1. **Mode d'élévation explicite et testable.** Extraire le calcul dans une fonction pure (par exemple `elevation_mode(args, deja_admin, demo)`) qui distingue : `ReadOnly`, `AdminAtStart`, `StrictReadOnly`, `Demo`, `AlreadyElevated`. Refuser la combinaison `--admin --no-admin` avec un message clair ; `--demo` reste sans UAC.
2. **Défaut en lecture seule.** Ne plus appeler `win::elevate()` dans le chemin par défaut. Seul `--admin` demande l'UAC au lancement. Si l'UAC est refusé, continuer en lecture seule et laisser l'interface expliquer l'état ; ne pas boucler.
3. **Mode strict.** `--no-admin` interdit `/api/elevate` (réponse `403` explicite), masque le bouton d'élévation et reste en lecture seule même après une tentative d'appel direct. Le mode est une propriété du serveur, pas du client.
4. **Propagation au serveur et à l'interface.** Ajouter un booléen `can_elevate` à la configuration du serveur et à `meta` (ou à `/api/health`), puis l'utiliser dans `app.js` pour masquer le bouton et afficher un message adapté. `/api/apply` et `/api/restore` conservent leurs garde-fous actuels.
5. **Arguments de relance.** `elevated_args` retire `--admin` et `--no-admin`, conserve le port, l'hôte, `--no-browser`, `--demo`, et n'ajoute qu'un seul `--elevated`.
6. **Documentation.** Mettre à jour `--help`, `README.md` et `SAFETY.md` après implémentation : le lancement normal ne demande plus UAC, `--no-admin` est strict, et la session élevée reste longue jusqu'à D9-2b.

**Tests ajoutés.**
- `le_defaut_ne_demande_pas_uac` : lancement sans option → `ReadOnly`.
- `admin_demande_l_uac_une_seule_fois` : mode `AdminAtStart`, pas de boucle.
- `no_admin_est_strict` : mode `StrictReadOnly`, `can_elevate=false`, `/api/elevate` → `403`.
- `admin_et_no_admin_sont_rejetes` : erreur de démarrage explicite.
- `elevated_args_retire_admin_et_no_admin` : port/hôte/`--no-browser` conservés, `--elevated` unique.
- `le_serveur_strict_ne_propose_pas_elevate` : contrôle serveur/JS, pas seulement le texte du README.
- `le_snapshot_annonce_le_droit_d_elevation` : propagation de `can_elevate` au snapshot.
- `une_relance_elevee_reessaie_le_port_avant_de_changer` : le retry de port est limité à `--elevated`, et seulement tant que le quota de tentatives n'est pas épuisé.
- `le_snapshot_strict_ne_propose_pas_de_relance_admin` : retire la note générique de relance admin en `--no-admin`, la conserve en lecture seule autorisant l'élévation et laisse les autres notes intactes.
- Test dans une VM avec un **compte standard** : défaut sans UAC ; clic « Passer en administrateur » → UAC ; annulation → l'ancien serveur lecture seule reste vivant ; acceptation → une seule instance élevée ; `--admin` → UAC au démarrage ; `--no-admin` → aucune proposition d'élévation. **Exécuté le 2026-11-15 dans un Windows Sandbox de test : les scénarios `--admin` d'acceptation et d'annulation, `POST /api/elevate` avec `admin:false` → UAC acceptée → `admin:true` sur le même port, le clic DOM et le rechargement avec Chrome for Testing portable sont validés. Le compte standard a aussi validé `--no-admin` (snapshot `can_elevate=false`, `POST /api/elevate` 403, aucun `consent.exe`) et `--host localhost` (page et API via `localhost`).**

**Résultats du test standard (2026-11-15).** Le Sandbox est livré avec UAC désactivé ; `EnableLUA=1` a été appliqué puis le Sandbox a été redémarré. Un compte `RamScopeStd` (Utilisateurs) et un compte administrateur distinct ont été créés. Les deux scénarios ont produit une vraie fenêtre `consent.exe` (`Credential Dialog Xaml Host`) avec les boutons Oui/Non :

| Scénario | Action UAC | Résultat RamScope |
|---|---|---|
| Acceptation, `--admin --no-browser --port 8899` | clic Oui | instance élevée : `admin:true`, `can_elevate:false`, `ok:true` ; processus standard sorti ; `/api/quit` 200 |
| Annulation, `--admin --no-browser --port 8898` | clic Non | instance standard toujours vivante : `admin:false`, `can_elevate:true`, `ok:true` ; `/api/quit` 200 |
| Route du bouton, `POST /api/elevate` sur défaut 8898 | clic Oui | avant `admin:false`, `can_elevate:true` ; après `admin:true`, `can_elevate:false` sur 8898 ; `original_exited:true` ; `/api/quit` 200 |
| Clic DOM, Chrome for Testing, défaut 8898 | clic Oui | `button_text=Passer en administrateur`, `click_method=cdp_dom` ; après `admin:true`, `can_elevate:false`, `original_exited:true` ; rechargement : badge `Administrateur`, bouton absent ; `/api/quit` 200 |
| `--no-admin`, compte standard, 8898 | aucune UAC | `admin:false`, `can_elevate:false` ; snapshot `can_elevate=false` ; `POST /api/elevate` 403 ; `consent_count=0` ; `/api/quit` 200 |
| `--host localhost`, compte standard, 8899 | aucune UAC | page et API via `localhost` ; `admin:false`, `can_elevate:true` ; `--host localhost` conservé ; `/api/quit` 200 ; alias `127.0.0.1` non joignable |

Le bureau sécurisé a été désactivé uniquement dans ce Sandbox pour permettre l'automatisation du clic ; le chemin applicatif reste `ShellExecuteW("runas")` puis la relance `--elevated`. Les deux premiers scénarios utilisent `--admin` ; le bouton « Passer en administrateur » a été cliqué dans Chrome for Testing portable, et la page rechargée affiche `Administrateur` sans bouton d'élévation. Edge n'a pas pu être installé dans le Sandbox (MSI 1603/1722) ; le test DOM utilise donc Chromium, pas Edge. Les preuves sont conservées dans l'archive locale `sandbox-test-2026-11-15.zip`, non versionnée ; elle contient les relevés `d92` d'acceptation, d'annulation, d'endpoint, de bouton et de reste. Le Sandbox C6 a été fermé le 15 novembre 2026 après archivage ; le Sandbox de test d92 a été fermé après relevé.

**Observation `--no-admin`.** Le snapshot expose bien `can_elevate=false`, mais sa liste `notes` contenait encore « Mode lecture seule : lance RamScope en administrateur pour appliquer les changements » ; `app.js` ajoutait la note stricte, ce qui produisait un message contradictoire. **Corrigé** : `apply_elevation_policy` retire cette note en mode strict ; le test `le_snapshot_strict_ne_propose_pas_de_relance_admin` couvre la réversion, et le rejeu Sandbox après correction observe `notes:[]`.

**Défaut de port corrigé.** Avant correction, `/api/elevate` répondait `ok:true` et lançait bien `--elevated`, mais l'instance élevée se rabattait sur 8899 parce que l'ancien serveur occupait encore 8898 ; le relevé après acceptation échouait. `main.rs` réessaie désormais le port demandé pendant 5 s pour `--elevated`. Après rebuild, le test d'endpoint a observé `admin:true`, `can_elevate:false`, `original_exited:true` et `/api/quit` 200 sur 8898.

**Réversion.** `le_defaut_ne_demande_pas_uac` échoue si le code réintroduit l'élévation automatique ; `no_admin_est_strict` échoue si `/api/elevate` reste accessible.

**Critère de sortie.**
- [x] Aucun UAC n'est demandé sans action explicite ou `--admin` (tests unitaires).
- [x] `--no-admin` est réellement strict et testé côté serveur (403 et `can_elevate=false`).
- [x] Les options, la relance et les parcours C2 restent inchangés.
- [x] Les scénarios `--admin` d'acceptation et d'annulation sur compte standard sont consignés ; C6, compte administrateur, ne peut pas valider le consentement UAC.
- [x] La route `POST /api/elevate` du bouton est testée sur compte standard, y compris le retour sur le même port après acceptation ; le défaut de port est corrigé.
- [x] Le clic DOM et le rechargement après acceptation sont validés avec Chrome for Testing portable ; Edge n'a pas pu être installé dans le Sandbox.
- [x] `--no-admin` sur VM standard : `can_elevate=false`, `POST /api/elevate` 403, aucun `consent.exe` ; la note contradictoire a été corrigée et couverte par `le_snapshot_strict_ne_propose_pas_de_relance_admin`.
- [x] `--host localhost` sur VM standard : page et API via `localhost` ; l'alias `127.0.0.1` n'était pas joignable sur cette liaison.

**Effort indicatif :** 0,5 à 0,75 jour, plus le test VM. Aucun changement de dépendance.

**Risques et parades.**
- **Confusion utilisateur** : l'interface doit expliquer la lecture seule et proposer un bouton nommé, pas seulement refuser l'action.
- **UAC annulé** : conserver l'instance lecture seule ; ne pas laisser de processus élevé orphelin.
- **Fausse promesse de sécurité** : cette étape réduit la durée d'exposition avant l'UAC, pas la surface d'une instance déjà élevée. D9-2b (helper one-shot) reste séparé et ne doit pas être revendiqué avant tests.
- **D9-1 inchangé** : un processus local hostile du même compte reste hors modèle de menace.

**Hors périmètre de ce lot.** Le helper élevé par lot, le named pipe/DACL, la suppression de l'UAC et toute revendication de protection contre un malware local.

### 9.12 — Helper elevated one-shot : D9-2b

**État :** note de conception, **non implémenté**. Aucun code, aucune dépendance et aucun test de production ne sont ajoutés par cette note.

**Objectif.** Réduire la durée pendant laquelle RamScope détient un jeton administrateur. Le parcours normal ne doit plus élever une instance complète de l'interface : une opération d'écriture explicitement confirmée lance un helper élevé, qui reçoit une seule demande, la revérifie, l'applique, renvoie son résultat puis se termine. L'interface standard reste en lecture seule.

**Décision de principe.** Le helper est un **one-shot par lot**, pas un serveur administrateur résident :
- une seule demande par processus ;
- aucun écouteur HTTP, aucune page, aucun navigateur ;
- sortie après la réponse, l'annulation ou le délai de sécurité ;
- une nouvelle opération exige une nouvelle demande et une nouvelle UAC.

**Non-objectifs.**
- Remplacer HTTP par un named pipe pour toute l'interface : seul le canal helper est concerné.
- Supprimer l'UAC ou promettre une protection contre un processus local hostile du même compte (D9-1 reste accepté).
- Garder un broker administrateur en arrière-plan.
- Exécuter une commande arbitraire ou une action absente du snapshot courant.
- Remplacer immédiatement `--admin`, qui reste le mode explicite d'une session entièrement élevée.

**Architecture retenue.**
1. Le serveur standard valide la sélection et construit un lot (`apply` ou `restore`) avec les états observés nécessaires.
2. Il crée un named pipe **à une seule instance**, avec un nom aléatoire et une DACL limitée à `SYSTEM` et au groupe `Administrators`, puis `PIPE_REJECT_REMOTE_CLIENTS`.
3. Il lance le même exécutable avec un mode interne `--helper --pipe <nom> --nonce <32 octets hex> --server-pid <pid>` via `ShellExecuteExW(..., "runas", SEE_MASK_NOCLOSEPROCESS)`. L'UAC reste le consentement utilisateur.
4. Le helper se connecte au pipe, s'authentifie, reçoit **une** demande encadrée, la revérifie sur une collecte fraîche, applique le lot et renvoie un résultat structuré.
5. Le serveur standard affiche le résultat et reprend son état lecture seule ; le helper se termine.

**Pourquoi un pipe créé par le serveur standard.** Le helper n'ouvre aucun port et n'a pas à écouter. Le serveur possède déjà le pipe avant l'UAC, peut vérifier le PID du client et fermer le canal en cas d'annulation. Un fichier de demande temporaire serait plus simple mais offrirait un canal moins contrôlable et une course de fichiers supplémentaire.

**Canal et authentification.**
- Nom de pipe imprévisible ; une seule instance ; messages encadrés par une longueur et bornés (demande maximale 64 Kio).
- `nonce` CSPRNG de 32 octets, à usage unique, jamais journalisé ; il est passé au helper par la ligne de commande, donc lisible par un processus du même compte. Ce n'est pas une frontière contre un malware local (D9-1).
- Le serveur vérifie le PID du client avec `GetNamedPipeClientProcessId`, le compare au PID retourné par `ShellExecuteEx` et vérifie la session interactive. Il refuse un client qui n'est pas le helper lancé.
- Le helper vérifie `GetNamedPipeServerProcessId` contre `--server-pid`, vérifie qu'il est réellement élevé, puis envoie `HELLO {nonce, version}`. Un `nonce` invalide, un PID différent ou une deuxième connexion ferment le canal sans action.
- La DACL du pipe réduit les connexions accidentelles ou inter-sessions ; elle ne protège pas contre un processus malveillant du même compte, qui peut déjà lire la page et le jeton (D9-1). Cette limite doit rester écrite dans `SAFETY.md`.

**Protocole minimal.**
- `HELLO` : nonce, version du protocole, PID du serveur.
- `REQUEST` : `request_id`, `operation` (`apply` ou `restore`), lot d'actions avec leurs états observés, libellé, limite de taille.
- `RESULT` : `request_id`, `ok`, message, entrées de rollback éventuelles, code d'erreur.
- Le serveur n'envoie qu'une demande ; le helper ne lit qu'une demande ; le canal est fermé après la réponse. Un `request_id` rejoué est refusé.

**Validation côté helper.** Le helper ne fait pas confiance au lot reçu :
- il vérifie qu'il est élevé et que la demande vient du PID serveur attendu ;
- il recollecte l'état courant et refuse toute action absente du snapshot, déjà appliquée, protégée, hors limite ou dont l'état a changé ;
- il réutilise les mêmes fonctions pures de `plan` que le serveur, sans dupliquer les listes de protection ;
- il refuse `StartService` sur une route utilisateur et réserve les inverses de restauration au chemin interne ;
- il borne le nombre d'actions (400) et la taille de la demande ;
- il n'accepte aucune action de type « kill » non proposée par le snapshot courant.

**Durée de vie et annulation.**
- Attente d'une demande : 30 s maximum ; opération : délai global de 120 s, à ajuster après mesure.
- Une annulation avant le début de l'application ne produit aucune écriture.
- Une annulation ou une coupure pendant l'application peut laisser un lot partiel : le helper renvoie les entrées de rollback déjà produites ; le serveur marque le résultat `partial` et ne rejoue jamais automatiquement.
- Si le helper dépasse le délai, le serveur ferme le pipe et attend sa sortie ; tuer un helper élevé doit être exceptionnel, journalisé et ne doit pas être présenté comme un rollback.
- Aucun processus élevé ne doit rester après un résultat, une erreur, une annulation ou un timeout.

**Interface et compatibilité.**
- Le bouton global « Passer en administrateur » ne peut pas conserver son sens actuel avec un helper one-shot. Le parcours recommandé est un bouton **« Appliquer avec élévation »** sur le lot confirmé ; sans lot sélectionné, aucune UAC n'est demandée.
- Une route dédiée, par exemple `POST /api/apply-elevated` et `POST /api/restore-elevated`, porte le lot et le consentement. L'ancienne route `/api/elevate` ne doit pas lancer silencieusement une instance longue durée dans le nouveau parcours.
- `--admin` reste disponible pour une session explicitement élevée ; `--no-admin` continue d'interdire toute demande. Le mode `--elevated` existant reste interne tant que `--admin` en dépend.
- Le helper ne doit pas être exposé dans `--help` comme une option utilisateur.

**Tests de réversion nommés.**
- `le_helper_refuse_un_nonce_rejoue` : une deuxième connexion avec le même nonce ne reçoit aucune demande.
- `le_helper_refuse_un_pid_serveur_different` : un pipe usurpé ne peut pas faire exécuter une action.
- `le_helper_refuse_une_session_differente` : le PID et la session du client sont vérifiés.
- `le_helper_exige_l_elevation` : un helper non élevé refuse la demande.
- `le_helper_refuse_une_action_hors_snapshot` : un lot forgé ne passe pas la revérification fraîche.
- `le_helper_refuse_un_lot_trop_grand` : la borne de 400 actions et la taille de message sont testées.
- `le_helper_refuse_une_restauration_non_demandee` : le chemin restore n'accepte que l'identifiant prévu.
- `le_helper_sort_apres_une_reponse` : aucun processus élevé ne survit au résultat.
- Test Sandbox compte standard : acceptation UAC, une action réelle appliquée, helper sorti, serveur standard toujours `admin:false` et `can_elevate:true`, puis deuxième opération avec un nouveau PID de helper.
- Test Sandbox annulation : UAC annulée ou pipe fermé avant `REQUEST` ; aucune action, aucun helper restant.
- Test Sandbox coupure : helper interrompu pendant un lot ; résultat `partial` ou `unknown` explicite, jamais de rejeu automatique.

**Critères de sortie.**
- [ ] Aucune instance RamScope élevée ne reste après une opération normale ; le helper ne sert qu'une demande.
- [ ] Le serveur standard ne devient jamais administrateur et son interface reste en lecture seule.
- [ ] Un lot forgé, rejoué ou hors snapshot est refusé par le helper.
- [ ] Une annulation UAC ne laisse ni pipe ouvert ni processus élevé.
- [ ] Les tests de réversion ci-dessus échouent si le contrôle correspondant est retiré.
- [ ] La documentation ne présente pas le pipe comme une protection contre un malware du même compte.

**Risques et parades.**
- **Course UAC/pipe** : créer le pipe et le nom avant `ShellExecuteEx`, conserver le handle, fermer en cas d'échec ; ne jamais réutiliser un nom.
- **DACL trop large ou trop étroite** : test avec compte standard et compte administrateur distinct ; `SYSTEM` + `Administrators`, pas `Everyone`.
- **PID réutilisé** : comparer le handle de processus et la session, pas seulement le numéro après attente.
- **Lot partiel** : retour explicite `partial` avec entrées de rollback ; pas de compensation automatique non testée.
- **Duplication de validation** : réutiliser les fonctions pures de `plan` et ajouter un test qui contourne le serveur.
- **UX** : le bouton global actuel ne doit pas être conservé avec une promesse d'élévation persistante ; la confirmation doit nommer le lot et l'action élevée.

**Décisions ouvertes avant implémentation.**
1. Périmètre initial : apply seulement, ou apply + restore dès la première version.
2. Conserver `--admin` comme mode documenté ou le déprécier après validation du helper.
3. Forme exacte de la route et du bouton de confirmation du lot.
4. DACL précise et comportement avec un administrateur UAC différent du compte standard.
5. Politique de timeout et de résultat `unknown` après mort brutale du helper.

**Effort indicatif.** 2 à 3 jours de développement et de tests, hors validation Sandbox étendue ; aucune dépendance externe nouvelle (FFI Windows et named pipes natifs).

**Hors périmètre de cette note.** Implémentation, migration de l'interface, suppression de `--admin`, refonte D9-3 du canal local et toute revendication de protection contre un processus local hostile.

## 6. Ordre de livraison recommandé

1. **9.1** : chemins système et données (C7), indépendant et testable rapidement.
2. **9.2** : propagation des options CLI (C6), correctif fonctionnel concret.
3. **9.3** : frontière de confiance (C2), après décision D9-1.
4. **9.4** : origine obligatoire et corps borné (C3, C4).
5. **9.5** : verrous indépendants ou correction documentaire (C5).
6. **9.6** : provenance réelle des règles (C1), traité le 2026-11-15.
7. **9.7** : zéro mesuré (C14), traité le 2026-11-15.
8. **9.8** : robustesse et finitions (C9–C13), traité le 2026-11-15.
9. **9.9** : tests comportementaux (C8), traité le 2026-11-15.
10. **9.10** : décision de licence (C15), MIT appliquée le 2026-11-15.
11. **9.11** : réduction de l'élévation UAC (D9-2a), implémentée ; les parcours `--admin`, `POST /api/elevate`, le clic DOM et le rechargement sur compte standard sont testés.

Chaque étape doit être un commit séparé, avec son test nommé, sa commande et son code retour. Ne pas regrouper 9.1 à 9.4 dans un même commit : une revue de sécurité doit pouvoir être relue isolément.

C1, C14, C9–C13, C8 et C15 sont désormais clos. C2 et C7 le sont aussi ; la distribution publique est possible sous MIT, le dépôt est public et la CI est verte.

## 7. Matrice de tests à ajouter

| Domaine | Test attendu |
|---|---|
| Chemins | `SystemRoot` forgé, `LOCALAPPDATA` forgé, `schtasks`/`explorer` absolus |
| Élévation | arguments conservés, `--elevated` unique, port/hôte effectifs, défaut sans UAC, `--no-admin` sans élévation demandée |
| API | `Host` invalide, jeton absent/invalide, origine absente/invalide |
| État | `from` nul refusé, `from` divergent refusé, restauration interne autorisée |
| Corps | `Content-Length` énorme, corps chunked énorme |
| Verrous | appel direct à `win::apply` sur cible protégée |
| Provenance | source exposée, date « non datée » et date ISO acceptées |
| Gains | hôte complet/partiel, `measured(0)`, `estimated`, `unavailable` |
| Robustesse | `extract_exe` Unicode, port 65 535, arrêt propre |
| Tests | décisions pures extraites, tests de chaînes restants justifiés |
| Preuves | `sha256sum`, CI, absence de fichiers de test non suivis |

Les tests d'écriture réelle restent limités à une VM ou un Sandbox jetable. La collecte réelle sur l'hôte reste en `--no-admin` et ne doit jamais appliquer d'action.

## 8. Critères de sortie du lot 9

- [x] Les 43 tests historiques passent toujours ; les 83 tests Rust actuels passent, ainsi que les quatre tests Node.
- [x] Tests de réversion nommés des chantiers clos (C2, C3–C6, C7, C9–C13, C14, D9-2a) — voir PLAN-LOT9 §9.3–9.8 et §9.11. Le chantier D9-2b (9.12) reste non implémenté et ses tests de réversion sont donc hors périmètre.
- [x] `cargo fmt --check`, `cargo test`, `cargo clippy --all-targets -- -D warnings`, `cargo build --release` : code retour 0.
- [x] C2 est implémenté et couvert par les tests de réversion nommés ; le risque local résiduel est documenté dans `SAFETY.md` (D9-1).
- [x] La CI est verte sur `c9771fc` : run Windows `43663058587`, toutes les étapes au vert.
- [x] C7 est corrigé ou explicitement documenté comme hors modèle de menace ; C1 est corrigé.
- [x] C3–C6 sont corrigés ou explicitement acceptés dans `SAFETY.md` ; C14 est corrigé.
- [x] C9–C13 sont corrigés.
- [x] C8 est traité par des tests comportementaux ciblés, avec les limites documentées.
- [x] C15 est décidé (MIT) et consigné.
- [x] D9-2a est implémenté et couvert par les tests unitaires ; `--no-admin` est strict côté serveur, le défaut est en lecture seule s'il n'est pas déjà élevé, et la relance ne boucle pas.
- [x] D9-2a est testé avec un compte standard (consentement, annulation, route `POST /api/elevate`, clic DOM et rechargement) ; Edge n'a pas pu être installé dans le Sandbox, le clic DOM a été validé sous Chrome for Testing. Le helper one-shot reste hors périmètre.
- [x] La documentation de D9-2a ne promet plus de lecture seule absolue ni de session rétrogradée ; la revue des autres lots reste ouverte.
- [x] Des actions réelles ont été testées dans un Windows Sandbox jetable : `dmwappushservice` Manuel → Désactivé → Manuel, `StartupAppTask` Ready → Disabled → Ready, et une entrée de démarrage `StartupApproved` 3 → 2, avec les API natives `Win32_Service`, `Get-ScheduledTask` et le registre vérifiés avant, après application et après restauration.
- [x] Le dépôt distant est à jour après accord de Ro, y compris le compte rendu du test de démarrage.
- [x] `PLAN.md` contient les commandes, codes retour et artefacts du lot (sections 12–17, confirmé par relecture le 2026-11-21).
- [x] C7 et C6 sont implémentés et couverts par des tests unitaires locaux ; C6 est validé dans Windows Sandbox pour la propagation des options (`--port`, `--host`, `--no-browser`, `--elevated` unique, `--no-admin` retiré). Le consentement UAC, le clic DOM et `--host localhost` ont depuis été testés sur compte standard.

## 9. Hors périmètre

- Refonte de l'interface ou changement de framework front.
- Remplacement complet de HTTP par un canal local dédié, sauf décision explicite après 9.3.
- Nouvelles règles métier ou nouvelle base de règles.
- Signature de code, installeur et distribution publique : la licence MIT est choisie, mais le push et la CI doivent être finalisés.
- Télémétrie, collecte distante ou nettoyage automatique.

## 10. Risques et parades

| Risque | Parade |
|---|---|
| Casser le chemin par défaut en corrigeant l'élévation | test d'arguments + test réel en VM avec port/hôte explicites |
| Introduire une régression de sécurité en déplaçant la validation | tests `from` nul/divergent, appel direct au moteur, revue dédiée |
| Refonte du canal local trop large | décision séparée après mesure ; ne pas mélanger avec 9.1–9.4 |
| Test d'audit qui verrouille la dette de provenance | accepter explicitement `non datée` ou une date ISO, avec un test daté |
| Tests de texte source qui remplacent de vrais tests | extraire les fonctions pures et limiter `include_str!` aux gardes justifiées |
| Fausse preuve de sécurité | ne jamais marquer un P1 « corrigé » sans test de réversion et, si possible, PoC local |
| Licence choisie par défaut sans décision | décision MIT consignée le 2026-11-15 ; toute évolution ultérieure reste explicite |
| Perte de sauvegarde distante | push après accord, vérifier `git ls-remote` |

## 11. Prochaine action

**Point au 2026-11-15.** C7, C6, C2, C3, C4, C5, C1, C14, C9–C13, C8, C15 et D9-2a sont traités ; la suite compte **83 tests Rust** et **4 tests Node**, tous verts, et la revue documentaire de D9-2a est alignée sur le code (commit `bf5c7fb`). D9-2b reste une conception, non implémentée ; la seule réserve de test de D9-2a est Edge, absent du Sandbox utilisé (le clic DOM a été validé sous Chrome for Testing, même moteur Chromium).

D9-2a est clôturé avec les preuves UAC, route, DOM, `--no-admin` et `--host localhost` ; la session déjà élevée n'est pas rétrogradée par `--no-admin`, et cette limite est désormais explicite dans `README.md`, `SAFETY.md`, `PLAN.md` et l'aide CLI.

La note contradictoire `--no-admin` est corrigée et couverte par `le_snapshot_strict_ne_propose_pas_de_relance_admin`. L'alias `127.0.0.1` non joignable avec `--host localhost` est désormais documenté comme comportement de liaison : `localhost` suit la résolution locale (souvent `::1`), et l'adresse affichée au démarrage doit être utilisée. Aucun changement de code n'est retenu pour ce point.

Ordre recommandé pour la suite :
1. **D9-2b — helper elevated one-shot** (conception en **9.12**) : valider les décisions ouvertes, puis implémenter le canal local, l'authentification, la durée de vie et les tests de réversion. Ne pas le mélanger à D9-1 ni à C2.
2. **Publication** : traiter les fichiers non suivis, vérifier la CI, puis pousser la décision MIT après accord.
3. **Validation Edge spécifique** : sur une image où Edge est préinstallé, sans bloquer le mode par défaut déjà validé sous Chromium.

La revue Hermes-Beam ne change pas cette priorité : elle confirme C7 et C6, ajoute C1/C8/C15, et justifie de renforcer 9.3 sur l'état fourni par le client. C3/C4/C5 sont désormais clos ; le plan est aligné sur ces constats et sur le durcissement livré.
