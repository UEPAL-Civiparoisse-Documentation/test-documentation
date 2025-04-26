# Civiparoisse : Aspects fonctionnels

Le projet CiviParoisse est un projet récent, qui fournit aux paroisses un outil de gestion opérationnelle. Le projet a été conforté par le virage numérique suite à la crise sanitaire de 2020.  
Ce document présente les différents aspects du projet (notamment fonctionnels, techniques, organisationnels).

## Introduction

Les fonctionnalités qui sont requises par les paroisses peuvent être globalement regroupées en 4 familles :

* La connaissance des personnes en lien avec la paroisse
* La communication avec ces personnes
* Les statistiques de la paroisse
* Une aide pour certains sujets de gestion liés à la paroisse.

## La connaissance des personnes en lien avec la paroisse (les acteurs de la paroisse)

La connaissance des personnes en lien avec la paroisse est un élément essentiel : une paroisse est en premier lieu une communauté de personnes, dont les interactions vont contribuer à la vie de la paroisse.  
Cette connaissance requiert de connaître et de mettre à jour des données personnelles usuelles (nom, prénom, adresse, téléphone, mail, …), mais également des données familiales (filiation), de même que des données sensibles (ayant trait à la religion).

On parle ici **d’acteurs de la paroisse**, car des personnes de différents profils peuvent y figurer :

* Les **paroissiens**, c’est à dire les membres de la paroisse, ainsi que des personnes ressources pour la paroisse (*exemple : le trésorier de la paroisse*)
* Des **personnes externes** mais dont il faut conserver les données pour des situations particulières (exemple : *informations sur les parents séparés d’un catéchumène*)
* Des **personnes morales** peuvent également être enregistrées (exemples : *entreprises, associations, institutions locales comme la municipalité ou une communauté de communes*)
* Des **groupes** comme par exemple les activités paroissiales (exemples : *chorale, catéchumènes, sacristains, organistes, ouvroir, …*), ou aussi des groupes ponctuels de communication

Outre les données sur les acteurs en eux-mêmes, il est également important de pouvoir disposer de précisions sur la **relation entre les acteurs** : par exemple, dans le cadre d’une famille, savoir qui sont les parents et qui sont les enfants. Ou bien dans le cadre d'une activité, savoir qui est le responsable de l'activité.  
Ces relations peuvent d’ailleurs évoluer dans le temps (ex : les conseillers presbytéraux, qui ont des mandats limités dans la durée), mais peuvent également être multiples (ex : certains conseillers presbytéraux sont en plus membre du bureau ; des paroissiens peuvent être des membres consultatifs du conseil presbytéral pour des domaines d’expertise spécifique, ...).

## La communication avec les acteurs de la paroisse

La communication des paroisses repose traditionnellement sur plusieurs vecteurs :

* les annonces lors des cultes
* le journal paroissial papier
* le bouche à oreille
* les appels téléphoniques

Ces méthodes sont peu adaptées aux évolutions technologiques du XXIème siècle :

* le mail
* les réseaux sociaux
* les sites web.

Civiparoisse cherche à optimiser la communication par mail envers les différents acteurs. On peut distinguer plusieurs cas d’utilisation :

* **La newsletter** : c’est le pendant électronique du journal paroissial, avec des informations « générales ». Ces mails s'adressent souvent à un public très large.
* **Les campagnes de mails ciblés** : on envoie des mails avec des informations sur des sujets précis à des personnes sélectionnées en fonction des critères liés à la connaissance des acteurs
* **La communication vers une personne** : Civiparoisse peut afficher dans les mails les coordonnées d’une personne, et ainsi permettre l'envoi de mails individualisés. Il sera même à terme envisageable de simplifier les appels téléphoniques, par clic sur le numéro de téléphone.
* **Les mails système d’administration** : logs, alertes, …

Dans le cas d’une communication pour le compte de la paroisse, il y a des besoins complémentaires, mais qui sortent du cadre du projet :

* L'identité numérique de la paroisse sur internet : son site web
* L'identité visuelle de la paroisse, dont découlent des chartes graphiques pour les différents médias
* Des connaissances et compétences en webmarketing, notamment pour les communications de masse
* L'organisation à un niveau plus global : projet de paroisse, organisation interne de la paroisse pour réaliser ses projets, stockage des données numériques, ...

> Le projet Civiparoisse n’a pas vocation à héberger un site web de paroisse, en raison de la nature des données opérationnelles déjà hébergées, qui ne sont pas publiques. De même, les réseaux sociaux sont des outils externes qui se situent plutôt sur des plateformes hébergées tierces (YouTube, Facebook, Twitter, LinkedIn…) avec d’éventuelles intégrations sur un site web qui ne sont pas non plus prévues dans ce projet.

## Les statistiques de la paroisse

Les paroisses sont consommatrices de statistiques, pour des fins diverses :

* Rapport sur les différents évènements : les baptêmes, confirmations, mariage, décès sur l’année écoulée
* Rapports opérationnels : liste des personnes ayant un anniversaire proche, liste des données qui commencent à dater et doivent donc être vérifiées, les paroissiens présents sur un secteur donné, etc...
* Analyses des données collectées, pour mieux ajuster le projet de paroisse : rapports démographiques, rapports sur les compétences…

## L'aide pour certains besoins de gestion

La paroisse est tenue de maintenir le **registre paroissial**, duquel on doit pouvoir sortir au besoin une proposition de liste des électeurs. La gestion de ces documents nécessite le traitement d’informations précises, que ce soit sur les croyances, le baptême, le lieu de résidence.

Un autre pôle de ce besoin est la **génération de reçus de dons**, ce qui implique donc le traitement de données comptables.

Ces besoins d'aide s’intègrent dans les autres besoins évoqués ci-dessus, sans se substituer aux obligations légales de la paroisse, et seront amenés à évoluer pour s’adapter aux évolutions des textes (droit local, règlement intérieur des églises, décisions du Directoire de l’UEPAL, décisions préfectorales…).
