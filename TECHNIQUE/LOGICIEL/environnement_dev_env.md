# Personnalisation de l'environnement de développement

!!! warning "Attention !"

    Les éléments de ce fichier n'ont pas été vérifiés ; les lignes de commandes n'ont pas été testées.

L'environnement de développement a été pensé initialement pour utiliser les images issues des Dockerfiles avec les variables d'environnement par défaut. Toutefois, il est parfois nécessaire de pouvoir personnaliser les valeurs, dont notamment le nom du serveur, et ceci peut avoir un impact sur les clefs TLS.

## Aspects théoriques

### Personnalisation de l'environnement

L'approche qui est prévue pour Helm est d'utiliser le fichier de valeurs, qui hérite des valeurs par défaut, pour préparer les éléments de configuration nécessaires (ConfigMap, Secrets, entre autres) des pods. En revanche, cette approche est plus difficile à mettre en oeuvre dans le cas de Compose.

L'approche retenue est d'utiliser les fichiers d'environnement. On trouve trois type de fichiers :

* le fichier d'environnement spécifié au niveau de la ligne de commande (`--env-file`) : ce fichier permet d'utiliser des variables qui sont référencées directement dans le fichier yml de docker-compose, et les valeurs peuvent être propagées, si elles sont au minimum mentionnées dans le docker-compose, jusqu'au container
* le premier fichier d'environnement spécifié au niveau des configurations via `env_file` : ce fichier est prévu pour contenir des valeurs "par défaut" qui sont normalement celles présentes dans les images. Ces valeurs peuvent se retrouver par `docker image inspect <nom image>`.
* le deuxième fichier d'environnement spécifié via `env_file` (fichier avec suffixe `override dans la configuration`): comme c'est le deuxième fichier, il sera pris en compte en priorité par rapport au premier fichier.

Lorsque l'on souhaite modifier l'environnement, il faut donc modifier le fichier envfile principal et les fichiers d'override, en gardant la correspondance des valeurs.

!!! warning "Attention !"
    Si on modifie `SERVERNAME`, il faut également penser à inclure la modification dans le `TRUSTED_HOST_PATTERNS`.

### Clefs TLS
Au niveau de Helm, le template qui gère les clefs TLS est capable de préparer les hiérarchies de certificats (avec CA autosignés), et est prévu pour ne remplacer les clefs que si elles n'existent pas.

Au niveau de docker-compose, ce mécanisme n'existe pas, et il a été nécessaire de préparer une image KEYS_INIT pour préparer les jeux de clefs qui vont être stockés dans un montage de type bind (vers le répertoire `genkeys`).

Pour utiliser cette image, il faudra passer la valeur de la variable d'environnement SERVERNAME au container. Le fichier docker-init-envkeys.yml est prévu pour cela, en utilisant en plus le fichier principal d'environnement (`--env-file`).

## Aspects pratiques

En pratique, on suivra le cheminement suivant pour utiliser ces éléments :

* récupération du dépôt des Dockerfiles, et se positionner sur la branche (peut-être `ENV_DEV_KEYS`)
* personnalisation des fichiers d'environnement `envfile` principal et les fichiers `envfile_*_override`, en fonction des besoins du développeur 
* build des images, grâce au script `build.sh`
* génération des clefs :
 
```bash
#!/bin/bash
docker-compose -f docker-init-envkeys.yml --env-file envfile up
docker-compose -f docker-init-envkeys.yml --env-file envfile down
```

* installation de l'environnement (volumes et BD) : 

```bash
#!/bin/bash
docker-compose -f docker-init-env.yml --env-file envfile up
docker-compose -f docker-init-env.yml --env-file envfile down
```

* démarrage avec l'environnement des containers :

```bash
#!/bin/bash
docker-compose -f docker-compose-env.yml --env-file envfile up
```

* se logguer dans le système pour vérifier que tout s'est bien passé
* récupérer le /app du container de serveur web interne comme décrit dans l'environnement de dev (en pensant à régler aussi les droits)
* arrêter l'environnement des containers :

```bash
#!/bin/bash
docker-compose -f docker-compose-env.yml --env-file envfile down
```

* démarrer l'environnement bindé avec les répertoires locaux (comme décrits dans l'environnement de développement)

```bash
#!/bin/bash
docker-compose -f docker-compose-bind-env.yml --env-file envfile up
```

* faire le travail désiré (développement)

* arrêter l'environnement

```bash
#!/bin/bash
docker-compose -f docker-compose-bind-env.yml --env-file envfile down
```