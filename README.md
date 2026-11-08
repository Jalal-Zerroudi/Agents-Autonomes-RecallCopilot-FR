# RecallCopilot FR

RecallCopilot FR est un projet de copilote destiné à accélérer le traitement des rappels de produits en France. L’objectif est de rapprocher les avis publiés sur RappelConso d’un inventaire utilisant des identifiants EAN/GTIN, puis de préparer les éléments nécessaires à l’information des équipes et des consommateurs.

> **Statut :** ce dépôt décrit actuellement le concept du projet. Aucun code exécutable n’est encore versionné.

## Objectifs envisagés

- surveiller les nouveaux avis de rappel publiés par RappelConso ;
- normaliser les références produit et les identifiants EAN/GTIN ;
- comparer les produits rappelés avec un inventaire interne ;
- produire une synthèse traçable des correspondances trouvées ;
- préparer une affiche de rappel au format PDF ;
- générer des notifications pour validation et diffusion ;
- conserver un journal des sources, décisions et actions.

## Flux fonctionnel cible

1. Collecter un avis de rappel depuis une source officielle.
2. Valider et normaliser les données reçues.
3. Rechercher les produits correspondants dans l’inventaire.
4. Évaluer la qualité de chaque correspondance.
5. Soumettre les résultats ambigus à une validation humaine.
6. Générer les documents et notifications approuvés.
7. Archiver les preuves et les horodatages associés.

## Architecture envisagée

| Composant | Responsabilité |
| --- | --- |
| Connecteur RappelConso | Collecte contrôlée des avis de rappel |
| Agent de normalisation | Nettoyage des références et EAN/GTIN |
| Agent de rapprochement | Comparaison avec l’inventaire |
| Service RAG | Recherche dans les procédures et documents de référence |
| Agent documentaire | Préparation de l’affiche et des notifications |
| API FastAPI | Exposition des opérations et statuts |
| Orchestration LangGraph | Coordination des étapes et validations |
| Journal d’audit | Traçabilité des données, décisions et sorties |

## Principes de fiabilité

- conserver l’URL et la date de chaque source consultée ;
- ne pas diffuser automatiquement une correspondance incertaine ;
- valider les identifiants avec leurs chiffres de contrôle ;
- rendre chaque décision reproductible et explicable ;
- séparer les données de test des inventaires réels ;
- stocker les secrets et identifiants hors du dépôt ;
- prévoir une validation humaine avant toute publication.

## Règle minimale de validation EAN/GTIN

Avant tout rapprochement, l'identifiant produit devra être traité comme une chaîne de caractères afin de conserver ses éventuels zéros initiaux, puis validé de manière déterministe :

1. retirer uniquement les espaces ou séparateurs explicitement autorisés ;
2. rejeter toute valeur contenant encore un caractère non numérique ;
3. accepter uniquement les longueurs GTIN-8, GTIN-12, GTIN-13 ou GTIN-14 ;
4. considérer le dernier chiffre comme chiffre de contrôle ;
5. parcourir les autres chiffres de droite à gauche en appliquant alternativement les poids 3 et 1 ;
6. calculer le chiffre attendu avec `(10 - (somme % 10)) % 10` et le comparer au dernier chiffre.

