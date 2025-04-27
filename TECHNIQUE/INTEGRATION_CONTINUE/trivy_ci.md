# Trivy

Trivy est un scanner de vulnérabilités open-source (<https://github.com/aquasecurity/trivy> et pour la documentation : <https://aquasecurity.github.io/trivy/v0.44/>). L'utilisation de ce genre d'outil est essentiel, car il permet notamment de garder un oeil sur le panel de vulnérabilités d'un système, et il permet de définir un Software Bill Of Materials (SBOM), et de chercher les vulnérabilités par rapport au SBOM.

Trivy dispose de plusieurs modes de recherches de vulnérabilités, dont le mode fs, le mode repo (git repository) et le mode image (pour les images de containers). Il faut remarquer que les manières de rechercher les vulnérabilités diffèrent en fonction du mode. Il est donc intéressant d'utiliser plusieurs moyens en parallèle pour avoir une vision plus complète des dépendances d'un projet.

De plus, Trivy dispose déjà d'une action Github disponible (<https://aquasecurity.github.io/trivy/v0.44/tutorials/integrations/github-actions/> et <https://github.com/aquasecurity/trivy-action/blob/master/action.yaml>) - et cette action semble assez souple pour s'adapter à un grand nombre de scénarios. Il y aurait même moyen d'intégrer le SBOM dans Github via une action Github (<https://github.com/marketplace/actions/spdx-dependency-submission-action>), ce qui permettrait également de profiter de la surveillance des dépendances au moyen de Dependabot.

Au niveau de Civiparoisse, on peut donc envisager plusieurs points où l'on va mettre Trivy en oeuvre :

* avant le tunnel : lors de la création d'une release, lorsqu'on a préparé le composer.json et le composer.lock, générer deux SBOM.
* durant le tunnel d'intégration : analyser l'image qui est forgée dans le tunnel.
* durant le tunnel d'intégration : générer un SBOM qui va être ajouté à un endroit spécifique, afin de pouvoir exécuter des analyses de vulnérabilités en se basant sur le SBOM.
* lors du workflow cron : analyser les vulnérabilités via les SBOM gérés.

L'utilisation de trivy est en ligne de commande, et est très simple lorsqu'il est installé directement sur une machine.
