# Environnement de développement


## Principe de fonctionnement
L'idée de l'environnement de développement est de fournir des outils rapides et efficaces pour faire les travaux de développement, sans s'encombrer d'une infrastructure complète.
Cette structure est prévue pour des travaux sur des fonctionnalités internes à Civiparoisse, mais n'est pas suffisante s'il y a besoin de tester des éléments plus importants qui mettent en jeu l'infrastructure (ex : envoi / réception de mail).

Dans le dépot des Dockerfiles, il y a non seulement le nécessaire pour constituer les images, mais également trois docker-compose : 

* `docker-init.yml` : contient un container d'initialisation
* `docker-compose.yml` : contient une stack minimale, avec la DB, l'authenticateur, et le proxy et le serveur web
* `docker-compose-bind.yml` : contient la même stack, mais avec une variable d'environnement pour faire un montage en bind du répertoire `/app`

L'idée est la suivante :

* initialiser un environnement pour avoir une installation propre
* récupérer en local l'ensemble des fichiers de l'environnement
* remplacer les emplacements qui contiennent le code spécifique de civiparoisse par des clones git
* utiliser la copie pour la faire tourner dans Docker

## Résolution de noms

Le nom utilisé dans les images est `civicrm.test`. Il y a lieu d'écrire une entrée dans `/etc/hosts` pour résoudre ce nom vers `127.0.0.1`.

## Initialisation du répertoire de destination

```bash
#!/bin/bash

echo "export CIVIPAROISSEDEV=~/devciviparoisse" >>~/.bashrc
source ~/.bashrc
mkdir -p ${CIVIPAROISSEDEV}
```

On définit la variable CIVIPAROISSEDEV dans le bashrc de sorte que la variable soit disponible, et chargée via le shell (bash).

## Récupération des Dockerfiles

Les dockerfiles se trouvent dans un dépôt spécifique :
`git@github.com:UEPAL-CiviParoisse/Dockerfiles.git`

