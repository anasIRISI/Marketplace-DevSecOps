# Cahier des charges individuel — ELKHANFAF Mohamed

**Domaine : Back-end Express.js et sécurité des API**  
**Projet : Secure Marketplace — AI-Augmented DevSecOps**  
**Établissement : FST Marrakech, Université Cadi Ayyad — Cycle ingénieur IRISI 3**  
**Module : Sécurité des applications et des services**  
**Sujet officiel : Projet 3 — Conception et mise en œuvre d'une pipeline DevSecOps augmentée par IA pour les applications de l'écosystème JavaScript (Express.js).**

Statut : spécification prévisionnelle. Aucun endpoint, schéma, contrôle ou test n'est déclaré opérationnel. Les chemins, modèles et seuils proposés devront être stabilisés avec l'équipe avant implémentation ; aucun code ni fichier de configuration n'est produit par ce document.

## 1. Présentation du rôle

Mohamed possède le socle Express.js, le modèle PostgreSQL, les migrations, les contrats REST et la politique d'accès serveur. Sa mission est de fournir des opérations métier utilisables par Web et mobile, tout en appliquant systématiquement authentification, autorisation, validation et limitation des ressources.

Il est responsable de la sécurité des lectures comme des écritures, y compris lorsqu'un client contourne son interface. Il ne doit faire confiance ni aux identifiants de propriétaire fournis par un client, ni aux rôles présents dans un formulaire, ni aux propositions d'un LLM.

Responsabilités partagées : Anas conçoit le protocole et les scénarios de messagerie, Mohamed garantit sa persistance et ses contrôles serveur ; Abdelilah conçoit le module IA, Mohamed en revoit l'intégration et les permissions ; Hanane et Anas consomment les contrats validés. Tous les membres participent aux tests, à la sécurité et aux corrections de leur partie.

## 2. Technologies prévues et justification

| Technologie | Rôle et justification |
|---|---|
| Node.js, Express.js, TypeScript | API modulaire, middlewares explicites et contrats typés ; fixer une version maintenue lors de l'implémentation. |
| PostgreSQL | Intégrité relationnelle, transactions, contraintes et index pour les annonces et conversations. |
| Prisma | Accès typé aux données, schéma et migrations traçables ; l'ORM ne dispense pas du contrôle d'accès. |
| Zod | Validation des paramètres, corps, filtres et champs autorisés aux frontières. |
| Helmet | Ensemble d'en-têtes de protection à adapter et tester selon les clients et l'hébergement. |
| CORS | Politique d'origines Web explicite ; ce mécanisme ne remplace ni l'authentification ni les autorisations. |
| Supertest | Tests HTTP du contrat, des erreurs et des accès interdits sur une application de test. |
| Bibliothèque de hachage de mots de passe à sélectionner | Prévoir Argon2id avec paramètres justifiés et mesurés ; ne jamais stocker un mot de passe en clair. |
| Bibliothèque JWT, rate limiter et journalisation structurée à sélectionner | Validation stricte des jetons, quotas et corrélation des événements avec masquage des secrets. |
| Socket.IO | Transport temps réel partagé avec Anas ; contrôles et persistance hébergés dans le serveur. |
| OpenAPI, moteur de tests à sélectionner | Contrat des endpoints, exemples anonymisés et tests reproductibles. |

Aucune version précise ni dépendance complémentaire ne sera présumée acquise ; Abdelilah et Mohamed devront consigner les choix de maintenance et les limites.

## 3. Architecture technique et modèle de données

### 3.1 Architecture modulaire

Chaîne HTTP cible : identifiant de corrélation → journalisation expurgée → en-têtes/CORS → limite du corps → quotas → authentification si requise → validation → autorisation → contrôleur → service métier → accès Prisma → réponse DTO. Gestionnaire d'erreurs central et traitement des routes inconnues en fin de chaîne.

Modules : `auth`, `users`, `products`, `conversations`, `messages`, `realtime` et `assistant`. Les services métier doivent porter les règles réutilisées entre REST, Socket.IO et outils IA. Les contrôleurs gèrent le transport ; les accès aux données doivent rendre explicites les filtres d'utilisateur et de visibilité.

