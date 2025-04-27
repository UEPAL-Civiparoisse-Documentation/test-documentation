# Distribution du code de Civiparoisse

La distribution du code de Civiparoisse est un point crucial de l’industrialisation de l’installation des instances CiviCRM. On a déjà vu que Composer procurait une facilité d’installation et de mise à jour des instances de Drupal et du coeur de CiviCRM. Il est donc intéressant de capitaliser sur Composer. Pour ce faire, il était tout d’abord nécessaire d’unifier les modules civiparoisse existants en un seul, de le préparer pour devenir un package, puis générer les packages, et enfin les utiliser.

## Unification des extensions Civiparoisse

L’unification des extensions Civiparoisse s’est imposée pour plusieurs raisons :

* les différentes extensions avaient fonctionnellement le même but : fournir la gestion opérationnelle des paroisses, et dépendaient les unes des autres pour fournir la fonctionnalité globale : ces éventuelles interdépendances auraient pu devenir problématique pour la maintenance des extensions, d’aunt plus que certains éléments nécessitent des « racines » (par exemple : création d’un menu « racine » Uepal)

* différentes extensions peuvent conduire à de la duplication de code : en effet, certains mécanismes comme la gestion des menus, les settings, auraient pu être requis dans le temps par plusieurs modules

* la gestion de versions est facilitée : une seule version était déjà prévue pour tout le code via les tags git : on pourra donc aligner la version de info.xml et les tags

* les séparations qui semblaient nécessaires restent présentes, mais sont intégrées dans des niveaux plus profonds : on retrouve différents « namespaces », comme CRM_Civiparoisse_Parametres,CRM_Civiparoisse_Pages… Il en est de même pour les templates associés
 