Il faut compiler les images du Dockerfile (pour l'instant, branche `ENV_DEV`), via le build.sh présent dans le dépôt.
La compilation des images prend un certain temps (et un temps certain).


## Initialisation d'une installation

```bash
docker-compose -f docker-init.yml up
docker-compose -f docker-init.yml down --remove-orphans
```

Ceci initialise surtout les volumes dont on aura besoin

## Récupération des données en local
On lance l'environnement :

```bash
docker-compose -f docker-compose.yml up -d
```

Le but est d'avoir le conteneur du serveur web lancé. On utilise `docker container ls` pour voir la liste des conteneurs lancés. Le nom du conteneur dépendra du répertoire contenant le clone du dépôt des dockerfiles.

```bash
docker container ls
CONTAINER ID   IMAGE                        COMMAND                  CREATED              STATUS              PORTS                            NAMES
7d607761b474   uepal_test/proxy             "apache2-foreground"     About a minute ago   Up About a minute   80/tcp, 127.0.0.1:443->443/tcp   git_dockerfiles_civiproxy_1
00d5bf0ad884   uepal_test/authenticator     "/exec/exec.sh"          About a minute ago   Up About a minute                                    git_dockerfiles_civiauthenticator_1
d960762f63b0   uepal_test/httpd             "apache2-foreground"     About a minute ago   Up About a minute   80/tcp, 444/tcp                  git_dockerfiles_civihttpd_1
898dca63e562   ubuntu/mysql                 "docker-entrypoint.s…"   About a minute ago   Up About a minute   3306/tcp, 33060/tcp              git_dockerfiles_cividb_1
22b88b8c9cde   uepal_test/mkdocs-material   "mkdocs serve --dev-…"   2 hours ago          Up 2 hours          127.0.0.1:8000->8000/tcp         mkdocs_material

```

Ici, le nom du conteneur est : `git_dockerfiles_civihttpd_1`. 

On va récupérer l'ensemble du répertoire /app en local, et on va utiliser des règles ACL pour positionner des droits pour l'utilisateur courant (et ainsi, on aura moins besoin de jongler entre les droits dans le conteneur et l'environnement de travail).
La copie créant de nouveaux fichiers, et comme on va binder les volumes, il faut également positionner les droits sur les fichiers du bind. Dans le conteneur courant, www-data est un utilisateur dont l'UID est 33 (pour le moment, mais ça peut changer en fonction de la vie des images); d'où la mise en place des droits pour l'utilisateur 33.

```bash
sudo docker cp git_dockerfiles_civihttpd_1:/app/ ${CIVIPAROISSEDEV}
sudo setfacl -R -m u:$(whoami):rwx ${CIVIPAROISSEDEV}
sudo setfacl -R -m u:33:rxw ${CIVIPAROISSEDEV}
```

On arrête les conteneurs :

```bash
docker-compose -f docker-compose.yml down
```

## Branchement des dépôts gits
Les dépôts gits viendront remplacer les fichiers de l'image, via de l'effacement puis des clones.

|Dépôt|Description|Chemin|
|---|---|---|
|`git@github.com:UEPAL-CiviParoisse/Extensions_CiviCRM.git`|Fichiers extension civiparoisse|`${CIVIPAROISSEDEV}/app/vendor/uepal/fr.uepalparoisse.civiparoisse`|
|`git@github.com:UEPAL-CiviParoisse/CIVIP_INSTALL.git`|Setup civicrm-civiparoisse|`${CIVIPAROISSEDEV}/app/vendor/uepal/civisetup`|
|`git@github.com:UEPAL-CiviParoisse/Modules_Drupal.git`|Module drupal civiparoisse|`${CIVIPAROISSEDEV}/app/web/modules/contrib/fr.uepalparoisse.druparoisse`|

Lorsqu'on branche les dépôts gits, on va certainement faire :

* un mv ou rm des fichiers présents en local dans les répertoires cibles, 
* des `git clone` pour remplacer les fichiers
* des `git checkout` pour se positionner dans les bonnes branches
* des `setfacl` pour positionner les droits pour le serveur web

Lorsqu'on change les branches, il faudra penser à vider les caches (par exemple via les interfaces d'administration).


## Lancement de l'environnement pour dev
L'environnement pour le développement est un peu particulier, dans la mesure où l'on va utiliser un conteneur httpd modifié pour inclure Xdebug, ainsi qu'un montage bind des fichiers locaux. De même, le réseau utilisé est un seul réseau (en raison de problèmes de mise au point sur les scopes). Pour le lancer : 

```bash
docker-compose -f docker-compose-bind.yml up -d
```

Pour arrêter : 

```bash
docker-compose -f docker-compose-bind.yml down
```

## Mots de passe par défaut

Les mots de passe sont présents dans les secrets liés à l'image `init`.

## Configuration xdebug pour vscode : launch.json

On ouvre le dossier ${CIVIPAROISSEDEV}/app dans vscode.
Vscode propose une extension pour xdebug : identificateur : `xdebug.php-debug`.

Cette extension utilise un fichier `launch.json` pour configurer une instance de xdebug. Il convient de faire attention d'écouter sur toutes les IP et sur le bon port (9003), ainsi que de configurer le pathMapping, pour la correspondance des fichiers sources.

```json
{
    // Utilisez IntelliSense pour en savoir plus sur les attributs possibles.
    // Pointez pour afficher la description des attributs existants.
    // Pour plus d'informations, visitez : https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003,
            "hostname": "::",
            "pathMappings": {
                "/app":"${workspaceFolder}"
            }
            
        }

    ]
}
```

Une fois que cela est fait, il suffit donc de positionner les points d'arrêts et de lancer le débugger.

A noter que xdebug, tel que configuré, est prévu pour envoyer des requêtes dans tous les cas à l'IDE.