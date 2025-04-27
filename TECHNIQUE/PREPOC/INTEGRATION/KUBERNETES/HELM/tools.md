# Les outils

Un stateful set un peu particulier est présent dans le chart Helm : il s'agit d'un pod pour exécuter une image des outils. Le pod est équipé également avec l'accès à la BD.
Ce pod est prévu pour pouvoir entrer rapidement dans un environnement Civiparoisse pour effectuer des opérations manuelles - et comme il s'agit d'un stateful set, le nom du pod généré sera stable dans le temps. Ainsi, ce pod pourra par exemple servir pour des imports de données. Toutefois, comme il n'est normalement pas utilisé, son nombre de réplicas est initialement à zéro.

En revanche, il est à noter qu'il est bien possible d'arriver dans des situations où le pod ne peut pas être schedulé simplement : en effet, il prévoit également l'accès au volume de bases de données. La difficulté pourrait donc survenir si le pod de la base de données serait sur un autre noeud que les autres services.

Enfin, le nombre de replicas des différentes charges, ainsi que la suspension ou non du cronjob, est configurable en settant les valeurs adéquates : il est ainsi alors possible de disposer par exemple uniquement des outils, ce qui pourrait faciliter certaines opérations manuelles, comme une éventuelle restauration manuelle de données.

