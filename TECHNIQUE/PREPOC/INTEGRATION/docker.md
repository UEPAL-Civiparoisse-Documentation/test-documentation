# Docker

Docker permet de mettre en oeuvre une architecture microservices, avec des images spécialisées, adaptées aux environnements grâce par exemple à des variables d'environnement que l'on peut passer en argument de docker-run (par exemple).

Le but des images est d'être des images les plus stables possibles - idéalement des images que l'on pourrait utiliser en lecture seule. Les parties variables seraient idéalement toutes dans des volumes.

Chaque Dockerfile du projet dispose d'une directive `LABEL` directement après la directive `FROM` initiale, de sorte à pouvoir invalider simplement le cache en modifiant les valeurs des labels.

Un script de build des images ( `build.sh`) est fourni dans le dépôt.

```bash
#!/bin/bash
set -xev
export EXPBUILDVERSION=`cat VERSION`
echo ${EXPBUILDVERSION}
docker build --rm --no-cache -t uepal_test/selfkeys:${EXPBUILDVERSION} -t uepal_test/selfkeys --build-arg BUILDVERSION=${EXPBUILDVERSION} SELFKEYS 
docker build --rm --no-cache -t uepal_test/keys_init:${EXPBUILDVERSION} -t uepal_test/keys_init --build-arg BUILDVERSION=${EXPBUILDVERSION} KEYS_INIT
docker build --rm --no-cache -t uepal_test/mysql_tls_router:${EXPBUILDVERSION} -t uepal_test/mysql_tls_router --build-arg BUILDVERSION=${EXPBUILDVERSION} MYSQL_TLS_ROUTER
docker build --rm --no-cache -t uepal_test/mysql_tls_server:${EXPBUILDVERSION} -t uepal_test/mysql_tls_server --build-arg BUILDVERSION=${EXPBUILDVERSION} MYSQL_TLS_SERVER
docker build --rm --no-cache -t uepal_test/composer_base:${EXPBUILDVERSION} -t uepal_test/composer_base --build-arg BUILDVERSION=${EXPBUILDVERSION} COMPOSER_BASE
docker build --rm --no-cache -t uepal_test/composer_files:${EXPBUILDVERSION} -t uepal_test/composer_files --build-arg BUILDVERSION=${EXPBUILDVERSION} COMPOSER_FILES
docker build --rm --no-cache -t uepal_test/tools:${EXPBUILDVERSION} -t uepal_test/tools --build-arg BUILDVERSION=${EXPBUILDVERSION} TOOLS
docker build --rm --no-cache -t uepal_test/tools_debug:${EXPBUILDVERSION} -t uepal_test/tools_debug --build-arg BUILDVERSION=${EXPBUILDVERSION} TOOLS_DEBUG
docker build --rm --no-cache -t uepal_test/init:${EXPBUILDVERSION} -t uepal_test/init --build-arg BUILDVERSION=${EXPBUILDVERSION} INIT
docker build --rm --no-cache -t uepal_test/cron:${EXPBUILDVERSION} -t uepal_test/cron --build-arg BUILDVERSION=${EXPBUILDVERSION} CRON
docker build --rm --no-cache -t uepal_test/httpd:${EXPBUILDVERSION} -t uepal_test/httpd --build-arg BUILDVERSION=${EXPBUILDVERSION} HTTPD
docker build --rm --no-cache -t uepal_test/httpd_debug:${EXPBUILDVERSION} -t uepal_test/httpd_debug --build-arg BUILDVERSION=${EXPBUILDVERSION} HTTPD_DEBUG
docker build --rm --no-cache -t uepal_test/update:${EXPBUILDVERSION} -t uepal_test/update --build-arg BUILDVERSION=${EXPBUILDVERSION} UPDATE
docker build --rm --no-cache -t uepal_test/test_opensmtpd_dovecot:${EXPBUILDVERSION} -t uepal_test/test_opensmtpd_dovecot --build-arg BUILDVERSION=${EXPBUILDVERSION} TEST_OPENSMTPD_DOVECOT
docker build --rm --no-cache -t uepal_test/gen_hashed_ca:${EXPBUILDVERSION} -t uepal_test/gen_hashed_ca --build-arg BUILDVERSION=${EXPBUILDVERSION} GEN_HASHED_CA
docker build --rm --no-cache -t uepal_test/traefik_docker:${EXPBUILDVERSION} -t uepal_test/traefik_docker --build-arg BUILDVERSION=${EXPBUILDVERSION} TRAEFIK_DOCKER

```