Frontières : clients non fiables/Express ; Express/PostgreSQL ; transport Socket.IO/services ; module IA/fournisseur externe. Ne pas ouvrir PostgreSQL aux clients et ne pas donner au LLM un accès SQL direct.

### 3.2 Modèles PostgreSQL/Prisma proposés

Une annonce et un produit sont une même entité `Product`. Les attributs suivants constituent le contrat initial à faire valider.

| Modèle | Champs et relations prévus | Contraintes et exposition |
|---|---|---|
| User | Identifiant opaque, courriel normalisé, nom public, empreinte du mot de passe, rôle `USER` ou `ADMIN`, état du compte, dates | Courriel unique ; rôle attribué côté serveur ; empreinte et courriel exclus du DTO vendeur public. |
| Session | Identifiant, utilisateur, empreinte du jeton de renouvellement, famille de rotation, expiration, révocation et dates | Jeton opaque aléatoire, empreinte seule en base ; session active exigée pour les accès protégés ; index utilisateur/expiration. |
| Category | Identifiant, libellé, état actif | Catalogue administré ou jeu de référence contrôlé ; clé étrangère depuis Product. |
| Product | Identifiant, vendeur, catégorie, titre, description texte, prix décimal, devise, état actif/retiré, version et dates | Prix positif avec plafond convenu ; vendeur non modifiable ; index état/catégorie/date et recherche à décider ; DTO public filtré. |
| Conversation | Identifiant, produit, acheteur, vendeur, dernier numéro de message, dates | Unicité produit/acheteur/vendeur ; vendeur dérivé du produit ; acheteur différent du vendeur ; contrôle transactionnel. |
| ConversationParticipant | Conversation, utilisateur, rôle acheteur/vendeur, curseur de lecture facultatif | Unicité conversation/utilisateur ; exactement les participants autorisés dans le périmètre initial, aucun ajout arbitraire. |
| Message | Identifiant, conversation, expéditeur, texte, `clientMessageId`, numéro séquentiel, date serveur | Unicité expéditeur/`clientMessageId` et conversation/numéro ; index conversation/numéro ; expéditeur dérivé de la session. |

Le prix est stocké en type décimal et transmis en chaîne avec devise ; définir précision, échelle et règle d'arrondi. Les clés opaques limitent l'énumération sans constituer une permission. Les contraintes métier non exprimables par une contrainte simple doivent être garanties par le service et sa transaction.

Prévoir des migrations versionnées et une stratégie d'évolution sans perte silencieuse. Les jeux de tests doivent être fictifs et isolés. La suppression d'une annonce retire sa visibilité publique et interdit les nouvelles conversations ; la conservation des conversations existantes doit être bornée par une politique de rétention, sans cascade accidentelle. Définir avant développement la suppression de compte et le sort des données associées ; ne pas promettre une suppression totale non spécifiée.

### 3.3 Session et révocation proposées

- Accès : jeton signé de courte durée, cible initiale de dix minutes, contenant seulement les claims nécessaires, dont utilisateur et session. Valider signature, algorithme explicitement autorisé, émetteur, audience et expiration.
- Renouvellement : jeton opaque aléatoire rotatif, cible initiale de sept jours ; empreinte en base, remplacement atomique et détection de réutilisation conduisant à la révocation de la famille.
- Web : accès en mémoire et renouvellement par cookie `HttpOnly`, `Secure`, politique `SameSite` adaptée, chemin limité ; protéger les endpoints à cookie par contrôle d'origine et mécanisme CSRF approprié.
- Mobile : accès en mémoire et renouvellement conservé avec `expo-secure-store`. Prévoir `POST /auth/mobile/login`, `POST /auth/mobile/refresh` et `POST /auth/mobile/logout` comme variantes explicites des mêmes services métier : jeton opaque dans un corps non journalisé, rotation dans la réponse non mise en cache, aucun cookie Web utilisé comme justificatif sur ces routes. Ne pas activer ce transport depuis un paramètre libre des routes Web.
- Déconnexion : révoquer la session et supprimer le cookie selon ses attributs d'origine. Vérifier l'état de session sur les opérations protégées et les événements Socket.IO pour rendre la révocation effective.
- Les clients regroupent les renouvellements concurrents. Définir et tester avec eux le traitement d'un doublon de renouvellement afin d'éviter une révocation accidentelle ou une fenêtre de rejeu non maîtrisée.

