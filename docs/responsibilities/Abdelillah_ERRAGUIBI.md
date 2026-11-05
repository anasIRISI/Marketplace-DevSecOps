# Cahier des charges individuel — ERRAGUIBI Abdelilah

**Domaine : Assistant IA et pipeline DevSecOps**  
**Projet : Secure Marketplace — AI-Augmented DevSecOps**  
**Établissement : FST Marrakech, Université Cadi Ayyad — Cycle ingénieur IRISI 3**  
**Module : Sécurité des applications et des services**  
**Sujet officiel : Projet 3 — Conception et mise en œuvre d'une pipeline DevSecOps augmentée par IA pour les applications de l'écosystème JavaScript (Express.js).**

Statut : cahier des charges prévisionnel. Aucun assistant, scanner, conteneur, workflow, déploiement ni mécanisme de protection n'est déclaré opérationnel. Les outils et seuils ci-dessous sont des choix prévus ou des propositions à valider, puis à démontrer par des résultats réels. Ce document ne crée aucune configuration ni aucun code.

## 1. Présentation du rôle

Abdelilah est responsable du module d'assistance IA et de l'orchestration de la pipeline DevSecOps. Il doit rendre l'assistant utile pour rechercher des produits publics, borner les capacités du LLM, puis organiser les contrôles automatiques et la traçabilité des corrections de l'ensemble du projet.

Il possède l'adaptateur fournisseur, le mode mock, les prompts, la politique d'outils, les validations IA, les tests adversariaux et la conception des workflows, analyses, rapports et indicateurs. Mohamed possède le socle Express, les autorisations et la base ; Hanane possède le client Web ; Anas possède le mobile et la messagerie. Abdelilah ne reprend pas le périmètre d'Anas.

**La sécurité, les tests et la correction des vulnérabilités concernent tous les membres.** Abdelilah organise la détection et les preuves ; chaque propriétaire de composant reproduit, corrige et teste ses défauts. La mise en place d'un scanner ne constitue ni une correction, ni une preuve de sécurité complète.

## 2. Technologies prévues et justification

| Technologie ou outil | Usage prévu et justification |
|---|---|
| Node.js, Express.js, TypeScript | Module IA intégré au backend existant, contrats typés et services séparés du transport. |
| Zod | Validation des entrées, appels d'outils et sorties ; les réponses du fournisseur restent des données non fiables. |
| Adaptateur LLM et client HTTP/SDK à choisir | Isoler fournisseur, authentification, délai, modèle et erreurs ; choix selon coût, confidentialité et compatibilité. |
| Fournisseur mock déterministe | Développement et tests reproductibles sans coût ni réseau ; panne et sortie hostile simulables. |
| GitHub Actions | Orchestration future de contrôles sur propositions de changement et branche principale. |
| Semgrep | SAST du code ; recherche de motifs de vulnérabilités avec règles et exclusions justifiées. |
| npm audit | SCA des dépendances npm déclarées dans les lockfiles ; limites sur code natif et images à expliciter. |
| Gitleaks | Détection de secrets dans les contenus et l'historique analysés ; un secret détecté doit être révoqué s'il a été exposé. |
| OWASP ZAP | Analyse dynamique d'une application démarrée sur environnement isolé et autorisé. |
| SonarQube, optionnel | Analyse de qualité complémentaire si installation, accès et coût d'entretien sont compatibles avec le projet. |
| Docker et Docker Compose | Empaquetage et environnement d'intégration reproductible avec API, base et dépendances nécessaires. |
| Outils de tests propres aux composants | Réutiliser les tests fournis par Hanane, Mohamed et Anas ; ne pas imposer un second moteur sans besoin. |

Fixer et consigner les versions et identifiants immuables des composants lors de l'implémentation. Aucun fournisseur, compte, abonnement, coût ou service hébergé n'est supposé disponible sans vérification.

# Partie A — Assistant IA

## 3A. Architecture du module IA

Flux cible : interface Web → `POST /api/v1/assistant/chat` → middleware de session/quota → validation → orchestrateur IA → adaptateur mock ou fournisseur → proposition d'outil éventuelle → vérification serveur → recherche de produits publics → validation de sortie → réponse au client.

Le module se compose d'un routeur, d'un contrôleur, d'un service d'orchestration, d'une interface fournisseur, d'un adaptateur mock, d'un adaptateur distant, d'un catalogue d'outils autorisés, de schémas et de modèles de prompts versionnés. Mohamed valide l'intégration au socle Express et les accès aux services métier.

### Contrat de chat proposé

- Entrée : texte utilisateur obligatoire et identifiant éventuel d'une conversation IA appartenant à la session. Limite initiale de 2 000 caractères par entrée, dix tours d'historique au maximum et volume total de contexte borné ; seuils à mesurer et à valider.
- Le client ne fournit ni prompt système, ni rôle privilégié, ni outil libre, ni modèle ou URL de fournisseur arbitraire. Tout champ inconnu sensible est rejeté.
- Sortie : texte, liste de références de produits autorisés, identifiant de requête et mode `mock` ou `provider`. Les erreurs utilisent le format commun de l'API.
- Historique initial en mémoire et borné côté service ou transmis selon un contrat revu ; aucune persistance durable de conversations IA sans politique d'accès, de rétention et d'effacement explicite. Un identifiant de chat, s'il est retenu, n'est jamais une autorisation.
- Le service ne doit pas recevoir les tokens, mots de passe, courriels privés, messages acheteur-vendeur ou secrets de pipeline. Le catalogue public doit être filtré avant transmission au modèle.

### Outil de recherche autorisé

L'outil initial de recherche accepte des paramètres structurés : texte, catégorie, bornes de prix et limite. Il appelle le service `products` de Mohamed, avec visibilité publique active et liste de champs restreinte. Cible : dix produits au maximum par appel et deux appels d'outil par tour, sans boucle ouverte.