Ce script utilise un fichier nommé `VERSION` pour tagger chaque image générée avec la version contenue dans le fichier, ainsi que le tag `latest`. La version est également positionnée dans un argument d'un label dans chaque image Docker générée. Ceci permet d'avoir un numéro de version uniforme pour l'ensemble des images générées.

## Liste des images

Un certain nombre d'images Docker est nécessaire pour exécuter un environnement Civiparoisse complet.

Ces images, dans l'ordre de build, sont :

* [selfkeys](DOCKER/selfkeys.md)
* [keys_init](DOCKER/keys_init.md)
* [mysql_tls_router](DOCKER/mysql_tls_router.md)
* [mysql_tls_server](DOCKER/mysql_tls_server.md)
* [composer_base](DOCKER/composer_base.md)
* [composer_files](DOCKER/composer_files.md)
* [tools](DOCKER/tools.md)
* [tools_debug](DOCKER/tools_debug.md)
* [init](DOCKER/init.md)
* [cron](DOCKER/cron.md)
* [httpd](DOCKER/httpd.md)
* [httpd_debug](DOCKER/httpd_debug.md)
* [update](DOCKER/update.md)
* [test_opensmtpd_dovecot](DOCKER/test_opensmtpd_dovecot.md)
* [gen_hashed_ca](DOCKER/gen_hashed_ca.md)
* [traefik_docker](DOCKER/traefik_docker.md)

## Dépendances des images entre elles

En se basant sur la directive FROM des Dockerfiles, on obtient les dépendances suivantes :

* ubuntu:focal
   * test\_opensmtpd\_dovecot
* ubuntu:lunar
   * mysql\_tls\_router
   * keys\_init
   * gen\_hashed\_ca
   * selfkeys
   * composer\_base
       * composer\_files
       * tools
          * tools\_debug
          * cron
          * init
          * update
* ubuntu/apache2
   * httpd
   * httpd\_debug
* ubuntu/mysql
   * mysql\_tls\_server
* traefik
   * traefik\_docker

Toutefois, il y a également des injections de données d'une image à l'autre  via `COPY --from` :

* selfkeys
    * httpd
       * indirectement (cf from): httpd\_debug
    * traefik\_docker
    * mysql\_tls\_router
    * mysql\_tls\_server
    * test\_opensmtpd\_dovecot
* composer\_files
    * httpd
        * indirectement (cf from): httpd\_debug
    * tools
        * indirectement (cf from): cron
        * indirectement (cf from): init    
        * indirectement (cf from): tools\_debug
        * indirectement (cf from): update

Le but de ces dépendances est de "stabiliser" le contenu des images buildées, pour que le contenu qui doit être présent dans plusieurs images soit identique.

## Principe de fonctionnement

Traefik va récupérer tout le trafic entrant, va servir de terminaison HTTPS avec le navigateur du client web, et va utiliser le hostname pour déterminer quelle instance de Civiparoisse contacter (via une liaison HTTPS "interne").

##Gestion des droits sur les fichiers

La gestion des droits sur les fichiers est un sujet complexe. On suppose que les systèmes de fichiers mis en oeuvre dans le système proposent une gestion des droits traditionnels "à la Unix", avec des droits read-write-exec que l'on peut attribuer à un propriétaire, un groupe, et le reste du monde ; et on suppose qu'on peut identifier le propriétaire et le groupe de référence des fichiers.

Etant donné qu'en principe seul le propriétaire et root peuvent changer les permissions sur un fichier, et qu'il est préférable que le compte root ne soit pas trop utilisé (par exemple pour éviter par la suite une exécution en root via du bit SUID), il a été privilégié de créer un compte spécifique, verrouillé, dont le but est d'être propriétaire des fichiers : le compte paroisse (UID 1001). Un groupe paroisse (GID 1001) est également provisionné.

!!! Warning "Compte ubuntu"
    Dans ubuntu:lunar, a été constatée l'existence d'un compte ubuntu (UID 1000). Ce compte a été verrouillé dans les images dérivant de ubuntu:lunar.

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