* Composer peut utiliser un dépôt git comme un repository qui contient un package versionné avec les tags ; si on ne souhaite pas utiliser directement le dépôt git, il est également possible de générer un site web « statique » avec Satis (<https://github.com/composer/satis> ) qui contient les packages générés, et utiliser ce site web comme repository Composer.

La fusion des différentes extensions a eu lieu dans l’issue 39.

## Préparation à l’utilisation du dépôt Git comme repository Composer

Le code, une fois unifié, peut être utilisé assez simplement comme un package composer : il suffit d’un fichier composer.json qui indique le vendor et le nom du package, mis à la racine du code, comme par exemple :

``` json
{
"name":"uepal/fr.uepalparoisse.civiparoisse"
}
```

Ensuite, les versions du code sont gérées avec des tags git qui prennent la forme du préfixe « v » auquel on accole le numéro de version (on obtient par exemple v0.0.1, v0.0.2,…). (Voir <https://getcomposer.org/doc/articles/versions.md> , <https://getcomposer.org/doc/02-libraries.md> et <https://getcomposer.org/doc/05-repositories.md#vcs> ).

CiviCRM facilite la maintenance des extensions qui sont des packages Composer : en effet, CiviCRM inclut dans les lieux où il cherche les extensions le lieu « CMS_ROOT/vendor », qui n’est autre que le lieu où Composer met par défaut les fichiers issus d’un composer update. Il est à
noter que le plugin d’installeur n’a pas fonctionné avec la procédure d’installation, car un répertoire ext situé à la racine du projet n’est pas scanné par défaut pour les extensions.

## Mise à disposition du package

On arrive ensuite à la distribution du package, qui peut être réalisée par deux approches complémentaires : distribuer le package via un dépôt git, et éventuellement distribuer le package via un site web.

### Mise à disposition via dépôt Git

La mise à disposition via dépôt Git est plus ou moins obligatoire, dans la mesure où les générateurs de site web utiliseront le dépôt Git comme source.

Il convient de ne pas utiliser directement comme source le dépôt de développement, car celui-ci contient l’historique du travail, dont les messages de commit, ainsi que les emails des développeurs : ces éléments n’ont pas besoin d’être vus par tous. On utilisera de ce fait un autre dépôt Git (sur Github) que l’on va utiliser uniquement pour publier les releases.

La procédure de mise à disposition va faire intervenir trois dépôts au total :

* le dépôt de publication
* le dépôt des sources
* un dépôt local de travail

La procédure est au bout du compte assez simple, on va travailler uniquement depuis le dépôt local de travail:

* on crée le dépôt de travail en clonant le dépôt de publication (git clone)
* on ajoute une référence au dépôt source au dépôt de travail : depuis le dépôt de travail :

``` sh
git remote add source <url ou chemin du dépôt d’origine>
```
* depuis le dépôt de travail, on récupère le commit correspondant au tag, et qui sera mis dans `FETCH_HEAD` selon ce qu’on voit dans la sortie de la commande :

``` sh
git fetch --depth 1 source tags/v2.0
```

* on crée une branche en orphan (sans historique et sans commit) depuis `FETCH_HEAD` :

```sh
git checkout --orphan release_v2 FETCH_HEAD
```

* on commite : attention à l’identité configurée (nom et mail d’utilisateur), et au message de commit : ces infos seront publiées ensuite.

``` sh
git commit -m "Release v2.0"
```

* on taggue :

``` sh
git tag v2.0 release_v2
```

* on pousse le tag vers le dépôt de release (qui est origin depuis le clone) :

``` sh
git push origin tags/v2.0
```

On peut tester ensuite le fonctionnement : on crée un composer.json du genre :

``` json
{
"repositories":[
{
"type":"vcs",
"url":"file:///home/ubuntu/DRUPAL_TEST/test3/release"
}],
"require":{
"uepal/fr.uepalparoisse.civiparoisse":"2.0"
}
}
```

Et ensuite, on peut utiliser composer pour lister les packages :

```
ubuntu@wpcore:~/DRUPAL_TEST/test3/usage$ ./composer.phar show -a uepal/fr.uepalparoisse.civiparoisse
Xdebug: [Step Debug] Could not connect to debugging client. Tried:
127.0.0.1:9003 (through xdebug.client_host/xdebug.client_port) :-(
Warning from https://repo.packagist.org: Support for Composer 1 is deprecated and some packages will not be available. You should upgrade to Composer 2. See https://blog.packagist.com/deprecating-composer-1-support/
name     : uepal/fr.uepalparoisse.civiparoisse
descrip. :
keywords :
versions : v2.0
type     : library
homepage :
source   : [git] /home/ubuntu/DRUPAL_TEST/test3/release
a85389ed3454e345d2bc67d4b0873f5d06e5dab8
dist     : []
names    : uepal/fr.uepalparoisse.civiparoisse
```
On peut également installer le package pour tester.

Si l’on veut mettre le code à la disposition de tous, on peut très bien faire du dépôt de release un dépôt « public ».

### Mise à disposition via site web

La génération des packages à mettre sur un site web présuppose de l’utilisation d’un système de versionning (en l’occurence, un dépôt git) qui va être interrogé pour générer les packages de code qui vont être mis à disposition. Le plus propre est de connecter l’outil sur le dépôt de release généré dans le point au-dessus, car il apparaît que les sources sont référencées dans les fichiers json générés.

Si les sources sont publiques, et donc mises à la disposition de tous, alors on peut éventuellement penser à publier directement chez packagist.org, qui va s’occuper lui-même de parcourir périodiquement le dépôt git de release et générer les packages (voir : <https://packagist.org/about> ). En revanche, il faut éventuellement mettre en place une configuration de hook sur le dépôt git pourque github avertisse packagist quand une modification est effectuée, de sorte à ne pas avoir besoin d’attendre le délai de la semaine.

Si on veut utiliser un dépôt privé, il faudra alors plutôt se tourner vers packagist.com (<https://packagist.com/> ), mais il y aura alors des coûts associés. Un autre type de solution est la génération du site web via Satis (<https://github.com/composer/satis> ). Satis est configuré via un
ficher de configuration, qui est par défaut `satis.json`. Un exemple de configuration est le suivant :

``` json
{
"name":"uepal/satis",
"homepage":"http://satis.test",
"repositories":[
{
"type":"vcs",
"url":"/home/ubuntu/DRUPAL_TEST/test3/release"
}
],
"require-all":true,
"archive":{
"directory":"dist",
"format":"tar",
"checksum":true,
"skip-dev":true
},
"output-dir": "TEST_RELEASE",
"output-html":true,
"providers":true,
"pretty-print":true
}
```

Les paramètres variants sont la homepage, qui sera l’URL de publication finale du contenu, les repositories qui seront analysés pour générer les packages, et l’output-dir, qui est le répertoire de destination. Un simple appel de satis avec le paramètre build permettra ensuite de générer le site. La documentation de satis est assez complète (dont <https://github.com/composer/satis/blob/main/docs/using.md> et <https://github.com/composer/satis/blob/main/docs/config.md> ), et complète également celle de
Composer pour les dépôts privés (<https://getcomposer.org/doc/articles/handling-private-packages.md> ). Comme il s’agira d’un site web « indépendant » dans une certaine mesure, on pourra mettre en œuvre des éléments comme l’authentification des clients.

La génération peut par exemple donner des fichiers comme :

``` sh
ubuntu@wpcore:~/DRUPAL_COMPOSER2/TOOLS/satis/TEST$ ls -lR
.:
total 156
drwxrwxr-x 3 ubuntu ubuntu 4096 nov. 13 16:14 dist
-rw-rw-r-- 1 ubuntu ubuntu 140235 nov. 13 16:14 index.html
drwxrwxr-x 3 ubuntu ubuntu 4096 nov. 13 16:14 p
drwxrwxr-x 3 ubuntu ubuntu 4096 nov. 13 16:14 p2
-rw-rw-r-- 1 ubuntu ubuntu 369 nov. 13 16:14 packages.json
./dist:
total 4
drwxrwxr-x 3 ubuntu ubuntu 4096 nov.13 16:14 uepal
./dist/uepal:
total 4
drwxrwxr-x 2 ubuntu ubuntu 4096 nov.13 16:14 fr-uepalparoisse-civiparoisse./dist/uepal/fr-uepalparoisse-civiparoisse:
total 1128
-rw-rw-r-- 1 ubuntu ubuntu 382464 nov. 13 16:14 uepal-fr-uepalparoisse-civiparoisse-v0.0.1-66a904.tar
-rw-rw-r-- 1 ubuntu ubuntu 382464 nov. 13 16:14 uepal-fr-uepalparoisse-civiparoisse-v0.0.2-ff0663.tar
-rw-rw-r-- 1 ubuntu ubuntu 382464 nov. 13 16:14 uepal-fr-uepalparoisse-civiparoisse-v0.0.3-283727.tar
./p:
total 4
drwxrwxr-x 2 ubuntu ubuntu 4096 nov.13 16:14 uepal
./p/uepal:
total 4
-rw-rw-r-- 1 ubuntu ubuntu 3889 nov. 13 16:14 'fr.uepalparoisse.civiparoisse$1060f03e1bdec900db4bd55eb8eb8bb1ad05795428765663e44247de1f6d7ecd.json'
./p2:
total 4
drwxrwxr-x 2 ubuntu ubuntu 4096 nov. 13 16:14 uepal
./p2/uepal:
total 8
-rw-rw-r-- 1 ubuntu ubuntu 744 nov. 13 16:14 fr.uepalparoisse.civiparoisse~dev.json
-rw-rw-r-- 1 ubuntu ubuntu 3167 nov. 13 16:14 fr.uepalparoisse.civiparoisse.json
```

La consommation du package se fait ensuite en spécifiant dans le composer.json du projet d’installation à la fois le dépôt privé et le package requis, comme par exemple pour l’expérimentation :

``` json
{
"repositories":[{
"type":"composer",
"url":"http://satis.test"
}],
"extra":{
"enable-patching":true,
"compile-whitelist": ["civicrm/civicrm-core", "civicrm/composer-compile-lib"]
},
"require":{
"drupal/recommended-project":">=9.2",
"drush/drush":">=10",
"drupal/console":">=1",
"civicrm/civicrm-core":">=5.38",
"civicrm/civicrm-packages":">=5.38",
"civicrm/civicrm-drupal-8":">=5.38",
"civicrm/civicrm-asset-plugin":">=1.1",
"civicrm/composer-downloads-plugin":">=3",
"uepal/fr.uepalparoisse.civiparoisse":"0.0.3"
},
"minimum-stability":"dev",
"prefer-stable":true,
"config":{
"secure-http":false
}
}
```
Le secure-http à FALSE a été utilisé pour ne pas avoir besoin de faire une configuration fastidieuse du SSL dans l’expérimentation, étant donné que dans le cadre d’une mise en production on utiliserait probablement des certificats reconnus sans avoir besoin de réglages spécifiques.

Après utilisation du composer update, on retrouve ensuite dans le répertoire `vendor/uepal/fr.uepalparoisse.civiparoisse` l’ensemble de l’extension.

On comprend donc que chaque méthode a des avantages et des inconvénients, mais on peut globalement supposer que l’utilisation du dépôt git uniquement convient pour un petit nombre d’instances, tandis que l’utilisation d’un site web fournira des facilités d’installation pour des instances plus nombreuses, dans la mesure où le site web pourra disposer de fonctionnalités supplémentaires (par exemple un proxy cache qui pourra être utile pour économiser du trafic réseau pour la récupération des packages Composer, sans compter une séparation plus nette entre les outils de développements et les outils de production).