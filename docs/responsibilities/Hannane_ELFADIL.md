# Cahier des charges individuel — ELFADIL Hanane

**Domaine : Web et sécurité front-end**  
**Projet : Secure Marketplace — AI-Augmented DevSecOps**  
**Établissement : FST Marrakech, Université Cadi Ayyad — Cycle ingénieur IRISI 3**  
**Module : Sécurité des applications et des services**  
**Sujet officiel : Projet 3 — Conception et mise en œuvre d'une pipeline DevSecOps augmentée par IA pour les applications de l'écosystème JavaScript (Express.js).**

Statut : cahier des charges prévisionnel. Toutes les fonctions, arborescences, protections et preuves ci-dessous sont à réaliser ou à vérifier. Ce document n'atteste aucune implémentation. Les valeurs chiffrées constituent des cibles initiales à valider collectivement avant développement.

## 1. Présentation du rôle

Hanane est responsable de la conception, du développement, de l'accessibilité et de la sécurité du client Web. Elle doit rendre utilisables les parcours d'authentification, de consultation, de publication, de profil, de messagerie et d'assistance IA, y compris en cas d'erreur réseau ou d'expiration de session.

Son périmètre propre couvre les composants React, les formulaires, la navigation, les services HTTP, l'affichage des événements et les tests du client Web. Mohamed possède les règles métier et les autorisations de l'API ; Anas possède le protocole de messagerie et son intégration temps réel ; Abdelilah possède le module IA et l'orchestration DevSecOps. Hanane intègre leurs contrats et fournit les preuves concernant son client.

**L'autorisation doit toujours être vérifiée côté serveur.** Masquer un bouton, protéger une route React ou valider un formulaire ne constitue jamais une autorisation de sécurité. Tous les membres participent à la sécurité, aux tests et à la correction des vulnérabilités de leur périmètre.

## 2. Technologies prévues et justification

| Technologie | Usage prévu et justification |
|---|---|
| React et TypeScript | Composants réutilisables, contrats typés et séparation des états ; les types ne remplacent pas la validation à l'exécution. |
| Vite | Environnement de développement et production de ressources Web ; toute variable exposée au navigateur doit être considérée publique. |
| Tailwind CSS | Styles cohérents, adaptations aux tailles d'écran, visibilité des états de focus et d'erreur. |
| React Router | Navigation publique et authentifiée, pages introuvables, préservation contrôlée de la destination après connexion. |
| Axios | Client HTTP commun, délais, traitement des statuts, annulation des requêtes et gestion maîtrisée du renouvellement de session. |
| Zod | Validation des formulaires et des réponses externes aux frontières utiles. |
| React Hook Form | Gestion des champs, erreurs, soumission et intégration des schémas de validation. |
| Vitest | Tests unitaires des composants logiques, schémas et services. |
| React Testing Library, outil de parcours navigateur à choisir | Compléments prévus pour les interactions accessibles et les parcours intégrés ; choix à documenter avant ajout. |
| Socket.IO Client | Consommation du protocole défini par Anas pour la messagerie Web. |

Les versions compatibles seront fixées lors de l'implémentation avec Abdelilah. Aucune dépendance n'est installée par ce cahier des charges.

## 3. Architecture technique et contrats

### 3.1 Composants et flux

Flux cible : pages React → composants de fonctionnalités → services typés → API Express `/api/v1` → données autorisées. Les composants visuels ne doivent pas multiplier les appels Axios directs. Prévoir un client commun, des services par domaine et des fonctions de transformation des réponses.

Séparer l'état de session, l'état des formulaires et l'état des données distantes. Utiliser initialement un contexte de session et des hooks ciblés ; n'introduire une bibliothèque supplémentaire que si un besoin est établi. Réinitialiser les caches et abonnements à la déconnexion. Définir les états vide, chargement, succès, erreur, session expirée et action en cours pour chaque page.

### 3.2 Contrats communs proposés

