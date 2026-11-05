# Cahier des charges individuel — HALLOUMI Anas

**Domaine : Application mobile et messagerie temps réel**  
**Projet : Secure Marketplace — AI-Augmented DevSecOps**  
**Établissement : FST Marrakech, Université Cadi Ayyad — Cycle ingénieur IRISI 3**  
**Module : Sécurité des applications et des services**  
**Sujet officiel : Projet 3 — Conception et mise en œuvre d'une pipeline DevSecOps augmentée par IA pour les applications de l'écosystème JavaScript (Express.js).**

Statut : cahier des charges prévisionnel. Les écrans, protocoles, contrôles et tests décrits restent à réaliser et à prouver. Les seuils proposés seront validés collectivement ; les arborescences ne représentent pas des fichiers déjà créés.

## 1. Présentation du rôle

Anas est responsable de l'application mobile et de la conception de la messagerie temps réel. Il doit fournir un client mobile utilisable sur les plateformes de démonstration retenues, ainsi qu'un protocole de conversation cohérent entre mobile, Web et serveur.

Son périmètre propre couvre navigation et écrans Expo, services mobiles, stockage local sécurisé, états réseau, Socket.IO Client, contrat des événements, synchronisation et tests temps réel. Les handlers Socket.IO serveur sont réalisés avec Mohamed : Anas porte le protocole et les scénarios, Mohamed possède les services de persistance et la validation des autorisations serveur.

Hanane possède l'interface Web de messagerie ; Anas lui fournit les contrats, exemples et règles de reprise. Abdelilah possède l'assistant IA et la pipeline DevSecOps. Ces responsabilités ne doivent pas être inversées. Tous les membres participent à la sécurité, aux tests et à la correction des vulnérabilités.

## 2. Technologies prévues et justification

| Technologie | Usage et justification |
|---|---|
| React Native et TypeScript | Écrans mobiles et logique typée réutilisable ; validation à l'exécution toujours nécessaire. |
| Expo | Socle mobile et intégration des capacités natives ; vérifier les limites sur les plateformes et builds réellement retenus. |
| Expo Router | Navigation par écrans, groupes publics/authentifiés, gestion des retours et liens entrants. |
| Axios | Client REST central, délais, annulation, statuts et renouvellement de session coordonné. |
| Socket.IO Client et serveur avec Mohamed | Connexion temps réel, accusés et reconnexion ; la reprise fonctionnelle doit être conçue au-delà de la reconnexion transport. |
| expo-secure-store | Conservation du jeton de renouvellement dans le stockage sécurisé de la plateforme ; ne pas l'assimiler à une protection absolue sur appareil compromis. |
| Zod ou schémas de validation compatibles convenus | Validation des entrées et des charges d'événements ; partager le contrat métier sans importer de secrets serveur. |
| Moteur de tests compatible Expo et React Native Testing Library à choisir | Tests des services et des interactions d'écran ; compléter par tests sur appareil ou émulateur disponible. |

Les versions et outils de tests seront fixés avant l'implémentation. Aucun SDK fournisseur LLM ni secret IA ne doit être embarqué dans l'application mobile.

# Partie A — Application mobile

## 3A. Architecture technique mobile

### Navigation, état et services

Prévoir un groupe public avec inscription/connexion, puis un groupe authentifié avec catalogue, détail, profil, conversations et conversation détaillée. Le catalogue pourra rester accessible au visiteur selon le même contrat public que le Web ; les actions de compte et de contact demandent une session. Aucun écran conditionnel ne remplace le contrôle d'accès du serveur.

Flux : écran → hook de fonctionnalité → service typé → client Axios → API `/api/v1`. Un contexte de session gère le démarrage, l'utilisateur courant, l'accès en mémoire et la déconnexion. Séparer chargement initial de session, chargement des données et soumission d'un formulaire. Les événements de premier plan/arrière-plan doivent déclencher une vérification contrôlée de session et une synchronisation, sans multiplier les connexions.

### Session et stockage local

