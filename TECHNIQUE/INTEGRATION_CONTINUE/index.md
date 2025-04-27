# Intégration continue

L'intégration est un sujet complexe. Le but de l'intégration continue est de vérifier si un système vérifie des exigences, et que ces vérifications soient effectuées de manière automatique.
En fonction de la nature du système impacté, il peut être envisagé d'utiliser un ou plusieurs tunnels : on pourrait par exemple vouloir exécuter des tests unitaires si du "code métier", à dominante fonctionnelle, serait intégré dans Civiparoisse. Pour l'heure, il est surtout important de disposer d'un tunnel de test et validation d'une image Civiparoisse avant sa publication, et son éventuel déploiement.

Les tunnels d'intégrations sont des éléments qui sont complexes à mettre en oeuvre. Il est nécessaire de traiter différents sujets de manière spécifique :

* [Docker : build et push](docker_build_push.md)
* [Tests automatisés : Codeception](tests_automatises.md)
* [Tests automatisés : Phpunit](tests_phpunit.md)
* [Composer](composer_ci.md)
* [Trivy](trivy_ci.md)
* [SonarLint et SonarQube](sonarqube_ci.md)