- Une annonce et un produit désignent la même ressource `Product`. Champs publics : identifiant, titre, description en texte, prix décimal sérialisé en chaîne, devise, catégorie, état de publication, vendeur public et dates. Ne jamais calculer un prix monétaire en flottant sans règle d'arrondi.
- Catalogue : `GET /products` avec recherche, catégorie, prix, tri et pagination par curseur ; détail : `GET /products/:id` ; catégories actives : `GET /categories`. Cible : 20 résultats par page, maximum serveur de 100.
- Compte : `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`, `GET/PATCH /users/me`. Réponses d'erreur normalisées avec code stable, message neutre, identifiant de corrélation et erreurs de champs éventuelles.
- Écriture : `POST /products`, `PATCH/DELETE /products/:id`. Le serveur attribue le propriétaire, filtre les champs autorisés et vérifie chaque mutation.
- Messagerie : `/conversations`, `/conversations/:id/messages` et événements `conversation:join`, `message:send`, `message:created`, `conversation:leave`, selon le contrat d'Anas.
- IA : `POST /assistant/chat`, réponse bornée contenant texte, références de produits et identifiant de requête ; messages du modèle traités comme non fiables.

### 3.3 Session Web cible

Prévoir un jeton d'accès de courte durée conservé en mémoire et transmis par en-tête Bearer. Le renouvellement utilise un jeton opaque rotatif dans un cookie `HttpOnly`, `Secure`, avec politique `SameSite` adaptée à l'hébergement et chemin limité. Aucun jeton dans `localStorage`, une URL ou un journal. Le serveur doit protéger les endpoints utilisant des cookies contre les attaques CSRF et vérifier les origines.

Le client tente au plus un renouvellement par requête en échec d'authentification et regroupe les renouvellements concurrents. En cas d'échec, vider la session, arrêter les abonnements et rediriger sans boucle. À la fermeture ou au rechargement, rétablir la session par l'endpoint prévu ; ne pas déduire la connexion d'une simple variable persistée. La déconnexion doit révoquer la session côté serveur.

## 4. Fonctionnalités à développer

| Référence | Fonctionnalité et réalisation attendue | Protection et vérification associées |
|---|---|---|
| WEB-01 | Inscription : nom public, courriel, mot de passe et confirmation ; connexion avec erreurs de champs, soumission unique et destination interne après succès. | Validation Zod, aucune journalisation des mots de passe, message de connexion générique ; tester champs invalides, doublon traité sans fuite excessive, 401, 429 et expiration. |
| WEB-02 | Accueil : présentation Marketplace, accès aux catégories et dernières annonces actives, liens vers catalogue et connexion. | Affichage textuel sûr, liens internes contrôlés, état vide et panne API ; tester navigation au clavier et contenu sans donnée personnelle. |
| WEB-03 | Catalogue : recherche temporisée, filtres combinables, tri et pagination ; synchroniser les paramètres utiles avec l'URL. | Borner la longueur des filtres, annuler les requêtes obsolètes, ne pas confondre validation locale et validation serveur ; tester résultats vides et réponses arrivant dans le désordre. |
| WEB-04 | Détail : prix, description, catégorie, vendeur public et action contacter ; gérer annonce absente ou retirée. | Texte sans HTML actif, URL d'image autorisée si les images sont retenues ; tester charge XSS, ressource inexistante et bouton adapté au propriétaire. |
| WEB-05 | Publication : formulaire titre, description, catégorie, prix et devise ; aperçu et confirmation après persistance. | Champs autorisés uniquement, prix positif et borné, limites cohérentes avec l'API ; tester soumission répétée, erreurs serveur et conservation du brouillon en mémoire. |
| WEB-06 | Modification : charger les données autorisées, conserver les champs saisis en cas d'erreur ; suppression avec confirmation explicite et retour au catalogue. | Ne pas envoyer de propriétaire ni de rôle ; vérifier traitement de 403, 404 et conflit de version ; tester accès direct à l'annonce d'un tiers et tentative de mutation rejetée par l'API. |
| WEB-07 | Profil : consulter et modifier les champs publics autorisés, consulter ses annonces, se déconnecter. | Aucun champ de rôle modifiable ; données privées limitées au compte courant ; tester compte A puis compte B pour repérer un cache résiduel. |
| WEB-08 | Messagerie : liste des conversations, historique paginé, saisie, états envoi/confirmé/échec et reprise après reconnexion. | Abonnement à une conversation autorisée, déduplication par identifiants et texte brut ; tester troisième utilisateur refusé, session expirée et reconnexion sans doublon. |
| WEB-09 | Assistant IA : saisie, indication de traitement, réponse textuelle et liens vers produits référencés ; mode indisponible et quota atteint. | Ne jamais interpréter la sortie comme HTML ou commande ; liens limités aux références validées ; tester texte malveillant, JSON invalide, délai dépassé et référence inconnue. |
| WEB-10 | Ergonomie transversale : responsive, chargement, erreurs, vides, absence de connexion et annulation des requêtes au démontage. | Étiquettes, focus visible, contraste, annonces accessibles des erreurs ; tester clavier seul, zoom 200 % et largeurs cibles 360, 768 et 1280 pixels. |

