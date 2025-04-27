# Serveur de médias

   Archivé le 1er mai 2023

<!-- Pour qu'on en reparle. J'ai du mal à distinguer le besoin, l'étude et le résultat mis en oeuvre -->

## Besoin

Le serveur de médias est une fonctionnalité nécessaire, complémentaire à l'envoi de mails par CiviCRM. En effet, pour que les mails ne soient pas de taille trop importante, on dépose souvent les fichiers souvent utilisés ou plus lourds (images par exemple) dans un serveur de médias, où le média sera récupéré à la volée lors des consultations des mails. Bien entendu, les fichiers confidentiels ne doivent pas être envoyés par mail sans précaution (chiffrement par exemple), et on évitera également de les laisser à disposition de tous sur un serveur non authentifié.

Un deuxième besoin complémentaire est la possibilité de déposer une page web qui représente un mail envoyé, et qui serait difficile à lire du côté de l'expéditeur. Ce besoin doit toutefois être étudié au cas par cas, car  ce genre de mail est en contradiction avec les mails personnalisés (mails sous forme de templates avec des champs de données par exemple). Il semble par ailleurs que réaliser une page web qui afficherait des données personnalisées par le biais d'un élément comme un token d'authentification dans l'URL serait hasardeux, et poserait des questions quant à la protection des données (sauf à disposer par exemple d'une clef publique de chiffrement pour envoyer du contenu à un utilisateur).

Il est à noter que même dans le cadre des mails personnalisés, les mails ne sont la plupart du temps pas chiffrés par des clefs asymétriques, comme par exemple via des technologies telles que S/MIME. De ce fait, bien que les transferts entre serveurs de mails peuvent être chiffrés, le contenu des mails pourrait toutefois être découvert lors du transit sur un serveur (et les mails sont généralement analysés avec des outils comme des antispams ou des antivirus). Il est donc essentiel de faire particulièrement attention à ce que les données qui transitent dans les mails ne soient pas des données sensibles.

Pour en revenir au besoin, il est donc nécessaire de s'authentifier auprès du système pour pouvoir déposer des fichiers, mais il n'est pas nécessaire ensuite de s'authentifier pour les récupérer.

## Eventuel besoin complémentaire : serveur d'échanges de fichiers ; analyse de possibles solutions

Certaines paroisses peuvent avoir besoin d'une solution simple de stockage de fichiers pour certains projets, avec des accès réservés à certaines personnes. Ce besoin nécessite donc des outils supplémentaires : 

* une authentification de l'ensemble des utilisateurs
* un mécanisme de contrôles des droits sur les fichiers

Pour autant, il reste nécessaire que certains documents soient accédés sans authentification et via le protocole HTTP

Plusieurs types d'outils semblent fournir des solutions :

* le stockage sur des services clouds : Google Drive est un exemple qui permet le stockage de documents, et la gestion des droits sur les documents ; néanmoins, certains utilisateurs peuvent se montrer réticents à utilisateur des services directement issus des GAFAS. On pourrait également penser à Azure Blob Storage.

* les logiciels de type Enterprise Content Management / Document Management System : elles fournissent des solutions qui dépassent fonctionnellement le besoin, et semblent pour la plupart être assez lourdes et impliqueraient des ressources assez importantes aussi bien humaines que machines pour être exploitées correctement. Par ailleurs, certains logiciels disposent d'une version communauté et d'une version payante, et que des écarts importants dans les numéros de version existent. Enfin, on trouve quelques projets de logiciel libre DMS plus légers, mais dont le code source n'est parfois plus maintenu ou ne semble plus être en phase avec les pratiques actuelles

* les solutions construites sur des briques importantes système/infrastructure : dans ce genre de solution, les paroisses devraient disposer de compétences techniques (ex : annuaire LDAP, gestion des utilisateurs et groupes Unix, SSH...) qui semblent difficilement à leur portée

Les recherches n'ont pas permis de déterminer une solution satisfaisante à intégrer dans le projet pour le besoin. Etant donné que ce besoin n'est pas immédiatement en lien avec CiviCRM, et que les solutions Drive semblent être les plus simples à mettre en oeuvre, ce besoin complémentaire ne sera pas adressé par le projet.

## Solution proposée (hors scope projet)

La solution que l'on retient pour le moment pour le serveur de médias est une instance de Drupal avec une base de données sqlite, dans le but de faire un serveur aussi léger que possible.

Le projet comporte des versions modifiées des images utilisées pour CiviCRM : 

* composer\_media\_base : la version de composer livrée avec Ubuntu ne supporte pas le `auth-plugin : true` ; du coup il fallait récupérer une version qui le supporte
* media\_files : la récupération des fichiers nécessaires
* media\_tools : l'image avec les fichiers et les outils
* media\_init : l'image pour l'installation des volumes
* media\_httpd : l'image pour le serveur web
* media\_cron : l'image pour le cron

L'image selfkeys a été modifiée pour intégrer la signature d'un certificat pour ce serveur.

Etant donné que ces images sont dérivées des images principales, on se reportera à la documentation de ces dernières pour comprendre le fonctionnement des images dérivées.