Le LLM propose un appel ; le serveur décide s'il est autorisé. Aucun SQL libre, accès fichiers, exécution de commande, téléchargement d'URL, envoi de message, modification de produit ou paiement n'est accessible au modèle. Un outil inconnu ou des arguments invalides doivent être refusés et journalisés sans contenu sensible.

## 4A. Fonctionnalités IA à développer

| Référence | Réalisation détaillée | Sécurité et tests |
|---|---|---|
| IA-01 — Endpoint Express | Route protégée, validation du corps, service d'orchestration et réponse normalisée. | Session active et quotas ; tester invité, entrée vide/trop longue, champ système injecté et JSON invalide. |
| IA-02 — Mode mock | Réponses déterministes : texte simple, recherche de produits, quota, timeout, sortie malformée et contenu hostile simulé. | Aucun appel fournisseur ni clé nécessaire ; tester reproductibilité et identification explicite du mode. |
| IA-03 — Adaptateur fournisseur | Interface commune de génération, conversion des messages, délais, compteurs et classification d'erreurs. | Clé uniquement serveur, destination autorisée, réponse bornée, retries limités ; tester panne, dépassement de temps et format incompatible. |
| IA-04 — Prompts | Séparer consignes système, demande utilisateur et données catalogue ; versionner et expliquer chaque changement. | Un prompt ne contient aucun secret et ne remplace pas un contrôle d'autorisation ; tester injection directe et indirecte via description produit. |
| IA-05 — Recherche de produits | Transformer la demande en paramètres validés, appeler le service autorisé et retourner des références existantes. | Ne transmettre que produits publics actifs, champs minimaux ; tester produit retiré, identifiant inconnu et demande d'une donnée privée. |
| IA-06 — Outils contrôlés | Catalogue fermé d'outils et limites du nombre d'appels, taille des résultats et durée. | Refus par défaut, pas d'outil dynamique fourni par le client ; tester outil inconnu, arguments excessifs et récursion. |
| IA-07 — Sorties sûres | Valider structure, longueur et références, puis retourner du texte au client ; réponse de repli en cas d'échec. | Ne pas exécuter HTML, SQL ou lien suggéré ; tester XSS, référence inventée et sortie tronquée. |
| IA-08 — Quotas et erreurs | Limites utilisateur/IP, coût et concurrence, annulation fournisseur, messages neutres et corrélation. | Cibles initiales : dix requêtes/minute/utilisateur, une requête IA simultanée/utilisateur, délai total de vingt secondes ; tester 429, panne et saturation contrôlée. |
| IA-09 — Confidentialité et documentation | Documenter données transmises, fournisseur retenu, paramètres de conservation connus et limites du modèle. | Minimisation et logs sans prompts complets ; tester demandes de secrets et lecture des données d'un autre compte. |

Prévoir un budget maximal d'entrée/sortie en jetons adapté au modèle sélectionné, un plafond de coût par période et un mécanisme de coupure. Les chiffres définitifs dépendront du fournisseur et des mesures ; aucun coût ou délai réel n'est garanti par ce document.

## 5A. Tâches de développement IA

### IA-T1 — Fixer les contrats et le modèle de confiance

- **Objectif :** borner les capacités avant de brancher un modèle.
- **Actions/composants :** spécifier entrée/sortie, données autorisées, adaptateur fournisseur, règles d'outils et mode mock ; revoir les flux avec Mohamed et Hanane.
- **Contrôles :** aucune donnée privée par défaut, refus des rôles/outils libres, analyse des frontières fournisseur et catalogue.
- **Tests :** exemples de contrats, champs inconnus et jeu d'injections initial.
- **Livrable :** contrat IA, schéma de flux, catalogue d'outils et registre de risques.
- **Validation :** chaque donnée sortante et chaque capacité du modèle possède une justification et une règle de contrôle serveur.

### IA-T2 — Livrer le module mock et l'endpoint

- **Objectif :** permettre l'intégration et les tests sans dépendre d'un fournisseur.
- **Actions/composants :** route, contrôleur, orchestrateur et adaptateur mock ; réponses de succès, refus, délai et sortie hostile.
- **Contrôles :** authentification, quotas, schémas et format d'erreur commun ; mode toujours identifiable.
- **Tests :** contrat HTTP, absence d'appel réseau externe et tous les états IA-01/IA-02/IA-08.
- **Livrable :** module mock et tests déterministes, puis intégration Web avec Hanane.
- **Validation :** le parcours complet fonctionne sans clé fournisseur ; une sortie invalide n'est jamais transmise comme réponse valide.

### IA-T3 — Brancher la recherche et sécuriser les outils

- **Objectif :** rendre l'assistant utile sans lui attribuer de privilèges.
- **Actions/composants :** outil de recherche, schémas d'arguments, service produits public, références de sortie et limites d'itération.
- **Contrôles :** filtres de visibilité imposés côté serveur, pas de SQL ni URL libre, sortie minimale et refus par défaut.
- **Tests :** produit retiré, accès à compte tiers, appel d'outil arbitraire, arguments malformés, injection dans description et dépassement d'appels.
- **Livrable :** IA-05/IA-06, tests d'outils et preuves de refus.
- **Validation :** seules les références publiques existantes sont retournées ; aucune mutation ni donnée privée n'est accessible au LLM.

### IA-T4 — Intégrer le fournisseur et la politique de prompts

- **Objectif :** proposer une génération réelle maîtrisée lorsque le fournisseur est disponible.
- **Actions/composants :** adaptateur distant, prompts versionnés, gestion des erreurs, délais, annulation et budget.
- **Contrôles :** secret serveur, destination fixe, aucune donnée sensible en contexte ou journaux, validation complète des sorties.
- **Tests :** tests d'intégration bornés sur fournisseur réel si accès disponible, et tests simulés de 429, 5xx, timeout et réponse malformée.
- **Livrable :** IA-03/IA-04/IA-07, fiche fournisseur et mesure réelle des limites.
- **Validation :** échec fournisseur sûr et identifié ; distinction explicite entre tests mock et résultats d'intégration réelle. Si l'accès manque, déclarer l'adaptateur non validé.

