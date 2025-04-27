# Composer et intégration continue

Composer permet d'industrialiser une grande partie de la gestion du code de la solution Civiparoisse. L'écosystème Composer permet de faciliter la mise à jour des packages logiciels de la solution, et permet même d'être averti de certaines failles de sécurité.
Les vérifications pourront devenir en fonction des nécessités une ou plusieurs étapes des tunnels d'intégration continue.

## Importance du fichier composer.json et composer.lock

En théorie, le fichier composer.lock est une conséquence du fichier composer.json, et n'aurait pas besoin d'être versionné. C'est vrai lorsqu'on a une installation à faire, puisque le composer.lock est généré à ce moment là. En revanche, dans le cadre d'une maintenance opérationnelle, le composer.lock devient très important, car il fait parti des éléments qui pourront être utilisés dans le cadre du Software Bill Of Materials, et qui pourra être utilisé par des outils comme Dependabot pour vérifier si des packages utilisés dans le projet ont des failles de sécurité (voir <https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file>), ou si des mises à jour sont disponibles.

Il devient donc important de pouvoir générer un fichier de lock, et de le versionner.

## Plateforme d'utilisation de Composer

!!! Danger "Problématique de la plateforme d'utilisation de Composer"
    Etant donné que les packages Composer listés dépendent également de la plateforme courante d'exécution de Composer (version PHP, extensions PHP chargées), il semble nécessaire d'exécuter Composer depuis une image qui aura des caractéristiques PHP équivalentes à l'image de production.

## Génération du fichier de lock uniquement

On va utiliser la commande `composer validate --check-lock` pour vérifier la correspondance entre les fichiers composer.json et composer.lock principaux.

## Création d'un fichier de lock

On peut utiliser `composer update -n --no-install` pour générer le fichier de lock.

## Synchronisation entre composer.json et composer.lock

On peut utiliser `composer update -n --no-install` pour mettre à jour le fichier de lock. 

!!!Danger "Paramètre --lock : risque de désynchronisation"

    A noter que l'utilisation du paramètre `--lock` mettrait uniquement à jour les sommmes de contrôle sans mettre à jour le contenu du fichier lock.

Eventuellement, si des plugins sont chargés, on peut vouloir ne pas utiliser les plugins, et rajouter l'option `--no-plugins` si des plugins sont présents et que l'on ne souhaite pas les charger.

## Vérification des vulnérabilités

Deux types de vérifications sont possibles :

* On peut souhaiter vouloir vérifier un lock file : `composer audit --locked` va retourner via le code de retour le nombre de vulnérabilités (on espère 0, mais ça peut monter jusqu'à 255).
* On peut aussi vouloir vérifier en fonction de ce qui est installé : `composer audit` - mais il faut que les fichiers aient été installés, sans quoi il n'y a pas de vérification effectuée.

## Vérification des mises à jour possibles

On peut utiliser `composer show -lo` pour voir quels packages pourraient bénéficier de mises à jour, et quelle est la dernière version qui est disponible. En plus, Composer analyse le niveau de difficulté (et donc intègre intrinsèquement une partie de la dimension du risque) de la mise à jour.

On peut travailler en se basant sur le lock file en utilisant `composer show -lo --locked` ; sans cette option, la commande travaille avec ce qui est installé. De même, on peut souhaiter avoir un code de retour différent de zéro s'il y a des mises à jours qui sont proposées, grâce à l'option `--strict`.

## Vérification des extensions de la plateforme

La commande `composer check-platform-reqs` peut vérifier si tous les prérequis d'extensions PHP sont présents dans la plateforme. Par défaut, le répertoire *vendor* est utilisé pour recenser les prérequis, mais s'il n'est pas présent ou si l'option `--lock` est précisée, on utilise les préconisations du fichier composer.lock.

## Installation de cv et drush

Etant donné qu'on réduit le nombre d'images et qu'on ne construit qu'une image Civiparoisse, il devient intéressant d'intégrer cv et drush depuis Composer, puisque cela semble à nouveau possible. On peut éventuellement envisager d'installer cv et drush dans des lieux différents, et utiliser des fichiers composer.json et composer.lock supplémentaires, de sorte à pouvoir continuer à bénéficier des outils de maintenance intégrés par Composer.

## Composer bump

La commande `composer bump` permettrait de modifier le composer.json pour que les versions visées par le composer.json sont au minimum égales à celles installées, pour éviter que des dépendances soient downgradées. Il faudrait voir dans le temps si cette commande peut être utile.