- Contrat commun : accès signé de courte durée en mémoire, cible initiale de dix minutes ; renouvellement opaque rotatif, cible initiale de sept jours, stocké via `expo-secure-store` ; session révoquée côté serveur à la déconnexion.
- Prévoir avec Mohamed les variantes explicites `POST /auth/mobile/login`, `POST /auth/mobile/refresh` et `POST /auth/mobile/logout` sous `/api/v1`, séparées du cookie Web. Le renouvellement opaque est transmis dans un corps exclu des journaux, puis remplacé par celui de la réponse non mise en cache. Les formats doivent être approuvés avant intégration ; ne pas contourner un endpoint Web protégé contre CSRF.
- Regrouper les renouvellements concurrents et retenter une requête au plus une fois après un renouvellement réussi. En cas d'échec, effacer les jetons, caches privés et abonnements, puis revenir à la connexion.
- Au démarrage, lire le renouvellement sécurisé et demander une session valide avant d'afficher des données privées. Une donnée en stockage local ne prouve pas l'authentification.
- Ne pas conserver mots de passe, conversations ou prompts dans AsyncStorage, journaux, URL, fichiers publics ou données analytiques. Prévoir initialement les historiques en mémoire ; tout cache persistant futur nécessite une analyse séparée de chiffrement, durée et effacement.
- Tester le comportement de SecureStore, de sauvegarde, de réinstallation et de verrouillage sur les plateformes réellement utilisées ; ne pas promettre un effacement ou une persistance identique partout.

### Contrats REST consommés

Compte : inscription commune via `POST /auth/register`, variantes mobiles de connexion/renouvellement/déconnexion et `GET/PATCH /users/me`. Marketplace : `GET /products` et `GET /products/:id`, avec texte, catégorie, prix, tri et curseur ; `GET /categories` fournit les catégories actives. Le prix décimal est transmis en chaîne avec devise. Conversation et messages sont détaillés dans la partie B.

Prévoir des erreurs normalisées avec code, message neutre et identifiant de corrélation ; traiter validation, 401/403/404/409/429, indisponibilité et délai. HTTPS est requis pour les environnements cibles ; ne pas désactiver les vérifications TLS dans le client de livraison. Une exception locale de développement devra rester explicitement isolée et documentée.

## 4A. Fonctionnalités mobiles

| Référence | Réalisation détaillée | Risques, protections et tests |
|---|---|---|
| MOB-01 — Inscription | Champs nom public, courriel, mot de passe et confirmation ; erreurs associées, clavier adapté et soumission unique. | Validation des tailles et formats, masquage du mot de passe, aucun log sensible ; tester champs invalides, erreur API et soumission répétée. |
| MOB-02 — Connexion et session | Connexion, restauration au démarrage, expiration, renouvellement coordonné et déconnexion explicite. | SecureStore pour renouvellement, accès mémoire, révocation serveur ; tester 401, token expiré/révoqué, stockage inaccessible et compte A puis B. |
| MOB-03 — Navigation | Groupes public/authentifié, retour cohérent, lien entrant vers un produit et retour après connexion. | Valider identifiants et destinations, ne pas ouvrir d'URL arbitraire, ne pas afficher un écran privé avant validation ; tester lien malformé et retour après déconnexion. |
| MOB-04 — Catalogue | Liste paginée et virtualisée, cartes, rafraîchissement, fin de liste et état vide. | Réponses validées, maximum de page serveur, annulation de requête obsolète ; tester rafraîchissement, doublons de page et liste vide. |
| MOB-05 — Recherche et filtres | Recherche temporisée, catégorie, prix, tri et réinitialisation du curseur lors d'un changement de filtre. | Entrées bornées, pas de requête illimitée à chaque frappe ; tester filtres combinés et réponses arrivant dans le désordre. |
| MOB-06 — Détail produit | Présenter titre, texte, prix, catégorie, vendeur public et action contacter ; traiter produit retiré. | Aucune WebView pour du HTML non fiable, images limitées aux sources convenues si retenues ; tester liens dangereux, identifiant inconnu et données malformées. |
| MOB-07 — Profil | Voir et modifier les champs publics autorisés, afficher le compte courant et accéder à la déconnexion. | Pas de modification de rôle, aucun cache du compte précédent ; tester champ interdit, panne de mise à jour et suppression des traces locales à la sortie. |
| MOB-08 — Réseau et états | Messages d'erreur utiles, nouvelle tentative explicite, chargement distinct et absence de connexion. | Ne pas réémettre automatiquement une mutation non idempotente ; tester interruption, timeout, 429 avec attente et passage arrière-plan/premier plan. |
| MOB-09 — Ergonomie | Zones tactiles lisibles, labels accessibles, taille de police adaptable, clavier ne masquant pas la saisie. | Pas de données sensibles dans notifications ou aperçus ajoutés sans analyse ; tester orientation retenue, grande police et navigation avec aides d'accessibilité. |