### IA-T5 — Évaluer les menaces et produire les preuves

- **Objectif :** rendre les protections et leurs limites démontrables.
- **Actions/composants :** corpus adversarial versionné, tests fonctionnels/sécurité, mesures de quota/coût/latence et documentation.
- **Contrôles :** pas de secret dans prompts ou rapports, suivi des incidents, revue des résultats par Mohamed et Hanane.
- **Tests :** injection directe/indirecte, extraction de secrets, confusion d'utilisateur, sortie hostile et consommation excessive.
- **Livrable :** rapport IA, résultats par scénario et correctifs avec nouvelle exécution.
- **Validation :** aucune exécution d'outil interdit, fuite de donnée privée ou interprétation active de sortie dans le corpus ; limites de résistance clairement décrites.

## 6A. Organisation prévisionnelle IA

Arborescence documentaire uniquement, au sein du backend partagé de Mohamed.

```text
backend/src/modules/assistant/
  routes/                contrat HTTP
  controllers/           adaptation requête/réponse
  services/              orchestration et limites
  providers/             interface, mock et fournisseur retenu
  prompts/               modèles de consignes versionnés
  tools/                 registre fermé et recherche publique
  schemas/               entrées, arguments et sorties
backend/tests/assistant/
  unit/                  orchestration, quotas et schémas
  integration/           endpoint et adaptateur
  security/              injections, fuite et outils interdits
```

# Partie B — DevSecOps

## 3B. Architecture prévisionnelle de la pipeline CI/CD

La future pipeline devra séparer les contrôles rapides sur changement, les analyses nécessitant une application démarrée et la préparation d'une livraison. L'organisation suivante décrit des workflows à réaliser ultérieurement ; aucun fichier GitHub Actions n'est créé par cette mission.

1. **Sur proposition de changement :** récupération de la révision, vérification des secrets, installation reproductible depuis les lockfiles disponibles, types/qualité, tests et analyses SAST/SCA. Les opérations indépendantes pourront être parallélisées avec ressources bornées.
2. **Sur branche principale après intégration :** mêmes contrôles obligatoires, production des artefacts et des images, lancement de l'environnement Compose isolé, migrations de test, vérification de disponibilité et parcours d'intégration.
3. **DAST :** scan passif de base sur environnement de test, puis scan actif et authentifié lorsqu'il est prévu, configuré et autorisé ; périmètre limité à la cible isolée et à ses comptes fictifs.
4. **Rapports et décision :** agréger statuts, conserver résultats expurgés même lorsqu'un job échoue et appliquer les quality gates ; arrêt de la promotion si contrôle bloquant absent, inexploitable ou en échec.
5. **CD conditionnelle :** préparer une livraison identifiée par révision et empreinte d'image, puis déployer uniquement vers la cible et selon les permissions qui seront définies par l'équipe. Prévoir approbation d'environnement si requise, smoke tests et retour à la version précédente. Ne pas prétendre disposer déjà d'une production.
6. **Analyses périodiques ou manuelles :** réanalyser dépendances et règles afin de détecter les vulnérabilités nouvellement connues, avec fréquence à convenir ; ne pas limiter la sécurité aux changements de code.

### Sécurité de l'orchestrateur

Prévoir permissions minimales du jeton de workflow, dépendances/actions épinglées à des identifiants immuables, secrets limités aux jobs nécessaires, jobs avec délais et nettoyage garanti des environnements temporaires. Un changement externe non fiable ne reçoit pas de secret privilégié. Ne pas exécuter du code de proposition externe avec un contexte de workflow privilégié.

Séparer les caches par confiance et éviter qu'un artefact non vérifié soit promu comme livrable. Les rapports d'analyse doivent eux-mêmes être considérés comme potentiellement sensibles. Les règles de protection de branche devront rendre obligatoires les contrôles retenus, afin qu'un résultat rouge ne soit pas contourné par simple intégration manuelle.

## 4B. Contrôles, outils et résultats attendus

### DEV-01 — Qualité du code et types

- **Objectif et risques :** détecter erreurs de types, règles de qualité et incohérences avant intégration ; cette étape ne prouve pas l'absence de faille.
- **Emplacement/déclenchement :** phase rapide, chaque proposition et branche principale ; utiliser les scripts fournis ultérieurement par les propriétaires des composants.
- **Résultats :** diagnostic par composant et statut explicite.
- **Échec :** erreur de types ou règle bloquante, script obligatoire absent, exécution interrompue ou rapport inutilisable.
- **Preuves :** révision, outils/versions, journaux expurgés et résultat horodaté.
- **Corrections :** corriger le composant, préciser une règle trop large avec justification, puis relancer le même contrôle.

### DEV-02 — Tests automatisés

- **Objectif et risques :** détecter régressions fonctionnelles, erreurs de permission et défauts de résilience.
- **Emplacement/déclenchement :** tests unitaires sur chaque changement ; intégration PostgreSQL, temps réel et parcours sur environnement dédié ; mock IA dans le chemin reproductible.
- **Résultats :** nombre de tests réussis/échoués/ignorés, couverture mesurée lorsqu'utile et détail des scénarios critiques.
- **Échec :** tout test obligatoire en échec ou non exécuté sans décision documentée ; la disponibilité d'un fournisseur réel ne doit pas rendre les tests mock aléatoires.
- **Preuves :** rapports par suite, environnement, jeux fictifs et limites des tests mobiles manuels.
- **Corrections :** reproduire, corriger et ajouter une régression pertinente ; traiter un test instable au lieu de le neutraliser silencieusement.

### DEV-03 — SAST avec Semgrep

- **Objectif et risques :** repérer motifs d'injection, usages dangereux et défauts de code couverts par les règles retenues.
- **Emplacement/déclenchement :** après disponibilité des sources, sur propositions, branche principale et réanalyses manuelles/périodiques.
- **Résultats :** constats avec fichier, emplacement, règle, gravité et contexte ; couverture réelle des répertoires à documenter.
- **Échec :** constat de sécurité bloquant selon la politique collective, règle critique applicable ou échec technique du scanner ; aucun scan vide ne vaut succès.
- **Preuves :** rapport JSON/SARIF selon la configuration retenue, version du scanner, règles et exclusions, révision analysée.
- **Corrections :** supprimer l'usage dangereux, renforcer validation/autorisation et tester ; faux positif exclu uniquement de façon ciblée avec justification et revue.