## 4. Fonctionnalités et endpoints prévus

Tous les chemins ci-dessous sont relatifs à `/api/v1`. Prévoir des réponses JSON typées, des curseurs opaques et des erreurs avec code stable, message neutre et identifiant de corrélation. Les erreurs d'accès à une ressource privée peuvent répondre 404 pour limiter l'énumération ; consigner une politique cohérente. Distinguer 401, 403, 404, 409, erreur de validation, 429 et erreur interne.

| Référence et endpoint | Comportement et données | Autorisation et tests essentiels |
|---|---|---|
| API-01 `POST /auth/register` | Valider courriel, nom public et mot de passe ; normaliser, hacher et créer un compte USER. | Public sous quotas ; ignorer ou rejeter tout rôle injecté ; tester doublon concurrent, limites et absence d'empreinte en réponse. |
| API-02 `POST /auth/login`, variante `POST /auth/mobile/login` | Vérifier les identifiants et créer une session selon le transport client convenu. | Erreur générique, protection contre essais massifs et différences révélatrices ; tester mauvais mot de passe, compte désactivé et quota partagé entre variantes. |
| API-03 `POST /auth/refresh`, `POST /auth/logout`, variantes `POST /auth/mobile/refresh`, `POST /auth/mobile/logout` | Rotation atomique, révocation et invalidation du cookie Web ou du renouvellement mobile selon la variante. | Vérifier session, origine/CSRF pour le Web ; tester jeton expiré, réutilisé, déconnexion répétée, mauvais transport et tentative après révocation. |
| API-04 `GET /users/me`, `PATCH /users/me` | Lire le compte courant ; modifier uniquement les champs publics autorisés. | Session active ; courriel et mot de passe nécessiteront des parcours dédiés s'ils sont retenus ; tester injection de rôle, autre utilisateur et champ inconnu. |
| API-05 `GET /products` | Rechercher annonces actives par texte, catégorie, prix et tri ; pagination cible 20, plafond 100. | Public ; validation des bornes, tri en liste blanche, durée de requête bornée ; tester filtres combinés et injection SQL. |
| API-06 `GET /products/:id` | Exposer le détail public d'une annonce active et un vendeur minimal. | Pas de courriel ni donnée de session ; annonce retirée non publique ; tester identifiant invalide et annonce absente. |
| API-07 `GET /users/me/products`, `POST /products` | Lister ses annonces, puis créer avec vendeur issu de la session. | USER actif ; catégorie valide, prix et tailles bornés ; tester propriétaire forgé, création sans connexion et champs supplémentaires. |
| API-08 `PATCH /products/:id`, `DELETE /products/:id` | Éditer les champs autorisés avec version ; retrait logique de l'annonce. | Propriétaire uniquement dans le périmètre initial ; admin sans accès implicite universel ; tester BOLA, conflit de version et double retrait. |
| API-09 `POST /conversations`, `GET /conversations` | Créer/retrouver la conversation pour une annonce et l'acheteur connecté ; lister les siennes. | Vendeur dérivé du produit, acheteur différent, annonce active pour création ; tester course de création et liste d'un tiers. |
| API-10 `GET /conversations/:id/messages` | Historique paginé, borné, ordonné par numéro serveur, avec curseur de reprise. | Participant actif ; tester substitution d'identifiant, curseur invalide et absence de fuite du dernier message. |
| API-11 `POST /conversations/:id/messages` | Voie REST de secours utilisant le même service que `message:send` ; accusé après persistance. | Participant, texte limité, expéditeur issu de session et clé d'idempotence ; tester doublon, charge excessive et utilisateur tiers. |
| API-12 `POST /assistant/chat` | Brancher le module d'Abdelilah, appliquer session, quotas, validation et erreur sûre. | Aucune permission dérivée du modèle ; outil de recherche limité aux produits publics ; tester utilisateur non connecté, quota et panne fournisseur. |
| API-13 `GET /health` | Réponse minimale pour vérifier la disponibilité sans révéler les paramètres internes. | Public avec information minimale ; état détaillé éventuel limité à l'exploitation ; tester absence de secrets et panne de dépendance. |
| API-14 `GET /categories` | Retourner les identifiants et libellés des catégories actives pour les filtres et formulaires Web/mobile. | Public, liste bornée et champs minimaux ; tester catégorie inactive exclue et cohérence avec la validation des annonces. |

