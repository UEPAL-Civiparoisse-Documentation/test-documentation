# Qualité de code : SonarLint et SonarQube

## Contexte général

La qualité de code est un élément fondamental pour la maintenance d'un code. Cette qualité de code se traduit souvent par le respect de règles. La première règle à respecter étant d'ailleurs le respect de la syntaxe et de la grammaire du langage de programmation utilisé, puis également les manières de programmer prévues (programmation impérative, fonctionnelle, orientée objet...).

Les règles sont aussi connues dans une certaine mesure comme des bests practices, qui sont par ailleurs souvent implémentées dans des frameworks - comme par exemple Symfony, comme utilisé dans Drupal et CiviCRM. Les frameworks permettent de réutiliser une base de code éprouvée et de limiter le code spécifique à une application, ce qui limite donc la maintenance du code spécifique.

Un code est souvent l'oeuvre de plusieurs développeurs : les développeurs doivent donc garder une certaine cohérence au travers de l'ensemble du code, ce qui aboutit à des conventions utilisées sur un projet, qui peuvent avoir trait à des règles de nommages de classes, de méthodes, à la largeur que doit faire une tabulation, à l'utilisation ou non de tabulation. De ce fait, la convention doit être connue et partagée par les développeurs.

On en arrive donc à des outils qui vont chercher à implémenter des modèles de règles, ainsi que des vérifications si le code respecte lesdites règles, ou si le code présente des problèmes. Au niveau de PHP, il existe un certain nombre d'outils comme par exemple `php_cs`, `php_md`, pour ne citer qu'eux. 

## SonarLint

Au niveau du développement de Civiparoisse, un autre outil est particulièrement appréciable : il s'agit de SonarLint.

SonarLint est plutôt à considérer comme un ensemble d'outils multilangages qui peuvent s'intégrer dans un certain nombre d'IDE, ce qui fait qu'on ne force pas un développeur à utiliser un IDE en particulier. De plus, SonarLint intègre déjà un certain nombre de règles (dont certaines ont d'ailleurs déjà permis de trouver des bugs dans le code de l'extension Civiparoisse !). Avec l'intégration à l'IDE, on cherche donc dès la conception du code à éviter un certain nombre d'écueils, et donc on cherche à éviter de la dette technique. Certaines implémentations de SonarLint (puisque ce sont des plugins d'IDE) sont sous licence LGPL-3.0, donc Open Source.

SonarLint est particulièrement appréciable également en raison de la qualité des explications qui sont fournies au niveau des règles : ces explications sont une plus-value importante, car elles pourront parfaire la formation des développeurs.

## SonarQube et SonarScanner

SonarLint est encore plus appréciable si on utilise conjointement SonarQube : alors qu'on cherche avec SonarLint à éviter des problèmes sur chaque fichier en particulier, avec des règles configurées pour son IDE, SonarQube peut centraliser les règles à appliquer sur un projet (on notera que ce partage est fait dans d'autres outils via des fichiers dans le dépôt git).

L'intérêt principal de SonarQube (dont certaines versions sont également Open Source) est qu'il va récupérer les résultats d'analyse de code qui sont effectués par un autre outil : sonar-scanner, qui dispose également de certaines versions Open Source. Les résultats du scanner vont être remontés et exploités via l'interface de SonarQube. SonarQube peut remonter les problèmes de chaque fichier, mais peut également remonter des problèmes plus globaux, comme des duplications de blocs de codes, et dispose d'une interface qui semble ergonomique et efficace.

Avec les analyses successives, on peut également suivre la qualité du développement. Il est également possible d'intégrer ce suivi dans certains tunnels d'intégration continue, de sorte à éviter de publier un code qui n'est pas satisfaisant aux normes définies pour le projet.

## SonarCloud

SonarCloud est un produit payant sous forme de Software As A Service.  On peut donc le voir dans un certain point de vue comme un SonarQube managé. Le modèle de licence courant dépend du nombre de lignes de codes analysées ; pour l'heure, Civiparoisse ne dépasse pas les 100 000 lignes (plutôt moins de 15 000 lignes selon une exécution `phploc` avec les paramètres par défaut), et serait donc facturé (selon le modèle de licensing courant - voir <https://www.sonarsource.com/plans-and-pricing/#sonarcloud>) dans la plage de prix la plus basse.

## En conclusion

L'usage qui a déjà été fait des versions Open Source gratuites de SonarLint et SonarQube a montré un grand potentiel dans l'utilisation de ces outils, et présentent un intérêt certain pour le projet. L'adoption de SonarCloud sera à étudier spécifiquement, également en fonction de la place que l'on veut faire (ou non) à l'outil dans les tunnels d'intégration du projet Civiparoisse - d'autant plus que des coûts supplémentaires pourraient s'ajouter (ex : minutes de workflow github).
