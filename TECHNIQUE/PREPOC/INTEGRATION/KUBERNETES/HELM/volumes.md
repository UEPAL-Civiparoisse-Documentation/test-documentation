# Volumes (persistants)

Les volumes dont il est question ici sont les volumes persistants. Les volumes temporaires sont la plupart du temps utilisé dans le contexte d'une mise à disposition d'une socket Unix via un emptyDir, ou de génération du répertoire de CA hashés (dans le cadre du déploiement du serveur mail de démo).

Les volumes sont gérés au travers de PersistentVolumeClaim sur une classe de stockage, avec un mode qui est positionné à ReadWriteOnce, ce qui signifie qu'un seul noeud K8S peut monter le volume en lecture écriture à la fois. Néanmoins, les pods exécutés sur le noeud peuvent accéder en ReadWrite au volume, et ce en parallèle. Les tailles présentes dans le chart seront à adapter à l'usage.

On retrouve trois volumes persistants :

* dbvolclaim : le stockage des fichiers de la BD
* filevolclaim : le stockage des fichiers du site civicrm
* privatevolclaim: le stockage des fichiers privés du site civicrm

D'autres volumes pourront faire leur apparition, notamment pour la sauvegarde, les logs, les mails.

