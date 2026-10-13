# TODO_LIST — Feuille de route de professionnalisation de Spawnly

**À l'attention de :** Olivier DUTOIT
**Projet :** Spawnly — location de serveurs de jeux (Minecraft, Valheim, Rust, CS2, ARK, Palworld)
**Base :** `DOCUMENTATION.md` (version complète), `AGENTS.md`
**Objet :** Faire évoluer votre application vers la qualité d'un site professionnel, **tout en la faisant tourner sur WAMP dans le cadre des TP**, sans renier les principes qui font sa valeur.

---

Bonjour Olivier,

Votre documentation est remarquable : architecture **MVC minimaliste** avec front controller et routeur maison, PDO natif préparé (`EMULATE_PREPARES = false`), autoloader PSR-4, helpers typés (`json()` / `redirect()` en `never`), codes de vérification hachés avec anti-énumération des comptes, design system complet et animations Canvas soignées. La section **« 16. Limites connues »** est le signe d'un vrai professionnel : vous identifiez vous-même vos faiblesses.

Cette feuille de route part donc de **votre code réel**, pas d'une hypothèse. Elle vise trois choses :

1. **Corriger** les limites que vous avez déjà documentées (robustesse).
2. **Industrialiser** les pratiques d'ingénierie absentes (tests, lint, outillage local).
3. **Étendre** vers les fonctions attendues d'un vrai service d'hébergement (simulation de paiement, provisioning simulé, back-office).

> 🖥️ **Cadre d'exécution : WAMP en local (TP).** Le site finalisé continue de tourner **sous WAMP sur un poste Windows** (`http://localhost/cfai/`). **Aucun serveur web réel, aucun VPS, aucune conteneurisation (Docker) ne sont demandés.** Le volet « exploitation » est donc traité comme un ensemble de **bonnes pratiques documentées et testables en local sur WAMP**, jamais comme des tâches d'infrastructure. Tout ce qui exigerait un hébergement réel est explicitement signalé **« hors périmètre TP »**.

> ⚠️ **Respect de vos règles (`AGENTS.md`)** : vos principes KISS et « aucune nouvelle dépendance (librairie, framework, CDN) sans accord » sont conservés. Partout où je recommande un outil externe (Composer, build CSS, PHPUnit, services externes…), il est marqué **« sous accord tuteur »** : à valider avant installation. Aucune de ces recommandations n'exige de réécrire le noyau.

## Comment lire cette liste

| Balise | Signification |
|---|---|
| 🔴 **P0 — Critique** | Bug avéré, faille, ou blocage de la livraison. À traiter en premier. |
| 🟠 **P1 — Important** | Attendu d'un site professionnel (fiabilité, maintenabilité, conformité). |
| 🟡 **P2 — Montée en gamme** | Différenciateur / ouvre des fonctions métier premium. |
| **Complexité** | Difficulté de mise en œuvre : **[simple]**, **[moyen]** ou **[complexe]**. |

---

## 0. Définition de « Terminé »

Une tâche n'est cochée que si :

- [ ] 🔴 Le code passe `php -l`, **PHPStan niveau 6+** et **PHP_CodeSniffer (PSR-12)** sans erreur (outils **sous accord tuteur**).
- [ ] 🔴 Tout correctif de la section §1 est couvert par un **test** (nominal + erreur).
- [ ] 🟠 La ligne correspondante de **`DOCUMENTATION.md`** (§7 API, §8 BDD, §16 Limites) est mise à jour dans la même itération.
- [ ] 🟠 Aucun secret (config BDD, clés Mailjet) n'est présent dans un fichier versionné.
- [ ] 🟠 Les messages utilisateur restent **en français**, le code et les colonnes **en anglais** (votre règle §15).

---

## 1. Correctifs prioritaires issus de vos « Limites connues » 🔴

Chaque ligne ci-dessous correspond à une limite que **vous avez vous-même identifiée** en §16. Ce sont les gains de robustesse les plus rentables.

