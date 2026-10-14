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
