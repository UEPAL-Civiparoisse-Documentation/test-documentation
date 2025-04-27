# Dépôt de documentation de Civiparoisse

Ce dépôt a pour but de réunir la documentation de Civiparoisse, sous forme de fichiers Markdown qui sont processés ensuite pour générer un site web statique via mkdocs.

Le site est prévu pour utiliser le thème [material](https://squidfunk.github.io/mkdocs-material/) car ce thème propose des fonctionnalités intéressantes :

* l'intégration du moteur LUNR pour les recherches dans les documents
* un mode sombre.

De plus, il propose directement une [image Docker](https://github.com/squidfunk/mkdocs-material.git) prête à être compilée (ou à l'emploi, en fonction des architectures matérielles de chacun), ce qui simplifie nettement la mise en oeuvre du thème et de mkdocs.

La prise en main par des débutants est également simplifiée car les commandes courantes de travail ont été préparées dans un Makefile, ce qui les rend très facilement accessibles.

L'ensemble de la documentation, une fois validée, est également disponible dans le GitHub *uepal-civiparoisse-documentation* 

## Création de la documentation dans le Github

* La documentation doit être validée dans le Github *Civiparoisse*, et intégré à la branche Main
* La documentation est ensuite à incorporer dans le Github *Civiparoisse-documentation*, dans le répertoire **A PRECISER**
* Github génèrera alors automatiquement la mise à jour du site, consultable à l'URL suivante : https://uepal-civiparoisse-documentation.github.io/index.html

## Création de la documentation dans Docker
### Prérequis pour créer l'image Docker

* Linux : nécessaire pour disposer de la variable CURDIR dans le GNU Make
* GNU Make : pour avoir en particulier la variable CURDIR écrite au-dessus
* Git : récupération de dépôt à la volée
* Docker : construction d'une image selon le Dockerfile et l'environnement récupérés via Git.

### Options et cibles pour Make

Un certain nombre de cibles ont été prévues. Elles sont simplement à lancer en se plaçant dans le répertoire où on a récupéré le dépôt Git de la documentation et en lançant 
```make <nom de la cible>```

Make supporte des options, en particulier : 

* -n : dry-run : voir ce que Make fera pour atteindre une cible
* -C `<dirpath>`: permet de changer le répertoire
* assignation de variable : on peut rajouter à la ligne de commande nom_var=valeur pour remplacer la valeur définie dans le makefile (en particulier servip et servport, ip et port pour faire tourner la documentation)

Contrairement à une utilisation "classique" de Make, l'ensemble des cibles prévues sont des cibles factices (PHONY), ce qui fait qu'on se retrouve plutôt face à un appel rapide de commandes plutôt qu'à un processus de compilation optimisé.

Les cibles sont :

* all : crée l'image Docker et génère le site
* image: génération de l'image Docker
* build : génération du site statique (dans le répertoire site)
* serve: génération du site statique et présentation du site sur [http://127.0.0.1:8080](http://127.0.0.1:8080)
* clean: suppression du répertoire de build de l'image et des fichiers générés dans site

***Attention : il faut penser à préparer l'image (cible image ou all) avant de lancer la cible build ou serve.***