### DEV-04 — SCA avec npm audit

- **Objectif et risques :** identifier les vulnérabilités connues des dépendances npm directes et transitives, y compris de développement dans le périmètre déclaré.
- **Emplacement/déclenchement :** après résolution reproductible des dépendances, à chaque changement et périodiquement même sans modification du code.
- **Résultats :** dépendance concernée, avis, gravité, chaîne de dépendances et correction disponible lorsqu'indiquée.
- **Échec :** vulnérabilité haute ou critique selon la politique initiale, avis de moindre gravité mais bloquant par son impact, ou impossibilité d'obtenir un résultat exploitable.
- **Preuves :** rapport structuré, lockfile/révision analysés, version de l'outil, périmètre et exceptions éventuelles.
- **Corrections :** mettre à jour, remplacer ou retirer la dépendance, vérifier compatibilité et relancer tests/scan ; éviter une correction forcée non évaluée.
- **Limite :** cette analyse n'atteste pas la sécurité du système de base d'une image ni de toutes les dépendances natives mobiles.

### DEV-05 — Secret Scanning avec Gitleaks

- **Objectif et risques :** détecter clés, jetons et autres secrets accidentellement présents dans les contenus ou l'historique couvert.
- **Emplacement/déclenchement :** au début de chaque pipeline ; prévoir un passage initial couvrant l'historique disponible, puis périmètre de changement et contrôles complets selon la politique retenue.
- **Résultats :** constats expurgés avec règle et emplacement, sans reproduire la valeur sensible dans les artefacts publics.
- **Échec :** secret potentiel non qualifié ou secret confirmé ; échec technique et périmètre insuffisant non signalé interdisent de conclure à l'absence de secret.
- **Preuves :** rapport masqué, portée de l'historique analysé, version et règle ; preuve de révocation expurgée en cas d'exposition réelle.
- **Corrections :** révoquer/renouveler le secret exposé, retirer son utilisation du dépôt et externaliser l'injection ; tout nettoyage d'historique éventuel sera traité séparément par l'équipe. Retirer la chaîne du dernier fichier ne suffit pas à invalider un secret compromis.

### DEV-06 — DAST avec OWASP ZAP

- **Objectif et risques :** observer l'application en fonctionnement : configurations Web, en-têtes, erreurs, injections et autres défauts couverts par les règles et chemins parcourus.
- **Emplacement/déclenchement :** après démarrage et disponibilité de l'environnement de test ; scan de base passif puis scan actif/authentifié sur branche principale, demande manuelle ou fréquence approuvée.
- **Résultats :** alertes, URLs et méthodes couvertes, contexte d'authentification, niveau de confiance et limites. Importer le contrat API et préparer des parcours pour améliorer la couverture.
- **Échec :** alerte haute, alerte moyenne confirmée présentant un impact de sécurité, perte d'authentification rendant le scan annoncé authentifié invalide, cible indisponible ou scan inexploitable.
- **Preuves :** rapports HTML/structurés expurgés, version, cible exacte, révision, règles et identités fictives ; aucun token dans un rapport diffusé.
- **Corrections :** reproduire le constat, corriger côté API/Web selon propriétaire, ajouter un test et rescanner le même périmètre.
- **Limites :** un scan passif ne remplace pas un scan actif ; ZAP seul ne prouve pas les autorisations métier, la sécurité Socket.IO, mobile ou LLM. Compléter par les tests A/B/C et les scénarios dédiés.

### DEV-07 — SonarQube si retenu

- **Objectif et risques :** compléter l'analyse de maintenabilité, duplication et qualité, et les constats de sécurité couverts ; ne pas remplacer Semgrep ou les tests.
- **Emplacement/déclenchement :** après tests et collecte de couverture, sur changements et branche principale, seulement si l'équipe retient et peut exploiter le service.
- **Résultats :** tableau de qualité et quality gate dont les seuils exacts seront fixés avant adoption.
- **Échec :** gate retenu en échec ou résultat non récupérable lorsque le contrôle est déclaré obligatoire ; s'il n'est pas retenu, le signaler explicitement comme hors pipeline.
- **Preuves :** export ou capture expurgée, paramètres du gate, version et révision.
- **Corrections :** traiter les défauts pertinents et vérifier l'analyse suivante ; ne pas modifier le seuil uniquement pour rendre un résultat vert.

### DEV-08 — Docker et Docker Compose

- **Objectif et risques :** rendre l'environnement reproductible et détecter défauts d'assemblage, dépendances manquantes et absence de disponibilité ; ce ne sont pas des scanners de vulnérabilités.
- **Emplacement/déclenchement :** construction après contrôles rapides, puis environnement isolé pour intégration/DAST ; production d'image de livraison après gates.
- **Résultats :** images identifiées, services joignables, migrations de test réussies et état de santé mesuré.
- **Échec :** construction impossible, santé non atteinte dans le délai défini, tests d'intégration en échec ou secret incorporé à l'image.
- **Preuves :** révision, empreintes d'images, base utilisée, résultats de santé et journaux expurgés ; secrets exclus des couches et contextes de construction.
- **Corrections :** réduire les privilèges et dépendances, corriger assemblage/migrations, revoir base et paramètres ; prévoir utilisateur non privilégié, ports minimaux et séparation des secrets.
- **Nettoyage :** détruire uniquement les ressources temporaires explicitement identifiées du futur environnement de test après collecte des preuves, jamais des données partagées par défaut.

### DEV-09 — Quality Gates et gestion des vulnérabilités

Politique initiale à valider avant le premier usage :