| # | Limite documentée (§16) | Action attendue | Priorité | Complexité |
|---|---|---|---|---|
| 1.1 | `available >= 1` + `INSERT` hors transaction → survente possible | Encapsuler vérification + insertion dans une **transaction** avec `SELECT ... FOR UPDATE` sur la ligne `servers`, dans `RentalController` / `Rental::create()` | 🔴 | moyen |
| 1.2 | `+N months` déborde (31 janv. + 1 mois = 3 mars) | Calculer `end_date` avec une règle **fin de mois explicite** (`DateTimeImmutable` + `modify('last day of ...')` selon le jour de départ) et documenter la règle métier | 🔴 | simple |
| 1.3 | Fuseaux : dates PHP (`Europe/Paris`) vs `CURDATE()` MySQL (fuseau serveur) | Fixer le fuseau côté PDO (`SET time_zone = '+01:00'`) **et** comparer dans un seul référentiel ; tester le changement d'heure. Supprimer `CURDATE()` des colonnes calculées au profit d'une date passée par PHP | 🔴 | moyen |
| 1.4 | Cookie de session non `Secure` | Rendre l'attribut `secure` **conditionnel** (actif derrière HTTPS, désactivé en local HTTP) et passer à `samesite=Strict` sur les cookies sensibles. Documenter la bascule (le WAMP local reste en HTTP). `session_regenerate_id(true)` déjà en place — à confirmer | 🟠 | simple |
| 1.5 | Pages `/login` et `/register` accessibles une fois connecté | Rediriger un utilisateur déjà connecté (`if (Auth::check()) redirect('/rentals')`) dans `PageController::login/register` | 🟠 | simple |
| 1.6 | JS obligatoire : formulaires sans `action`, aucune page ne fonctionne sans JS | **Amélioration progressive** : donner un `action` + `method="post"` réel aux formulaires, avec repli serveur (POST classique → validation → réponse HTML ou 303). Le JS reste une amélioration, pas un prérequis | 🟠 | moyen |
| 1.7 | Aucune limitation des tentatives de connexion | Ajouter un **rate limiting** par IP + par compte sur `POST /api/auth/login` (compteur en base ou en session), avec délai progressif et message générique | 🔴 | simple |
| 1.8 | Endpoints GET de l'API non utilisés (`/api/me`, `/api/servers*`, `/api/rentals*`) | Décider : soit les **consommer** en JS (recherche serveur, rafraîchissement), soit les **assumer comme API publique** et les documenter/versionner. Pas de code mort | 🟡 | simple |
| 1.9 | Paiement fictif, adresse de connexion simulée `*.spawnly.test` | Voir §10 : **simulation de paiement** (aucune passerelle bancaire réelle) + **provisioning simulé** du serveur | 🟡 | — |
| 1.10 | Pas de tests ni de lint | Voir §6 (PHPUnit + CI + PHPStan/PSR-12) | 🟠 | — |

**Critère d'acceptation :** la section « 16. Limites connues » de la documentation ne contient plus aucune limite 🔴/🟠 non traitée ou explicitement reportée avec justification.

---

## 2. Sécurité 🔴

Vos bases sont bonnes (PDO préparé, `e()` partout, 2FA, anti-énumération). Voici ce qu'il manque à un site exposé à Internet ; tout se teste **en local contre `http://localhost/cfai`**, sans hébergement.