Le périmètre mobile obligatoire couvre consultation, compte et messagerie. La publication mobile et une interface IA mobile ne seront ajoutées qu'après validation collective de leur priorité ; elles ne doivent pas être présentées comme fonctions acquises.

## 5A. Tâches de développement mobile

### MOB-T1 — Définir le socle et la navigation

- **Objectif :** disposer d'écrans et de services séparés, avec états explicites.
- **Actions/composants :** définir routes Expo, disposition, composants communs, contexte de session, DTO et client HTTP ; choisir les plateformes de validation.
- **Contrôles :** frontières client/API, variables publiques sans secrets, destinations de navigation autorisées.
- **Tests :** routes, liens invalides, chargement initial et absence de données privées sans session.
- **Livrable :** architecture et navigation MOB-03, contrats validés puis socle à développer.
- **Validation :** chaque écran obligatoire dispose d'un état chargé, vide, erreur et session expirée lorsqu'applicable.

### MOB-T2 — Livrer le compte et le stockage sécurisé

- **Objectif :** authentifier sans laisser de secrets dans le stockage ordinaire.
- **Actions/composants :** formulaires MOB-01, contexte MOB-02, adaptateur SecureStore, renouvellement unique et nettoyage de déconnexion.
- **Contrôles :** aucun mot de passe conservé, jeton d'accès en mémoire, renouvellement rotatif et révocation convenus avec Mohamed.
- **Tests :** connexion invalide, app relancée, jeton révoqué, renouvellement concurrent et échec du stockage sécurisé.
- **Livrable :** parcours compte, tests des services et fiche de comportement par plateforme.
- **Validation :** aucun token dans AsyncStorage/logs/URL ; déconnexion invalide la session serveur et efface les données privées visibles.

### MOB-T3 — Livrer catalogue, recherche, détail et profil

- **Objectif :** rendre les données publiques consultables et le profil maîtrisable.
- **Actions/composants :** écrans MOB-04 à MOB-07, filtres, pagination, annulation et formulaires du profil.
- **Contrôles :** validation des DTO, prix/format cohérents, texte non exécutable et liste blanche de champs.
- **Tests :** filtre rapide, résultat vide, produit retiré, données malformées, compte A/B et erreur de modification.
- **Livrable :** écrans et tests de parcours de consultation.
- **Validation :** aucun résultat obsolète ne remplace la recherche courante ; aucun champ de rôle ou donnée d'un autre compte n'est exposé/modifiable.

### MOB-T4 — Finaliser résilience et sécurité mobile

- **Objectif :** conserver une interface cohérente sous réseau instable et changements de cycle de vie.
- **Actions/composants :** MOB-08/MOB-09, annulations, reprise explicite, accessibilité et vérification des données locales.
- **Contrôles :** HTTPS sans désactivation TLS, absence de secret dans le paquet et les journaux, aucune mutation réémise sans idempotence.
- **Tests :** mode avion, délai, 429, arrière-plan, rotation retenue et grande police ; vérification sur appareil/émulateur disponible.
- **Livrable :** rapport mobile et limites par plateforme, preuves expurgées.
- **Validation :** cas réseau et stockage exécutés sur chaque plateforme revendiquée ; toute plateforme non testée signalée comme telle.

## 6A. Organisation prévisionnelle mobile

Arborescence documentaire uniquement.

```text
mobile/
  app/
    (public)/            connexion et inscription
    (main)/              catalogue, profil et conversations
    products/            écran détail
    conversations/       écran conversation
  src/
    components/          champs, cartes et états accessibles
    features/            auth, products, profile, messaging
    services/            HTTP, sessions et stockage sécurisé
    realtime/            client Socket.IO et reprise
    schemas/             formulaires, DTO et événements
    hooks/               session, cycle de vie, requêtes
    tests/               services et interactions d'écrans
```

# Partie B — Messagerie temps réel

## 3B. Architecture fonctionnelle et données

Le parcours cible est : acheteur connecté → détail d'une annonce → création/retrouvaille d'une conversation → historique REST → abonnement Socket.IO autorisé → envoi → validation et transaction serveur → accusé et diffusion aux participants. Les clients Web et mobile utilisent le même contrat.

Modèles convenus avec Mohamed : `Conversation` référence produit, acheteur et vendeur dérivé du produit ; `ConversationParticipant` lie les utilisateurs autorisés ; `Message` possède identifiant serveur, conversation, expéditeur issu de session, texte, `clientMessageId`, numéro séquentiel par conversation et date serveur.