| Contrôle | Condition de passage |
|---|---|
| Qualité et tests | Tous les contrôles et scénarios critiques obligatoires réussissent ; aucun job requis ignoré ou techniquement invalide. |
| Secrets | Aucun secret confirmé actif ni constat potentiel non qualifié dans le périmètre analysé. |
| SAST/SCA | Aucun constat haut/critique ouvert sans exception bornée approuvée ; tout défaut d'accès, injection ou fuite avéré reste bloquant selon son impact réel. |
| DAST | Aucune alerte bloquante non résolue ; couverture et authentification du scan vérifiées. |
| Artefacts | Révision, rapports et empreintes cohérents ; absence de secret dans rapports et images. |
| Promotion | Gates obligatoires passés ; exceptions documentées et non expirées ; cible de livraison et retour arrière définis. |

Une exception doit comporter constat, impact, justification, propriétaire, mitigation compensatoire, périmètre, échéance et revalidation. Elle ne doit pas servir à conserver un secret exposé actif. Sa revue ne revient pas à son seul auteur. À expiration, le constat redevient bloquant. Les faux positifs doivent être qualifiés et exclus au plus petit périmètre possible.

Cycle d'un constat : détection → qualification/reproduction → attribution au propriétaire → correction → test de régression → nouvelle analyse sur révision corrigée → clôture avec preuves. Ne pas fermer un ticket sur une simple promesse de correction.

### DEV-10 — Rapports, preuves et performance

Chaque exécution doit relier révision, environnement, outil/version, règles, déclenchement, début/fin, portée, résultat et décision. Conserver séparément rapport brut à accès limité si nécessaire et version expurgée pour soutenance. Proposition de rétention : trente jours pour rapports courants, conservation de la campagne finale jusqu'à validation académique selon la politique de l'équipe ; ne pas inclure de données personnelles réelles.

Mesurer durée totale, durée par étape, attente, taux d'échec technique, tests instables, nombre de constats par gravité, délai de qualification/correction et temps de nouvelle analyse. Comparer une exécution sans cache et une avec cache sur ressources comparables. Cibles initiales à confirmer après référence mesurée : chemin rapide en moins de quinze minutes, campagne DAST isolée en moins de vingt minutes ; ces cibles ne sont pas des performances déjà obtenues.

Paralléliser les contrôles indépendants, mettre en cache uniquement les données appropriées et fixer les délais des jobs. Ne pas réduire les contrôles ou supprimer les rapports pour améliorer artificiellement la durée. Un nombre d'alertes plus faible ne prouve pas une meilleure sécurité si la couverture a diminué.

### DEV-11 — Pratiques Tier 1 et analyse Tier 2

La demande mentionne **les onze pratiques Tier 1 de Prates et Pereira**, sans fournir la publication ni leurs intitulés exacts. Il est interdit d'inventer cette liste ou d'affirmer que les outils ci-dessus constituent ces onze pratiques.

Abdelilah devra obtenir la référence scientifique ou la grille officielle du module, vérifier édition, définitions et classement, puis construire une matrice avec exactement onze entrées issues de cette source : identifiant/intitulé exact, exigence du projet, contrôle concret, propriétaire, preuve attendue, statut prévu/réalisé/vérifié, limite et justification de non-applicabilité si nécessaire. Tant que la source manque, le statut de couverture Tier 1 reste **à confirmer**.

Analyser les pratiques Tier 2 uniquement après identification de leur classement dans la même source. Pour chaque pratique candidate : bénéfice, coût, prérequis, risque couvert, effort, preuve et décision motivée. Une SBOM, une signature d'image, un scan d'image ou une analyse d'infrastructure peuvent être des pistes techniques optionnelles ; elles ne doivent pas être étiquetées Tier 2 sans vérification scientifique. Les options ne doivent pas retarder les contrôles obligatoires et les corrections bloquantes.

## 5B. Tâches de développement DevSecOps

### DEV-T1 — Définir la politique et le périmètre

- **Objectif :** rendre les résultats interprétables et les décisions traçables.
- **Actions/composants :** inventaire des composants, événements de déclenchement, contrôles obligatoires, seuils, propriétaires, données de test et politique de preuves ; rechercher la source Tier 1 exacte.
- **Contrôles :** moindre privilège, secrets de test uniquement, séparation des changements non fiables et des jobs privilégiés.
- **Tests :** revue collective des cas succès/échec/absence de rapport et d'une exception expirée.
- **Livrable :** architecture CI/CD, matrice des gates, responsabilités et traçabilité scientifique à compléter.
- **Validation :** chaque contrôle possède périmètre, résultat, seuil, propriétaire et preuve ; la liste Tier 1 non vérifiée n'est jamais présentée comme couverte.

### DEV-T2 — Construire les contrôles rapides

- **Objectif :** fournir un retour exploitable sur chaque changement.
- **Actions/composants :** intégrer ultérieurement DEV-01 à DEV-05 et tests de chaque membre ; rapports même en cas d'échec et verrouillage des versions.
- **Contrôles :** permissions minimales, masquage des secrets, caches séparés et erreurs techniques bloquantes.
- **Tests :** changements de démonstration bénins déclenchant un échec de qualité/test ; fixtures de scanner sans secret réel et hors artefact final applicatif.
- **Livrable :** workflows à développer plus tard, rapports et preuve de blocage.
- **Validation :** une exécution correcte passe et un défaut contrôlé échoue au gate attendu, sans faux succès dû à une étape ignorée.

### DEV-T3 — Préparer l'environnement intégré et DAST

- **Objectif :** analyser les composants en fonctionnement de façon reproductible.
- **Actions/composants :** DEV-06/DEV-08, Compose isolé, base fictive, migrations de test, santé, comptes dédiés et contexte ZAP authentifié.
- **Contrôles :** cible autorisée, réseau restreint, secrets éphémères, collecte expurgée et nettoyage ciblé.
- **Tests :** cible indisponible, perte d'authentification de scan, anomalie contrôlée sur cible de test et parcours A/B/C.
- **Livrable :** futur environnement CI intégré, rapports de couverture et résultats de scan.
- **Validation :** rapport rattaché à la révision testée, authentification vérifiée et échec bloquant si le scan prévu n'a pas réellement eu lieu.

