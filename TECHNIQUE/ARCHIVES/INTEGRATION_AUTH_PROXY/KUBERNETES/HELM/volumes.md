# Volumes (persistants)

Documentation archivée le 1er mai 2023.

Les volumes dont il est question ici sont les volumes persistants. Les volumes temporaires sont la plupart du temps utilisé dans le contexte d'une mise à disposition d'une socket Unix via un emptyDir.

Les volumes sont gérés au travers de PersistentFolumeClaim sur une classe de stockage, avec un mode ReadwriteMany. Les tailles présentes dans le chart seront à adapter à l'usage.

On retrouve trois volumes persistants :

* dbvolclaim : le stockage des fichiers de la BD
* filevolclaim : le stockage des fichiers du site civicrm
* privatevolclaim: le stockage des fichiers privés du site civicrm

D'autres volumes pourront faire leur apparition, notamment pour la sauvegarde, les logs, les mails.

