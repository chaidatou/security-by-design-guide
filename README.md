# Security by Design — Guide de sécurité du code pour applications et SaaS

Un guide de sécurité **prêt à intégrer dans n'importe quel projet**, à placer dans le dépôt dès le premier commit pour que la sécurité soit prise en compte **avant** d'écrire le code, et non corrigée après coup.

Il est pensé pour être lu par les développeurs **et** par les assistants IA de code (Claude Code, Cursor, Copilot), qui appliquent les règles automatiquement.

## Pourquoi ce guide ?

En audit et en test d'intrusion, on retrouve presque toujours les mêmes failles : contrôle d'accès oublié, jetons stockés dans le `localStorage`, secrets publiés sur GitHub, mode debug actif en production, données d'un client visibles par un autre…

Ces failles coûtent très peu à éviter au début du projet, et très cher à corriger une fois l'application en ligne. Ce guide transforme ces constats de terrain en règles claires, applicables dès la première ligne de code.

## Ce qu'il couvre

| # | Thème |
|---|---|
| 0 | Profil du projet + instructions pour les assistants IA |
| 1 | Authentification (hachage, force brute, 2FA) |
| 2 | Sessions et jetons JWT |
| 3 | Autorisation et contrôle d'accès (IDOR, assignation de masse) |
| 4 | Validation des entrées, injections SQL/NoSQL, XSS, CSRF |
| 5 | Gestion des secrets |
| 6 | Communication et API (CORS, rate limiting, SSRF) |
| 7 | En-têtes de sécurité HTTP |
| 8 | Upload de fichiers |
| 9 | Protection des données personnelles |
| 10 | Spécificités SaaS multi-tenant |
| 11 | Applications mobiles |
| 12 | Dépendances et CI |
| 13 | Journalisation et surveillance |
| 14 | Configuration du serveur |
| 15 | Check-lists avant fusion et avant mise en production |

## Adaptable à votre stack

Chaque section contient un tableau d'adaptation avec les fonctions et outils concrets pour :

**PHP natif · Laravel · Django · Node.js (Express / NestJS) · Next.js · Spring Boot · ASP.NET Core · Flutter · React Native**

## Utilisation

1. Copiez [`SECURITY-GUIDELINES.md`](./SECURITY-GUIDELINES.md) à la racine de votre projet.
2. Remplissez la **section 0** (stack du projet).
3. Si vous codez avec un assistant IA, ajoutez cette ligne dans votre `CLAUDE.md`, `AGENTS.md` ou `.cursorrules` :
   ```
   Respecte toutes les règles de SECURITY-GUIDELINES.md pour tout code généré.
   ```
4. Utilisez les check-lists de la section 15 avant chaque fusion et avant chaque mise en production.

## Référentiels

OWASP Top 10 · OWASP ASVS · OWASP API Security Top 10 · OWASP MASVS

## Contribuer

Les suggestions sont les bienvenues : ouvrez une *issue* ou une *pull request* pour proposer une règle, corriger une erreur ou ajouter un framework.

## Auteure

**Larissa Chaidatou Koinda** — Cybersécurité (audit, pentest, sécurisation d'applications)
[LinkedIn](https://linkedin.com/in/larissa-koinda) · koindalarissa@gmail.com

## Licence

[CC BY 4.0](./LICENSE) — libre d'utilisation, de modification et de partage, y compris en contexte commercial, à condition de citer l'auteure.
