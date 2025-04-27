# Documentation du Pré-POC (Proof Of Concept)

Avant d'effectuer le POC proprement dit, il s'agit d'affiner le système global envisagé pour pouvoir déployer une version représentative de Civiparoisse.

## Utilisabilité 

L'UEPAL souhaite privilégier l'utilisabilité de CiviParoisse. En conséquence, le reverse proxy envisagé pour authentifier l'ensemble des requêtes ne sera pas mis en oeuvre dans cette version.

De même, le système semble plus facilement opérationnel s'il n'a pas d'impacts directs sur les systèmes d'information existants dans les paroisses (notamment DNS).

## Efficience des ressources attribuées

Il nous est demandé de chercher à optimiser l'utilisation des ressources, ce qui correspond au moins partiellement à une démarche Finops.

Le livre Cloud FinOps d'Oreilly est considéré comme livre de référence ; ce livre propose une double approche : une optimisation des ressources utilisées par les équipes techniques (en particulier, supprimer les ressources réservées et plus utilisées) d'une part, et une optimisation du montage financier avec le fournisseur cloud d'autre part.

En ce qui concerne l'optimisation des ressources utilisées, plusieurs leviers peuvent éventuellement être actionnés :

* Les réservations de ressources comportent une demande de base, et un niveau d'utilisation maximum. Au lieu de faire correspondre les deux niveaux de réservations, ce qui correspond à des ressources garanties si les charges de travail sont planifiées, on va faire une différence entre les deux niveaux de ressource, en espérant que les charges de travail maximales ne soient pas atteintes simultanément sur l'ensemble des instances. *Cette approche n'a pas été retenue dans notre projet pour le moment*.

* Toutes les instances de Civiparoisse mettent en oeuvre un moteur mysql : il pourrait être plus efficace de provisionner un seul moteur mysql, mais avec plus de ressources : les ressources sont donc mises plus en commun en comptant sur le fait que l'ensemble des utilisateurs ne lanceront pas des travaux lourds sur la BD en même temps. Par ailleurs, le service MySQL pourrait éventuellement sorti hors de Kubernetes, puisque les providers clouds proposent souvent MySQL comme un service qui peut être acheté à part. De ce fait, il convient alors de garder une connexion "classique" vers la base de données, ce qui conduit à utiliser un proxy "classique" pour la connexion chiffrée vers la BD : mysqlrouter semble être une solution de compromis intéressante dans ce cas. De même, il faut également mettre à jour le système d'installation pour ne pas donner SUPER à l'utilisateur de BD. *Cette approche n'a pas été retenue dans notre projet pour le moment*.

* La mise en oeuvre de PHP-FPM pourrait éventuellement aider également à optimiser l'utilisation des ressources : on peut en effet chercher à définir la taille du pool PHP d'une instance, et limiter en parallèle le nombre de requêtes traitées simultanément par Apache : la différence entre les deux permet de servir les fichiers statiques plus rapidement, et pourrait concourir à des charges de travail mieux connues et mieux maîtrisées. *Cette approche n'a pas été retenue dans notre projet pour le moment*.

* Pour le moment, il a été décidé de se passer de conteneurs spécialisés pour l'envoi et la récupération des mails ; CiviCRM fera donc également ces travaux, et disposera de ce fait des identifiants nécessaires.

## Constitution d'une instance Civiparoisse
L'instance Civiparoisse sera constituée par :

* un pod avec la BD, et son service associé (mysql)
* un pod avec le container serveur web, et un mysqlrouter, et le service associé (httpd)
* un pod de cronjob avec le container cron et un mysqlrouter
* un pod de cronjob pour la sauvegarde avec un mysqlrouter : sauvegarde avec mysqldump et tar gz des fichiers

## Services centraux
DNS : à voir si on peut passer avec le plugin kubernetes pour la découverte, metadatas pour la publication de metadonnées et le plugin template en se basant sur les metadatas pour avoir une réponse générique.

EFK : à voir si on ne peut pas faire un Fluentd général et balancer tous les logs vers un seul service.

Ces deux pistes n'ont pas encore été étudiées dans le cadre de notre projet.