### DEV-T4 — Organiser corrections, artefacts et livraison

- **Objectif :** relier détection à remédiation et empêcher la promotion d'un résultat invalide.
- **Actions/composants :** DEV-09, tickets par propriétaire, liens de revalidation, empreintes d'images, smoke tests et procédure de retour arrière sur cible à définir.
- **Contrôles :** artefacts issus de la révision validée, exceptions revues et bornées, aucun secret exposé laissé actif.
- **Tests :** gate rouge empêchant promotion, exception expirée, artefact incohérent et échec de smoke test.
- **Livrable :** registre de vulnérabilités, procédure de promotion et preuves d'une correction complète.
- **Validation :** une alerte représentative est détectée, corrigée, retestée et clôturée avec nouvelle analyse ; une livraison invalide reste bloquée.

### DEV-T5 — Mesurer et préparer la démonstration

- **Objectif :** justifier les choix avec des mesures et des preuves consultables.
- **Actions/composants :** DEV-07 si retenu, DEV-10/DEV-11, durées par étape, matrice Tier 1 exacte après réception et étude des options Tier 2.
- **Contrôles :** comparaison à périmètre constant, rapports sans secrets, distinction entre contrôle installé, exécuté et validé.
- **Tests :** répétition de campagne finale, contrôle des liens preuve/révision et vérification des limites déclarées.
- **Livrable :** dossier de soutenance, métriques et bilan de couverture réel.
- **Validation :** tout résultat présenté peut être retrouvé ; toute pratique non prouvée ou source manquante est marquée explicitement.

## 6B. Organisation prévisionnelle DevSecOps

Arborescence descriptive uniquement. Aucun workflow, Dockerfile, fichier Compose, règle de scanner ou autre configuration n'est créé par ce cahier des charges.

```text
.github/workflows/       futurs workflows qualité, sécurité, intégration et livraison
devsecops/
  policies/              gates, exceptions et règles de preuve
  scans/                 paramètres futurs Semgrep, Gitleaks et ZAP
  environments/          description des cibles de test autorisées
  evidence/              index des preuves, sans secrets ni données réelles
infra/                   futurs fichiers de conteneurisation et orchestration locale
docs/                    guides, modèle de menaces et matrice scientifique
```

# Exigences transversales aux parties A et B

## 7. Sécurité et référentiels

L'analyse IA ci-dessous est une application au projet du [OWASP LLM Top 10:2025](https://genai.owasp.org/llm-top-10/). Elle ne garantit pas l'immunité aux injections et doit être étayée par les tests.

| Risque LLM 2025 | Mesures prévues et limites |
|---|---|
| LLM01 — Injection de prompt | Séparer instructions et données, considérer descriptions/entrées comme hostiles, contrôler les outils hors modèle ; le prompt seul ne constitue pas une frontière de sécurité. |
| LLM02 — Divulgation d'informations sensibles | Minimiser et filtrer contexte, interdire données privées, expurger logs et preuves ; tester demandes d'extraction. |
| LLM03 — Chaîne d'approvisionnement | Identifier fournisseur, modèle, SDK et changements ; revoir dépendances et destination des données. |
| LLM04 — Empoisonnement des données et modèles | Ne pas entraîner sur des données arbitraires ; considérer le catalogue utilisateur comme non fiable et limiter son influence aux données de recherche. |
| LLM05 — Traitement inadéquat des sorties | Schéma, longueur, références validées et texte non exécutable ; aucune concaténation dans SQL, HTML ou commande. |
| LLM06 — Autonomie excessive | Outils fermés en lecture, permissions serveur minimales et nombre d'itérations borné ; aucune mutation autonome. |
| LLM07 — Fuite du prompt système | Aucun secret ou mécanisme d'autorisation caché dans les consignes ; tester extraction et accepter que le texte d'un prompt puisse être divulgué. |
| LLM08 — Faiblesses des vecteurs/embeddings | Pas de base vectorielle prévue dans le socle ; déclarer cette limite de périmètre et refaire l'analyse si une RAG est ajoutée. |
| LLM09 — Désinformation | Références produits vérifiées, prix/état venant de l'API, signaler les limites ; ne pas présenter une réponse comme garantie de disponibilité ou d'achat. |
| LLM10 — Consommation non bornée | Quotas, concurrence, taille du contexte, jetons de sortie, délais, nombre d'outils et budget ; tests des plafonds et arrêt contrôlé. |