Une seule conversation par triplet produit/acheteur/vendeur. Un utilisateur ne crée pas de conversation avec lui-même. Une annonce retirée interdit une nouvelle conversation ; le maintien d'un historique existant suit la politique de rétention et d'accès décidée avec Mohamed. Le client ne choisit jamais librement un vendeur, un expéditeur ou un participant supplémentaire.

La reconnexion transport ne garantit pas la livraison fonctionnelle des événements. Prévoir des envois réessayables avec idempotence et une reprise REST depuis le dernier numéro serveur confirmé. Le réseau peut dupliquer ou perdre un événement ; ne pas promettre une livraison réseau « exactement une fois ».

## 4B. Fonctionnalités et protocole à prévoir

### REST et événements

REST : `POST /conversations` avec identifiant de produit ; `GET /conversations` limité au compte ; `GET /conversations/:id/messages` paginé avec curseur de reprise ; `POST /conversations/:id/messages` comme secours utilisant exactement le même service serveur que Socket.IO. Tous les chemins sont sous `/api/v1`.

| Événement | Charge et résultat prévus | Contrôle et cas de test |
|---|---|---|
| Connexion Socket.IO | Jeton d'accès en mémoire transmis dans la zone d'authentification du handshake, jamais dans l'URL. | Mohamed vérifie signature et session active ; tester jeton absent, expiré et révoqué. |
| `conversation:join` | Identifiant de conversation ; accusé succès/refus ; room calculée côté serveur. | Vérifier participation avant entrée, jamais de nom de room libre ; tester C contre conversation A/B. |
| `conversation:leave` | Identifiant de conversation ; désabonnement de la socket courante. | N'altère pas les participants persistés ; tester changement rapide de conversation. |
| `message:send` | Conversation, `clientMessageId`, texte ; accusé contenant le message canonique ou une erreur. | Expéditeur dérivé de session, appartenance, validation et quota à chaque envoi ; tester usurpation et session révoquée après connexion. |
| `message:created` | Message canonique avec identifiant, numéro et date serveur. | Diffuser uniquement après persistance, aux participants autorisés ; tester réception Web/mobile et absence de diffusion à C. |
| Échec de connexion / déconnexion | Code neutre et raison exploitable sans détail interne. | Rafraîchir la session si approprié, reconnexion bornée ; tester panne prolongée et éviter une boucle agressive. |

Les accusés doivent utiliser un format stable : réussite avec donnée canonique, ou échec avec code, corrélation et délai de reprise si quota. Les événements de présence, saisie et notifications système sont des extensions optionnelles ; aucun indicateur « lu » n'est affiché sans protocole de lecture effectivement défini et testé.

### Fonctionnalités détaillées

| Référence | Réalisation | Risques, protections et tests |
|---|---|---|
| MSG-01 — Créer et lister | Depuis un produit, demander une conversation ; afficher ses conversations avec dernier aperçu autorisé et pagination. | Création concurrente et BOLA ; serveur dérive le vendeur et garantit unicité ; tester double appui et liste d'un tiers. |
| MSG-02 — Historique | Charger les messages paginés, trier par numéro serveur, fusionner avec les événements reçus pendant le chargement. | Fuite ou trou de synchronisation ; appartenance vérifiée, reprise par curseur ; tester arrivée d'un message pendant chargement et ordre inversé. |
| MSG-03 — Envoyer et recevoir | Générer un `clientMessageId` stable pour l'envoi, afficher en attente, puis confirmer avec le message persisté. | Doublon, expéditeur forgé et faux succès ; serveur idempotent, accusé après transaction ; tester timeout d'accusé puis renvoi identique. |
| MSG-04 — Rooms et sessions | Entrer uniquement dans la conversation ouverte ; quitter à la fermeture ; synchroniser après reprise. | Abonnement tiers, session expirée et fuite après déconnexion ; vérification serveur et fermeture des sockets révoquées ; tester C, révocation et changement de compte. |
| MSG-05 — Reconnexion | Reconnecter avec attente progressive bornée et aléa, renouveler si nécessaire, rejoindre après nouvelle autorisation et récupérer les manques. | Rafale de connexions et messages perdus ; une socket par client actif, état réseau visible et reprise REST ; tester coupure entre transaction et accusé. |
| MSG-06 — Validation et limites | Texte brut, non vide après normalisation, maximum initial 2 000 caractères, enveloppe maximale 8 Kio. | Spam, contenu exécutable et épuisement ; validation serveur, quotas initiaux trente messages/minute/utilisateur et budget par conversation ; tester taille limite/dépassement et 429. |

