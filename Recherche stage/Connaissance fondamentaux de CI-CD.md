
CI vs CD vs CD

![[Pasted image 20260914082835.png]]

### Continuous Integration (CI), intégration continue

**Ce que c'est** : la pratique de fusionner fréquemment le code de tous les développeurs dans une **branche principale**, avec vérification automatique à chaque ajout.

Son fonctionnement :

1. Un développeur pousse son code sur Git
2. Un serveur CI (Jenkins, Gitlab CI, Github Actions...) détecte le changement
3. Le code est compilé automatiquement
4. Les testes Unitaires et intégrations s'exécutent
5. Des analyses des qualités et des sécurités sont lancées
6. Si tout se passe bien : le code est validé
7. Si quelque chose échoue : l'équipe est alerté immédiatement

> [!NOTE] La règle d'or de la CI
> **Le build ne doit jamais rester cassé.** Si un commit casse le build, c'est la **priorité absolue** de le réparer. Une équipe mature corrige un build cassé en moins de 10 minutes, avant même de continuer à développer de nouvelles fonctionnalités.

### Continuous Delivery, livraison continue

**Ce que c'est** : l'extension de la CI où le code est automatiquement préparé pour être déployé en production, mais **un humain appuie sur le bouton** pour le déploiement final.

**Quand l'utiliser** :

- Applications critiques nécessitant une validation métier
- Secteurs réglementés (finance, santé)
- Équipes en transition vers le DevOps
- Changements impactant fortement les utilisateurs (UI, fonctionnalités majeures)

### Continuous Deployment, déploiement continu

**Ce que c'est** : la version ultime où **tout est automatique**. Dès que le code passe les tests, il est déployé en production sans intervention humaine.

**Quand l'utiliser** :

- Applications avec une **excellente couverture de tests**
- Équipes matures avec une forte culture qualité
- Systèmes avec **rollback automatique**
- Microservices indépendants les uns des autres


> [!NOTE] Delivery vs Deployment
> La confusion vient de l'abréviation « CD » utilisée pour les deux pratiques. Retenez : **Delivery** = prêt à déployer (bouton manuel) ; **Deployment** = déploiement automatique.

En pratique, la plupart des équipes font du **Continuous Delivery** avec déploiement automatique sur les environnements de test, et bouton manuel réservé à la production.

## Anatomie d'un pipeline CI/CD

Un **pipeline** est une séquence d'étapes automatisées qui transforme le code source en application déployée. Chaque étape ne démarre que si la précédente a réussi : c'est ce qui garantit qu'un code défaillant ne progresse jamais jusqu'à la production. Voyons chaque étape en détail.

![[Pasted image 20260914095849.png]]

### Étape 1 : déclenchement (trigger)
Le pipeline démarre quand un **événement** se produit sur le dépôt Git. Choisir le bon déclencheur évite deux écueils opposés : lancer le pipeline trop souvent (gaspillage de ressources) ou pas assez (feedback trop tardif).

| Événement                    | Quand l'utiliser                   |
| ---------------------------- | ---------------------------------- |
| Push sur une branche         | Tests à chaque modification        |
| Pull Request / Merge Request | Validation avant fusion            |
| Tag / Release                | Déploiement d'une version          |
| Cron (planifié)              | Tests nocturnes, scans de sécurité |
| Manuel                       | Déploiements contrôlés             |

### Étape 2 : build (compilation)

Le **build** transforme le code source en un **artefact** exécutable, c'est-à-dire le livrable final que l'on va tester puis déployer : une image Docker, un binaire compilé ou un paquet de dépendances. Cette étape doit être **reproductible** : deux builds du même commit doivent produire un résultat strictement identique.

- **Application compilée** (Java, Go, Rust...)
- **Image Docker** (conteneurs)
- **Bundle optimisé** (JavaScript, TypeScript)
- **Package** (pip, npm, nuget...)


> [!NOTE] Builds reproductibles
> Utilisez **npm ci** au lieu de **npm install** en CI. La commande `ci` installe exactement les versions du `package-lock.json`, garantissant un build identique à chaque exécution, contrairement à `install` qui peut résoudre des versions légèrement différentes.

### Étape 3 : tests
Un pipeline exécute plusieurs **niveaux de tests**, du plus rapide au plus lent, pour donner un feedback progressif. L'idée : détecter les régressions évidentes en quelques secondes, avant de lancer les vérifications les plus coûteuses.

1. **Tests unitaires** : vérifient les fonctions isolément (millisecondes).
    
2. **Tests d'intégration** : vérifient les interactions entre composants (secondes).
    
3. **Tests End-to-End** : vérifient les parcours utilisateurs complets (minutes).

### Étape 4 : analyse de qualité et sécurité

Avant de publier un artefact, le pipeline le **scanne** pour détecter les vulnérabilités et les erreurs de configuration. Cette étape incarne le principe du **shift-left** : détecter les problèmes de sécurité le plus tôt possible, quand ils coûtent le moins cher à corriger.