Les images, si retenues collectivement, nécessiteront un contrat de stockage et de validation serveur avant leur intégration. Leur présence dans une maquette ne vaut pas livraison d'un service de téléversement.

## 5. Tâches de développement ordonnées

Chaque tâche comprend un objectif, des actions, les composants, les contrôles, les tests et sa preuve de fin.

### WEB-T1 — Stabiliser les contrats et le socle

- **Objectif :** rendre les pages cohérentes et les échanges prévisibles.
- **Actions et composants :** établir les routes, composants de disposition, types des DTO, schémas des formulaires, services HTTP et catalogue des erreurs ; préparer des données fictives sans données réelles.
- **Sécurité :** répertorier les frontières navigateur/API et interdire les secrets dans les variables publiques ; ne pas créer de maquette avec des droits implicitement privilégiés.
- **Tests :** valider les schémas aux bornes, les erreurs normalisées et les routes inconnues.
- **Livrable :** architecture Web documentée, contrats revus par Mohamed, Anas et Abdelilah, puis socle lors du développement.
- **Validation :** chaque parcours WEB-01 à WEB-10 possède une route ou un composant identifié et un état d'erreur spécifié.

### WEB-T2 — Réaliser l'authentification et les sessions

- **Objectif :** connecter et déconnecter un utilisateur sans exposition durable des jetons.
- **Actions et composants :** développer les formulaires, contexte de session, protection de navigation et renouvellement coordonné ; limiter les redirections à des chemins internes autorisés.
- **Sécurité :** mémoire pour l'accès, cookie serveur pour le renouvellement, protection CSRF convenue, nettoyage des données au changement de compte.
- **Tests :** succès, refus, 429, renouvellement concurrent, révocation, deux sessions distinctes et reconnexion après rechargement.
- **Livrable :** parcours compte et preuves de requêtes expurgées.
- **Validation :** aucun jeton dans le stockage Web persistant ; les requêtes interdites restent refusées côté API même en contournant l'interface.

### WEB-T3 — Livrer la consultation Marketplace

- **Objectif :** permettre de trouver une annonce et de comprendre son état.
- **Actions et composants :** développer accueil, cartes, filtres, curseurs, détail et sérialisation des paramètres ; gérer les réponses obsolètes.
- **Sécurité :** traiter titres, descriptions et vendeurs comme données non fiables ; contrôler les protocoles des liens.
- **Tests :** combinaisons de filtres, caractères spéciaux, zéro résultat, page suivante, disparition d'annonce et recherche rapide successive.
- **Livrable :** WEB-02 à WEB-04, tests de services et d'interactions.
- **Validation :** résultat conforme aux paramètres courants et aucune exécution de charges XSS du jeu de test.