Rôles : `USER` peut publier et gérer ses propres ressources ; `ADMIN` n'obtient que des fonctions explicitement définies et testées. Aucun endpoint d'administration n'est supposé livré ici. Un administrateur ne lit pas automatiquement les messages privés. Aucune action d'achat ou de paiement n'est incluse sans cadrage supplémentaire.

## 5. Tâches de développement ordonnées

### API-T1 — Contrats, données et architecture

- **Objectif :** stabiliser une base commune pour les trois clients et le module IA.
- **Actions/composants :** produire schéma relationnel, dictionnaire de données, DTO, conventions d'erreur et OpenAPI ; séparer contrôleurs, services et accès Prisma.
- **Contrôles :** classification public/privé/secret, liste de champs exposés, contraintes relationnelles et frontières de confiance.
- **Tests :** revue des relations, exemples valides/invalides, contrainte de conversation unique et migrations sur base de test.
- **Livrable :** contrats validés et stratégie de migration à implémenter.
- **Validation :** chaque endpoint API-01 à API-14 possède entrée, sortie, permission et cas d'échec documentés.

### API-T2 — Authentification, rôles et sessions

- **Objectif :** identifier l'acteur et rendre ses sessions révocables.
- **Actions/composants :** hachage de mots de passe, création/rotation/révocation, middleware de session, DTO utilisateur et transport Web/mobile explicite.
- **Contrôles :** signature stricte, jetons absents des logs, CSRF Web, quotas de connexion, comparaison et messages ne révélant pas le compte.
- **Tests :** mot de passe incorrect, token altéré, expiré, mauvaise audience, réutilisation de renouvellement, rôle injecté, concurrence et révocation Socket.IO.
- **Livrable :** API-01 à API-04, politique de session et résultats de tests.
- **Validation :** aucun mot de passe ou jeton de renouvellement en clair en base ; une session révoquée ne peut plus effectuer une opération protégée.

### API-T3 — Catalogue et gestion d'annonces

- **Objectif :** exposer et modifier les produits selon visibilité et propriété.
- **Actions/composants :** requêtes paginées, filtres, création, édition avec version, retrait et annonces personnelles.
- **Contrôles :** filtres de propriété dans les requêtes, liste blanche des champs, requêtes paramétrées, limites de recherche et contraintes de prix.
- **Tests :** A propriétaire/B tiers, champs de rôle/propriétaire, injection, bornes de prix, pagination et modification concurrente.
- **Livrable :** API-05 à API-08, tests et documentation.
- **Validation :** B ne lit pas de donnée privée et ne modifie aucune annonce de A ; une annonce retirée disparaît du catalogue public.

### API-T4 — Persistance et autorisation de messagerie

- **Objectif :** garantir qu'un message appartient à une conversation autorisée et n'est persisté qu'une fois.
- **Actions/composants :** services de conversation/message, transactions d'unicité, séquence par conversation et reprise historique ; intégrer les handlers d'Anas.
- **Contrôles :** appartenance vérifiée à la lecture, à l'envoi, à l'entrée de room et lors des événements sensibles ; session révoquée déconnectée, aucune confiance dans les noms de rooms clients.
- **Tests :** créations concurrentes, doublons de `clientMessageId`, expéditeur usurpé, tiers C, ordre des messages et reprise après perte de connexion.
- **Livrable :** API-09 à API-11, service partagé REST/Socket.IO et tests avec Anas.
- **Validation :** un accusé de succès implique une persistance confirmée ; un doublon de clé renvoie le même message si sa demande est identique, sinon un conflit.

### API-T5 — Intégration IA et durcissement transversal

