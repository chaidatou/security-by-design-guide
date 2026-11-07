# Guide de sécurité du code — Security by Design

> **À lire avant d'écrire la première ligne de code.**
> Ce fichier définit les règles de sécurité obligatoires du projet. Il s'applique à tous les développeurs **et à tous les assistants IA** (Claude Code, Cursor, Copilot…) qui génèrent du code pour ce dépôt.

| | |
|---|---|
| **Version** | 1.0 — novembre 2026 |
| **Auteure** | Larissa Chaidatou Koinda — Cybersécurité |
| **Contact** | koindalarissa@gmail.com |
| **Référentiels** | OWASP Top 10, OWASP ASVS, OWASP API Security Top 10, OWASP MASVS (mobile) |

---

## 0. Profil du projet (à remplir au démarrage)

Ce guide est universel. Remplissez ce tableau une fois, puis appliquez les lignes « Adaptation par stack » de chaque section qui correspondent à votre choix.

| Élément | Choix du projet |
|---|---|
| Langage / framework backend | _ex. Laravel, Django, Express, NestJS, Spring Boot, ASP.NET Core, PHP natif_ |
| Frontend | _ex. React, Vue, Next.js, Blade, templates serveur_ |
| Application mobile | _ex. aucune, Flutter, React Native, Kotlin, Swift_ |
| Base de données | _ex. PostgreSQL, MySQL, MongoDB_ |
| Authentification | _ex. sessions + cookies, JWT, OAuth/SSO_ |
| Hébergement | _ex. VPS, AWS, mutualisé, Vercel_ |
| Serveur web / proxy | _ex. Nginx, Apache, Caddy, géré par l'hébergeur_ |
| Type d'application | _ex. app mono-client, SaaS multi-clients (multi-tenant)_ |
| Données sensibles traitées | _ex. paiement, localisation, santé, pièces d'identité_ |

### Instruction pour les assistants IA