Pour [OWASP Web Top 10:2025](https://top10.owasp.org/2025/0x00_2025-Introduction/), coordonner contrôles d'accès A01 avec Mohamed, configuration A02, chaîne logicielle A03, secrets/transport A04, injections A05, conception A06, session A07, intégrité A08, logs/alertes A09 et gestion des exceptions A10.

Pour [OWASP API Security Top 10:2023](https://api-security.owasp.org/editions/2023/en/0x11-t10/), l'assistant doit conserver l'autorisation objet/fonction/propriété, limiter les ressources et valider le fournisseur externe ; le LLM n'est jamais une identité privilégiée. Pour [OWASP Mobile Top 10:2024](https://owasp.github.io/www-project-mobile-top-10/), collecter les preuves d'Anas sur stockage, transport, dépendances et paramètres de build, sans prétendre que ZAP vérifie un binaire mobile.

## 8. Threat Modeling — STRIDE

Actifs : clé fournisseur, prompts, données publiques autorisées, budgets, secrets CI, sources, rapports, images et jetons de l'orchestrateur. Frontières : utilisateur/API, orchestrateur IA/fournisseur, modèle/outils, dépôt/runner, runner/cible DAST et artefact/livraison.

| Menace | Scénario | Protection et preuve |
|---|---|---|
| Usurpation | Appel IA sous compte tiers ou workflow utilisant un token trop puissant | Session/API, permissions minimales et secrets ciblés ; tests de session et revue des permissions. |
| Altération | Description produit injectant un outil ou cache CI contaminé | Arguments validés, outils fermés, cache par confiance, artefact identifié ; tests d'injection et de provenance. |
| Répudiation | Résultat présenté sans révision ou correction sans nouvelle analyse | Corrélation et index de preuves ; contrôle révision/outil/rapport avant clôture. |
| Divulgation | Clé dans prompt, log ou image ; rapport contenant un token | Minimisation, injection serveur, masquage, contrôle d'accès aux artefacts et scan ; preuve expurgée et révocation si exposition. |
| Déni de service | Boucle d'outils, coût LLM ou jobs sans fin | Budgets, quotas, concurrence et délais ; tests de limites et mesures par étape. |
| Élévation de privilèges | LLM exécutant une mutation ou code externe exécuté avec secrets CI | Aucun outil de mutation, contrôles hors modèle et contexte CI isolé ; tentative interdite et scénario de workflow non fiable revu. |

Consigner impact, vraisemblance, propriétaire, mitigation, test et risque restant. Revoir après ajout d'outil, fournisseur, environnement de livraison ou workflow privilégié.

## 9. Tests à prévoir

- **IA unitaires :** schémas, registre d'outils, limites d'itérations, budget, choix mock/fournisseur et classification des erreurs.
- **IA intégration :** endpoint avec session/quota, recherche publique réelle sur base fictive, DTO de sortie et erreurs fournisseur simulées ; essais fournisseur réel limités et identifiés séparément.
- **IA fonctionnels :** recherche par catégorie/prix, absence de résultat, référence de produit retiré, état de quota et mode indisponible avec Hanane.
- **IA sécurité :** injections directes et via catalogue, extraction de clés/prompt, autre utilisateur, outil interdit, XSS en sortie, référence inventée, taille excessive et boucle d'appels.
- **Pipeline :** exécution verte de référence, défaut contrôlé de qualité, test en échec, fixture de scanner sans secret réel, cible DAST indisponible, rapport absent et gate bloquant.
- **Intégration système :** migrations, disponibilité, parcours Web/API/mobile/temps réel avec comptes fictifs ; distinction entre tests automatiques et manuels.
- **Résilience :** fournisseur lent, 429, DB indisponible, job expiré, perte d'authentification ZAP et collecte des preuves après échec.

Les simulations de vulnérabilités doivent rester dans des fixtures de test ou une cible isolée prévue à cet effet lors de l'implémentation ; ne pas introduire un défaut exploitable dans l'application destinée à la livraison. Chaque test doit conserver résultat attendu/réel et limites de portée.

## 10. Intégration DevSecOps et traçabilité

Abdelilah maintient le lien exigence → risque → contrôle → responsable → test/analyse → rapport → correctif → nouvelle preuve. Le statut doit distinguer prévu, configuré, exécuté, résultat exploitable et validé ; un outil installé n'est pas un contrôle prouvé.

Hanane fournit les preuves Web et corrige le client ; Mohamed fournit les preuves API/DB et corrige les services ; Anas fournit les preuves mobile/temps réel et corrige sa partie avec Mohamed pour le serveur. Abdelilah corrige l'assistant et les workflows et vérifie que les résultats de tous les membres sont intégrables et expurgés.

La couverture des pratiques Tier 1 reste à confirmer tant que la source exacte n'est pas identifiée. Après obtention, établir les onze correspondances et distinguer absence de contrôle, preuve manquante et non-applicabilité justifiée. Les options Tier 2 doivent disposer d'une décision d'adoption ou de report motivée.

## 11. Collaboration et dépendances

| Partenaire | Interface et responsabilité | Condition de passage |
|---|---|---|
| Mohamed | Montage Express, session, service de recherche publique, quotas partagés et environnement PostgreSQL | Autorisation hors LLM, données filtrées, erreurs et santé documentées. |
| Hanane | Contrat de chat, mode mock, états de quota et rendu de texte sûr | Parcours complet mock et réponse hostile sans exécution côté navigateur. |
| Anas | Tests mobile/Socket.IO, limites d'automatisation et rapports de plateforme | Résultats traçables et non confondus avec DAST Web/API. |
| Tous | Scripts futurs de qualité/tests, tickets, politiques et pratiques scientifiques | Propriétaire de correction nommé, preuve de revalidation et décisions collectives sur gates/exceptions. |

Commencer par les contrats et le mock IA pour ne pas bloquer le Web. Préparer les étapes CI à partir des scripts réellement fournis par les membres ; ne pas inventer un succès de test sur un composant encore absent.

## 12. Livrables attendus

1. Module IA, mock, adaptateur fournisseur, prompts, outils et validations à développer ultérieurement.
2. Contrat du chat et documentation de confidentialité, quotas, coût et limites de résistance.
3. Corpus et résultats des tests fonctionnels/adversariaux, séparant mock et fournisseur réel.
4. Workflows, conteneurisation et paramètres de scanners à créer dans une future phase d'implémentation, conformément à la politique validée.
5. Rapports SAST/SCA/secrets/DAST, qualité/tests et SonarQube uniquement si retenu.
6. Registre de vulnérabilités, exceptions, corrections, nouvelles analyses et index de preuves expurgées.
7. Mesures de performance, procédure de livraison/retour arrière selon cible définie, matrice Tier 1 exacte et étude Tier 2 sourcée.
8. Support de démonstration basé sur les résultats réellement exécutés.

## 13. Critères d'acceptation mesurables

| Domaine | Condition de validation |
|---|---|
| Contrat IA | IA-01 à IA-09 disposent d'un scénario nominal et d'un scénario négatif ; chaque entrée/sortie/outil est validé. |
| Mock | Parcours et erreurs reproductibles sans appel externe ni clé ; mode explicitement identifié. |
| Fournisseur | Destination et paramètres revus, clé serveur seule, délai et quotas testés ; intégration réelle identifiée ou déclarée non validée si indisponible. |
| Autorisation IA | Aucun outil hors catalogue, accès privé, mutation ou exécution active de sortie dans le corpus ; références limitées aux produits publics existants. |
| Consommation | Limites de requêtes, concurrence, contexte, itérations et durée vérifiées ; dépassement produit un refus ou arrêt contrôlé. |
| Pipeline | Chaque contrôle obligatoire produit un résultat exploitable pour la bonne révision ; échec/absence/scan invalide empêche promotion. |
| Remédiation | Au moins un scénario représentatif montre détection, attribution, correction, test puis nouvelle analyse liée au correctif. |
| Preuves | Aucun secret ni donnée privée dans les artefacts partagés ; outil, révision, environnement, portée et limites identifiables pour chaque résultat. |
| Performance | Durées par étape et campagne de référence mesurées ; comparaison à périmètre constant et écarts aux cibles documentés. |
| Tier 1/Tier 2 | Source scientifique exacte identifiée avant toute affirmation ; onze pratiques Tier 1 traçables ou écarts explicitement déclarés ; options Tier 2 sourcées et arbitrées. |

La couverture scientifique ne peut pas être acceptée sur une liste inventée. Un mode mock validé ne remplace pas la validation du fournisseur et une pipeline verte ne remplace pas les tests d'autorisation ou les preuves mobiles.

## 14. Planning sans dates

1. **P0 — Cadrage commun :** contrats, modèle de confiance, risques, politiques de données/gates et obtention de la source scientifique.
2. **P0 — IA mock et contrôles rapides :** endpoint, outils bornés, tests déterministes ; orchestration qualité/tests/SAST/SCA/secrets lorsque les composants sont disponibles.
3. **P1 — Fournisseur et outils :** recherche réelle filtrée, prompts, coûts, quotas et tests d'intégration ; dépendance : service produit de Mohamed.
4. **P0 — Environnement intégré :** conteneurisation future, base fictive, santé, tests croisés et ZAP sur cible autorisée.
5. **P0 — Remédiation et preuves :** corriger les constats, relancer, protéger les artefacts et compléter la matrice Tier 1 exacte.
6. **P1 — Mesure et options :** performance, SonarQube si retenu, analyse Tier 2 sans retarder une correction obligatoire.
7. **P1 — Soutenance :** campagne finale, démonstration d'un blocage puis d'une correction, résultats IA et limites vérifiables.

## 15. Checklist détaillée

- [ ] Valider le périmètre IA et les données autorisées avec Mohamed.
- [ ] Définir le contrat de chat et le faire relire par Hanane.
- [ ] Séparer route, service, adaptateurs, prompts, outils et schémas.
- [ ] Réaliser le mode mock sans dépendance réseau.
- [ ] Identifier explicitement mock et fournisseur dans les résultats.
- [ ] Protéger le chat par session, quotas et validation.
- [ ] Limiter la recherche aux produits publics actifs et champs minimaux.
- [ ] Refuser les outils inconnus, URL, SQL, commandes et mutations libres.
- [ ] Borner appels d'outils, contexte, jetons, concurrence et durée.
- [ ] Garder la clé fournisseur côté serveur et hors journaux.
- [ ] Revoir confidentialité, destination et paramètres du fournisseur retenu.
- [ ] Valider sorties, longueurs et références de produits.
- [ ] Tester injections directes et indirectes via les annonces.
- [ ] Tester extraction de secrets, utilisateur tiers et sorties XSS.
- [ ] Mesurer coût/latence réels ou signaler l'absence d'essai fournisseur.
- [ ] Inventorier les contrôles de chaque composant et leurs propriétaires.
- [ ] Définir déclenchements, permissions et délais des futurs workflows.
- [ ] Isoler les changements externes des secrets et contextes privilégiés.
- [ ] Prévoir qualité, types et tests avec rapports exploitables.
- [ ] Prévoir Semgrep avec règles, exclusions et seuils explicites.
- [ ] Prévoir npm audit avec périmètre de dépendances déclaré.
- [ ] Prévoir Gitleaks avec portée d'historique et rapports masqués.
- [ ] Prévoir Compose isolé, migrations de test et vérification de santé.
- [ ] Définir la cible ZAP autorisée et les comptes fictifs.
- [ ] Distinguer scan passif, actif et authentifié ; vérifier leur couverture.
- [ ] Décider SonarQube et consigner le gate si retenu.
- [ ] Définir politique de blocage, exceptions bornées et revue collective.
- [ ] Relier chaque vulnérabilité à son propriétaire et sa nouvelle analyse.
- [ ] Conserver des preuves expurgées liées à la révision et aux outils.
- [ ] Définir la cible de livraison et le retour arrière avant déploiement futur.
- [ ] Mesurer les durées et les échecs techniques à périmètre constant.
- [ ] Obtenir la publication/grille exacte de Prates et Pereira.
- [ ] Compléter les onze pratiques Tier 1 sans en inventer les intitulés.
- [ ] Classer les options Tier 2 uniquement à partir de la source validée.
- [ ] Préparer les résultats, preuves, limites et démonstrations de soutenance.

## 16. Soutenance

Présenter d'abord la frontière entre modèle, orchestrateur et outils. Montrer une recherche de produits, le mode mock identifié, un fournisseur réel uniquement s'il a été testé, puis une injection directe et une description produit hostile n'obtenant ni outil interdit ni donnée privée. Démontrer un dépassement de quota et une sortie non conforme traitée de façon sûre.

Présenter ensuite les étapes de la pipeline et un résultat pour chaque outil effectivement retenu. Montrer un contrôle bloquant sur défaut de démonstration isolé, son attribution, sa correction et la nouvelle analyse autorisant la suite. Expliquer les limites de ZAP et des scans de dépendances, les mesures de durée et la conservation expurgée des preuves.

Relier les résultats aux onze pratiques Tier 1 seulement après validation de la source exacte. Présenter les options Tier 2 avec bénéfices et coûts, les écarts restants et la contribution de chaque membre. Distinguer clairement contrôle prévu, exécuté, validé et non démontré.