- [ ] 🔴 **[simple] — Jeton CSRF explicite.** Aujourd'hui la protection repose uniquement sur `SameSite=Lax` + JSON. Ajoutez un jeton de session (`csrf`), vérifié sur **tous** les POST (`/api/*`), et incluez-le dans l'en-tête `X-CSRF-Token` envoyé par `app.js`.
- [ ] 🔴 **[simple] — Durcir le hachage.** Passer à **Argon2id** (`PASSWORD_ARGON2ID`) avec `password_needs_rehash()` à la connexion ; conserver la compatibilité des hachages existants.
- [ ] 🔴 **[moyen] — En-têtes de sécurité HTTP** dans `public/.htaccess` (testables sur l'Apache de WAMP) : `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, `X-Frame-Options`. **Attention :** une CSP stricte suppose de sortir du CDN Tailwind (§7). `Strict-Transport-Security` n'a de sens que derrière HTTPS : préparez-le en commentaire et documentez-le pour une future mise en ligne.
- [ ] 🟠 **[simple] — Webhook Mailjet authentifié.** Si vous consommez les événements de l'API Mailjet, vérifier la signature/le secret pour éviter des événements forgés.
- [ ] 🟠 **[simple] — Revue OWASP Top 10** formalisée : pour chacune des 10 catégories, indiquer la mesure en place dans Spawnly (excellent complément à votre §14 Sécurité).
- [ ] 🟠 **[simple] — Message d'erreur 500 générique** côté utilisateur (page dédiée) ; détails uniquement dans les logs.
- [ ] 🟡 **[moyen] — 2FA TOTP en complément de l'e-mail** (authenticator), plus robuste et plus professionnel ; l'e-mail peut rester en secours.
- [ ] 🟡 **[simple] — Journalisation des actions sensibles** (changement de mot de passe, changement d'e-mail, activation 2FA, résiliation) dans une table d'audit.

**Critère d'acceptation :** un scan OWASP ZAP (baseline, lançable en local contre `http://localhost/cfai`) ne remonte aucune vulnérabilité haute ; l'en-tête CSP est actif sans casser l'interface.

---

## 3. Base de données & intégrité 🔴

- [ ] 🔴 **[complexe] — Migrations versionnées.** `database/schema.sql` fait un `DROP TABLE` destructeur : remplacez-le (ou complétez-le) par des migrations incrémentales et réversibles (Phinx **sous accord tuteur**, ou runner maison documenté). Le `DROP` reste réservé au poste de développement.
- [ ] 🔴 **[simple] — Jeu de données de démonstration (seed) séparé du schéma** (vous l'avez aujourd'hui mélangé aux tables).
- [ ] 🟠 **[simple] — Index** sur les colonnes de jointure/filtre (`rentals.user_id`, `rentals.server_id`, `rentals.status`, `rentals.end_date`, `verification_codes.user_id`) et vérification des plans via `EXPLAIN`.
- [ ] 🟠 **[simple] — Sauvegardes automatiques** (dump quotidien, rétention) + **procédure de restauration testée**.
- [ ] 🟠 **[simple] — Suppression utilisateur maîtrisée.** `ON DELETE CASCADE` sur `verification_codes` est bien ; définissez explicitement le sort des `rentals` (conservation de l'historique de location vs effacement). Voir §8 RGPD.
- [ ] 🟡 **[simple] — Purge planifiée** des codes expirés et des comptes non vérifiés anciens (script `.bat` lancé par le **Planificateur de tâches Windows**, équivalent du `cron`), pour ne pas laisser croître `verification_codes`.

**Critère d'acceptation :** une base existante peut être mise à jour par migration, sans perte de données (le `DROP` restant réservé au poste de développement local), et la restauration est démontrée.

---

## 4. Qualité du code & architecture 🟠

Votre règle KISS est saine : ne complexifiez pas, mais clarifiez.

- [ ] 🟠 **[simple] — PHPStan (level 6) + PHP_CodeSniffer (PSR-12)**, configurés et documentés, exécutables via un script unique (`composer lint` **sous accord tuteur**, sinon scripts `.bat`/Makefile).
- [ ] 🟠 **[simple] — Journalisation PSR-3** (Monolog **sous accord tuteur** ou logger maison conforme) en remplacement des `error_log` dispersés, avec niveaux et contexte structuré.
- [ ] 🟠 **[simple] — Gestion d'erreurs unifiée** : exceptions typées (`ValidationException`, `NotFoundException`, `ConflictException`), mapping vers les codes HTTP déjà en place (400/404/409/422).
- [ ] 🟠 **[simple] — Objets de transfert typés** pour les entrées (au lieu de `(string) ($data['x'] ?? '')` répété partout), sans introduire d'ORM.
- [ ] 🟡 **[simple] — Enums PHP 8.1+** pour `purpose` (`register|login|reset_password`) et `status` (`active|cancelled`), pour supprimer les chaînes magiques.
- [ ] 🟡 **[simple] — ADR (Architecture Decision Records)** : formaliser les choix déjà commentés (« pourquoi pas de framework », « pourquoi pas d'ORM », « pourquoi le faux `pending` ») — très valorisant en dossier de projet.
- [ ] 🟡 **[moyen] — (Optionnel, à évaluer) Migration vers Composer** pour l'autoload PSR-4 et les dépendances de développement. **Sous accord tuteur** : à ne faire que si le gain est démontré, au regard de votre règle KISS.

---

## 5. API & conventions 🟠

- [ ] 🟠 **[simple] — Versionner l'API** (`/api/v1/...`) depuis `routes/api.php`, pour pouvoir évoluer sans casser les clients.
- [ ] 🟠 **[simple] — Corps d'erreur homogène.** Aujourd'hui `{"error": "..."}`. Adoptez `{"error": {"code": "...", "message": "..."}}` (code machine + message français) sur tous les endpoints.
- [ ] 🟠 **[simple] — Pagination et tri serveur** sur `/api/servers` et `/api/rentals` (`?page=`, `?per_page=`, `?sort=`), au lieu de tout renvoyer.
- [ ] 🟠 **[simple] — Idempotence** de `POST /api/rentals` (en-tête `Idempotency-Key`) pour éviter une double location lors d'un double-clic ou d'un retry réseau (en lien avec §1.1).
- [ ] 🟡 **[simple] — Limitation de débit** (rate limiting) aussi sur `/api/auth/register`, `/api/auth/forgot-password` et `/api/rentals`.
- [ ] 🟡 **[moyen] — Spécification OpenAPI 3** (`openapi.yaml`) + page de doc générée. Vos endpoints GET inutilisés deviennent alors une vraie **API publique**.

**Critère d'acceptation :** un client tiers peut consommer l'API à partir de la seule spec OpenAPI, avec pagination et erreurs homogènes.

---

## 6. Tests & assurance qualité 🟠

C'est le principal écart restant avec une pratique professionnelle (votre §16 le reconnaît).

- [ ] 🟠 **[moyen] — PHPUnit** (**sous accord tuteur**) : premiers tests unitaires sur la logique métier — calcul `total_price`, `end_date` (§1.2), `available` (§1.1), résiliation, hachage des codes.
- [ ] 🟠 **[moyen] — Tests d'intégration** des parcours : inscription → `/verify` → location → résiliation ; mot de passe oublié ; 2FA.
- [ ] 🟠 **[simple] — Base de test isolée** (base dédiée ou SQLite en mémoire) avec rollback entre tests.
- [ ] 🟠 **[simple] — Couverture de code** mesurée, seuil réaliste sur le cœur métier (ex. 60 %). Optionnel : CI GitHub Actions (runners hébergés — **rien à installer en local**, pas de Docker) affichant le taux.
- [ ] 🟡 **[complexe] — Tests de bout en bout** (Playwright **sous accord tuteur**) : parcours de location, bascule de thème, navigation clavier.
- [ ] 🟡 **[simple] — Fixtures** pour générer utilisateurs/serveurs/locations de test.

**Critère d'acceptation :** une commande unique lance toute la suite ; une régression sur le calcul de prix, le stock ou l'auth fait échouer l'intégration continue.

---

## 7. Front-end & UX 🟠

- [ ] 🔴 **[moyen] — Sortir Tailwind du CDN navigateur.** `@tailwindcss/browser` compile dans le navigateur du visiteur : coût de performance, flash de style, dépendance à jsDelivr, et **incompatible avec une CSP stricte** (§2). Produisez un **CSS statique purgé** à l'aide du **CLI autonome Tailwind** (un seul exécutable Windows, **sans Node, sans Docker, sans serveur** — **sous accord tuteur**) et référencez-le depuis `public/assets/`. Sans accord, conservez le CDN mais documentez la limite et affaiblissez la CSP en conséquence.
- [ ] 🟠 **[simple] — Optimisation des polices/icônes** : `preconnect`, préchargement des polices réellement utilisées, sous-ensemble de Material Symbols, cache long (`Cache-Control`) avec hash dans le nom de fichier.
- [ ] 🟠 **[simple] — SEO & partage social** : `title`/`meta description` uniques par page, Open Graph / Twitter Cards, `sitemap.xml`, `robots.txt`, données structurées JSON-LD `Product`/`Offer` avec prix.
- [ ] 🟠 **[moyen] — Accessibilité WCAG 2.1 AA** : contraste des deux thèmes (à valider chiffres à l'appui), focus visible, `aria-*` des composants custom, `label` associés, `alt` des images. Votre §13.5 est un bon départ à auditer.
- [ ] 🟡 **[simple] — États d'interface** charger/vide/erreur/succès sur chaque écran (aujourd'hui surtout succès/erreur via `data-form-*`).
- [ ] 🟡 **[simple] — PWA** (manifest + service worker prudent) et respect de `prefers-reduced-motion` (déjà bien pris en compte).
- [ ] 🟡 **[moyen] — Internationalisation** : externaliser les libellés FR (le site est très textuel) et préparer une seconde langue ; formats de date/devise selon la locale.
- [ ] 🟡 **[simple] — Cohérence du design system** : documenter les composants (`.btn`, `.card`, `.status-dot`…) et les tokens dans un mini « design system » de référence.

**Critère d'acceptation :** Lighthouse (mobile) ≥ 90 en Performance, Accessibilité et SEO sur `/` et `/servers` ; aucun CLS visible.

---

## 8. Conformité juridique & RGPD 🔴

Tout site professionnel qui collecte des données personnelles doit s'y conformer — le RGPD s'applique indépendamment du caractère simulé du paiement.

- [ ] 🔴 **[simple] — Mentions légales, CGV/CGU et politique de confidentialité** accessibles depuis le footer.
- [ ] 🔴 **[simple] — Bandeau de consentement cookies** si vous ajoutez de l'analytique (refus aussi simple que l'acceptation).
- [ ] 🔴 **[simple] — Registre des traitements** : données, finalités, durées, destinataires (Mailjet = sous-traitant).
- [ ] 🔴 **[simple] — Droit à l'effacement** : un parcours « supprimer mon compte » qui efface réellement les données personnelles, avec confirmation ; ne conserve ensuite que l'historique de location strictement nécessaire.
- [ ] 🟠 **[simple] — Durées de conservation** et purge automatique (codes, logs, comptes inactifs).
- [ ] 🟠 **[simple] — Aucune donnée sensible dans les e-mails** (codes, jetons) ; mention du sous-traitant Mailjet.

**Critère d'acceptation :** la suppression de compte efface les données personnelles (vérifiable en base) en ne conservant que l'historique de location strictement nécessaire.

---

## 9. Robustesse & bonnes pratiques d'exploitation 🟠

> 🖥️ **Hors périmètre TP :** aucune installation de serveur web réel, de VPS, de Nginx/PHP-FPM ni de Docker. Le site continue de tourner sous **WAMP**. Cette section regroupe donc des **pratiques locales testables** et de la **documentation de mise en ligne**, pas de l'administration d'infrastructure.

- [ ] 🟠 **[simple] — Configuration par environnement.** Distinguez `config/config.dev.php` (erreurs affichées) de `config/config.prod.php` (erreurs masquées, logs uniquement, **mode « production » simulé**), sélectionnée par une variable. Les deux tournent sur le même WAMP.
- [ ] 🟠 **[simple] — Endpoint `GET /health`** (hors `/api`) qui vérifie la connexion PDO et l'API Mailjet, et renvoie un statut simple. Testable en local ; c'est le point d'entrée de toute supervision future.
- [ ] 🟠 **[simple] — Journalisation structurée et rotation.** Écrivez les logs dans `logs/app.log` (au lieu du `php_error.log` de WAMP) avec niveaux et dates, et prévoyez une rotation simple (fichier par jour).
- [ ] 🟠 **[simple] — Sauvegardes locales scriptées.** Script `.bat` lançant `mysqldump` (inclus dans WAMP) vers `backups/`, avec rétention de N jours, lancé par le **Planificateur de tâches Windows** ; **procédure de restauration testée**.
- [ ] 🟠 **[simple] — Préparation HTTPS.** Documentez la marche à suivre pour activer HTTPS (cookie `Secure`, HSTS, redirection HTTP→HTTPS) **sans l'appliquer en local** : le code doit être prêt, l'activation restant un choix d'hébergement futur.
- [ ] 🟡 **[simple] — Checklist de mise en ligne** (documentation) : prérequis, configuration, import du schéma, variables de config, vérifications, **rollback manuel**. C'est le substitut local au « déploiement automatisé ».
- [ ] 🟡 **[simple] — Plan de reprise (documentaire)** : RTO/RPO visés, procédure d'incident, contact.
- [ ] 🟡 **[simple] — CI légère (optionnel).** GitHub Actions (runners hébergés) pour lint + tests à chaque `push` : **aucune installation locale requise**, donc compatible avec le cadre TP.

**Critère d'acceptation :** le site se réinstalle sur un WAMP vierge en suivant uniquement la checklist documentée, la sauvegarde/restauration fonctionne, et le code est prêt pour HTTPS sans modification ultérieure.

---

## 10. Modélisation métier avancée (simulations) 🟡

Spawnly reste aujourd'hui une démonstration : paiement fictif et adresse de connexion simulée. Cette section propose de **modéliser proprement** ces mécanismes — toujours en **simulation**, toujours **sur WAMP**. On ne cherche pas à ouvrir un service commercial, mais à porter le code au niveau d'une architecture métier professionnelle.

> 🔒 **Interdit dans le cadre du TP : tout paiement en ligne réel.** Un étudiant ne peut pas légalement recourir au système bancaire. Sont donc **proscrits** : toute passerelle de paiement (Stripe, PayPal, Mollie…), y compris en mode test, toute clé d'API bancaire, toute saisie de données de carte et tout prélèvement. L'objectif est une **simulation de paiement** techniquement soignée, sans aucune transaction financière.

> 🧪 **Faisabilité en local — à lire avant de commencer.** Toutes les opérations ci-dessous sont **déclenchables et vérifiables sur un simple WAMP**, sans service payant, sans serveur externe et sans dépendance non validée :
> - le mode simulation est piloté par **une constante de configuration** (ex. `payment.mode = 'simulation'`) ;
> - les e-mails passent par le `Mailer` existant : **sans clé Mailjet, leur contenu est écrit dans un log** (mécanisme déjà en place) — donc entièrement testable sans boîte mail ;
> - les traitements différés (relances, expiration, renouvellement) s'exécutent via un **script PHP lancé à la main** ou par le **Planificateur de tâches Windows** ;
> - **aucune** opération ne dépend d'une passerelle bancaire, d'un panel d'hébergement, d'un serveur de jeu réel ou d'une librairie PDF.

- [ ] 🟠 **[complexe] — Module de paiement simulé, propre et testable.** Créer les tables `orders` / `payments` (montant, devise, statut, référence interne, horodatage) et un `PaymentService` qui **simule** le cycle : création d'un ordre → paiement → confirmation → échec/remboursement. **Aucune donnée bancaire, aucun appel à un prestataire, aucun numéro de carte.**
- [ ] 🟠 **[moyen] — Machine à états explicite des paiements** (`pending → paid → refunded` / `failed` / `expired`), transitions contrôlées côté serveur, journalisation des changements et **idempotence** (une commande ne peut être payée deux fois). L'état `expired` est **calculé à la lecture** (aucune tâche planifiée obligatoire). En lien direct avec §1.1.
- [ ] 🟠 **[simple] — Abstraction « passerelle ».** Définir une interface `PaymentGatewayInterface` (créer, confirmer, rembourser) avec une implémentation `SimulatedGateway`. Le code doit être **prêt à recevoir une vraie passerelle plus tard, sans qu'elle soit implémentée** (pas de clé, pas d'appel réseau).
- [ ] 🟠 **[simple] — Simulation des cas d'échec et de remboursement.** Prévoir un mode « paiement refusé » (déclencheur de démo) pour tester les chemins d'erreur, ainsi qu'un parcours de remboursement simulé.
- [ ] 🟡 **[simple] — Endpoint de confirmation interne (au lieu d'un « webhook »).** Une route de simulation (ex. `POST /api/payments/{id}/confirm`) fait passer une commande de `pending` à `paid`, déclenchable depuis l'interface **ou** par un simple `curl` en local. Aucun service externe, aucun appel entrant à sécuriser.
- [ ] 🟠 **[moyen] — Reçu imprimable (sans dépendance PDF).** Une vue HTML dédiée au reçu (numérotation séquentielle, mentions structurées) avec une feuille de style `@media print` : l'export PDF se fait par l'**impression du navigateur**. **Aucune librairie PDF requise.** Un export PDF côté serveur reste **optionnel et sous accord tuteur**.
- [ ] 🟠 **[simple] — Cycle de vie des locations (simulé).** Renouvellement simulé (réémission d'un ordre de paiement), rappel avant échéance, résiliation en fin de période (et non immédiate si déjà réglée), statuts (actif, en retard, expiré, résilié). Les transitions différées sont **déclenchables manuellement** ou par script planifié.
- [ ] 🟠 **[moyen] — Provisioning simulé des serveurs (abstraction d'intégration).** Modéliser un cycle de vie de serveur (`creating → running → suspended → deleted`) stocké en base, derrière une interface `ProvisioningServiceInterface` et une implémentation `SimulatedProvisioner` qui génère l'adresse de connexion. **Aucun panel, aucun serveur de jeu lancé** ; l'interface reste prête à recevoir une vraie API d'hébergeur plus tard, sans être implémentée. C'est **la** fonctionnalité qui distingue une véritable modélisation métier d'une simple maquette.
- [ ] 🟠 **[complexe] — Back-office d'administration** : gestion du catalogue/plans/stock/clients/locations, recherche, filtres, **journal d'actions** et **rôles/permissions (RBAC)** (une colonne `role` sur `users` suffit). Indispensable pour une exploitation professionnelle, même en démonstration.
- [ ] 🟡 **[moyen] — Notifications.** E-mails transactionnels (échéance, reçu, incident) et préférences par utilisateur, envoyés par le `Mailer` existant ; **sans clé Mailjet, le contenu est journalisé** — donc testable sans boîte mail.
- [ ] 🟡 **[moyen] — Support client** : tickets ou formulaire de contact tracé en base, avec historique par client et délai de réponse visé.
- [ ] 🟡 **[moyen] — Tableau de bord d'activité** : volume de locations, taux de conversion, taux de résiliation (churn) et locations actives — calculés par de simples requêtes SQL d'agrégation sur les données simulées.
- [ ] 🟡 **[simple] — Statut simulé des serveurs** (états du cycle de vie provisionné) et page de statut **consultable en local**.
- [ ] 🟡 **[moyen] — API authentifiée par clé (testable en local).** Clés d'API stockées en base et **quota par clé** (compteur), pour qu'un client externe (script, `curl` ou Postman) consomme l'API. Aucune inscription de partenaire réelle.
- [ ] 🟡 **[simple] — Relances d'échéance.** Script PHP envoyant les rappels J-7 et J-1, **exécuté à la main ou planifié** ; envois via le `Mailer` (avec repli en journal).

**Critère d'acceptation :** le parcours complet (location → paiement simulé → reçu → cycle de vie simulé du serveur → résiliation) se déroule **de bout en bout sur un WAMP**, déclenché par des actions utilisateur ou un script lancé localement, **sans aucun service externe payant, sans transaction financière, sans serveur de jeu réel et sans nouvelle dépendance**.

---

## 11. Documentation & dossier de projet 🟠

Votre documentation technique est déjà d'un très bon niveau ; poussez-la et complétez-la.

- [ ] 🔴 **[simple] — Compléter les `TODO` implicites** : mettez à jour §7 (API), §8 (BDD) et §16 (Limites) au fur et à mesure des correctifs de §1.
- [ ] 🟠 **[simple] — `README.md` autonome** : description, prérequis, installation pas à pas, variables de config, commandes, licence. (Aujourd'hui l'installation vit dans la doc technique.)
- [ ] 🟠 **[simple] — Diagrammes complémentaires** : schéma d'architecture, MCD/MLD de la base (§8 a déjà un `erDiagram` — ajouter un MLD relationnel), diagrammes de séquence (déjà bons en §4/§9), cycle d'une requête versionné.
- [ ] 🟠 **[simple] — Guide utilisateur** (louer, gérer, résilier) et **guide administrateur** (back-office, support, modération).
- [ ] 🟠 **[simple] — `CHANGELOG.md`** (Keep a Changelog) + versionnage SemVer de Spawnly.
- [ ] 🟡 **[simple] — `CONTRIBUTING.md`**, `LICENSE`, `SECURITY.md` (procédure de signalement).
- [ ] 🟡 **[simple] — Dossier portfolio** : contexte, besoins, choix techniques justifiés, difficultés résolues (le débordement de date, la survente, l'anti-énumération…), indicateurs de qualité. Précieux pour votre poursuite d'études.

---

## 12. Séquencement recommandé (jalons)

| Jalon | Contenu | Priorités | Complexité |
|---|---|---|---|
| **J1 — Robustesse** | Correctifs §1 (transaction stock, dates, fuseaux, cookie, anti-brute-force) | §1 | moyen |
| **J2 — Socle qualité** | Lint, PHPStan, PHPUnit, CI, en-têtes, CSRF, Argon2id | §2, §3, §4, §6 | complexe |
| **J3 — Front & API pro** | Build Tailwind local, progressive enhancement, SEO/a11y, versionnement API, pagination | §5, §7 | complexe |
| **J4 — Exploitation locale** | Config dev/prod, en-têtes, logs, sauvegardes/restauration, checklist de mise en ligne (documentation) | §9 | moyen |
| **J5 — Conformité** | CGV, confidentialité, RGPD, suppression de compte, purges | §8 | moyen |
| **J6 — Modélisation métier** | Simulation de paiement, provisioning simulé, reçus, back-office, notifications | §10 | complexe |
| **J7 — Rayonnement** | Documentation complète, guides, dossier portfolio | §11 | moyen |

---

## 13. Indicateurs de réussite

- [ ] 🔴 Section « Limites connues » vidée de tout point 🔴/🟠 non traité ou reporté avec justification.
- [ ] 🔴 Zéro secret dans le dépôt ; scan de vulnérabilités sans faille haute ; CSP active.
- [ ] 🟠 Couverture de tests ≥ 60 % sur le cœur métier ; CI verte ; tests sur transactions et dates.
- [ ] 🟠 Lighthouse ≥ 90 (Performance, Accessibilité, SEO) sur les pages clés.
- [ ] 🟠 Parcours complet testé : inscription → **paiement simulé** → **provisioning simulé** → reçu → résiliation (sans transaction ni serveur réel).
- [ ] 🟠 Réinstallation sur un WAMP vierge reproductible, en suivant uniquement la checklist documentée.
- [ ] 🟡 Page de statut publique + tableau de bord d'activité opérationnels.

---

Bon courage, Olivier. Votre savoir-faire technique n'est pas en cause : vous maîtrisez déjà PHP natif, PDO, la gestion de session, le front sans framework. Ce qui vous manque pour passer au niveau professionnel relève de **l'ingénierie** (tests, lint, transactions, exploitation locale, conformité) et de **la profondeur métier** (simulation de paiement, provisioning simulé, back-office). Traitez les jalons dans l'ordre, documentez chaque décision, et vous transformerez une excellente application d'étudiant en une architecture digne d'un véritable hébergeur — **qui continue de tourner sur WAMP**.

*Feuille de route établie à partir de la documentation complète fournie par l'étudiant (`DOCUMENTATION.md`, 16 sections) et de ses propres règles (`AGENTS.md`). Cadre d'exécution : **WAMP en local (TP)** — aucun serveur web réel, VPS ni Docker requis. Les points marqués « sous accord tuteur » introduisent une dépendance externe et sont à valider au regard du principe KISS.*