Si vous êtes un assistant IA qui génère du code pour ce projet :
1. Lisez la section 0 pour connaître la stack, puis appliquez les règles de ce fichier **sans attendre qu'on vous le demande**.
2. Ne désactivez jamais une protection (CSRF, validation, vérification TLS, contrôle d'accès) pour « faire marcher » un bout de code. Signalez le problème à la place.
3. Ne mettez jamais de secret en dur dans le code, même pour un exemple ou un test.
4. Quand une règle ne peut pas être respectée, écrivez un commentaire `// SECURITY-TODO:` expliquant pourquoi.

---

## Principes fondamentaux

1. **Ne jamais faire confiance au client.** Tout ce qui vient du navigateur, de l'appli mobile ou d'une API externe peut être modifié : paramètres, en-têtes, cookies, champs cachés, prix, rôles, identifiants.
2. **Refus par défaut.** Un utilisateur n'a accès qu'à ce qui lui est explicitement autorisé. Une route sans contrôle d'accès est une faille.
3. **Moindre privilège.** Chaque compte, service, clé API ou utilisateur de base de données a uniquement les droits dont il a besoin.
4. **Paramètres sûrs par défaut.** Les options de confidentialité (partage de position, profil public…) sont désactivées par défaut et activées avec le consentement de l'utilisateur.
5. **Défense en profondeur.** Une seule protection ne suffit jamais : validation + requête préparée + droits limités en base.
6. **Échouer en sécurité.** En cas d'erreur, l'application refuse l'accès et ne révèle aucun détail technique.

---

## 1. Authentification

**Risque :** vol de comptes par force brute, fuite de mots de passe réutilisables, prise de contrôle d'un compte administrateur.

**Règles :**
- Mots de passe hachés avec **Argon2id** (recommandé) ou **bcrypt** (coût ≥ 10). Jamais MD5, SHA-1, SHA-256 simple ni chiffrement réversible.
- Longueur minimale de 8 caractères (12 recommandé), pas de longueur maximale inférieure à 64. Refuser les mots de passe connus comme compromis si possible.
- Limiter les tentatives de connexion (ex. 5 essais puis blocage temporaire ou captcha), par compte **et** par IP.
- Message d'erreur identique que l'e-mail existe ou non : « Identifiants incorrects ».
- Réinitialisation de mot de passe : jeton aléatoire, à usage unique, expiration courte (15 à 60 min), stocké haché.
- **Double authentification (2FA)** obligatoire pour les comptes administrateurs, proposée à tous les utilisateurs.
- Après connexion ou changement de mot de passe : régénérer l'identifiant de session et invalider les autres sessions si demandé.

**Adaptation par stack :**

| Stack | Hachage | Limitation des tentatives |
|---|---|---|
| PHP natif | `password_hash($mdp, PASSWORD_ARGON2ID)` / `password_verify()` | Compteur en base ou Redis |
| Laravel | `Hash::make()` (configurer `argon2id` dans `config/hashing.php`) | Middleware `throttle`, `RateLimiter` |
| Django | Ajouter `Argon2PasswordHasher` en tête de `PASSWORD_HASHERS` | `django-axes` |
| Node.js (Express / NestJS) | Paquet `argon2` ou `bcrypt` | `express-rate-limit`, `@nestjs/throttler` |
| Next.js | Idem Node.js, ou Auth.js / fournisseur d'identité | Rate limit dans les route handlers |
| Spring Boot | `Argon2PasswordEncoder` ou `BCryptPasswordEncoder` | Bucket4j, filtre Spring Security |
| ASP.NET Core | ASP.NET Core Identity (PBKDF2 intégré) | `AddRateLimiter()`, lockout d'Identity |

---

## 2. Sessions et jetons (JWT)

**Risque :** vol de session via XSS, réutilisation d'un jeton volé, session qui ne meurt jamais.

**Règles :**
- **Ne jamais stocker un jeton de session ou d'accès dans `localStorage` ou `sessionStorage`** : n'importe quel script injecté (XSS) peut le lire.
- Utiliser des cookies avec les attributs : `HttpOnly`, `Secure`, `SameSite=Lax` (ou `Strict`), `Path=/`.
- Expiration : session inactive fermée après un délai raisonnable (ex. 30 min pour une app sensible, quelques jours maximum avec « se souvenir de moi »).
- Déconnexion = invalidation **côté serveur**, pas seulement suppression du cookie.

**Si JWT :**
- Jeton d'accès court (5 à 15 min) + jeton de rafraîchissement avec rotation et révocation.
- Fixer l'algorithme attendu côté serveur (ex. `HS256` ou `RS256`) ; refuser `alg: none`.
- Ne jamais mettre de données sensibles dans le payload (il est seulement encodé en base64, pas chiffré).
- Secret de signature long et aléatoire (≥ 256 bits), stocké dans les variables d'environnement.

**Mobile :** stocker les jetons dans le stockage sécurisé du système (`flutter_secure_storage`, `expo-secure-store` / `react-native-keychain`, Keychain iOS, EncryptedSharedPreferences / Keystore Android), jamais dans `SharedPreferences` ou `AsyncStorage` en clair.

---

## 3. Autorisation et contrôle d'accès

**Risque :** c'est la faille n°1 de l'OWASP. Un utilisateur change `?id=124` en `?id=125` et lit les données d'un autre (IDOR), ou appelle une route admin sans être admin.

**Règles :**
- **Chaque** route, contrôleur, action serveur et endpoint d'API vérifie : (1) l'utilisateur est connecté, (2) il a le droit d'agir **sur cette ressource précise**.
- Vérifier la propriété de la ressource côté serveur :
  ```
  // Mauvais : on fait confiance à l'id envoyé
  commande = Commande.find(id)

  // Bon : on filtre par le propriétaire connecté
  commande = Commande.find_where(id = id, user_id = utilisateur_connecte.id)
  ```
- Ne jamais lire le rôle ou l'identifiant utilisateur depuis le corps de la requête ou un champ caché : il vient de la session ou du jeton vérifié.
- Protéger contre l'**assignation de masse** : n'accepter que les champs autorisés (un utilisateur ne doit pas pouvoir envoyer `role=admin` ou `is_verified=true`).
- Masquer un bouton dans l'interface n'est **pas** une protection. Le contrôle se fait sur le serveur.
- Préférer des identifiants non devinables (UUID) pour les ressources exposées, **en plus** du contrôle d'accès (jamais à la place).

**Adaptation par stack :**

| Stack | Contrôle d'accès | Assignation de masse |
|---|---|---|
| Laravel | Policies et Gates, `$this->authorize()` | `$fillable` sur chaque modèle, jamais `$guarded = []` |
| Django | `permission_required`, permissions DRF, filtrer les querysets par `request.user` | `fields` explicites dans les serializers/forms (jamais `__all__`) |
| Express / NestJS | Middleware d'autorisation, Guards NestJS | Schéma de validation (Zod, Joi, `class-validator` avec `whitelist: true`) |
| Next.js | Vérifier la session **dans chaque** route handler et Server Action — le middleware seul ne suffit pas | Valider les entrées avec Zod |
| Spring Boot | `@PreAuthorize`, Spring Security | DTO dédiés, jamais l'entité JPA directement |
| ASP.NET Core | `[Authorize]`, policies | DTO / ViewModels, `[Bind]` |

---

## 4. Validation des entrées et injections

**Risque :** injection SQL (vol de toute la base), XSS (vol de sessions), injection de commandes système.

**Règles générales :**
- Valider **côté serveur** toute entrée : type, longueur, format, valeurs autorisées (liste blanche). La validation côté client sert seulement au confort.
- Rejeter ce qui ne correspond pas plutôt que d'essayer de « nettoyer ».

**Injection SQL / NoSQL :**
- Toujours des **requêtes préparées** ou l'ORM. Jamais de concaténation de chaînes dans une requête.
  ```
  // Interdit
  "SELECT * FROM users WHERE email = '" + email + "'"

  // Correct (requête préparée)
  "SELECT * FROM users WHERE email = ?"  avec  [email]
  ```
- Attention aux méthodes « brutes » des ORM : `DB::raw()` (Laravel), `.raw()` / `.extra()` (Django), `$queryRawUnsafe` (Prisma), `createNativeQuery` avec concaténation (JPA).
- MongoDB : refuser les objets là où on attend une chaîne (bloque les injections du type `{"$ne": null}`).

**XSS :**
- Laisser le moteur de templates échapper automatiquement : `{{ }}` (Blade, Twig, Jinja/Django), JSX (React/Vue).
- Interdire sauf justification écrite : `{!! !!}` (Blade), `|safe` / `mark_safe` (Django), `dangerouslySetInnerHTML` (React), `v-html` (Vue), `innerHTML` (JS).
- Si du HTML utilisateur est nécessaire (éditeur de texte riche) : le nettoyer avec **DOMPurify** (front) ou une bibliothèque équivalente côté serveur.

**CSRF :**
- Protection active sur toutes les requêtes qui modifient des données (POST, PUT, PATCH, DELETE).
- Laravel : `@csrf` ; Django : `{% csrf_token %}` ; ASP.NET : antiforgery ; Spring Security : activé par défaut ; Express : cookies `SameSite` + jeton (paquet `csrf-csrf`) ; Next.js : les Server Actions vérifient l'origine, les route handlers doivent être protégés manuellement.
- Ne jamais désactiver la protection CSRF globalement pour « débloquer » un formulaire.

**Commandes système :**
- Éviter d'appeler le shell. Si nécessaire : fonctions qui prennent les arguments sous forme de liste (`subprocess.run([...])` sans `shell=True`, `execFile` au lieu de `exec`), jamais de saisie utilisateur dans une chaîne de commande.

---

## 5. Gestion des secrets

**Risque :** une clé API ou un mot de passe de base de données publié sur GitHub est exploité en quelques minutes par des robots.

**Règles :**
- Aucun secret dans le code source, les commits, les captures d'écran, les tickets ou les messages.
- Secrets dans des variables d'environnement (`.env`) ou un gestionnaire de secrets (AWS Secrets Manager, Vault, secrets de la CI).
- `.env` dans `.gitignore` **dès le premier commit**. Fournir un `.env.example` sans valeurs réelles.
- Permissions du fichier `.env` sur le serveur : `chmod 600`, propriétaire = utilisateur de l'application.
- Clés différentes pour développement, test et production.
- Un secret publié par erreur est considéré comme **compromis** : on le révoque et on le remplace (le supprimer du dépôt ne suffit pas, il reste dans l'historique Git).
- Activer un scanner de secrets : `gitleaks` en pre-commit et dans la CI, ou GitHub secret scanning.

**Piège frontend :** tout ce qui est envoyé au navigateur ou embarqué dans une appli mobile est **public**.
- Next.js : les variables `NEXT_PUBLIC_*` sont visibles par tous. Vite : `VITE_*`. Create React App : `REACT_APP_*`.
- Une appli mobile peut être décompilée : aucune clé secrète dedans. Les appels sensibles passent par votre backend.

---

## 6. Communication et API

**Risque :** interception des données, abus de l'API par des robots, fuite d'informations par les messages d'erreur.

**Règles :**
- **HTTPS partout**, redirection automatique HTTP → HTTPS, TLS 1.2 minimum.
- **CORS strict** : liste explicite des domaines autorisés. Jamais `Access-Control-Allow-Origin: *` sur une API qui utilise des cookies ou des jetons.
- **Limitation de débit** (rate limiting) sur toutes les routes, renforcée sur : connexion, inscription, mot de passe oublié, envoi d'e-mails/SMS, formulaires publics.
- Formulaires publics : protection anti-robots (Cloudflare Turnstile, hCaptcha, reCAPTCHA).
- Pagination obligatoire avec une taille maximale (ex. 100 éléments), pour empêcher l'extraction massive.
- Réponses API : ne renvoyer que les champs nécessaires (jamais l'objet complet de la base avec le hash du mot de passe, les tokens, etc.).
- Erreurs : message générique côté client (`"Une erreur est survenue"` + identifiant de référence), détails uniquement dans les logs serveur.
- **Mode debug désactivé en production** : `APP_DEBUG=false` (Laravel), `DEBUG = False` (Django), `NODE_ENV=production`, `display_errors=Off` (PHP).
- Documentation d'API (Swagger, GraphQL introspection) désactivée ou protégée en production.
- Webhooks entrants : vérifier la signature de l'expéditeur.
- Requêtes vers des URL fournies par l'utilisateur (aperçu de lien, import d'image) : bloquer les adresses internes (`127.0.0.1`, `10.x`, `192.168.x`, `169.254.169.254`) pour éviter le SSRF.

---

## 7. En-têtes de sécurité HTTP

**Risque :** clickjacking, exécution de scripts injectés, fuite d'informations.

**En-têtes obligatoires :**

| En-tête | Valeur de départ recommandée |
|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` |
| `Content-Security-Policy` | `default-src 'self'; frame-ancestors 'none'; object-src 'none'; base-uri 'self'` (à ajuster selon les CDN et services utilisés) |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` (activer seulement ce qui est utilisé) |

**À supprimer :** `X-Powered-By`, numéro de version dans `Server`.

**Adaptation par stack :**

| Stack | Où configurer |
|---|---|
| Nginx | `add_header ... always;` dans le bloc `server`, `server_tokens off;` |
| Apache | `Header always set ...`, `ServerTokens Prod`, `ServerSignature Off` |
| Express | Paquet `helmet`, `app.disable('x-powered-by')` |
| Next.js | `headers()` dans `next.config.js`, `poweredByHeader: false` |
| Laravel | Middleware dédié ou config du serveur web |
| Django | `SECURE_HSTS_SECONDS`, `SECURE_CONTENT_TYPE_NOSNIFF`, `X_FRAME_OPTIONS`, `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`, paquet `django-csp` |
| Spring Boot | `http.headers(...)` dans Spring Security |
| ASP.NET Core | Middleware personnalisé, `app.UseHsts()` |

Vérifier le résultat sur [securityheaders.com](https://securityheaders.com).

---

## 8. Upload de fichiers

**Risque :** envoi d'un script (`.php`, `.jsp`) exécuté par le serveur, fichiers géants qui saturent le disque.

**Règles :**
- Liste blanche d'extensions **et** vérification du type réel (contenu du fichier, pas seulement l'extension ni le `Content-Type` envoyé).
- Taille maximale définie côté serveur.
- Renommer le fichier avec un nom aléatoire généré par le serveur.
- Stocker **hors du dossier public** ou sur un stockage objet (S3 & co.), servi via une route contrôlée ou une URL signée temporaire.
- Le dossier d'upload ne doit jamais pouvoir exécuter de code.
- Images : les ré-encoder (supprime aussi les métadonnées EXIF, dont la position GPS).
- Analyse antivirus (ClamAV) si les fichiers sont partagés entre utilisateurs.

---

## 9. Protection des données personnelles

**Risque :** fuite de données, exposition de la localisation ou de l'identité des utilisateurs, non-conformité légale (loi algérienne 18-07, RGPD pour les utilisateurs européens, lois nationales selon les pays visés).

**Règles :**
- **Minimisation :** ne collecter que ce qui est nécessaire.
- **Confidentialité par défaut :** profil, position, coordonnées non partagés tant que l'utilisateur ne l'a pas choisi.
- Consentement explicite avant toute collecte sensible (géolocalisation, contacts, caméra), au moment où on en a besoin.
- **Géolocalisation :** ne pas exposer de coordonnées exactes aux autres utilisateurs ; arrondir ou afficher une zone approximative.
- Chiffrement des données sensibles au repos (champs critiques, sauvegardes) et en transit (HTTPS, connexion BDD chiffrée si distante).
- La politique de confidentialité publiée doit correspondre à ce que le code fait réellement.
- Permettre à l'utilisateur de supprimer son compte et ses données.
- Sauvegardes chiffrées, testées régulièrement (restauration réelle), stockées hors du serveur principal.

---

## 10. Spécificités SaaS (multi-tenant)

À appliquer si plusieurs entreprises ou organisations clientes partagent la même application.

**Risque :** un client voit les données d'un autre client. C'est l'incident le plus grave pour un SaaS : il détruit la confiance de tous les clients.

**Règles :**
- Chaque table contenant des données client a une colonne `tenant_id` (ou `organization_id`).
- **Toutes** les requêtes sont filtrées par le tenant de l'utilisateur connecté, de façon automatique (scope global de l'ORM, middleware) plutôt que manuelle.
  - Laravel : Global Scope ; Django : Manager personnalisé ; Prisma : extension de client ; PostgreSQL : **Row Level Security (RLS)** ; Supabase : RLS obligatoire sur chaque table.
- Le `tenant_id` vient de la session, jamais de la requête.
- Fichiers stockés avec un préfixe par tenant (`tenant-42/factures/...`) et accès vérifié.
- Caches et files d'attente : la clé inclut le tenant.
- Tests automatisés qui vérifient qu'un utilisateur du tenant A ne peut **rien** lire ni modifier du tenant B.
- Rôles au sein d'une organisation (propriétaire, admin, membre, lecture seule) gérés côté serveur.
- Journal d'audit des actions sensibles (changement de rôle, export, suppression) consultable par l'admin du client.
- Facturation : le plan et les limites sont vérifiés côté serveur, jamais d'après une valeur envoyée par le client.

---

## 11. Applications mobiles

**Règles (OWASP MASVS) :**
- Aucun secret embarqué dans l'application (elle peut être décompilée).
- Jetons dans le stockage sécurisé du système (voir section 2).
- HTTPS uniquement : pas d'exception `cleartextTrafficPermitted` (Android) ni `NSAllowsArbitraryLoads` (iOS) en production.
- Ne jamais désactiver la vérification des certificats TLS, même temporairement.
- Le backend applique **toutes** les règles : l'appli mobile est un client comme un autre, contrôlable par un attaquant.
- Désactiver les logs de débogage et activer l'obfuscation en build de production (R8/ProGuard, `--obfuscate` Flutter).
- Liens profonds (deep links) : valider les paramètres reçus.
- Permissions : demander uniquement celles qui servent, au moment où elles servent.

---

## 12. Dépendances et chaîne d'intégration

**Risque :** une bibliothèque vulnérable ou malveillante compromet toute l'application.

**Règles :**
- Fichier de verrouillage versionné (`package-lock.json`, `composer.lock`, `poetry.lock`, `pubspec.lock`…).
- Audit automatique des vulnérabilités dans la CI :

| Écosystème | Commande |
|---|---|
| npm / Node | `npm audit` |
| Composer / PHP | `composer audit` |
| Python | `pip-audit` |
| Java | OWASP Dependency-Check |
| .NET | `dotnet list package --vulnerable` |
| Multi-langages | Dependabot, Trivy, Snyk |

- Vérifier le nom exact d'un paquet avant de l'installer (paquets pièges aux noms proches).
- Mettre à jour régulièrement ; les correctifs de sécurité critiques sont appliqués sous quelques jours.
- Analyse statique du code (SAST) dans la CI : Semgrep (multi-langages), Bandit (Python), PHPStan / Psalm (PHP).
- Les secrets de la CI sont accessibles uniquement aux branches protégées.
- Branche `main` protégée : revue de code obligatoire avant fusion.

---

## 13. Journalisation et surveillance

**Risque :** une attaque passe inaperçue pendant des mois ; ou les logs eux-mêmes deviennent une fuite de données.

**À journaliser :** connexions réussies et échouées, changements de mot de passe et de rôle, accès refusés, erreurs serveur, actions administrateur, exports de données.

**À ne jamais journaliser :** mots de passe, jetons, clés API, numéros de carte, contenu complet des requêtes contenant des données personnelles.

**Règles :**
- Chaque entrée contient : horodatage, identifiant utilisateur, IP, action, résultat.
- Logs stockés hors de la portée de l'application (pour qu'un attaquant ne puisse pas les effacer) et conservés selon une durée définie.
- Alertes sur les comportements anormaux : pics d'échecs de connexion, nombreuses erreurs 403/404, accès admin inhabituels.
- Outils possibles : fail2ban, Wazuh, CloudWatch, Sentry (en masquant les données sensibles).

