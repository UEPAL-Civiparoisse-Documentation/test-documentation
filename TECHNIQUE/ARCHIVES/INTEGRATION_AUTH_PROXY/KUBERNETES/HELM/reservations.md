# Réservations de ressources

Documentation archivée le 1er mai 2023.

## Considérations Générales

La réservation des ressources est un domaine important dans Kubernetes, car les réservations permettent de mieux planifier l'exécution des pods. Les ressources attribuées seraient garanties quand la demande et la limite sont équivalentes. D'où la mise en oeuvre de limitations et de requêtes de mêmes valeurs, et ce au niveau CPU et mémoire.

Le profilage des ressources consommées joue sur plusieurs éléments :

* la RAM et l'espace disque sont des ressources critiques : en particulier, si l'on n'a plus de RAM, le container peut crasher ; en revanche, au niveau CPU, on jouera plutôt sur les performances (à moins d'avoir des prolématiques de temporisation)
* la BD est prévue initialement pour tenir dans 512Mo ; en désactivant performance_schema, on devrait pouvoir tenir dans 350Mo
* on peut limiter au niveau d'Apache le nombre de requêtes servies simultanément ; ces requêtes, lorsqu'elles arrivent sur du code PHP, font entrer en jeu la limite mémoire de PHP. On arrive alors à borner la plus grosse dépense mémoire
* le cron travaille avec un seul environnement PHP à la fois : il devrait donc être possible de borner sa consommation de RAM
* il y a le problème de l'authentification de toutes les requêtes : si on passe à la double authentification, on peut envisager d'utiliser le cache d'authentification malgré sa structure qui n'est plus satisfaisante, ce qui permettrait une meilleure utilisation des ressources
* on ne peut pas encore déterminer avec précision les espaces disques qui seront à allouer, que ce soit pour les volumes persistants ou pour les volumes éphémères (dont logs)
* il a été prévu de pouvoir changer les valeurs des ressources dans chaque déploiement de release : en effet, la charge dépend aussi de l'utilisation (intensive ou non) qui sera faite de l'outil. Il peut être compréhensible de chercher à contenter les plus gros consommateurs tant que ce n'est pas au détriment des autres consommateurs
* il ne faut pas oublier que le nombre cible de déploiements est très conséquent : de ce fait, des petites économies de ressources peuvent avoir toutefois un effet notable
* on ne peut pas additionner aveuglément les ressources de tous les containers : il faut également considérer l'ordre de démarrage des pods, et le fait que les containers d'init d'un pod n'existent pas en même temps que les containers qui constituent la véritable charge de travail
* il ne faut pas oublier que toutes les ressources du cloud ont un coût associé, d'où l'apparition d'une nouvelle démarche : le FinOps, qui cherche à optimiser l'utilisation des ressources et le coût.

## Estimations des ressources d'une instance

### Hypothèses de travail

Il est nécessaire de poser un certain nombre d'hypothèses de travail :

* **on va supposer que l'on va utiliser le cache d'authentification** ; en effet, le coût d'une authentification multifacteur via du SSL est un coût quasiment considérable comme fixe (prix des clefs de sécurité, et éventuellement, prix des certificats (annuels ou multiannuels)s'ils ne sont pas autogénérés), alors que la puissance consommée dépend du nombre de requêtes, et influe donc sur le coût de revient mensuel de l'instance : on va limiter le nombre de requêtes traitables simultanément.

* on va se baser sur des ressources garanties (request=limit sur les réservations), pour ne pas avoir en théorie de problèmes d'éviction

* il faudra rajouter le coût d'un probable firewall qui viendra en coupure des flux

* on va passer sur un modèle de threading Apache en mpm_prefork, qui est de toute façon utilisé pour le PHP si on n'utilise pas php-fpm, et on va utiliser par simplicité ce modèle de threading à la fois sur le serveur web interne et le reverse proxy

