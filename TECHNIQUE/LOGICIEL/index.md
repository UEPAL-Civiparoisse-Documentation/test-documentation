# Aspects logiciels

## Développements spécifiques

Les besoins fonctionnels seront traités via des développements spécifiques s’appuyant sur CiviCRM, qui est un CRM écrit en PHP nécessitant un CMS et une stack LAMP. Le CMS Drupal a été choisi à cet effet. 

Les développements spécifiques sont répartis à ce jour en trois modules :

* un module Civicrm : ce module intègre les adaptations de CiviCRM nécessaires pour les fonctions « coeur de métier ».

* un module plugin d'installation pour Civicrm : ce module intègre certains aspects d'installation initiale de CiviCRM qui ne sont pas ou difficilement prévus pour la ligne de commande.

* le code des tests automatisés : les tests automatisés sont basés sur codeception, et sont déployés via une stack technique en parallèle de CiviCRM.

A noter qu'historiquement, il existait également un module Drupal qui remplissait un besoin technique (permettre l’utilisation de l’authentification de type basic).

Chaque module est versionné dans git, et constitue un package composer. Le choix d’utiliser composer n’est pas anodin : il est utilisé à la fois par CiviCRM et Drupal, ce qui uniformise dans une certaine mesure la gestion des fichiers source du projet.

Il a été décidé de suivre la gestion git « gitflow » (<https://www.atlassian.com/fr/git/tutorials/comparing-workflows/gitflow-workflow> ). Le suivi des développements se fait via les outils mis à disposition sur Github (dont notamment les issues).

## Qualité du code

Le code spécifique est pour l’heure globalement stabilisé. Néanmoins, il devra être envisagé de mettre en place des tests automatisés pour fiabiliser les développements, voire même de faire de l’intégration continue.

## Distribution du code spécifique

Le code a pour vocation d’être sous licence « logiciel libre » (licence exacte à définir). De ce fait, il conviendra de publier le code source dans un dépôt git « public ». Toutefois, mettre à disposition le code n’implique pas de mettre à disposition le détail des interactions des intervenants (ni leur identité). De ce fait, il est prévu que la publication sur le dépôt git public se fasse à travers un fetch remote de profondeur 1 et un checkout en orphan release de FETCH_HEAD, de sorte à conserver uniquement le code, et pas l’historique. Cette manière de procéder permettra toutefois les comparaisons d’une version à l’autre, ainsi que la mise en place des tags qui sont nécessaires pour identifier les versions.

## Utilisation de modules tiers

D’autres modules / extensions peuvent également être nécessaires au projet : il s’agira de les intégrer, dans la mesure du possible, comme des packages composer. Au pire des cas, on pourra utiliser un repository de type « package » dans composer (cf <https://getcomposer.org/doc/05-repositories.md#package-2> ).

## Documents
Cette section répertoire les documentations relatives au développement logiciel de Civiparoisse.

* [Distribution du code](distribution_code_composer.md)
* [Environnement de dev](environnement_dev.md)
* [Customisation environnement de dev](environnement_dev_env.md)
* [Importation des données](importation.md)
