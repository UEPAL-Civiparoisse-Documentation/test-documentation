# Réservations de ressources

## Considérations Générales

La réservation des ressources est un domaine important dans Kubernetes, car les réservations permettent de mieux planifier l'exécution des pods. Les ressources attribuées seraient garanties quand la demande et la limite sont équivalentes. D'où la mise en oeuvre de limitations et de requêtes de mêmes valeurs, et ce au niveau CPU et mémoire.

Le profilage des ressources consommées joue sur plusieurs éléments :

* la RAM et l'espace disque sont des ressources critiques : en particulier, si l'on n'a plus de RAM, le container peut crasher ; en revanche, au niveau CPU, on jouera plutôt sur les performances (à moins d'avoir des prolématiques de temporisation)
* la BD est prévue initialement pour tenir dans 512Mo ; en désactivant performance_schema, on devrait pouvoir tenir dans 350Mo
* on peut limiter au niveau d'Apache le nombre de requêtes servies simultanément ; ces requêtes, lorsqu'elles arrivent sur du code PHP, font entrer en jeu la limite mémoire de PHP. On arrive alors à borner la plus grosse dépense mémoire
* le cron travaille avec un seul environnement PHP à la fois : il devrait donc être possible de borner sa consommation de RAM
* on ne peut pas encore déterminer avec précision les espaces disques qui seront à allouer, que ce soit pour les volumes persistants ou pour les volumes éphémères (dont logs)
* il a été prévu de pouvoir changer les valeurs des ressources dans chaque déploiement de release : en effet, la charge dépend aussi de l'utilisation (intensive ou non) qui sera faite de l'outil. Il peut être compréhensible de chercher à contenter les plus gros consommateurs tant que ce n'est pas au détriment des autres consommateurs
* il ne faut pas oublier que le nombre cible de déploiements est très conséquent : de ce fait, des petites économies de ressources peuvent avoir toutefois un effet notable
* on ne peut pas additionner aveuglément les ressources de tous les containers : il faut également considérer l'ordre de démarrage des pods, et le fait que les containers d'init d'un pod n'existent pas en même temps que les containers qui constituent la véritable charge de travail
* il ne faut pas oublier que toutes les ressources du cloud ont un coût associé, d'où l'apparition d'une nouvelle démarche : le FinOps, qui cherche à optimiser l'utilisation des ressources et le coût.

## Estimations des ressources d'une instance

### Hypothèses de travail

Il est nécessaire de poser un certain nombre d'hypothèses de travail :

* on va se baser sur un seul utilisateur simultané d'une instance

* on va se baser sur des ressources garanties (request=limit sur les réservations), pour ne pas avoir en théorie de problèmes d'éviction

* il faudra rajouter le coût d'une solution de firewalling qui viendra également probablement en coupure des flux ; toutefois une telle solution sera probablement mutualisée du fait du certificat wildcard utilisé : il ne faut donc pas la comptabiliser au niveau de chaque instance

* on va passer sur un modèle de threading Apache en mpm_prefork, qui est de toute façon utilisé pour le PHP si on n'utilise pas php-fpm

* le minimum de mémoire (memory_limit requis par Drupal est 64M); toutefois selon <https://docs.civicrm.org/installation/en/latest/general/requirements/> il faudrait passer au moins à 256M

* les logs passeront par un DaemonSet EFK ou équivalent : du coup, le conteneur de gestion des logs n'est pas comptabilisé dans l'infrastructure d'une instance

* l'ingress Traefik est partagé par l'ensemble des instances, et n'est donc pas à prendre en compte ici. Toutefois, il entre en compte au niveau des ressources du cluster, puisque prévu en DaemonSet. 

* on considère que le service de mails de démo n'a pas à être activé : la consommation de ce service n'est donc pas comptabilisée.

* on considère que le service des tools n'a pas à être activé en situation normale : la consommation de ce service n'est donc pas comptabilisée.


* configuration mpm_prefork :

```
MAX_CONNECTIONS_PER_CHILD=1
MAX_REQUEST_WORKERS=4
SERVER_LIMIT=4
START_SERVERS=4
MIN_SPARE_SERVERS=0
MAX_SPARE_SERVERS=4
LISTEN_BACKLOG=1
```
Du coup, on sert 4 connexions simultanées, ce qui pourrait être suffisant pour un seul utilisateur simultané.



|Conteneur|Description|Limite mémoire|Limite CPU|
|-----|-----|-----|----|
|cron|exécution cron|384Mi|250m|
|cron_dbrouter|sidecar d'accès à la BD cron|100Mi|100m|
|cron_init|initialisation du pod cron|50Mi|100m|
|httpd|serveur web interne|1536Mi|1000m|
|httpd_dbrouter|sidecar d'accès à la BD pour serveur web|100Mi|100m|
|httpd_init|initialisation du pod server web interne|50Mi|100m|
|db|conteneur BD|384Mi|250m|
|db_init|initialisation environnement BD|1024Mi|1000m|
|db_update|update de l'environnement fichiers et BD|1024Mi|1000m|


Ordre de démarrage (sans spécialisation du mail sortant):

Les containeurs d'initialisation implémentent les contraintes suivantes :

* `db_init` permet de lancer `db` 
* `httpd_init` dépend de `dbproxy` pour lancer `httpd_dbrouter` et `httpd`
* `cron_init` dépend de dbproxy pour lancer `cron` et `cron_dbrouter`


On en obtient un équivalent de graphe de dépendances :

* `db_init`
* `db_update`
	- `db`
		+ `httpd_init`
			* `httpd_dbrouter`
			* `httpd`				
		+ `cron_init`
			* `cron`
			* `cron_dbrouter`



On peut déduire deux phases de lancement d'une instance :

* installation initiale : seul `db_init` est lancé : 1024Mi, 1000m
* ensuite : mise à jour : seul `db_update` est lancé : 1024Mi, 1000m
* ensuite, le pire cas est quand tous les services tournent, car `httpd_init` et `cron_init` consomment moins que les services proprement dits. Toutefois, l'unité étant le pod d'exécution (à cause du scheduling sur les noeuds), il faut faire des sommes séparées pour chaque pod :
	- pod db : 384Mi et 250m
	- pod cron : 484Mi et 350m
	- pod httpd : 1636Mi et 1100m

Donc une consommation totale max de : __2504Mi et 1800m__