### WEB-T4 — Livrer publication, mutations et profil

- **Objectif :** donner au propriétaire la maîtrise de ses annonces et de son profil.
- **Actions et composants :** formulaires communs création/édition, messages de validation, confirmation de suppression et invalidation des données après mutation.
- **Sécurité :** liste blanche de champs, soumission verrouillée pendant traitement, prise en charge de 403 et conflit ; protection contre l'affichage de données du compte précédent.
- **Tests :** prix invalide, champ excessif, rôle injecté, modification concurrente, propriétaire différent et échec de suppression.
- **Livrable :** WEB-05 à WEB-07 et matrice des cas de mutation.
- **Validation :** aucune modification n'est affichée comme réussie avant confirmation serveur ; chaque refus conserve une interface cohérente.

### WEB-T5 — Intégrer messagerie et assistant

- **Objectif :** offrir les deux parcours interactifs sans dupliquer les règles serveur.
- **Actions et composants :** brancher le contrat Socket.IO d'Anas, charger l'historique REST, dédupliquer les messages et intégrer le service IA d'Abdelilah.
- **Sécurité :** jeton en mémoire à la connexion Socket.IO, sortie IA en texte, absence de données privées dans le contexte envoyé au chat.
- **Tests :** acheteur/vendeur Web-mobile, reconnexion, message dupliqué, utilisateur tiers, quota IA et sortie malformée.
- **Livrable :** WEB-08 et WEB-09 avec scénarios croisés validés par les responsables.
- **Validation :** aucun message d'une autre conversation ; une seule représentation par message confirmé ; tout résultat IA non conforme est rejeté ou affiché en erreur sûre.

### WEB-T6 — Finaliser accessibilité, sécurité et preuves

- **Objectif :** rendre le client démontrable et exploitable dans la pipeline.
- **Actions et composants :** revoir états, focus, contrastes et responsive ; corriger les défauts signalés par les analyses et préparer les parcours reproductibles.
- **Sécurité :** vérifier exposition de configuration, rendu de contenu et comportement sur session révoquée ; convenir avec Mohamed des en-têtes de protection servis au navigateur.
- **Tests :** parcours complets, clavier, trois largeurs cibles, inspection stockage et console, régression des vulnérabilités corrigées.
- **Livrable :** rapport Web, résultats de tests, preuves expurgées et limites connues.
- **Validation :** critères de la section 13 satisfaits et aucun risque bloquant laissé sans décision collective explicite.

## 6. Organisation prévisionnelle des fichiers

Arborescence documentaire uniquement ; aucun de ces fichiers applicatifs n'est créé par le présent document.

```text
web/
  src/
    app/                 navigation, fournisseurs de contexte, disposition
    components/          boutons, champs, dialogues, états communs
    features/
      auth/              pages, formulaires, contexte de session
      products/          accueil, catalogue, détail, édition
      profile/           profil et annonces personnelles
      messaging/         conversations et vues du protocole d'Anas
      assistant/         interface et service du chat IA
    services/            client HTTP, erreurs, contrats de réponses
    hooks/               requêtes annulables et comportements partagés
    schemas/             validation des entrées et DTO utiles
    styles/              styles et règles d'accessibilité
    tests/               tests unitaires, interactions et jeux fictifs
  e2e/                   parcours navigateur envisagés
```

## 7. Sécurité et référentiels