| Type d'analyse  | Ce qu'il détecte            | Outils                     |
| --------------- | --------------------------- | -------------------------- |
| **Linting**     | Style, conventions          | ESLint, Ruff, Pylint       |
| **SAST**        | Vulnérabilités dans le code | SonarQube, Semgrep, CodeQL |
| **Secrets**     | Credentials exposés         | GitLeaks, TruffleHog       |
| **Dépendances** | CVE dans les packages       | Dependabot, Snyk, Trivy    |
| **SBOM**        | Inventaire des composants   | Syft, CycloneDX            |
### Étape 5 : publication de l'artefact

Une fois validé, l'artefact est publié dans un **registre**, un serveur qui stocke et distribue les versions numérotées de vos livrables. C'est ce registre que l'étape de déploiement viendra interroger pour récupérer la bonne version à installer.

- **Docker Registry** (Docker Hub, GitHub Container Registry, Harbor)
- **Package Registry** (npm, PyPI, Maven Central)
- **Artifact Storage** (S3, GCS, Artifactory)

### Étape 6 : déploiement

Le déploiement installe l'artefact publié sur l'environnement cible, typiquement d'abord un **environnement de staging** (réplique de la production pour valider sans risque), puis la **production**. Beaucoup d'équipes matures remplacent les commandes `kubectl` impératives ci-dessous par une approche déclarative.

## Stratégies de déploiement

Comment mettre à jour une application en production sans interrompre le service ? Plusieurs **stratégies de déploiement** existent, chacune avec ses avantages et son niveau de complexité opérationnelle.

![[Pasted image 20260914111048.png]]

### Rolling Update (mise à jour progressive)

**Principe** : remplacer les instances **une par une**, ce qui permet de mettre à jour l'application sans jamais tout arrêter d'un coup. C'est la stratégie par défaut de la plupart des orchestrateurs de conteneurs.

**Comment ça marche** :

1. Vous avez 4 serveurs avec la version 1.0
2. Le déploiement arrête le serveur 1, le met à jour en 2.0, le redémarre
3. Puis le serveur 2, puis 3, puis 4
4. À chaque instant, au moins 3 serveurs sont disponibles

**Avantages** :

- Zero downtime
- Utilise l'infrastructure existante
- Rollback possible (mais lent)

**Inconvénients** :

- Période avec versions mixtes (1.0 et 2.0 en parallèle)
- Problème si l'ancienne et la nouvelle version sont incompatibles (API, base de données)

### Blue/Green (environnements jumeaux)

**Principe** : maintenir **deux environnements identiques** et basculer le trafic de l'un vers l'autre en une seule opération, ce qui rend le rollback instantané puisque l'ancien environnement reste disponible.

**Comment ça marche** :

1. **Blue** = production actuelle (version 1.0)
2. **Green** = nouvelle version (2.0) déployée en parallèle
3. Tests sur Green sans impact utilisateur
4. Changement DNS : le trafic bascule de Blue vers Green
5. Blue devient l'environnement de secours

**Avantages** :

- Rollback instantané (rebascule vers Blue)
- Test en conditions réelles avant mise en production
- Pas de versions mixtes

**Inconvénients** :

- Coût infrastructure doublé
- Complexité pour les migrations de base de données

### Canary (déploiement progressif)

**Principe** : déployer la nouvelle version pour un **petit pourcentage d'utilisateurs**, puis augmenter progressivement si aucun signal négatif n'apparaît. Le nom vient des canaris utilisés dans les mines de charbon : si le canari mourait, les mineurs savaient que l'air était toxique. Ici, si le groupe pilote rencontre des problèmes, on stoppe le déploiement.

**Comment ça marche** :

1. 5 % du trafic vers la nouvelle version
2. Monitoring des métriques (erreurs, latence, satisfaction)
3. Si tout va bien : 25 % puis 50 % puis 100 %
4. Si problème : rollback immédiat (seuls 5 % impactés)

**Avantages** :

- Risque limité (un petit pourcentage d'utilisateurs impacté)
- Feedback réel en production
- Détection précoce des problèmes

**Inconvénients** :

- Nécessite un monitoring sophistiqué
- Plus complexe à configurer

Quelle stratégie choisir ?

|Situation|Stratégie recommandée|
|---|---|
|Premier déploiement CI/CD|Rolling Update|
|Application critique, besoin de rollback rapide|Blue/Green|
|Forte incertitude sur la nouvelle version|Canary|
|Fonctionnalité expérimentale|Feature Flags|
|Startup, itérations rapides|Continuous Deployment + Rolling|
|Entreprise réglementée|Continuous Delivery + Blue/Green|

blog.stephane-robert.info/docs/devops/delivery/ci-cd/ : 14/09/2026 à 10 heures - 12 heurs