---

## 14. Configuration du serveur (si auto-hébergé)

- Système et paquets à jour, mises à jour de sécurité automatiques (`unattended-upgrades`).
- SSH : authentification par clé uniquement, `PermitRootLogin no`, fail2ban actif.
- Pare-feu : seuls les ports nécessaires ouverts (ex. 22, 80, 443) ; la base de données n'est **jamais** exposée sur Internet.
- L'application tourne avec un utilisateur dédié sans droits root.
- Utilisateur de base de données propre à l'application, avec uniquement les droits nécessaires.
- Si un proxy comme Cloudflare est utilisé : n'accepter le trafic web que depuis ses plages d'IP, pour ne pas exposer l'IP réelle du serveur.
- Fichiers sensibles inaccessibles depuis le web : `.env`, `.git/`, sauvegardes, `composer.json`, fichiers de logs.

---

## 15. Check-lists

### Avant chaque fusion (pull request)
- [ ] Toute nouvelle route vérifie l'authentification **et** l'autorisation sur la ressource
- [ ] Les entrées sont validées côté serveur
- [ ] Aucune requête construite par concaténation
- [ ] Aucun secret ajouté au code
- [ ] Aucun échappement HTML désactivé sans justification
- [ ] Aucune donnée sensible dans les logs ou les réponses d'API
- [ ] (SaaS) Les nouvelles requêtes sont filtrées par tenant
- [ ] L'audit des dépendances et le scan de secrets passent

### Avant la mise en production
- [ ] Mode debug désactivé, erreurs génériques
- [ ] HTTPS forcé, HSTS actif
- [ ] En-têtes de sécurité vérifiés (note A minimum sur securityheaders.com)
- [ ] Cookies `HttpOnly`, `Secure`, `SameSite`
- [ ] CORS limité aux domaines de production
- [ ] Rate limiting actif sur les routes sensibles
- [ ] 2FA activée sur tous les comptes admin (application, hébergeur, registrar, dépôt Git)
- [ ] Secrets de production différents de ceux de développement
- [ ] Sauvegardes configurées et restauration testée
- [ ] Logs et alertes en place
- [ ] Documentation d'API et routes de test désactivées
- [ ] Test d'intrusion ou audit de sécurité réalisé

---

## 16. Signaler une vulnérabilité

Toute faille découverte (par l'équipe, un utilisateur ou un chercheur) est signalée à : **[adresse e-mail sécurité du projet]**.
Ne jamais ouvrir de ticket public décrivant une faille non corrigée.

---

*Ce document évolue avec le projet. Toute exception à une règle doit être validée par la personne responsable de la sécurité et documentée.*