- **Objectif :** héberger l'assistant sans élargir ses privilèges et sécuriser tous les transports.
- **Actions/composants :** monter le routeur IA d'Abdelilah, transmettre uniquement le contexte autorisé ; finaliser limites de corps, temps, CORS, Helmet, erreurs et journaux.
- **Contrôles :** pas d'accès SQL/URL arbitraire via le modèle, sorties validées, quotas combinés utilisateur/IP, proxy de confiance configuré explicitement, absence de détails internes en erreur.
- **Tests :** injection de prompt, outil inconnu, fuite d'utilisateur tiers, 429, panne IA/DB, JSON malformé et origine Web interdite.
- **Livrable :** API-12/API-13, politiques de sécurité et preuves transversales.
- **Validation :** panne et contenu non conforme produisent une erreur sûre ; aucun résultat IA ne contourne les services autorisés.

### API-T6 — Intégration et clôture des preuves

- **Objectif :** fournir une API reproductible et vérifiable dans la pipeline.
- **Actions/composants :** stabiliser OpenAPI, tests Supertest sur PostgreSQL isolé, migrations, scénarios BOLA et correction des analyses.
- **Contrôles :** secrets injectés par l'environnement futur, données fictives, moindre privilège DB, clôture des vulnérabilités avec revalidation.
- **Tests :** suite positive/négative, environnement vierge et scénarios clients réels avec Hanane et Anas.
- **Livrable :** rapports, matrice endpoints/permissions/tests et procédure de diagnostic sans secrets.
- **Validation :** tous les critères de la section 13 sont démontrés sur la révision présentée.

## 6. Organisation prévisionnelle des fichiers

Arborescence indicative seulement, sans création de code ou de configuration dans cette mission documentaire.

```text
backend/
  prisma/                schéma, migrations et données fictives de test
  src/
    app/                 assemblage Express et cycle de vie
    modules/
      auth/              contrôleur, service, schémas et sessions
      users/             profil et rôles
      products/          catalogue et propriété
      conversations/     participants et historique
      messages/          persistance, ordre et idempotence
      realtime/          handlers partagés avec Anas
      assistant/         module d'Abdelilah intégré au socle
    middlewares/         session, permissions, validation, quotas, erreurs
    infrastructure/      Prisma, journalisation et fournisseurs
    contracts/           DTO et description OpenAPI
  tests/
    unit/                règles et schémas
    integration/         PostgreSQL et requêtes HTTP
    security/            accès inter-comptes, injections et abus
```

## 7. Sécurité des API et correspondances OWASP