* le minimum de mémoire (memory_limit requis par Drupal est 64M); toutefois selon <https://docs.civicrm.org/installation/en/latest/general/requirements/> il faudrait passer au moins à 256M

* les logs passeront par un DaemonSet EFK ou équivalent : du coup, le conteneur de gestion des logs n'est pas comptabilisé dans l'infrastructure d'une instance

* **on suppose qu'on n'intègre pas de serveur de médias, qu'on n'intègre pas le traitement des mails entrants (donc traitement manuel), et qu'on spécialise le traitement des mails sortants**

* configuration mpm_prefork :

```
MAX_CONNECTIONS_PER_CHILD=1
MAX_REQUEST_WORKERS=4
SERVER_LIMIT=4
START_SERVERS=4
MIN_SPARE_SERVERS=0
MAX_SPARE_SERVERS=4
```
Du coup, on sert 4 connexions simultanées, ce qui pourrait être suffisant pour un seul utilisateur.



|Conteneur|Description|Limite mémoire|Limite CPU|
|-----|-----|-----|----|
|proxy|reverse proxy d'authentification|256Mi|250m|
|proxy_init|init du reverse proxy|50Mi|100m|
|auth (proxy_authenticator)|authentificateur|256Mi|250m|
|cron|exécution cron|384Mi|250m|
|cron_dbproxy|accès tunnel BD SSL pour cron|50Mi|100m|
|cron_init|initialisation du pod cron|50Mi|100m|
|httpd|serveur web interne|1536Mi|1000m|
|httpd_dbproxy|accès tunnel BD SSL pour serveur web|100Mi|100m|
|httpd_init|initialisation du pod server web interne|50Mi|100m|
|db|conteneur BD|512Mi|250m|
|db_init|initialisation environnement BD|1024Mi|1000m|
|dbproxy| accès tunnel BD SSL pour le serveur BD|100Mi|100m|

Ordre de démarrage (sans spécialisation du mail sortant):

Les containeurs d'initialisation implémentent les contraintes suivantes :

* `db_init` permet de lancer `db` et `dbproxy` 
* `httpd_init` dépend de `dbproxy` pour lancer `httpd_dbproxy` et `httpd`
* `cron_init` dépend de dbproxy pour lancer `cron` et `cron_dbproxy`
* `proxy_init` dépend du lancement de `httpd` pour lancer `proxy` et `auth`

On en obtient un équivalent de graphe de dépendances :

* `db_init`
	- `db`
	- `dbproxy`
		+ `httpd_init`
			* `httpd_dbproxy`
			* `httpd`
				- `proxy_init`
					+ `proxy`
					+ `auth`
		+ `cron_init`
			* `cron`
			* `cron_dbproxy`



On peut déduire deux phases de lancement d'une instance :

* installation initiale : seul `db_init` est lancé : 1024Mi, 1000m
* ensuite, le pire cas est quand tous les services tournent, car `httpd_init`, `proxy_init` et `cron_init` consomment moins que les services proprement dits. Toutefois, l'unité étant le pod d'exécution (à cause du scheduling sur les noeuds), il faut faire des sommes séparées pour chaque pod :
	- pod db : 1124Mi et 1100m
	- pod cron : 434Mi et 350m
	- pod httpd : 1636Mi et 1100m
	- pod proxy : 512Mi et 500m
	
Ce à quoi on ajoute la spécialisation du mail sortant :

* Client d'accès à 50Mi et 100m pour le pod httpd et le pod proxy
* Pod spécifique d'envoi des mails : 256Mi et 200m

On en arrive donc, avec la spécialisation du mail sortant :

- pod db : __1124Mi et 1100m__
- pod cron : __484Mi et 450m__
- pod httpd : __1686Mi et 1200m__
- pod proxy : __512Mi et 500m__
- pod mail\_sortant : __256Mi et 200m__

Donc une consommation totale max de : __4062Mi et 3450m__





