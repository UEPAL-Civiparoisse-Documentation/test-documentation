# Docker

Documentation archivée le 1er mai 2023.

Docker permet de mettre en oeuvre une architecture microservices, avec des images spécialisées, adaptées aux environnements grâce par exemple à des variables d'environnement que l'on peut passer en argument de docker-run (par exemple).

Le but des images est d'être des images les plus stables possibles - idéalement des images que l'on pourrait utiliser en lecture seule. Les parties variables seraient idéalement toutes dans des volumes.

Chaque Dockerfile du projet dispose d'une directive `LABEL` directement après la directive `FROM` initiale, de sorte à pouvoir invalider simplement le cache en modifiant les valeurs des labels.

Un script de build des images ( `build.sh`) est fourni dans le dépôt.

## Liste des images

Un certain nombre d'images Docker est nécessaire pour exécuter un environnement Civiparoisse complet.

Ces images, dans l'ordre de build, sont :

* [composer_base](DOCKER/composer_base.md)
* [composer_files](DOCKER/composer_files.md)
* [tools](DOCKER/tools.md)
* [init](DOCKER/init.md)
* [cron](DOCKER/cron.md)
* [selfkeys](DOCKER/selfkeys.md)
* [httpd](DOCKER/httpd.md)
* [authenticator](DOCKER/authenticator.md)
* [proxy](DOCKER/proxy.md)

## Dépendances des images entre elles

En se basant sur la directive FROM des Dockerfiles, on obtient les dépendances suivantes :

* ubuntu
   * authenticator
   * selfkeys
   * composer_base
       * composer_files
       * tools
          * cron
          * init
* ubuntu/apache2
   * httpd
   * proxy
* ubuntu/mysql             

Toutefois, il y a également des injections de données d'une image à l'autre  via `COPY --from` :
* selfkeys
    * authenticator
    * httpd
    * proxy
* composer_files
    * httpd
    * tools
        * indirectement (cf from ): cron
        * indirectement (cf from ): init    

Le but de ces dépendances est de "stabiliser" le contenu des images buildées, pour que le contenu qui doit être présent dans plusieurs images soit identique.

## Principe de fonctionnement

On suppose l'existence d'un **forwarder TCP pricipal** qui va faire l'aiguillage vers le bon reverse-proxy  de paroisse en se basant sur le Server Name Indication, qui apparaît en clair dans la connexion TLS. Il y a donc une connexion TLS entre le client et le serveur TLS, mais deux liaisons TCP (client/forwarder et forwarder/reverse proxy).

Le reverse proxy va authentifier les utilisateurs, au travers des identifiants Drupal convoyés via du HTTP Basic au-dessus d'une liason TLS, et probablement un autre moyen (certificat SSL par exemple). En ce qui concerne les identifiants, ils sont testé au travers d'un **authenticateur** qui est branché sur le reverse proxy au travers d'une socket Unix. Chaque requête adressée au reverse proxy doit être authentifiée, et déclenchera un appel à l'authenticateur.

L'authenticateur va effectuer une requête vers le **serveur web interne** pour vérifier que l'utilisateur est bien reconnu. Si tel est le cas, le reverse proxy va transférer la requête vers le serveur web interne pour être traitée.

##Gestion des droits sur les fichiers

La gestion des droits sur les fichiers est un sujet complexe. On suppose que les systèmes de fichiers mis en oeuvre dans le système proposent une gestion des droits traditionnels "à la Unix", avec des droits read-write-exec que l'on peut attribuer à un propriétaire, un groupe, et le reste du monde ; et on suppose qu'on peut identifier le propriétaire et le groupe de référence des fichiers.

Etant donné qu'en principe seul le propriétaire et root peuvent changer les permissions sur un fichier, et qu'il est préférable que le compte root ne soit pas trop utilisé (par exemple pour éviter par la suite une exécution en root via du bit SUID), il a été privilégié de créer un compte spécifique, verrouillé, dont le but est d'être propriétaire des fichiers : le compte paroisse (UID 1000). Un groupe paroisse (GID 1000) est également provisionné.

Le principe général est que les fichiers (pour l'instant, non système) qui sont présents dans les images sont des fichiers destinés à être en lecture seule, tandis que les fichiers des volumes seront en lecture écriture. Il s'agit d'appliquer le principe des droits minimaux pour effectuer les opérations.

L'idée est de donner aux fichiers le propriétaire paroisse, et le groupe www-data. De là, on peut positionner les droits sur les fichiers.

De même, il y a un compte authenticator (UID 1001) et un groupe authenticator (GID 1001) qui ont été crées sur l'image d'authenticateur. Compte également verrouillé.

En ce qui concerne les clefs, il faut se rappeler qu'Apache lit les clefs en tant que l'utilisateur lancé (root) avant de descendre ses droits. Du coup, les clefs n'ont pas besoin d'être lues par www-data.




## Volumes principaux

Ces volumes sont :

* civicomp_dbvol: le volume de base de données
* civicomp_filevol: le volume des fichiers, avec les fichiers spécifiques à l'instance courante
* civicomp_privatevol : le volume des fichiers privés

Les volumes sont montés avec l'option `nocopy: true` car ce fonctionnement correspond au fonctionnement en environnement Kubernetes. En environnement Kubernetes, il y aura des montages supplémentaires (les secrets et les variables d'environnement, par exemple).

## Docker-compose


Deux docker-compose sont disponibles. L'un d'eux, `docker-init.yml` est prévu pour initialiser les volumes, tandis que l'autre (`docker-compose.yml`, nom par défaut) est utilisé pour l'exécution classique.

L'intérêt du docuker-compose réside dans le fait que docker-compose va s'occuper de faire les liaisons nécessaires entre les containers et les volumes, sans avoir besoin de saisir des lignes de commande fastidieuses. C'est donc un système analogue à ce qu'on retrouve dans la déclaration d'objets dans Kubernetes.

### Initialisation des volumes

On suppose qu'on se trouve dans le répertoire contenant les fichiers compose.

``` sh
docker-compose -f docker-init.yml up
docker-compose -f docker-init.yml rm
```

### Utilisation de l'environnement Docker au quotidien

Il faut penser à mettre à jour son `/etc/hosts` pour disposer de la résolution de noms nécessaire. 
Ensuite, on peut faire des opérations quotidiennes avec des commandes comme, par exemple (en étant dans le répertoire contenant les fichiers compose) :

``` bash
docker-compose -d up
# [on fait le travail]
docker-compose stop
```