Un identifiant vide, de longueur incorrecte ou dont le chiffre de contrôle ne correspond pas devra être classé comme invalide avant toute recherche approximative. Référence : [méthode officielle de calcul GS1](https://www.gs1.org/services/how-calculate-check-digit-manually).

## Politique de rapprochement et niveaux de confiance

Le rapprochement doit privilégier les règles déterministes avant toute comparaison approximative. Les seuils numériques devront être configurables, versionnés et évalués sur un jeu de données de test ; ils ne doivent pas être présentés comme une preuve à eux seuls.

| Résultat | Condition minimale | Traitement |
| --- | --- | --- |
| `exact` | GTIN valide et identique dans l'avis et l'inventaire | Peut être proposé pour approbation humaine |
| `candidat` | Aucun GTIN exact, mais des attributs concordants produisent un score supérieur au seuil configuré | Bloqué en attente de validation humaine |
| `absent` | Aucun candidat ne dépasse le seuil minimal | Aucune diffusion ; cas conservé pour suivi |
| `invalide` | GTIN invalide ou données sources insuffisantes | Rapprochement interrompu et anomalie journalisée |

Une correspondance approximative devra indiquer les attributs comparés — par exemple la marque, le libellé, le lot, le conditionnement ou la période de commercialisation — ainsi que leur contribution au score. Un modèle génératif pourra aider à expliquer ou ordonner les candidats, mais ne devra jamais transformer seul un résultat `candidat` en résultat `exact`.

Chaque décision de rapprochement devra au minimum enregistrer l'identifiant de l'avis source, la référence d'inventaire, le GTIN normalisé, le type de correspondance, le score éventuel, le statut, la version de la règle appliquée et l'horodatage. Une modification de règle devra entraîner une nouvelle évaluation traçable plutôt que l'écrasement silencieux du résultat précédent.

## Idempotence et gestion des mises à jour

La surveillance devra pouvoir relire plusieurs fois la même source sans recréer les mêmes actions. Chaque avis collecté devra donc recevoir une clé stable, issue en priorité de son identifiant officiel et, à défaut, d'une empreinte déterministe des champs sources pertinents.

Les règles minimales sont les suivantes :

- une collecte identique met à jour l'horodatage de dernière vérification sans dupliquer le dossier ;
- une modification de la source crée une nouvelle version liée à l'avis initial ;
- une nouvelle version invalide les brouillons non approuvés devenus obsolètes ;
- une notification ou une affiche possède une clé d'idempotence empêchant sa génération ou son envoi en double ;
- chaque transition d'état est horodatée et ajoutée au journal d'audit ;
- seuls les états explicitement approuvés peuvent atteindre l'étape de diffusion.

Le cycle minimal attendu est : `collecté` → `validé` → `rapproché` → `en attente de validation` → `approuvé` → `diffusé`. Les erreurs et rejets doivent rester enregistrés sans être assimilés à une diffusion réussie.

## Corrections et retraits après diffusion

Une diffusion réussie ne clôt pas définitivement le suivi d'un avis. Toute nouvelle
version de la source officielle devra être comparée à la version effectivement
approuvée et diffusée, y compris lorsqu'elle corrige un lot, une période, une
conduite à tenir ou retire l'avis.

Les règles minimales sont les suivantes :

- enregistrer pour chaque diffusion le `source_version_id`, l'empreinte de
  l'artefact, les canaux, les destinataires logiques et les accusés disponibles ;
- classer chaque changement officiel comme `sans_impact`, `correction_requise`
  ou `retrait`, avec justification et validation humaine ;
- invalider immédiatement les brouillons et approbations fondés sur une version
  devenue obsolète, sans effacer leur historique ;
- suspendre les envois encore en attente dès qu'une correction ou un retrait est
  détecté ;
- produire une action corrective liée à la diffusion initiale, avec sa propre
  approbation, sa clé d'idempotence et son état de livraison ;
- ne jamais modifier silencieusement un PDF ou un message déjà archivé : créer
  une nouvelle version explicitement marquée comme correction ou retrait ;
- conserver la chaîne complète source → décision → diffusion initiale → action
  corrective dans le journal d'audit.

Le dossier ne pourra être marqué `corrigé` ou `retiré` qu'après vérification de
chaque canal concerné. Un échec partiel restera visible et relançable sans créer
de doublon. Les tests devront couvrir une correction avant envoi, une correction
après envoi, un retrait, un canal indisponible et deux traitements concurrents de
la même nouvelle version.

## Contrat minimal de données

Le prototype devra convertir les données collectées vers un schéma interne stable avant tout rapprochement. Le contenu source brut ou son empreinte devra être conservé séparément afin de pouvoir expliquer et rejouer la normalisation.

### Avis de rappel normalisé

| Champ | Type | Obligatoire | Rôle |
| --- | --- | --- | --- |
| `recall_id` | chaîne | oui | Identifiant officiel stable ou empreinte de secours |
| `source_url` | URI | oui | Lien exact vers l’avis source |
| `collected_at` | date-heure ISO 8601 UTC | oui | Horodatage de la collecte |
| `source_hash` | chaîne | oui | Empreinte du contenu ayant produit cette version |
| `product_name` | chaîne | oui | Libellé du produit rappelé |
| `brand` | chaîne ou `null` | non | Marque telle qu’indiquée par la source |
| `gtins` | liste de chaînes | non | Identifiants conservant leurs zéros initiaux |
| `lots` | liste de chaînes | non | Lots concernés sans déduction implicite |
| `sale_start` / `sale_end` | date ou `null` | non | Période de commercialisation publiée |
| `reason` | chaîne ou `null` | non | Motif officiel du rappel |
| `consumer_action` | chaîne ou `null` | non | Conduite à tenir publiée |

### Ligne d’inventaire synthétique

| Champ | Type | Obligatoire | Rôle |
| --- | --- | --- | --- |
| `inventory_ref` | chaîne | oui | Référence interne unique |
| `product_name` | chaîne | oui | Libellé utilisé pour le rapprochement |
| `brand` | chaîne ou `null` | non | Marque connue dans l’inventaire |
| `gtin` | chaîne ou `null` | non | GTIN conservé comme texte puis validé |
| `lot` | chaîne ou `null` | non | Lot disponible lorsque l’inventaire le suit |
| `updated_at` | date-heure ISO 8601 UTC | oui | Fraîcheur de la ligne comparée |

Les espaces périphériques peuvent être supprimés, mais les valeurs sources ne doivent pas être traduites, complétées ou fusionnées silencieusement. Un champ facultatif absent doit devenir `null` ou une liste vide selon son type ; il ne doit pas être remplacé par une valeur inventée. Chaque erreur de validation doit préciser le champ, la valeur reçue et la règle violée.

## Contrat minimal du journal d'audit

Le journal devra être append-only : une correction crée un nouvel événement lié au
précédent, sans modifier ni supprimer l'événement historique. Chaque écriture
devra au minimum contenir :

| Champ | Rôle |
| --- | --- |
| `event_id` | Identifiant unique de l'événement |
| `event_type` | Type stable, par exemple `source_version_created`, `match_evaluated`, `review_recorded` ou `notification_sent` |
| `occurred_at` | Date-heure ISO 8601 en UTC générée par le service |
| `recall_id` / `source_version_id` | Avis et version concernés |
| `correlation_id` | Identifiant reliant une exécution complète et ses reprises |
| `actor_type` / `actor_id` | Service, agent ou réviseur à l'origine de l'action |
| `previous_state` / `new_state` | Transition demandée, avec une valeur absente si elle ne change pas l'état |
| `rule_version` | Version des règles de validation ou de rapprochement appliquées |
| `input_hash` / `output_hash` | Empreintes des données utilisées et produites |
| `decision_reason` | Motif structuré d'une validation, d'un rejet ou d'une erreur |

Un événement de diffusion devra référencer l'approbation humaine qui l'autorise
et la clé d'idempotence de la notification. Une reprise avec la même clé ne devra
pas créer un second envoi réussi. Les événements techniques pourront contenir
des identifiants et empreintes, mais pas l'inventaire complet, des secrets ou des
coordonnées personnelles en clair. L'accès au journal et sa durée de conservation
devront être définis séparément des données opérationnelles.

## Conservation et suppression des données

La conservation devra être fondée sur la catégorie de donnée et sur l'état du
dossier, et non sur une durée unique appliquée à tout le système. Les durées
effectives devront être configurées, versionnées et validées avec les responsables
métier, sécurité et protection des données avant tout usage réel.

| Catégorie | Contenu minimal conservé | Fin de conservation |
| --- | --- | --- |
| Source publique | version brute ou empreinte, URL et date de collecte | expiration de la durée définie pour les preuves du dossier |
| Données normalisées | champs nécessaires au rapprochement et version de règle | clôture du dossier suivie de la durée configurée |
| Instantané d'inventaire | uniquement les références nécessaires à la décision | fin de la contestation ou de la revue associée |
| Brouillons et artefacts | empreinte, statut et fichier lorsque sa conservation est justifiée | remplacement ou expiration selon le statut |
| Notifications | identifiant, canal, destinataire pseudonymisé et résultat d'envoi | expiration de la durée opérationnelle configurée |
| Journal d'audit | événements append-only et identifiants minimisés | durée d'audit approuvée, distincte des données métier |

Les règles minimales sont les suivantes :

- chaque enregistrement porte une catégorie, une date d'expiration et la version
  de politique qui les a déterminées ;
- une nouvelle politique s'applique aux nouvelles données et déclenche une
  migration explicite des données existantes, jamais une réécriture silencieuse ;
- une obligation de conservation suspend la suppression avec un motif, un acteur
  et une échéance journalisés ;
- la suppression est idempotente, produit un événement d'audit et ne journalise
  ni le contenu supprimé ni une donnée sensible en clair ;
- les sauvegardes ont une durée maximale documentée et les données expirées ne
  doivent pas réapparaître durablement après une restauration ;
- lorsque la preuve complète n'est plus nécessaire, seule une empreinte non
  réversible et les métadonnées minimales peuvent être conservées.

Un traitement planifié devra détecter les expirations, supprimer ou anonymiser
les données concernées et produire un rapport agrégé. Les tests devront couvrir
une expiration normale, une suspension de suppression, une seconde exécution
sans effet supplémentaire et une restauration de sauvegarde ne réactivant pas
un dossier expiré.

## Validation humaine et autorisation de diffusion

Une sortie générée restera un brouillon tant qu'un réviseur autorisé ne l'aura
pas approuvée explicitement. L'approbation devra porter sur une version précise
de l'avis, du rapprochement et du document à diffuser ; elle ne pourra pas être
réutilisée après modification de l'une de ces entrées.

| Champ d'approbation | Rôle |
| --- | --- |
| `review_id` | Identifiant unique de la décision humaine |
| `reviewer_id` / `reviewer_role` | Identité et rôle vérifiés du réviseur |
| `source_version_id` | Version exacte de l'avis examinée |
| `match_result_id` | Résultat de rapprochement effectivement validé |
| `artifact_hash` | Empreinte du PDF ou du message approuvé |
| `decision` | `approuvé`, `rejeté` ou `correction_demandée` |
| `reason` | Motif obligatoire, sans donnée sensible inutile |
| `decided_at` | Date-heure UTC générée par le service |

Le service de diffusion devra vérifier l'approbation au moment de l'envoi, et
pas seulement lors de la génération du brouillon. Il refusera l'opération si
l'empreinte de l'artefact, la version source, le résultat de rapprochement ou le
rôle du réviseur ne correspondent plus à la décision enregistrée.

Une nouvelle version de la source, un changement de règle affectant le résultat
ou une modification du brouillon annulera automatiquement l'approbation
précédente. L'annulation créera un événement d'audit ; elle ne supprimera ni ne
réécrira la décision historique.

Les écritures concurrentes devront utiliser un numéro de version ou un verrou
optimiste. Une seule transition depuis `en attente de validation` pourra
réussir ; une seconde décision fondée sur un état périmé sera refusée et devra
recharger le dossier. Lorsque les procédures internes l'exigent, le rôle ayant
préparé le brouillon ne pourra pas l'approuver lui-même.

Les tests devront démontrer qu'un brouillon non approuvé ne peut pas être
diffusé, qu'une modification invalide l'approbation, qu'un rôle non autorisé est
refusé et que deux décisions concurrentes ne déclenchent jamais deux diffusions.

## Politique minimale des erreurs et des reprises

Une erreur ne doit pas être relancée automatiquement sans classification préalable.
Le traitement devra distinguer au minimum les catégories suivantes :

| Catégorie | Exemples | Traitement attendu |
| --- | --- | --- |
| `transitoire` | délai réseau dépassé, indisponibilité temporaire, limitation de débit | reprise bornée avec attente progressive et conservation de la même clé d'idempotence |
| `permanente` | schéma invalide, champ obligatoire absent, GTIN invalide | aucun nouvel essai automatique ; anomalie journalisée et dossier rejeté |
| `métier` | correspondance ambiguë, règle non applicable, données contradictoires | suspension du flux et validation humaine obligatoire |
| `diffusion_inconnue` | délai dépassé après une demande d'envoi | vérifier d'abord l'état auprès du service de diffusion ; ne jamais supposer que l'envoi a échoué |

Chaque échec devra enregistrer un code stable, un message exploitable sans donnée
sensible, le nombre de tentatives, les dates du premier et du dernier échec, la
prochaine date de reprise et le caractère relançable ou non. Le nombre maximal
de tentatives et la stratégie d'attente devront être configurables et versionnés.

Après épuisement des tentatives, le dossier passe en `quarantaine`. Un élément en
quarantaine ne peut ni produire une sortie approuvée ni déclencher une diffusion.
Sa libération exige une action humaine enregistrant l'acteur, le motif et la
version de règle utilisée. Une reprise doit repartir du dernier point de contrôle
sûr et conserver la version de source initiale, au lieu de recommencer
silencieusement tout le flux.

Les tests devront vérifier au minimum qu'une panne transitoire peut réussir après
reprise, qu'une erreur permanente n'est pas relancée, que l'épuisement des
tentatives conduit à la quarantaine et que plusieurs reprises avec la même clé
d'idempotence ne produisent jamais deux notifications réussies.

## Contrôle d'accès et gestion des secrets

Chaque acteur humain ou technique devra utiliser une identité distincte et ne
recevoir que les permissions nécessaires à sa tâche. Une clé de service ne devra
jamais être partagée entre la collecte, l'approbation et la diffusion.

| Rôle | Autorisations minimales | Interdictions explicites |
| --- | --- | --- |
| `collecteur` | Lire les sources publiques et créer une version d'avis | Modifier une décision, approuver ou diffuser |
| `analyste` | Normaliser les données et proposer des rapprochements | Transformer seul un candidat en correspondance approuvée |
| `réviseur` | Consulter les preuves et enregistrer une décision humaine | Modifier les données sources ou envoyer une notification |
| `diffuseur` | Envoyer uniquement un artefact dont l'approbation est encore valide | Générer ou auto-approuver son propre artefact |
| `auditeur` | Lire les événements et les preuves nécessaires au contrôle | Modifier les événements ou les données opérationnelles |

L'API devra authentifier chaque requête, vérifier le rôle au niveau de
l'opération et refuser par défaut toute action non déclarée. Les actions
sensibles — approbation, annulation, libération de quarantaine, diffusion et
changement de politique — devront enregistrer l'identité, le rôle, la décision
et l'identifiant de corrélation dans le journal d'audit.

Les secrets devront provenir d'un gestionnaire de secrets ou de variables
d'environnement injectées au démarrage. Ils ne devront apparaître ni dans le
dépôt, ni dans les journaux, ni dans les messages d'erreur ou artefacts générés.
La rotation ou la révocation d'un secret devra être possible sans modifier les
données métier, et les identifiants techniques inactifs devront perdre
immédiatement leurs autorisations.

Les tests d'autorisation devront au minimum couvrir un accès sans identité, un
rôle insuffisant, la séparation entre préparation et approbation, la révocation
d'un jeton et la vérification qu'aucun secret n'est présent dans les événements
d'audit ou les réponses d'erreur.

## Fraîcheur de la collecte et détection des silences

L'absence de nouvel avis ne doit jamais être confondue avec un échec de collecte. Chaque
interrogation de la source officielle devra produire un état observable, même lorsqu'elle ne
retourne aucun résultat.

Pour chaque cycle, le connecteur devra enregistrer au minimum :

- `poll_started_at` et `poll_finished_at` en UTC ;
- le dernier curseur ou point d'eau confirmé par la source ;
- le nombre d'avis reçus, créés, mis à jour, ignorés et rejetés ;
- un résultat explicite : `succès_avec_données`, `succès_sans_nouveauté`, `incomplet` ou `échec` ;
- la version du connecteur et l'empreinte du schéma source observé ;
- le code d'erreur et la prochaine reprise lorsqu'un cycle n'est pas complet.

Le point d'eau ne pourra avancer qu'après validation et persistance de toutes les données du
cycle. La collecte utilisera une fenêtre de recouvrement configurable, puis l'idempotence par
`recall_id` et `source_hash`, afin de récupérer les publications tardives sans créer de doublons.
Après une panne, le rattrapage repartira du dernier point d'eau confirmé, jamais de l'heure du
redémarrage.

Une alerte devra être déclenchée lorsque le dernier cycle réussi dépasse le seuil de fraîcheur
configuré, lorsque le schéma source change, lorsqu'un curseur régresse ou lorsqu'un volume
s'écarte fortement de sa plage habituelle. Un état de santé pourra exposer les horodatages, le
retard et le résultat du dernier cycle, mais ni contenu d'inventaire, ni secret, ni donnée
personnelle.

Les tests devront couvrir un cycle réussi sans nouveauté, une interruption en cours de page, un
changement de schéma, une publication tardive dans la fenêtre de recouvrement et un rattrapage
rejoué deux fois sans duplication.

## Première feuille de route

- définir les schémas d’entrée et de sortie ;
- documenter les règles de correspondance EAN/GTIN ;
- développer un connecteur RappelConso testable ;
- créer un jeu de données synthétique d’inventaire ;
- implémenter le rapprochement déterministe avant les agents LLM ;
- ajouter les seuils de confiance et la validation humaine ;
- générer un modèle de document PDF ;
- exposer les fonctions par une API FastAPI ;
- ajouter des tests, des métriques et un journal d’audit.

## Périmètre du premier prototype

Le premier prototype doit rester local, reproductible et fondé uniquement sur des données synthétiques ou publiques.

### Entrées minimales

- un avis de rappel d’exemple, conservant son URL source et sa date de collecte ;
- un inventaire synthétique au format CSV avec au minimum une référence interne, un libellé et un EAN/GTIN ;
- une configuration locale définissant les seuils de correspondance, sans secret versionné.

### Sorties attendues

- un rapport structuré listant les correspondances exactes, incertaines et absentes ;
- la règle appliquée et le niveau de confiance pour chaque résultat ;
- un brouillon d’affiche PDF clairement identifié comme non validé ;
- un journal d’audit horodaté reliant chaque sortie à ses données sources.

### Hors périmètre initial

- connexion à un inventaire de production ;
- envoi automatique d’e-mails, SMS ou notifications publiques ;
- publication automatique d’une affiche ;
- décision fondée uniquement sur un modèle génératif ;
- déclaration de conformité juridique sans validation compétente.

## Critères d’acceptation du prototype

Le prototype pourra être considéré comme démontrable lorsque :

- un EAN/GTIN invalide est détecté avant le rapprochement ;
- une correspondance exacte est distinguée d’une correspondance approximative ;
- les cas sans résultat ou ambigus sont bloqués pour validation humaine ;
- chaque résultat indique sa source, son horodatage et la règle utilisée ;
- les scénarios « correspondance exacte », « aucune correspondance », « résultat ambigu » et « entrée invalide » sont couverts par des tests ;
- une exécution complète peut être reproduite sans donnée sensible ni service payant obligatoire.

## Limites

L’objectif de temps de traitement et la conformité des documents générés devront être mesurés et validés sur une implémentation réelle. Les sorties du futur système devront rester soumises aux procédures internes et aux obligations réglementaires applicables.