Les seuils sont des propositions à mesurer et à approuver. Conserver en mémoire un brouillon non envoyé ; ne pas ajouter une file persistante hors ligne sans spécification de protection et d'effacement. Pour un même `clientMessageId`, un contenu différent doit provoquer un conflit, pas une nouvelle émission silencieuse.

## 5B. Tâches de développement messagerie

### MSG-T1 — Stabiliser le contrat et le modèle partagé

- **Objectif :** donner aux trois composants une définition unique de conversation et message.
- **Actions/composants :** préciser charges, accusés, erreurs, curseurs et transitions d'état ; Mohamed réalise/revoit les contraintes de persistance, Hanane relit les besoins Web.
- **Contrôles :** identité issue de session, participants fixés côté serveur, données privées minimales.
- **Tests :** exemples valides/invalides, conversation de tiers et création concurrente.
- **Livrable :** contrat REST/Socket.IO, matrice d'autorisation et schéma de séquence.
- **Validation :** chaque événement possède schéma d'entrée, permission, sortie et scénario négatif.

### MSG-T2 — Réaliser les services et handlers avec Mohamed

- **Objectif :** persister avant de confirmer ou diffuser.
- **Actions/composants :** intégrer les services Conversation/Message aux handlers ; appliquer authentification, appartenance, quotas et idempotence aux deux transports.
- **Contrôles :** room calculée par le serveur, expéditeur non fourni par le client, session revalidée et erreur sûre.
- **Tests :** tiers C, session révoquée, duplication REST/Socket.IO, conflit de contenu et panne transactionnelle.
- **Livrable :** handlers réalisés en collaboration et tests serveur partagés ; Mohamed demeure responsable des autorisations et transactions.
- **Validation :** aucune diffusion avant transaction confirmée et aucun accès tiers par REST ou événement.

### MSG-T3 — Intégrer les interfaces mobile et Web

- **Objectif :** assurer l'interopérabilité acheteur-vendeur.
- **Actions/composants :** Anas réalise les vues mobiles et le client Socket.IO ; Hanane réalise les vues Web en suivant le contrat ; définir en attente/confirmé/échec et annulation des abonnements.
- **Contrôles :** contenu textuel non exécutable, socket liée à la session courante, pas de fuite d'historique au changement de compte.
- **Tests :** mobile → Web, Web → mobile, plusieurs onglets/appareils, changement de conversation et retour arrière.
- **Livrable :** parcours croisé et compte rendu d'intégration commun.
- **Validation :** messages visibles chez les seuls participants, un seul rendu par identifiant serveur.

### MSG-T4 — Garantir reprise, limites et preuves

- **Objectif :** rendre les coupures et abus vérifiables.
- **Actions/composants :** reprise REST, déduplication, nouveaux abonnements après autorisation, état de connexion et quotas ; préparer des scénarios de panne.
- **Contrôles :** bornes de reconnexion, pas de renvoi infini, limites de corps et sessions révoquées interrompues.
- **Tests :** coupure avant envoi, après persistance, avant accusé, perte d'événement, reprise après arrière-plan et envois excessifs.
- **Livrable :** tests de résilience, traces anonymisées et limites connues.
- **Validation :** après reconnexion et reprise, l'historique affiché correspond aux messages persistés autorisés sans doublon ; tout échec reste visible.

## 6B. Organisation prévisionnelle messagerie

Arborescence indicative : les dossiers serveur appartiennent au backend commun et ne doivent pas devenir un second backend divergent.

```text
backend/src/modules/
  conversations/         services et permissions possédés par Mohamed
  messages/              persistance transactionnelle et idempotence
  realtime/              authentification, rooms et handlers avec Anas
backend/tests/realtime/  accès, accusés, concurrence et reprise
mobile/src/
  features/messaging/    liste, historique, saisie et états
  realtime/              connexion, événements, reprise et déduplication
web/src/features/messaging/
                         interface réalisée par Hanane sur le même contrat
```

# Exigences transversales aux parties A et B

## 7. Sécurité et référentiels OWASP