Les correspondances suivantes sont une analyse du projet fondée sur [OWASP Web Top 10:2025](https://top10.owasp.org/2025/0x00_2025-Introduction/). Elles définissent des travaux et ne constituent pas une certification.

| Risque Web 2025 | Protection à prévoir côté Web et dépendance serveur |
|---|---|
| A01 — Contrôles d'accès défaillants | Routes conditionnelles pour l'ergonomie ; vérification API obligatoire de propriétaire et de participant. |
| A02 — Mauvaise configuration de sécurité | Variables publiques revues, erreurs neutres, politique CSP et en-têtes convenus avec le serveur, origines CORS limitées. |
| A03 — Défaillances de la chaîne logicielle | Revue des dépendances et corrections des résultats SCA ; éviter un paquet pour une fonction triviale. |
| A04 — Défaillances cryptographiques | HTTPS en cible, aucun secret embarqué, pas de mécanisme cryptographique artisanal ni de token persistant exposé au script. |
| A05 — Injection | Échappement textuel React, absence de rendu HTML arbitraire, interdiction des URL exécutables ; assainissement adapté si un rendu enrichi devient nécessaire. |
| A06 — Conception non sécurisée | Parcours de suppression explicite, limitation des requêtes et modèle de confiance navigateur/API documenté. |
| A07 — Défaillances d'authentification | Renouvellement borné, nettoyage de session et gestion de révocation ; aucun indice sensible dans les erreurs. |
| A08 — Défaillances d'intégrité logicielle ou des données | Dépendances maîtrisées, DTO validés, confirmation des mutations par réponse serveur. |
| A09 — Défaillances de journalisation et d'alerte | Identifiant de corrélation, aucun contenu de chat ou secret dans la console ; remonter les anomalies via les mécanismes convenus. |
| A10 — Mauvaise gestion des conditions exceptionnelles | Frontières d'erreur React, traitement des délais, erreurs réseau, réponses invalides et reprise sans boucle. |

Pour [OWASP API Security Top 10:2023](https://api-security.owasp.org/editions/2023/en/0x11-t10/), contribuer particulièrement aux scénarios API1/API3/API5 d'autorisation et API4 de consommation. Pour [OWASP LLM Top 10:2025](https://genai.owasp.org/llm-top-10/), traiter les sorties du chat comme non fiables et ne pas exposer de secrets ni de données privées au modèle. Pour [OWASP Mobile Top 10:2024](https://owasp.github.io/www-project-mobile-top-10/), partager avec Anas les contrats et jeux de tests de session ; les protections locales mobiles restent de son ressort.

## 8. Threat Modeling — STRIDE

Actifs : session, données du compte, annonces, conversations et contenu du chat. Frontières : navigateur/API, navigateur/Socket.IO, API/fournisseur IA. Réaliser avec l'équipe un schéma des flux et attribuer un propriétaire aux risques résiduels.

| Menace | Scénario concret | Mesure et preuve attendues |
|---|---|---|
| Usurpation | Réutilisation de jeton ou faux écran de connexion | Session courte, renouvellement contrôlé, navigation interne ; test de révocation et de redirection externe refusée. |
| Altération | Modification du propriétaire dans une requête forgée | Schémas et liste de champs, autorisation serveur ; preuve de rejet API. |
| Répudiation | Contestation d'une suppression | Confirmation claire et corrélation serveur ; preuve d'événement sans contenu sensible. |
| Divulgation | XSS lisant la session ou cache d'un autre compte | Rendu textuel, stockage limité, nettoyage ; test de charge XSS et changement A/B. |
| Déni de service | Rafale de recherches ou renouvellement infini | Temporisation, annulation, tentative de renouvellement unique ; tests de concurrence et 429. |
| Élévation de privilèges | Accès direct à une route cachée d'administration | Pas de confiance dans le client, refus serveur ; scénario utilisateur ordinaire contre fonction privilégiée. |

Consigner pour chaque scénario : préconditions, impact, vraisemblance qualitative, mitigation, test, responsable et risque restant. Réviser après toute évolution de session ou de rendu enrichi.

## 9. Tests à prévoir

- **Unitaires :** schémas Zod, formats de prix, filtres, normalisation des erreurs, renouvellement limité et déduplication des messages.
- **Composants :** soumissions par clavier, étiquettes, erreurs associées aux champs, dialogue de suppression, boutons désactivés et reprise après erreur.
- **Intégration :** services avec réponses conformes et malformées, 401/403/404/409/429/500, délai et interruption réseau ; tests croisés sur API de test.
- **Fonctionnels :** inscription → connexion → publication → modification → contact Web/mobile → suppression → déconnexion ; vérifier aussi le parcours visiteur.
- **Sécurité :** contenu XSS dans annonce, message et sortie IA ; URL dangereuse ; mutation d'un tiers ; token absent des stockages et des journaux ; CSRF testé avec Mohamed.
- **Accessibilité et responsive :** clavier seul, focus après navigation et erreur, zoom 200 %, largeurs cibles, lecteurs d'écran sur les parcours principaux si environnement disponible.

Les jeux de données devront comporter au moins un visiteur, un propriétaire A, un utilisateur B et un participant tiers C. Chaque anomalie corrigée doit obtenir un scénario de régression ciblé. Conserver résultats réels et limites, sans déclarer une couverture non mesurée.

## 10. Intégration DevSecOps et preuves

Hanane fournit les contrôles de types, qualité, tests et production du client qui seront branchés ultérieurement par Abdelilah. Elle analyse les alertes SAST, SCA, secrets et DAST liées à son périmètre, les reproduit, les corrige et fournit la preuve de nouvelle analyse.

Preuves attendues : identifiant de révision, résultat des tests, captures d'accessibilité, requêtes et réponses anonymisées, absence de tokens en stockage, scénario XSS rejeté et tickets de correction. Chaque preuve doit préciser environnement, outil/version lorsqu'applicable, résultat attendu/réel et limite de validité.

La référence exacte aux **11 pratiques Tier 1 de Prates et Pereira** n'est pas fournie. Ne pas en inventer les intitulés. Hanane devra relier ses contrôles aux identifiants exacts après obtention de la publication ou de la grille officielle par Abdelilah, puis fournir une preuve pour chaque pratique applicable. Les pratiques non applicables nécessiteront une justification relue collectivement.

## 11. Collaboration et dépendances

| Partenaire | Échange attendu | Condition de passage |
|---|---|---|
| Mohamed | DTO, endpoints, erreurs, sessions, CSRF, autorisations, règles de validation | Contrat et exemples de succès/refus approuvés avant branchement réel ; aucun contournement des refus API. |
| Anas | Événements Socket.IO, accusés, reprise, identifiants de messages et composants de scénario | Un échange Web-mobile sans doublon et un accès tiers refusé. |
| Abdelilah | Contrat IA, mode mock, limites, états de quota et contrôles de pipeline | Interface utilisable sans fournisseur réel ; sortie non conforme traitée de façon sûre. |
| Équipe | Revue des menaces et parcours communs | Décisions consignées, risque attribué et preuve de validation. |

Si l'API n'est pas prête, travailler avec des réponses fictives conformes au contrat en signalant explicitement le mode simulé. Aucun succès en simulation ne prouve la sécurité du serveur.

## 12. Livrables attendus

1. Client Web et composants réutilisables, à développer dans une phase ultérieure.
2. Contrats de consommation REST/temps réel/IA et catalogue des états d'interface.
3. Tests unitaires, d'interactions et de parcours avec données fictives.
4. Guide des parcours, choix de session et d'accessibilité, schéma des flux et registre STRIDE.
5. Rapports d'analyses, preuves de correction, captures expurgées et limites connues.

## 13. Critères d'acceptation mesurables

| Domaine | Condition de validation |
|---|---|
| Couverture fonctionnelle | Chaque référence WEB-01 à WEB-10 dispose d'au moins un scénario nominal et un scénario d'échec exécutés et consignés. |
| Formulaires | Chaque contrainte possède un test de valeur valide et de valeur invalide ; les erreurs serveur sont affichées sans effacer inutilement les données non sensibles. |
| Session | Aucun jeton dans le stockage Web persistant, URL ou console ; une seule tentative de renouvellement par requête ; déconnexion et révocation testées. |
| Autorisation | Une mutation de B sur une annonce de A et un accès de C à une conversation A/B sont refusés par le serveur, indépendamment de l'interface. |
| Rendu | Aucune exécution des charges XSS du corpus de test dans annonces, messages ou réponses IA. |
| Ergonomie | Parcours essentiels utilisables au clavier et à 360/768/1280 pixels sans perte d'action ni défilement horizontal général. |
| Résilience | États réseau, 429 et réponse malformée couverts ; aucun faux succès affiché après refus ou timeout. |
| Qualité et preuves | Contrôles obligatoires de la pipeline réussis ; toute alerte bloquante corrigée ou exception justifiée et bornée selon la politique commune. |

Un scénario non exécuté ou une preuve absente empêche de déclarer le critère correspondant terminé.

## 14. Planning sans dates

1. **Priorité P0 — Cadrage :** contrats, routes, maquettes des états, menaces et politique de session. Sortie : validation collective des interfaces.
2. **P0 — Socle et compte :** architecture React, services, formulaires, session et tests de refus. Dépendance : contrat d'authentification.
3. **P0 — Marketplace :** accueil, catalogue, détail, publication, édition, suppression et profil. Sortie : parcours propriétaire et utilisateur tiers vérifiés.
4. **P1 — Interactions :** messagerie Web avec Anas, assistant avec Abdelilah. Dépendance : contrats et mocks disponibles.
5. **P0 — Stabilisation :** responsive, accessibilité, sécurité, corrections et preuves de pipeline.
6. **P1 — Soutenance :** parcours reproductibles, résultats réels et limites. Aucune amélioration visuelle ne doit retarder une correction bloquante.

## 15. Checklist détaillée

- [ ] Faire valider la matrice des routes et les DTO avec Mohamed.
- [ ] Documenter les frontières de confiance et les états de chaque page.
- [ ] Définir les composants de formulaire et messages accessibles.
- [ ] Prévoir une validation Zod cohérente avec les règles serveur.
- [ ] Réaliser inscription et connexion avec erreurs neutres.
- [ ] Garder le jeton d'accès en mémoire et traiter le renouvellement concurrent.
- [ ] Vérifier CSRF, révocation et nettoyage au changement de compte avec Mohamed.
- [ ] Bloquer les redirections externes non autorisées.
- [ ] Réaliser accueil, catalogue, recherche, filtres et pagination.
- [ ] Annuler les recherches obsolètes et tester l'ordre des réponses.
- [ ] Réaliser le détail et le cas d'annonce retirée.
- [ ] Réaliser création, modification et confirmation de suppression.
- [ ] Vérifier le refus serveur des mutations d'un tiers.
- [ ] Réaliser profil et liste des annonces personnelles.
- [ ] Intégrer événements, historique, accusés et reprise avec Anas.
- [ ] Tester l'échange Web-mobile et la déduplication.
- [ ] Intégrer le chat mock puis le contrat IA réel avec Abdelilah.
- [ ] Tester sortie IA malformée, liens inconnus et quota dépassé.
- [ ] Tester les charges XSS sur tous les contenus non fiables.
- [ ] Vérifier clavier, zoom, focus et trois largeurs cibles.
- [ ] Exécuter les tests prévus pendant la phase de développement.
- [ ] Corriger les alertes du client Web et fournir les preuves de nouvelle analyse.
- [ ] Relier les preuves aux pratiques Tier 1 exactes après réception de la source.
- [ ] Préparer le scénario de démonstration et les limites connues.

## 16. Soutenance

Présenter l'architecture React et la frontière de confiance avec l'API. Montrer un visiteur recherchant un produit, un propriétaire publiant puis modifiant son annonce, une session expirée correctement traitée et un refus serveur malgré une requête forgée.

Démontrer un échange Web-mobile avec Anas, une réponse IA sûre avec Abdelilah, une charge XSS rendue comme texte et un parcours au clavier. Présenter les résultats réellement obtenus, l'identifiant des preuves de pipeline, une correction suivie d'une nouvelle analyse et les limites non résolues. Distinguer explicitement mode mock et intégration réelle.