Analyse du projet fondée sur [OWASP API Security Top 10:2023](https://api-security.owasp.org/editions/2023/en/0x11-t10/). Les mesures suivantes sont à implémenter et à tester.

| Risque | Mesure attendue | Test ou preuve |
|---|---|---|
| API1 — Autorisation au niveau objet, BOLA/IDOR | Filtrer par propriétaire ou participant pour chaque identifiant ; ne pas se contenter d'un UUID. | A/B/C : lecture et écriture d'un tiers refusées, aucune métadonnée de conversation exposée. |
| API2 — Authentification défaillante | Validation stricte des tokens, rotation, révocation, hachage et quotas. | Jetons altérés/expirés/rejoués et connexion abusive. |
| API3 — Autorisation au niveau des propriétés | DTO d'entrée/sortie en liste blanche, refus du rôle, vendeur et expéditeur imposés. | Tentatives d'affectation massive et absence de hash/courriel dans DTO public. |
| API4 — Consommation de ressources | Pagination, limites de corps, longueur et durée ; quotas HTTP/Socket.IO/IA. | Charge bornée, 429 avec indication de reprise, taille excessive rejetée. |
| API5 — Autorisation des fonctions | Politique explicite de rôles et refus par défaut. | USER contre fonction privilégiée et admin contre message privé sans droit explicite. |
| API6 — Abus des flux sensibles | Quotas de création de comptes, annonces, conversations et messages. | Rafales réparties entre routes ; absence de création illimitée. |
| API7 — SSRF | Aucun téléchargement d'URL arbitraire ; si besoin futur, liste d'hôtes et blocage des cibles internes après résolution. | URL locale ou de métadonnées refusée dans tout connecteur concerné. |
| API8 — Mauvaise configuration | Helmet adapté, CORS explicite, cookies sûrs, proxy contrôlé et erreurs neutres. | Origine interdite, absence de stack trace, revue des en-têtes. |
| API9 — Inventaire incomplet | API versionnée, OpenAPI maintenu, inventaire REST et événements, routes inutiles retirées. | Comparaison des routes exposées et documentées. |
| API10 — Consommation non sûre d'API | Valider réponse fournisseur IA et timeout ; absence de confiance automatique dans un service externe. | Sortie malformée, indisponibilité et référence produit non autorisée. |

Correspondances avec [OWASP Web Top 10:2025](https://top10.owasp.org/2025/0x00_2025-Introduction/) : A01 autorisations ; A02 configuration ; A03 chaîne logicielle ; A04 hachage/transport/secrets ; A05 requêtes SQL et validation ; A06 conception des sessions et quotas ; A07 authentification ; A08 intégrité des migrations et artefacts ; A09 logs/alertes ; A10 erreurs, pannes et rollback transactionnel.

Pour SQL, privilégier les opérations Prisma paramétrées ; si une requête brute devient nécessaire, utiliser une API paramétrée et une revue dédiée. Ne jamais concaténer une entrée dans du SQL, un nom de colonne ou un tri. Limiter le rôle PostgreSQL applicatif, séparer les droits de migration et empêcher la fuite des erreurs DB.

Politique initiale proposée : corps JSON général limité à 64 Kio, texte d'un message à 2 000 caractères, enveloppe temps réel à 8 Kio, page à 100 objets maximum ; connexion à cinq échecs par minute par combinaison compte/IP avec protection supplémentaire IP, messages à trente par minute par utilisateur et budget par conversation. Ces seuils devront être mesurés, ajustés et testés ; un stockage partagé des compteurs sera nécessaire si plusieurs instances sont retenues.

Journaliser connexions, refus d'accès, mutations sensibles, rotation/révocation, quotas et erreurs techniques avec corrélation. Ne pas journaliser mots de passe, tokens, cookies, corps de messages ou prompts complets. Prévoir accès limité aux journaux, durée de conservation et alerte sur abus répétés.

Les contrats de session doivent également soutenir [OWASP Mobile Top 10:2024](https://owasp.github.io/www-project-mobile-top-10/) avec Anas ; le module IA doit respecter l'analyse [OWASP LLM Top 10:2025](https://genai.owasp.org/llm-top-10/) d'Abdelilah. Ces référentiels se complètent et ne prouvent pas à eux seuls la conformité.

## 8. Threat Modeling — STRIDE

| Menace | Scénario | Protection et preuve à produire |
|---|---|---|
| Usurpation | Jeton volé, faux expéditeur Socket.IO | Session active, transport chiffré, identité issue du middleware ; refus de jeton révoqué et d'expéditeur forgé. |
| Altération | Propriétaire modifié ou SQL injecté | Liste blanche, filtre d'autorisation, requêtes paramétrées et version ; tests de mutation et d'injection. |
| Répudiation | Utilisateur contestant une publication ou un message | Corrélation, dates serveur et identifiant de message persisté ; traces sans contenu privé. |
| Divulgation | Historique d'un tiers, hash ou stack trace exposé | DTO minimaux, contrôle participant et erreur centralisée ; tests de non-divulgation. |
| Déni de service | Recherches massives, sockets multiples, panne DB ou IA | Limites, quotas, délais, pagination et fermeture propre ; tests de saturation contrôlée sur environnement dédié. |
| Élévation de privilèges | Rôle ADMIN envoyé à l'inscription ou outil IA privilégié | Rôle serveur, refus par défaut et outils en lecture autorisée ; tests d'affectation massive et d'outil interdit. |

Produire un schéma de flux, classifier les actifs et évaluer impact/vraisemblance. Chaque menace doit avoir un propriétaire, une mitigation, un test et un risque résiduel accepté ou ouvert. Revoir le modèle après ajout d'un endpoint ou d'un accès fournisseur.

## 9. Tests à prévoir

- **Unitaires :** schémas Zod, permissions de propriété/participation, règles de prix, rotation et classification des erreurs ; tester limites et valeurs absentes.
- **Intégration :** Supertest avec PostgreSQL isolé, migrations depuis une base vide, contraintes, transactions et nettoyage des jeux fictifs ; un mock de DB seul ne prouve pas l'intégrité relationnelle.
- **Fonctionnels :** inscription, connexion, annonce, conversation, message, assistant mock et déconnexion via les contrats clients.
- **Sécurité :** matrice visiteur/A/B/C/admin ; BOLA et IDOR sur toutes les routes à identifiant ; champs supplémentaires ; token altéré ; CSRF ; CORS ; SQL ; dépassement des limites ; logs expurgés.
- **Concurrence et temps réel :** double création de conversation, même message réémis, deux éditions concurrentes, renouvellements concurrents, révocation pendant socket ouvert et reprise d'historique.
- **Résilience :** DB indisponible, erreur après début de transaction, fournisseur IA lent, entrée malformée et arrêt propre ; ne pas laisser d'écriture partielle.

Pour chaque endpoint, prévoir au minimum succès, validation invalide et accès refusé lorsque protégé. Conserver résultat réel, révision, environnement et données fictives utilisées. Les tests de débit ne doivent viser que les environnements de test autorisés.

## 10. Intégration DevSecOps et preuves

Mohamed fournit à Abdelilah les contrôles de types, qualité, tests, migrations de test, démarrage minimal et endpoint de disponibilité nécessaires à la future pipeline. Il fournit les comptes fictifs et contrats d'authentification nécessaires aux analyses DAST, sans secret réel dans les rapports.

Traiter les alertes Semgrep, npm audit, Gitleaks et ZAP concernant l'API : reproduire, qualifier, corriger, ajouter un test pertinent et faire relancer l'analyse. Une analyse échouée techniquement n'est pas une analyse réussie sans résultat. Fournir rapports anonymisés, résultat de nouvelle analyse et lien vers le correctif lors de l'implémentation.

Les **11 pratiques Tier 1 de Prates et Pereira** doivent être prises en compte après identification de leur source exacte. La liste n'est pas disponible dans la demande : ne pas la reconstruire par intuition. Mohamed fournira la correspondance entre chaque pratique applicable, contrôle API, test et preuve ; Abdelilah coordonnera la consolidation. Toute pratique non applicable devra être justifiée.

## 11. Collaboration

| Membre | Responsabilité propre | Interface et dépendance avec Mohamed |
|---|---|---|
| Hanane | Interface Web, formulaires et sécurité du rendu | DTO, erreurs, cookies/CSRF, pagination et refus inter-comptes. Valider les exemples avant intégration. |
| Anas | Mobile, protocole messagerie, handlers temps réel en collaboration | Transport mobile des sessions, schémas conversations/messages, rooms et reprise ; Mohamed revoit les autorisations et transactions serveur. |
| Abdelilah | Assistant IA et orchestration de pipeline | Route IA, service de recherche public, quotas, logs expurgés, environnement de test et analyses. |
| Tous | Menaces, tests et corrections | Relecture des contrats, décisions sur seuils et risques, scénario de soutenance commun. |

Définir une modification de contrat par son impact sur les consommateurs, ses exemples et ses tests. En cas de service manquant, utiliser un mock identifié ; ne pas supprimer un contrôle serveur pour débloquer un client.

## 12. Livrables attendus

1. API Express modulaire, modèle Prisma et migrations à développer ultérieurement.
2. Dictionnaire de données et contrat OpenAPI versionné avec autorisations et exemples.
3. Politique de session Web/mobile, matrice des rôles et classification des données.
4. Tests unitaires, HTTP, relationnels, d'accès et de concurrence.
5. Modèle STRIDE, rapports de sécurité, registre de corrections et preuves de nouvelle analyse.
6. Guide d'exploitation minimal : erreurs, corrélation, limites, migrations et données de démonstration fictives.

## 13. Critères d'acceptation mesurables

| Domaine | Condition de validation |
|---|---|
| Contrat | API-01 à API-14 documentés, variantes mobiles comprises ; chaque entrée, sortie, statut et permission couverts par exemples et tests. |
| Données | Migrations applicables sur base de test vierge ; contraintes d'unicité et transaction vérifiées en concurrence. |
| Session | Aucun mot de passe ou renouvellement en clair en base ; tokens altérés/expirés/révoqués refusés ; déconnexion effective aussi sur socket. |
| Autorisation | Aucun scénario de B sur les annonces privées de A ou de C sur une conversation A/B ne réussit ; tous les endpoints concernés sont inclus dans la matrice. |
| Validation | Champs inconnus sensibles refusés, limites respectées, aucune injection SQL du corpus exécutée. |
| Messages | Un même `clientMessageId` identique produit un seul message ; conflit si contenu différent ; aucun accusé de succès avant transaction confirmée. |
| Erreurs et logs | Aucun secret, hash, corps privé ou stack trace dans réponses et preuves ; panne DB/IA traitée sans écriture partielle. |
| Quotas | Dépassements testés avec réponse 429, politique de reprise et compteurs adaptés à l'architecture retenue. |
| DevSecOps | Tests obligatoires réussis, analyses exploitables et aucune vulnérabilité bloquante non corrigée ou sans exception formellement bornée. |

## 14. Planning sans dates

1. **P0 — Cadrage :** contrats, modèle relationnel, rôle de chaque membre, menaces, politiques de session et données. Sortie : interfaces validées.
2. **P0 — Socle :** architecture, migrations de test, validation, erreurs et journaux. Sortie : tests du socle reproductibles.
3. **P0 — Compte et annonces :** sessions, profils, catalogue et mutations avec tests BOLA. Ces protections précèdent les intégrations complètes.
4. **P1 — Messagerie :** modèles et services avec Anas, concurrence et rooms ; dépendance : identité et annonces stables.
5. **P1 — IA :** intégration mock puis fournisseur avec Abdelilah, quotas et tests d'outils.
6. **P0 — Durcissement :** analyses, tests de résilience, corrections et revalidation ; aucune vulnérabilité bloquante repoussée pour une fonction secondaire.
7. **P1 — Soutenance :** preuves traçables et démonstration des refus autant que des succès.

## 15. Checklist détaillée

- [ ] Valider le modèle User/Session/Product/Conversation/Message et les DTO.
- [ ] Fixer les contraintes de prix, devise, tailles et pagination.
- [ ] Documenter les endpoints et statuts dans le contrat prévu.
- [ ] Séparer contrôleurs, services et accès Prisma.
- [ ] Prévoir migrations et jeux PostgreSQL fictifs isolés.
- [ ] Sélectionner et paramétrer le hachage de mots de passe.
- [ ] Réaliser session courte, rotation atomique et révocation.
- [ ] Vérifier cookies, CSRF Web et transport mobile explicite.
- [ ] Tester renouvellements concurrents et réutilisation d'un jeton.
- [ ] Protéger profil et rôle contre l'affectation massive.
- [ ] Réaliser catalogue public paginé et détail filtré.
- [ ] Réaliser créations et mutations avec propriété et version.
- [ ] Tester BOLA/IDOR sur chaque route à identifiant.
- [ ] Réaliser conversations uniques et participants contrôlés.
- [ ] Partager le service de message entre REST et Socket.IO.
- [ ] Garantir séquence, idempotence et accusé après persistance.
- [ ] Refuser les rooms et messages d'un tiers avec Anas.
- [ ] Intégrer le module IA sans accès SQL ou outil arbitraire.
- [ ] Définir et tester limites, CORS, Helmet et proxy de confiance.
- [ ] Expurger les journaux et prévoir corrélation/alertes.
- [ ] Tester pannes DB/IA, rollback et erreurs malformées.
- [ ] Fournir les données de test et contrats pour DAST.
- [ ] Corriger les résultats des analyses et conserver leur revalidation.
- [ ] Compléter la traçabilité Tier 1 après réception de la source exacte.
- [ ] Préparer la matrice d'acceptation et les démonstrations.

## 16. Soutenance

Présenter l'architecture Express/Prisma/PostgreSQL, les frontières de confiance et les contrats. Démontrer la connexion, la rotation puis la révocation d'une session, la création d'annonce, une tentative de modification par un tiers refusée et une tentative d'injection ou de rôle ignorée/rejetée selon la politique documentée.

Avec Anas, montrer la persistance unique d'un message réémis et l'impossibilité pour C d'entrer dans une conversation A/B. Avec Abdelilah, montrer un outil IA limité au catalogue public. Présenter un résultat de test et une vulnérabilité corrigée avec nouvelle analyse, en identifiant révision, environnement et limites. Aucun succès en mock ne doit être présenté comme validation du fournisseur réel.