Analyse fondée sur [OWASP Mobile Top 10:2024](https://owasp.github.io/www-project-mobile-top-10/), utilisée comme grille de risques à tester, sans présumer une certification.

| Risque Mobile 2024 | Mesure prévue et preuve |
|---|---|
| M1 — Mauvaise utilisation des identifiants | Aucun secret fournisseur embarqué, accès en mémoire et renouvellement sécurisé ; inspection de stockage et journaux expurgés. |
| M2 — Sécurité insuffisante de la chaîne logicielle | Versions maîtrisées, revue des packages natifs/JS et correction SCA ; rapports fournis à Abdelilah. |
| M3 — Authentification/autorisation non sûres | Session révocable et contrôle serveur de chaque conversation ; tests A/B/C et jeton révoqué. |
| M4 — Validation insuffisante des entrées/sorties | Schémas, liens entrants contrôlés et texte sans HTML actif ; tests de charges malformées. |
| M5 — Communication non sûre | HTTPS et transport Socket.IO chiffré en cible, TLS non désactivé ; vérification des URLs et comportement certificat invalide sur environnement de test. |
| M6 — Confidentialité insuffisante | Minimisation des données locales et des permissions, absence de contenu privé en télémétrie ; revue des permissions réellement demandées. |
| M7 — Protections binaires insuffisantes | Build de livraison sans mode debug, inspection des secrets embarqués ; documenter les limites Expo et appareil compromis, sans promettre d'inviolabilité. |
| M8 — Mauvaise configuration | Variables publiques revues, liens entrants et options de build contrôlés ; inventaire des paramètres exposés. |
| M9 — Stockage non sûr | SecureStore pour renouvellement, pas d'historique privé persistant par défaut, nettoyage de déconnexion ; tests sur plateformes revendiquées. |
| M10 — Cryptographie insuffisante | Utiliser les mécanismes de plateforme et TLS ; aucun chiffrement artisanal ni clé statique embarquée. |

Pour [OWASP API Security Top 10:2023](https://api-security.owasp.org/editions/2023/en/0x11-t10/), fournir particulièrement les scénarios API1 BOLA, API2 session, API3 propriétés, API4 consommation et API5 fonctions. La participation à une room n'est pas une autorisation permanente : les opérations serveur doivent toujours vérifier les droits courants.

Pour [OWASP Web Top 10:2025](https://top10.owasp.org/2025/0x00_2025-Introduction/), partager avec Hanane et Mohamed les tests d'accès A01, configuration A02, chaîne logicielle A03, injection A05, authentification A07 et exceptions A10. Pour [OWASP LLM Top 10:2025](https://genai.owasp.org/llm-top-10/), ne pas envoyer les messages privés à l'assistant ; toute future intégration IA mobile devra reprendre les sorties non fiables et quotas d'Abdelilah.

## 8. Threat Modeling — STRIDE

Actifs : jetons, compte, conversations, messages, données locales. Frontières : appareil/API, appareil/stockage sécurisé, client/Socket.IO et Socket.IO/base. Prévoir diagramme des flux et revue commune.

| Menace | Scénario | Protection et validation |
|---|---|---|
| Usurpation | Jeton volé ou expéditeur falsifié | SecureStore, session courte/révocable, identité serveur ; tests d'expéditeur forgé et token révoqué. |
| Altération | Modification de message ou participant dans la charge | Validation serveur, participant non arbitraire et idempotence contrôlée ; charges modifiées rejetées. |
| Répudiation | Désaccord sur un envoi affiché comme réussi | Identifiant et date serveur, accusé après persistance ; test de coupure avant accusé et corrélation expurgée. |
| Divulgation | Tiers rejoignant une room ou cache du compte précédent | Appartenance serveur, nettoyage, absence de cache privé persistant ; tests A/B/C et changement de compte. |
| Déni de service | Messages excessifs et reconnexion agressive | Quotas, charges bornées, attente progressive ; tests de saturation contrôlée et 429. |
| Élévation de privilèges | Client imposant le vendeur, le rôle ou une room | Valeurs dérivées côté serveur et refus par défaut ; test des champs injectés et room non autorisée. |

Pour chaque menace, consigner impact, vraisemblance, propriétaire, mitigation, test et risque restant. La sécurité d'un appareil compromis demeure une limite à exposer ; elle ne dispense pas de protéger le serveur.

## 9. Tests à prévoir

- **Unitaires mobiles :** schémas, adaptateur de stockage, gestion d'erreurs, renouvellement unique, fusion/déduplication et curseurs.
- **Écrans :** saisie, validation, navigation, chargement, erreur, état vide, grande police et clavier ; tests avec bibliothèque compatible Expo choisie.
- **Intégration REST :** compte, catalogue, profil et historique sur API de test ; token absent/expiré, 403/404/429, données invalides et délai.
- **Temps réel :** deux participants et un tiers, abonnements, envoi/accusé, course REST/Socket.IO, renvoi identique, conflit de contenu et ordre serveur.
- **Résilience :** mode avion, fermeture/réouverture, arrière-plan, coupure après transaction avant accusé, perte d'un événement et reprise par historique.
- **Sécurité mobile :** pas de secrets dans stockage ordinaire, journaux et paquet ; révocation ; HTTPS ; liens entrants malformés et absence de données d'un autre compte.
- **Fonctionnels croisés :** acheteur mobile/vendeur Web puis sens inverse ; même état canonique après reprise.

Le test de reconnexion doit comparer les identifiants et numéros serveur attendus, pas seulement constater qu'une socket est reconnectée. Documenter OS, appareil/émulateur, type de build, versions, conditions réseau et limites. Aucun résultat Android ne prouve automatiquement un comportement iOS.

## 10. Intégration DevSecOps et preuves

Anas fournit à Abdelilah les contrôles de types, qualité, tests mobiles et serveur temps réel à intégrer ultérieurement. Si un test sur appareil ne peut pas être automatisé, préciser sa procédure manuelle, son résultat et sa limite au lieu de le déclarer couvert par la CI.

Fournir rapports de tests, matrice des plateformes, preuve de stockage sans token ordinaire, refus d'accès tiers, traces d'accusés anonymisées et comparaison d'historique après coupure. Corriger les alertes SAST/SCA/secrets de sa partie ; traiter avec Mohamed les constats DAST sur les échanges serveur. Aucun contenu privé ni token ne doit entrer dans les artefacts.

Les **11 pratiques Tier 1 de Prates et Pereira** ne sont pas listées dans la demande. La liste scientifique exacte doit être obtenue et validée avant correspondance ; ne pas inventer des pratiques sous cette appellation. Anas devra relier les contrôles mobiles et temps réel aux pratiques applicables et produire leurs preuves, sous coordination d'Abdelilah.

## 11. Collaboration

| Partenaire | Échange attendu | Validation commune |
|---|---|---|
| Mohamed | Sessions mobiles, tables, permissions, services transactionnels, quotas et handlers serveur | Contrat fixé, accès tiers refusé, message idempotent et révocation effective. |
| Hanane | Événements, accusés, états, curseurs et exemples pour l'interface Web | Échange Web-mobile, reprise et absence de doublon sur les deux clients. |
| Abdelilah | Besoins CI mobile, secrets de test, rapports et correction des analyses | Contrôles automatisables branchés ultérieurement ; limites des essais manuels clairement indiquées. |
| Équipe | Politique de rétention, seuils et scénarios d'abus | Décisions documentées et données de démonstration fictives. |

Un serveur simulé permet de construire des écrans, mais ne valide ni permissions, ni transaction, ni sécurité de Socket.IO. Les preuves d'intégration doivent utiliser les services serveur réels de l'environnement de test.

## 12. Livrables attendus

1. Application Expo et services mobiles à développer ultérieurement, avec parcours compte/catalogue/profil/messagerie.
2. Contrat REST/Socket.IO, schémas des événements et diagrammes connexion/envoi/reprise.
3. Handlers de messagerie réalisés avec Mohamed et règles partagées avec Hanane.
4. Tests mobiles, temps réel, accès inter-comptes et résilience.
5. Fiche de stockage sécurisé et matrice de plateformes effectivement validées.
6. Modèle STRIDE, rapports d'analyse, corrections et preuves de revalidation.

## 13. Critères d'acceptation mesurables

| Domaine | Condition de validation |
|---|---|
| Mobile | MOB-01 à MOB-09 possèdent chacun un test nominal et un test d'erreur ; résultats rattachés aux plateformes réellement essayées. |
| Sessions | Aucun token dans stockage ordinaire/log/URL ; accès en mémoire ; renouvellement coordonné ; session révoquée inutilisable côté serveur. |
| Confidentialité | Après déconnexion de A puis connexion de B, aucun profil, message ou abonnement de A ne demeure accessible. |
| Conversation | Un triplet produit/acheteur/vendeur ne crée qu'une conversation même en concurrence ; aucun accès de C aux données A/B. |
| Envoi | Même clé et même contenu : un seul message persisté ; clé réutilisée avec contenu différent : conflit ; accusé positif après persistance uniquement. |
| Reprise | Après les coupures prévues, l'historique correspond aux messages persistés autorisés sans trou constaté ni doublon d'affichage. |
| Interopérabilité | Envoi mobile → Web et Web → mobile validé, y compris après reconnexion et changement de conversation. |
| Limites | Message trop long et enveloppe trop grande rejetés ; quota testé ; reconnexion sans boucle agressive. |
| Sécurité et preuves | Scénarios négatifs exécutés ; analyses obligatoires exploitables ; alertes bloquantes corrigées ou traitées par exception bornée commune. |

Un test non disponible sur une plateforme impose de déclarer cette limite ; ne pas élargir la portée des résultats.

## 14. Planning sans dates

1. **P0 — Cadrage :** plateformes, parcours, schémas et protocole, stockage, menaces et partage des tâches serveur.
2. **P0 — Socle mobile :** navigation, client HTTP, validation, session et données locales. Dépendance : contrat de Mohamed.
3. **P0 — Consultation :** catalogue, filtres, détail, profil et états réseau.
4. **P0 — Messagerie serveur :** contraintes, permissions et accusés avec Mohamed ; tests négatifs avant branchement complet.
5. **P1 — Clients temps réel :** vues mobiles, assistance à Hanane et parcours croisés.
6. **P0 — Reprise et durcissement :** coupures, arrière-plan, quotas, stockage et corrections ; preuves par plateforme.
7. **P1 — Soutenance :** scénarios reproductibles, résultats réels et limites assumées. Les notifications avancées restent secondaires.

## 15. Checklist détaillée

- [ ] Fixer les plateformes et les environnements de validation.
- [ ] Faire valider routes, DTO et transport mobile de session avec Mohamed.
- [ ] Séparer écrans, hooks, services HTTP et stockage sécurisé.
- [ ] Réaliser formulaires d'inscription et connexion avec validation.
- [ ] Garder l'accès en mémoire et le renouvellement dans SecureStore.
- [ ] Tester expiration, révocation et renouvellement concurrent.
- [ ] Vérifier démarrage, stockage indisponible et déconnexion complète.
- [ ] Sécuriser navigation, liens entrants et retour après connexion.
- [ ] Réaliser catalogue, recherche, filtres et pagination.
- [ ] Tester réponses obsolètes, données invalides et produit retiré.
- [ ] Réaliser profil sans champ de rôle modifiable.
- [ ] Gérer mode avion, timeout, 429 et arrière-plan.
- [ ] Vérifier grande police, clavier et labels accessibles.
- [ ] Fixer les modèles de conversation et message avec Mohamed.
- [ ] Documenter événements, accusés, erreurs et curseurs.
- [ ] Dériver rooms, vendeur et expéditeur côté serveur.
- [ ] Vérifier appartenance et session sur les opérations temps réel.
- [ ] Réaliser le service partagé REST/Socket.IO avec Mohamed.
- [ ] Garantir idempotence et accusé après persistance.
- [ ] Réaliser liste, historique, saisie et états mobiles.
- [ ] Transmettre le contrat d'interface Web à Hanane.
- [ ] Tester tiers C, expéditeur forgé et jeton révoqué socket ouverte.
- [ ] Tester perte d'accusé, événement manqué et reprise REST.
- [ ] Vérifier limites de message, quotas et reconnexion bornée.
- [ ] Comparer l'historique Web-mobile après une coupure.
- [ ] Inspecter stockage, logs et exposition de secrets du build retenu.
- [ ] Fournir rapports de tests et corrections à Abdelilah.
- [ ] Compléter la traçabilité Tier 1 après réception de la référence exacte.
- [ ] Préparer les démonstrations et la matrice des limites de plateforme.

## 16. Soutenance

Présenter séparément l'architecture mobile et le protocole de messagerie. Démontrer connexion, catalogue, recherche, détail et profil, puis stockage sécurisé et nettoyage au changement de compte. Identifier les plateformes et le type de build réellement testés.

Avec Hanane et Mohamed, montrer un échange acheteur mobile/vendeur Web, une conversation interdite à C, une déconnexion réseau suivie de reprise et un envoi réessayé sans double persistance. Présenter les messages canoniques et les preuves expurgées plutôt qu'un simple indicateur de socket connectée. Expliquer une vulnérabilité corrigée, sa nouvelle analyse, les limites réseau/plateforme et le lien avec les contrôles DevSecOps coordonnés par Abdelilah.
