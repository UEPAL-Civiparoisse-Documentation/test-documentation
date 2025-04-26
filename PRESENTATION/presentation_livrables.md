# Présentation des livrables Civiparoisse

La philosophie du projet est de laisser les paroisses qui souhaitent utiliser Civiparoisse **libres et responsables** de leurs choix. De ce fait, Civiparoisse définit un certain nombre de **livrables** que les paroisses pourront utiliser si elles le souhaitent. Ces livrables correspondent à des niveaux d'intégration de Civiparoisse dans des environnements de plus en plus étendus. 


## Approches envisagées pour les paroisses

La mise en œuvre de la solution est envisageable sous trois formes, en fonction des souhaits des paroisses :

* Mise à disposition d’une infrastructure d’hébergement « centralisée », où seront déployées les instances packagées décrites ci-dessous : **cette solution est la voie privilégiée**, avec à la clé des économies d’échelles (en particulier les moyens humains utilisés, mais également les matériels). Il s'agira ici d'un « produit standardisé d’hébergement » pour lequel les paroisses émettront leurs souhaits d'évolution.

* Les paroisses peuvent souhaiter disposer d’un « package logiciel » et assurer les aspects infrastructures elles-mêmes : ceci permet aux paroisses qui désirent administrer leur solution sur une infrastructure existante de le faire elle-même, tout en profitant toutefois d’une solution logicielle commune. Dans ce cas de figure, la paroisse assume également les éléments de gestion et de mise à jour de son infrastructure.

* Le code développé étant sous licence de logiciel libre, les paroisses peuvent éventuellement réutiliser le code et choisir d’héberger la solution elles-même, voire même de créer leur propre solution.  Notre expérience ayant montré que les ressources humaines ayant des compétences en informatique sont rares dans les paroisses, nous déconseillons fortement cette solution, et n'assurerons aucun service après-vente pour les paroisses retenant cette solution.

## Les livrables de Civiparoisse

On retrouve donc, dans un degré d'intégration croissant :

### Composants logiciels spécifiques

Ces composants sont pour l'heure au nombre de deux : 

* plugin d'installation de CiviCRM

* extension CiviCRM contenant le code de Civiparoisse.

### Images Docker

Si l’on regarde l’évolution des pratiques d’hébergement informatique, on est passé progressivement des serveurs physiques dédiés aux machines virtuelles, et depuis quelques années, avec l’avènement de la tendance devops, on assiste plutôt à un passage à des architectures microservices containarisés.

Dans le cadre du package logiciel pour la production, il est judicieux de passer par des containers et l’infrastructure microservices. En effet, ce type d’architecture est conçue pour faciliter les mises à jour des infrastructures de manière rapide, en recréant les images des containers via des fichiers de build. De plus, on peut avoir une séparation plus nette entre les chemins qui sont en lecture écrite et ceux en lecture seule, avec en particulier la notion de volumes. Cette notion de volume peut d’ailleurs aussi être exploitée pour chercher à concentrer les éléments de configuration des environnements dans des endroits spécifiques (config map kubernetes par exemple), et les données sensibles (clefs par exemple) peuvent être stockés dans des volumes de secrets. Enfin, les logs peuvent être disponibles via les entrées sorties standard.

En revanche, pour des démonstrations, l’installation dans une machine virtuelle est plus accessible et plus simple à mettre en œuvre.

Les images tendent à s'affiner progressivement ; pour l'heure, on retrouve les images suivantes :

* composer_base : une image avec une version de composer dont les dépendances sont compatibles avec le code de CiviCRM

* composer_files : image qui va mettre en oeuvre un fichier composer.json permettant de récupérer l'ensemble des fichiers nécessaires à l'installation du système

* tools : image qui ajoute des outils d'usage général et qui va servir en tant qu'image "tout terrain" pour des images à utilisation "éphémère"

* init : image pour initialiser une instance (au niveau des fichiers et de la base de données) de CiviCRM, avec une configuration initiale basique

* cron : image pour exécuter les tâches périodiques

* selfkeys : image pour figer un ensemble de clefs autosignées pour faciliter les déploiements de démonstration

* httpd : le serveur web interne, qui est accédé indirectement pour répondre aux utilisateurs

* proxy : reverse proxy qui va authentifier les requêtes des utilisateurs et les transmettre au serveur web interne.

Des docker-compose complètent ces images pour monter rapidement un environnement pour faire une démonstration ou des essais.

### Intégration Kubernetes

* un **package Helm** : le package permet de fournir un processus de déploiement dans des plateformes Kubernetes

### Hébergement

* un **hébergement dans une infrastructure** négociée par l'Uepal (pour les paroisses ayant choisi l'hébergement centralisé).

L'ensemble des codes seront livrés dans des dépôts Git à accès public. Il est à noter que ces dépôts à accès public ne sont pas les dépôts utilisés pour le développement, et ne contiendront que des "versions numérotées" des codes.

### Prérequis pour l'utilisation des livrables

Les prérequis constituent des éléments que les paroisses qui souhaitent utiliser les images devront gérer elles-mêmes. Nous n'avons pas l'ambitions de couvrir tous les sujets, ni les aborder spécifiquement ici, mais on peut citer en première intention les points suivants :

* les paroisses devront administrer Civicrm et Drupal, et savoir utiliser la solution : les paroisses devront avoir un responsable qui sera administrateur Civicrm et Drupal, pour effectuer les opérations courantes d’administration (création, désactivation de comptes, reset de mot de passe...)

* les paroisses devront savoir comment reconstruire une image : ceci est notamment nécessaire pour palier au cas où un patch de sécurité devrait rapidement appliqué

* les images, lorsqu’utilisées dans des containers, nécessitent des volumes de données pour les données persistantes. Ces volumes seront gérés par l’hôte. On aura besoin de plusieurs volumes, avec la particularité que les volumes de fichiers métiers seront partagés entre containers (cron, web, et sauvegarde, notamment). Les volumes devront être chiffrés, et la sauvegarde devra être chiffrée et déportée vers un site distant en France (pour offrir les meilleures garanties quant à la sécurisation des données)

* l’hôte : il faut le voir comme un environnement d’exécution qui a été fiabilisé, de haute disponibilité, et le plus sécurisé possible 

* un nom de domaine est requis, car il est utile en particulier pour les enregistrements DNS spécifiques pour les techniques antispam (DKIM, SPF)

* de plus, il faudra gérer un service de messagerie lié au domaine, en particulier pour récupérer et traiter les retours en erreur des envois de mails en masse. Les envois de mails en masse devront d’ailleurs passer par un prestataire spécialisé, qui saura mieux gérer des problèmes de mails considérés comme spams, et pourra éventuellement avoir des serveurs reconnus comme des sources de trafic légitime

* il faudra non seulement placer l’ensemble de l’infrastructure derrière un pare-feu, mais il faudra également prévoir un accès à CiviParoisse uniquement via un VPN, accédé via une authentification forte

* des certificats vont également être nécessaires, en particulier pour le serveur web, ce qui pose le problème de la gestion des clefs et d’une PKI. L’ensemble des flux réseaux devra être chiffré

* il faudra également mettre en place des outils de supervision / monitoring des containers, de même que la récupération et l’analyse des logs.

Vu la complexité, les compétences, et la disponibilité des ressources, autant matérielles qu’humaines, qui entrent en jeu, nous recommandons fortement de profiter des économies d'échelle proposées par l’hébergement (et intrinsèquement l’administration) centralisés des instances.
